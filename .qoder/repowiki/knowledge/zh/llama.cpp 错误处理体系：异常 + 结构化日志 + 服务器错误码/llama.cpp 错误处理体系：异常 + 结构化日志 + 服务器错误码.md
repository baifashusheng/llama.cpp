---
kind: error_handling
name: llama.cpp 错误处理体系：异常 + 结构化日志 + 服务器错误码
category: error_handling
scope:
    - '**'
source_files:
    - ggml/include/ggml.h
    - include/llama.h
    - src/llama-impl.h
    - common/log.h
    - common/arg.cpp
    - common/download.cpp
    - common/hf-cache.cpp
    - common/json-schema.cpp
    - common/jinja/runtime.cpp
    - tools/server/server-common.h
---

## 1. 总体方案

llama.cpp 是一个 C/C++ 推理框架，错误处理采用三层组合：

- **核心运行时**（`src/`, `ggml/`）：以 `GGML_ASSERT` / `GGML_ABORT` 作为不变量检查；不定义自定义异常类型，依赖标准库异常与 abort。
- **公共库**（`common/`）：通过 `std::runtime_error` / `std::invalid_argument` 等标准异常表达可恢复错误，并通过统一的日志宏 `LOG_*` / `LLAMA_LOG_*` 输出诊断信息。
- **HTTP 服务层**（`tools/server/`）：将内部异常转换为 OpenAI 兼容的 JSON 错误响应，使用自有的 `error_type` 枚举分类。

仓库没有独立的 `errors/` 目录、也没有统一的错误码枚举或 `Result<T>` 风格返回值——错误是分散在各模块中的。因此该类别属于“中等”强度信号：有清晰的日志基础设施和服务器端错误模型，但没有跨模块的错误类型抽象。

## 2. 关键文件与包

| 层次 | 文件 | 职责 |
|---|---|---|
| 日志基础设施 | `ggml/include/ggml.h` | 定义 `enum ggml_log_level { NONE=0, DEBUG=1, INFO=2, WARN=3, ERROR=4 }` 及 `GGML_ASSERT` / `GGML_ABORT` |
| llama 日志封装 | `src/llama-impl.h` | 提供 `LLAMA_LOG_INFO/WARN/ERROR/DEBUG/CONT` 宏，转发到 `llama_log_internal(ggml_log_level, ...)` |
| 公共日志 API | `common/log.h` | 暴露 `LOG_DBG/TRC/INF/WRN/ERR/CNT` 宏、`common_log_add`、`common_log_default_callback`、`common_log_get_verbosity` |
| 全局日志回调 | `include/llama.h` (L1670-L1671) | `llama_log_set(ggml_log_callback, void*)` / `llama_log_get(...)`，允许用户替换默认 stderr 输出 |
| 服务器错误模型 | `tools/server/server-common.h` (L58-L67) | `enum error_type { INVALID_REQUEST, AUTHENTICATION, SERVER, NOT_FOUND, PERMISSION, UNAVAILABLE, NOT_SUPPORTED, EXCEED_CONTEXT_SIZE }` |
| 服务器错误响应 | `tools/server/server-common.h` (L95) | `json format_error_response(const std::string & message, const enum error_type type)` |
| 参数解析异常集中地 | `common/arg.cpp` | 大量 `throw std::invalid_argument(...)` 用于 CLI 参数校验 |
| 下载/网络异常 | `common/download.cpp` | `throw std::runtime_error(...)` 包装 HTTP 状态码 |
| Jinja 模板异常 | `common/jinja/runtime.cpp` | 捕获并转换底层异常为 `std::runtime_error` |

## 3. 架构与约定

### 3.1 断言与不可恢复错误

- `GGML_ASSERT(x)` 在 debug 构建中展开为 `if (!(x)) GGML_ABORT(...)`（见 `ggml/include/ggml.h:L288`），用于内部不变量检查。
- `GGML_ABORT` 直接终止进程，不抛出异常。常见于后端实现中的非法张量形状、越界访问等。
- `src/llama-impl.h` 中的 `time_meas`、`buffer_view` 等工具结构体也依赖 `GGML_ASSERT` 做前置条件校验。

### 3.2 可恢复错误 → 标准 C++ 异常

公共库普遍使用标准异常而非返回码：

- `common/arg.cpp`：CLI 参数校验失败时抛 `std::invalid_argument`（如 L267 `"invalid value"`、L339 `"invalid boolean value"`、L919 `"--model is required"`、L1135 `"invalid device: %s"`）。
- `common/download.cpp`：网络请求失败抛 `std::runtime_error`，消息中包含 HTTP 状态码（L786 `"Download '%s' failed with status code: %d"`）。
- `common/hf-cache.cpp`：HF 缓存目录解析失败抛 `std::runtime_error`（L58）。
- `common/json-schema.cpp`：JSON Schema 解析错误抛 `std::runtime_error("JSON schema error at " + path + ": " + msg)`（L23）。
- `common/jinja/parser.cpp`：语法错误抛 `std::runtime_error("Unexpected token type" / "Unknown statement" / ...)`.  

调用方通常用 `catch (const std::exception & e)` 包裹这些调用点，例如 `common/chat.cpp` 多处 catch 后记录日志并返回空结果或继续执行。

### 3.3 日志系统

日志分为两层：

1. **底层**：`ggml_log_level` 枚举 + `ggml_log_callback` 函数指针。
2. **上层**：`common/log.h` 的 `LOG_INF/WRN/ERR` 宏 + `src/llama-impl.h` 的 `LLAMA_LOG_*` 宏。

所有日志最终通过 `llama_log_internal(level, fmt, ...)` 分发，默认回调 `llama_log_callback_default` 输出到 stderr。用户可以通过 `llama_log_set` 注册自定义回调（`include/llama.h:L1670-L1671`），注释明确说明：*"the logger state is global so these functions are NOT thread safe"*。

Android 示例 (`examples/llama.android/lib/src/main/cpp/logging.h`) 展示了如何把 `ggml_log_level` 映射到 Android Logcat 优先级（ERROR→WARN→INFO→DEBUG）。

### 3.4 服务器端错误模型

`tools/server/server-common.h` 定义了 OpenAI 兼容的错误分类：

```cpp
enum error_type {
    ERROR_TYPE_INVALID_REQUEST,
    ERROR_TYPE_AUTHENTICATION,
    ERROR_TYPE_SERVER,
    ERROR_TYPE_NOT_FOUND,
    ERROR_TYPE_PERMISSION,
    ERROR_TYPE_UNAVAILABLE,   // custom
    ERROR_TYPE_NOT_SUPPORTED, // custom
    ERROR_TYPE_EXCEED_CONTEXT_SIZE, // custom
};
```

配合 `format_error_response(message, type)` 生成统一 JSON 结构。任务结构体 `server_task` 中也包含 `error_type err_type = ERROR_TYPE_SERVER` 字段，用于在异步任务完成时携带错误类型。

## 4. 约定与约束

- **约定（描述性）**：
  - 核心运行时（`ggml/`、`src/`）优先使用 `GGML_ASSERT` 表达不可恢复的内部错误；仅在边界处（如 I/O、参数解析）才抛 `std::exception`。
  - 公共库（`common/`）对输入校验使用 `std::invalid_argument`，对运行时失败（网络、文件、模型加载）使用 `std::runtime_error`。
  - 所有日志走 `LOG_*` / `LLAMA_LOG_*` 宏，禁止直接 `printf` / `fprintf(stderr, ...)` 输出诊断信息。
  - HTTP 服务端将所有异常路径收敛为 `format_error_response` 输出的 OpenAI 兼容 JSON，错误类型取自 `error_type` 枚举。
  - 日志回调是全局且非线程安全的（由 `include/llama.h` 注释声明），应在应用初始化阶段设置一次。

- **规则（由代码强制）**：
  - `GGML_ASSERT` 在 debug 构建下必然触发 `GGML_ABORT`，这是由 `ggml/include/ggml.h` 的宏定义强制的。
  - `ggml_log_level` 的取值范围由 `ggml/include/ggml.h:L652-L656` 的枚举严格限定，任何日志宏都必须使用该枚举值。
  - 服务器端错误类型只能从 `tools/server/server-common.h` 的 `error_type` 枚举选取，新增错误需先扩展该枚举再添加对应 JSON 映射。
  - 外部用户可通过 `llama_log_set` 替换日志回调，但必须遵循 `ggml_log_callback` 签名（`void (*)(ggml_log_level, const char*, void*)`），否则编译失败。

- **未观察到的模式**：
  - 未发现统一的错误码枚举（除服务器端外）、`Result<T>` 风格返回值、panic/recover 机制、或跨模块的错误传播中间件。
  - Python 侧（`gguf-py/`、`conversion/`）主要使用 Python 原生异常，与本 C++ 错误体系解耦。