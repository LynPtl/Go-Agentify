# 产品需求文档 (PRD): Go-Agentify

## 1. 产品概述 (Product Overview)

`Go-Agentify` 是一款针对 Go 语言微服务设计的静态代码注入与 Agent-Native CLI 生成工具。
它通过在编译期解析抽象语法树 (AST)，将带有特定 Pragma 注释的内部业务方法，自动暴露为基于 UNIX Domain Socket (UDS) 通信的独立 CLI 客户端。本工具旨在为 AI Agent 或 SRE 提供 100% 结构化 (JSON) 的底层状态干预接口，同时实现零业务侵入、零网络端口暴露。

## 2. 核心架构设计 (Architecture Specification)

系统严格采用 **"编译期代码生成 + 运行期 In-Process UDS C/S"** 架构：

* **编译期 (Control Plane)**：扫描目标项目 AST，生成用于序列化的 DTO，在业务初始化流程后注入 Admin Server 注册代码，并独立生成基于 Cobra 的 CLI 客户端源码。
* **运行期 (Data Plane)**：
* **Admin Server (服务端)**：随目标微服务主进程启动，挂载于本地系统 UDS 路径，基于原生 `net/http` 提供 RPC 路由，并直接持有业务实例指针。
* **Agent CLI (客户端)**：作为独立进程被调用，通过 UDS 向 Admin Server 发送带参请求，并向终端 (`stdout`/`stderr`) 返回标准结果。



## 3. 核心功能模块规约 (Core Features)

### 3.1 编译期解析与实例劫持 (AST Parser & Injector)

* **扫描定界**：目标项目必须包含 `package main` 和 `main()` 启动函数，否则中断编译并报错。
* **安全暴露边界**：仅识别带有 `//go:agentify export` 编译指令的 `func`（普通函数或 Struct 方法）。
* **作用域与依赖注入优先级 (Critical)**：AST 引擎在处理实例劫持时，必须严格遵循以下扫描顺序：
1. **逃生舱检测 (优先)**：全量扫描 AST。若发现开发者已手动调用 `agentify.Register("StructName", instance)`，则直接将该 `StructName` 标记为已注册，**跳过**后续针对该类型的自动劫持和多实例冲突检测。
2. **自动劫持与冲突检测**：仅在 `main()` 函数作用域内寻找未被手动注册的目标 Struct 初始化语句。若在 `main()` 内发现同一个目标 Struct 被多次初始化（多实例冲突），直接抛出 `Fatal Error` 终止编译。



### 3.2 序列化处理 (DTO Generator)

* **基础类型**：直接映射并透传。
* **深度限定的复杂结构体**：针对包含未导出字段（小写字母开头）的参数/返回值，工具在 `agentify_gen/dto` 包下自动生成 `{Original}DTO` 结构体及 `ToDTO()`/`FromDTO()` 方法。**最大递归深度 (Max Depth) 严格限定为 1**。
* **越界阻断与错误提示**：若未导出字段本身是结构体且仍含未导出字段，抛出带有完整路径的编译错误，格式如下：
```text
[go-agentify] Fatal: cannot auto-serialize field "PaymentService.config" (type: *DatabaseConfig)
  → field "DatabaseConfig.password" is unexported (depth=2, max=1)
  → hint: simplify the return type, or define a fully-exported DTO manually

```


* **类型阻断**：遇无法跨进程序列化的类型（如 `chan`, `func`, `unsafe.Pointer`），直接抛出编译错误。

### 3.3 CLI 客户端生成 (CLI Generator)

* **命令路由**：采用嵌套子命令结构，规范为：`./agent-cli [struct-name] [method-name]`。
* **参数映射策略**：
* 基础类型映射为 Flag (如 `--uid=123`)。
* 复杂 Struct 映射为 JSON 字符串入参 (如 `--req='{"field": "value"}'`)。支持 `--generate-skeleton` 输出空 JSON 模版供 Agent 填充。



## 4. 通信协议与版本控制 (Protocol & Versioning)

### 4.1 UDS 路由规约

* **通信载体**：HTTP/1.1 over UDS。
* **路由格式**：`POST /{struct_name}/{method_name}`。
* **生命周期**：Admin Server 启动前执行 `os.Remove(sockPath)` 清理残留；监听 `SIGINT`/`SIGTERM` 执行 `os.RemoveAll()`。

### 4.2 版本协商 (Version Negotiation)

* **规则**：CLI 发起请求前在 Header 中注入 `X-Agentify-Client-Version: <SemVer>`。
* **比对**：Admin Server 发现 Major (主版本) 不一致时拒绝服务（HTTP `426 Upgrade Required`），Minor/Patch 不一致则向下兼容。

### 4.3 标准错误响应 Schema

服务端 `panic` 或 `error`，统一封装为以下 JSON 返回给 CLI：

```json
{
  "success": false,
  "error": {
    "code": "EXECUTION_ERROR",
    "message": "specific error reason",
    "trace": "stack trace string (optional)"
  }
}

```

## 5. 可观测性与流隔离 (Observability & Stream Isolation)

### 5.1 Admin Server 审计日志

业务进程内的 Admin Server 必须生成结构化审计日志。

* **输出路由控制**：
* 默认输出至业务进程的 `stderr`，严禁污染业务自身的 `stdout` 日志流。
* 支持通过启动配置指定独立文件路径（例如：`agentify.yaml` 中配置了 `audit_log_path: /var/log/agentify-audit.log` 时，写入该文件）。


* **日志 Schema**：包含 `timestamp`, `level`, `event`, `struct`, `method`, `args`, `result`, `latency_ms`。

### 5.2 CLI 客户端终端输出隔离 (Agent 友好规约)

CLI 客户端进程必须严格区分输出流：

* **`stdout` 独占**：仅允许输出最终的业务执行结果（纯 JSON 字符串）。
* **`stderr` 兜底**：CLI 运行时的状态日志（正在连接、文件不存在）、耗时统计、前置参数校验报错等，强制输出至 `stderr`。
* **人类可读模式 (`--human`)**：CLI 提供全局 `--human` flag。启用时，向 `stdout` 输出格式化、带有颜色高亮的纯文本结果以供 SRE 调试。**规范约定：Agent 调度该 CLI 时，绝对禁止追加 `--human` 参数。**

### 5.3 配置化防刷限流 (Rate Limiting)

内置基于令牌桶算法的限流中间件。读取项目根目录 `agentify.yaml`，触发限流返回 HTTP `429 Too Many Requests`。

## 6. 工程与交付规范 (Engineering & Delivery Standards)

### 6.1 编译产物结构与幂等管理

* **产物隔离**：生成的 Server 代码放至 `agentify_gen/` 包下；CLI 编译至 `bin/agentify-cli`。
* **`.gitignore` 幂等追加**：按行读取目标目录 `.gitignore`，仅当不存在对应规则时，向文件末尾追加 `/agentify_gen/` 和 `/bin/agentify-cli`。

### 6.2 客户端健壮性 (Client Robustness)

* **本地快速失败 (Fast-Fail Validation)**：依赖 Cobra 进行必填参数校验；针对复杂 Struct 字符串入参，在发起网络请求前调用 `json.Valid()` 进行本地验证。
* **灵活超时控制 (Timeout Control)**：CLI 侧 `http.Client` 默认设置 `5 * time.Second` 超时。提供全局 Flag `--timeout` 解析并覆盖默认值。

### 6.3 部署与并发契约

* **工具分发**：支持标准 Go 工具链 `go install github.com/xxx/go-agentify/cmd/agentify@latest` 安装。
* **依赖对齐**：代码生成完毕后，自动执行 `go mod tidy`。
* **并发模型**：Admin Server 依赖 `net/http` 原生多 Goroutine 处理。暴露的业务方法并发安全性由业务代码自行保证。
