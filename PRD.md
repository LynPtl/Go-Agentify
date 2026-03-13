# 产品需求文档 (PRD): Go-Agentify (v1.0-MVP)

## 1. 产品概述 (Product Overview)

`Go-Agentify` 是一款针对 Go 语言微服务设计的静态代码注入与 Agent-Native CLI 生成工具。
它通过在编译期解析抽象语法树 (AST)，将带有特定 Pragma 注释的内部业务方法，自动暴露为基于 UNIX Domain Socket (UDS) 通信的独立 CLI 客户端。本工具旨在为 AI Agent 或 SRE 提供 100% 结构化 (JSON) 的底层状态干预接口，同时实现零业务侵入、零网络端口暴露。

## 2. 核心架构设计 (Architecture Specification)

系统严格采用 **"编译期代码生成 + 运行期 In-Process UDS C/S"** 架构：

* **编译期 (Control Plane)**：扫描目标项目 AST，生成用于序列化的 DTO，在业务初始化流程后注入 Admin Server 注册代码，并独立生成基于 Cobra 的 CLI 客户端源码。
* **运行期 (Data Plane)**：
* **Admin Server (服务端)**：随目标微服务主进程启动，挂载于本地系统 UDS 路径，基于原生 `net/http` 提供 RPC 路由，并直接持有业务实例指针。
* **Agent CLI (客户端)**：作为独立进程被调用，通过 UDS 向 Admin Server 发送带参请求，并向终端 (`stdout`/`stderr`) 返回标准 JSON 结果。



## 3. 核心功能模块规约 (Core Features)

### 3.1 编译期解析与注入 (AST Parser & Injector)

* **扫描定界**：强制要求目标项目必须包含 `package main` 和 `main()` 启动函数，否则中断编译流程并报错。
* **安全暴露边界**：仅识别带有 `//go:agentify export` 编译指令的 `func`（普通函数或 Struct 方法）。
* **依赖注入 (实例劫持)**：AST 引擎需定位到目标 Struct 实例初始化的作用域，自动在其后插入 `agentify.Register("StructName", instance)` 代码，使 Admin Server 捕获运行时的内存状态。

### 3.2 序列化处理 (DTO Generator)

* **基础类型**：直接映射并透传。
* **复杂结构体**：若导出的方法参数/返回值包含未导出的字段（小写字母开头），工具必须在生成的 `agentify_gen/dto` 包下自动生成全导出的 `{Original}DTO` 结构体，并注入 `ToDTO()` / `FromDTO()` 转换逻辑。
* **阻断策略**：遇无法跨进程序列化的类型（如 `chan`, `func`, `unsafe.Pointer`），直接抛出编译错误。

### 3.3 CLI 客户端生成 (CLI Generator)

* **命令路由**：采用嵌套子命令结构。格式规范为：`./agent-cli [struct-name] [method-name]` (例如：`./agent-cli payment-service reset`)。
* **参数映射策略**：
* 基础类型直接映射为 Flag (如 `--uid=123`)。
* 复杂 Struct 映射为 JSON 字符串入参 (如 `--req='{"field": "value"}'`)。
* **增强体验**：为复杂参数命令自动生成 `--generate-skeleton` Flag，输出空 JSON 模版供 Agent 填充。



## 4. 通信协议与版本控制 (Protocol & Versioning)

### 4.1 UDS 路由规约

* **通信载体**：HTTP/1.1 over UDS。
* **路由格式**：`POST /{struct_name}/{method_name}`。
* **生命周期管理**：Admin Server 启动前必须执行 `os.Remove(sockPath)` 清理残留；监听 `SIGINT`/`SIGTERM` 实现优雅退出时的 `os.RemoveAll()`。

### 4.2 版本协商 (Version Negotiation)

* **请求头携带**：CLI 发起请求前，必须在 Header 中注入 `X-Agentify-Client-Version: <SemVer>`。
* **比对逻辑**：Admin Server 校验版本：
* **Major (主版本) 不一致**：拒绝服务，返回 HTTP `426 Upgrade Required`。
* **Minor/Patch**：向下兼容，放行。



### 4.3 标准错误响应 Schema

服务端 `panic` 或方法返回的 `error`，必须统一封装为以下 JSON 返回：

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

## 5. 可观测性与安全防护 (Observability & Protection)

### 5.1 结构化审计日志 (Audit Logging)

所有调用必须将审计日志输出至标准输出 (stdout)，格式如下：

```json
{
  "timestamp": "2026-03-13T10:30:00Z",
  "level": "INFO",
  "event": "agentify.call",
  "struct": "PaymentService",
  "method": "ResetUserState",
  "args": {"uid": 12345},
  "result": "success",
  "latency_ms": 12
}

```

### 5.2 配置化防刷限流 (Rate Limiting)

内置基于令牌桶算法的限流中间件。

* **配置读取**：读取项目根目录 `agentify.yaml`，无则使用缺省值。
* **拦截响应**：触发限流直接返回 HTTP `429 Too Many Requests`。

## 6. 工程与交付规范 (Engineering & Delivery Standards)

### 6.1 编译产物结构与幂等管理

* **产物隔离**：生成的 Server 与 DTO 代码统一放至项目根目录 `agentify_gen/` 包下；CLI 编译至 `bin/agentify-cli`。
* **`.gitignore` 幂等追加**：工具在执行时，需检测目标目录是否存在 `.gitignore`。若无则创建；若有，则按行读取校验，仅在不存在对应规则时，才向文件末尾追加 `/agentify_gen/` 和 `/bin/agentify-cli`，避免重复写入。

### 6.2 客户端健壮性 (Client Robustness)

* **本地快速失败 (Fast-Fail Validation)**：
* **必填校验**：依赖 Cobra `cmd.MarkFlagRequired()` 进行基础校验。
* **JSON 语法校验**：对于复杂 Struct 的字符串入参，CLI 必须在发起网络请求前，调用标准库 `json.Valid([]byte(reqString))` 进行合法性验证。校验失败直接退出，不消耗服务端资源。


* **灵活超时控制 (Timeout Control)**：
* CLI 侧的 `http.Client` 默认设置 `5 * time.Second` 超时。
* 提供全局 Flag `--timeout`，通过 `time.ParseDuration(flag.Value)` 解析并覆盖默认超时，以支持耗时较长的文件 IO 或批量 DB 操作。



### 6.3 部署与并发契约

* **工具分发**：支持标准 Go 工具链 `go install github.com/xxx/go-agentify/cmd/agentify@latest` 全局安装。
* **依赖对齐**：代码生成完毕后，自动执行 `go mod tidy`。
* **并发模型**：Admin Server 依赖 `net/http` 原生多 Goroutine 处理。被导出的业务方法，其内部的并发安全性由业务代码自行保证，Server 层不介入加锁。