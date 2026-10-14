---
kind: logging_system
name: common_log 线程安全日志子系统
category: logging_system
scope:
    - '**'
source_files:
    - common/log.h
    - common/log.cpp
    - common/common.cpp
    - common/arg.cpp
    - tests/test-log.cpp
    - tests/llama-context/test-log.cpp
    - tools/server/server-context.cpp
    - tools/mtmd/mtmd-cli.cpp
    - tools/tts/tts.cpp
---

## 1. 系统概览

llama.cpp 的日志系统由 `common/log.{h,cpp}` 实现，名为 `common_log`。它不是第三方框架，而是自研的轻量级、线程安全的结构化日志子系统，为 llama 核心、ggml 后端、common 公共库以及 tools/ 下的所有 CLI 工具提供统一的日志输出能力。

核心设计要点：
- **单例 + 后台 worker 线程**：通过 `common_log_main()` 获取全局单例（故意 leak 以避免 Windows DLL 卸载挂起），内部维护一个环形队列 + `std::thread` worker，生产者（任意线程）调用 `LOG_*` 宏入队，worker 线程负责格式化与落盘。
- **双通道输出**：可同时输出到 `stdout/stderr` 和可选文件 sink；JSONL 模式下只走 JSON 通道。
- **颜色与 ANSI**：默认关闭颜色，可通过 `tty_can_use_colors()` 自动检测 TTY 后启用。
- **与 ggml 日志桥接**：在 `common/common.cpp` 中通过 `llama_log_set(common_log_default_callback, NULL)` 将 ggml 内部的 `ggml_log_level` 消息转发到 common_log。

## 2. 关键文件

| 文件 | 作用 |
|---|---|
| `common/log.h` | 公共 API、宏定义、级别常量、颜色枚举 |
| `common/log.cpp` | `common_log` 结构体、环形队列、worker 线程、JSONL 序列化、ggml 回调 |
| `common/common.cpp` | 初始化时注册 `common_log_default_callback` 作为 ggml 日志回调 |
| `common/arg.cpp` | 解析 `--log-jsonl`、`-v`/`--verbose` 等参数并调用 `common_log_set_jsonl` / `common_log_set_verbosity_thold` |
| `tests/test-log.cpp` | 多线程并发压测用例 |
| `tools/server/server-context.cpp`、`tools/mtmd/mtmd-cli.cpp`、`tools/tts/tts.cpp` | 将下游组件（如 mtmd）的日志也桥接到 `common_log_default_callback` |

## 3. 架构与约定

### 3.1 日志级别体系

`common/log.h` 定义了应用层级别常量（数值越小越严重）：

```
LOG_LEVEL_DEBUG  = 5
LOG_LEVEL_TRACE  = 4
LOG_LEVEL_INFO   = 3
LOG_LEVEL_WARN   = 2
LOG_LEVEL_ERROR  = 1
LOG_LEVEL_OUTPUT = 0   // 工具输出通道
```

同时映射到 ggml 的 `enum ggml_log_level`（DEBUG/INFO/WARN/ERROR/CONT/NONE）。映射关系见 `common_log_get_verbosity`：
- `GGML_LOG_LEVEL_DEBUG` → `LOG_LEVEL_DEBUG`
- `GGML_LOG_LEVEL_INFO` / `GGML_LOG_LEVEL_CONT` → `LOG_LEVEL_TRACE`
- `GGML_LOG_LEVEL_WARN` → `LOG_LEVEL_WARN`
- `GGML_LOG_LEVEL_ERROR` → `LOG_LEVEL_ERROR`
- `GGML_LOG_LEVEL_NONE` / 其他 → `LOG_LEVEL_OUTPUT`

默认阈值：`LOG_DEFAULT_DEBUG = 5`，`LOG_DEFAULT_LLAMA = LOG_LEVEL_INFO`。

### 3.2 使用宏

所有日志点统一通过以下宏调用，避免在低阈值下计算昂贵参数：

| 宏 | 用途 | 底层 level | 默认阈值 |
|---|---|---|---|
| `LOG(...)` | 工具输出（stdout） | `GGML_LOG_LEVEL_NONE` | `LOG_LEVEL_OUTPUT` |
| `LOGV(verbosity, ...)` | 带自定义阈值的输出 | `GGML_LOG_LEVEL_NONE` | 显式传入 |
| `LOG_DBG(...)` | 调试信息 | `GGML_LOG_LEVEL_DEBUG` | `LOG_LEVEL_DEBUG` |
| `LOG_TRC(...)` | 追踪信息 | `GGML_LOG_LEVEL_INFO` | `LOG_LEVEL_TRACE` |
| `LOG_INF(...)` | 常规信息 | `GGML_LOG_LEVEL_INFO` | `LOG_LEVEL_INFO` |
| `LOG_WRN(...)` | 警告 | `GGML_LOG_LEVEL_WARN` | `LOG_LEVEL_WARN` |
| `LOG_ERR(...)` | 错误 | `GGML_LOG_LEVEL_ERROR` | `LOG_LEVEL_ERROR` |
| `LOG_CNT(...)` | 计数/进度（同 INFO） | `GGML_LOG_LEVEL_CONT` | `LOG_LEVEL_INFO` |

每个宏都有对应的 `*V` 变体（如 `LOG_INFV(verbosity, ...)`）用于动态阈值。

### 3.3 输出格式

非 JSONL 模式，每行形如：

```
[时间戳] [颜色前缀] 消息文本
```

- 时间戳格式：`M.mm.sss.us`（分钟.秒.毫秒.微秒），仅当开启 `timestamps` 时输出。
- 前缀：`I`/`W`/`E`/`D`，分别对应 info/warn/error/debug，带 ANSI 颜色（绿/洋红/红/黄）。
- 普通 INFO 不刷新 stderr，WARN/ERROR/DEBUG 会 `fflush`。
- 可配置是否输出前缀 (`common_log_set_prefix`)。

JSONL 模式（`--log-jsonl`）下，每条日志是独立 JSON 对象：

```json
{"type":"log","time":123456,"level":"info","msg":"..."}
```

结构化 JSON 数据通过 `LOG_JSON(type, data)` 宏输出，字段为 `{"type": type, "data": <common_json 对象>}`。

### 3.4 线程模型

`common_log` 内部状态：
- `std::mutex mtx` 保护环形队列 `queue` 及 head/tail 指针。
- `std::condition_variable cv_new`：有新条目时唤醒 worker。
- `std::condition_variable cv_full`：队列满时阻塞生产者。
- `std::thread thrd`：worker 线程，循环消费队列并调用 `print_entry`。
- `pause()` / `resume()`：暂停期间丢弃新消息，`pause()` 会向队列插入 `is_end=true` 哨兵以优雅退出 worker。

注意：`set_file` / `set_stdo` / `set_colors` 等方法内部先 `pause()` 再修改状态再 `resume()`，保证 worker 不会读到不一致的配置。

### 3.5 与 ggml 日志集成

`common/common.cpp` 在初始化时调用：

```cpp
llama_log_set(common_log_default_callback, NULL);
```

`common_log_default_callback` 根据当前 `common_log_verbosity_thold` 决定是否转发 ggml 的日志消息到 `common_log_add`。

## 4. 约定与约束

- **所有日志点必须通过 `LOG_*` 宏调用**，禁止直接 `fprintf(stderr, ...)` 或 `printf(...)` 输出诊断信息（`LOG` 宏除外，它是专门给工具输出用的 stdout 通道）。
- **阈值控制**：通过 `common_log_set_verbosity_thold(int)` 设置全局阈值，低于该阈级的 DEBUG/TRACE 消息会被静默丢弃（但写入文件时仍可能输出——参见 `print_entry` 中对 `stderr` 的特殊处理）。
- **JSONL 开关**：通过 `common_log_set_jsonl(bool)` 切换，CLI 端由 `--log-jsonl` 参数驱动（见 `common/arg.cpp`）。
- **颜色开关**：默认 `LOG_COLORS_AUTO`，由 `tty_can_use_colors()` 决定；也可强制 `LOG_COLORS_ENABLED` / `LOG_COLORS_DISABLED`。
- **线程安全限制**：文档注释明确标注 `common_log_set_verbosity_thold`、`common_log_set_jsonl`、`common_log_pause`、`common_log_resume`、`common_log_set_file`、`common_log_set_stdo`、`common_log_set_colors` 均 **not thread-safe**，应在程序启动阶段、多工作线程开始之前设置。
- **单例泄漏**：`common_log_main()` 返回的单例故意不释放（`// intentionally leaked to avoid Windows teardown hangs`），需在退出前调用 `common_log_flush()` 确保 flush；测试中还会显式 `common_log_free()` 以绕过 Windows 问题。
- **ggml 日志桥接**：ggml 后端的所有日志最终都经过 `common_log_default_callback`，因此统一受 `common_log_verbosity_thold` 控制。
- **结构化字段**：JSONL 模式下固定字段为 `type`、`time`、`level`、`msg`；`LOG_JSON` 额外包裹一层 `{"type", ..., "data", ...}`。

## 5. 相关构建/测试

- `tests/test-log.cpp` 用 8 个线程各写 1000 条随机级别的日志，并在循环中随机切换 `prefix`/`timestamps`，验证并发安全性。
- `tests/llama-context/test-log.cpp` 通过 `common_log_default_callback` 间接验证 ggml→common_log 的桥接路径。
- CLI 参数 `-v`/`--verbose` 与 `--log-jsonl` 在 `common/arg.cpp` 中解析并调用 `common_log_set_verbosity_thold` / `common_log_set_jsonl`。

## 6. 总结

llama.cpp 的日志系统是一个围绕 `common_log` 单例构建的轻量级方案：应用层宏 `LOG_*` 提供分级与阈值短路，内部环形队列 + worker 线程解耦生产与 I/O，支持 stdout/stderr 双通道与可选文件 sink，并通过 JSONL 模式输出结构化日志。ggml 后端日志通过 `llama_log_set` 回调统一接入，使整个推理栈共享同一套日志策略。