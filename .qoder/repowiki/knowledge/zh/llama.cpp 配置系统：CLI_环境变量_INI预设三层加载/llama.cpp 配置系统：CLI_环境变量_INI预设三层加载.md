---
kind: configuration_system
name: llama.cpp 配置系统：CLI/环境变量/INI预设三层加载
category: configuration_system
scope:
    - '**'
source_files:
    - common/arg.h
    - common/arg.cpp
    - common/preset.h
    - common/preset.cpp
    - tools/server/server.cpp
    - tools/server/main.cpp
    - common/common.cpp
    - common/speculative.cpp
    - common/llguidance.cpp
    - ggml/src/ggml-backend-reg.cpp
    - ggml/src/ggml-backend.cpp
    - ggml/src/ggml-backend-meta.cpp
    - ggml/src/ggml-cann/ggml-cann.cpp
    - ggml/src/ggml-cpu/kleidiai/kleidiai.cpp
    - ggml/src/ggml-cpu/spacemit/ime_env.cpp
---

## 1. 采用的方案

llama.cpp 没有引入外部配置框架，而是自实现了一套**命令行参数 + 环境变量 + INI 预设文件**的三层配置体系，核心位于 `common/` 公共库中：
- **CLI 参数层**：通过 `common/arg.{h,cpp}` 中的 `common_arg` / `common_params_context` 注册表驱动，所有可配置项以声明式方式定义（短参、长参、负参、类型处理器）。
- **环境变量层**：每个 `common_arg` 可通过 `set_env()` 绑定一个 `LLAMA_ARG_<NAME>` 风格的环境变量名；解析时 CLI 优先，未提供则回退到环境变量。同时兼容 `LLAMA_ARG_NO_<NAME>` 形式的否定环境变量。
- **INI 预设文件层**：`common/preset.{h,cpp}` 实现了一个轻量 INI 解析器（基于内部 PEG 解析器），支持多 section、注释（`;` / `#`）、转义换行、preset 合并与级联（cascade）。

此外，GGML 后端层也有独立的运行时开关（如 `GGML_BACKEND_PATH`、`GGML_DISABLE_VULKAN`、`GGML_CANN_*`、`GGML_OP_OFFLOAD_MIN_BATCH`、`GGML_META_DEBUG`、`GGML_SCHED_DEBUG*`、`LLAMA_TRACE`），这些是绕过 common 参数系统的底层硬件/调度开关。

## 2. 关键文件

- `common/arg.h` / `common/arg.cpp` — 参数注册表、CLI 解析、env 读取、negated arg/env 兼容逻辑。
- `common/preset.h` / `common/preset.cpp` — INI preset 结构体、解析器、apply_to_params、cascade/merge、to_args/to_ini。
- `tools/server/server.cpp` — server 入口，调用 `common_params_parse(..., LLAMA_EXAMPLE_SERVER)` 完成统一参数解析。
- `tools/server/main.cpp` — 最薄包装，转发给 `llama_server`。
- `ggml/src/ggml-backend-reg.cpp` / `ggml/src/ggml-backend.cpp` / `ggml/src/ggml-cann/ggml-cann.cpp` — GGML 后端直接读 `getenv("GGML_...")` 等开关。
- `common/common.cpp` — `NO_COLOR`、`TERM`、`LLAMACPP_TEST_MODELFILE` 等全局环境行为。
- `common/speculative.cpp` — `LLAMA_TRACE` 控制 trace 输出。
- `common/llguidance.cpp` — `LLGUIDANCE_LOG_LEVEL` 控制 llguidance 日志级别。

## 3. 架构与约定

### 3.1 参数声明式注册
`common_arg` 结构体集中描述一个配置项：支持的 CLI token（含正/负形式）、值提示、帮助文本、可选的 env 变量名、以及按类型分派的 handler（void/string/int/bool/str-str）。新增配置项只需构造一个 `common_arg` 并加入 `common_params_context::options`，无需修改解析主循环。

### 3.2 优先级顺序
1. **CLI 显式传入**（最高优先级）。
2. **环境变量**：若 CLI 未提供，解析器会检查该选项绑定的 `LLAMA_ARG_<NAME>` 环境变量；对带 negated 参数的选项还会额外检查 `LLAMA_ARG_NO_<NAME>`（见 `common_arg::get_value_from_env` 与 `has_value_from_env`）。
3. **默认值**：由各示例在注册参数时设置。

### 3.3 INI 预设机制
`common_preset` 将一组 `(common_arg → string)` 映射持久化为 INI 格式，section 名为 preset 名称。支持：
- `load_from_ini(path)` 从文件加载多个 preset。
- `cascade(base, added)` 类似 CSS 级联：同名 key 被覆盖，不存在则新增。
- `apply_to_params(params)` 通过 handler 回调把字符串值写回 `common_params`。
- `to_args(bin_path)` 反向生成 CLI 参数串（用于 preset-only 场景）。

Preset 还可从缓存模型目录或 HF repo 自动生成（`load_from_cache` / `load_from_models_dir` / `load_from_args`），server 的 router 模式即依赖此能力自动拉取 preset 仓库。

### 3.4 特殊伪环境变量
`arg.h` 定义了三个以 `__PRESET_` 开头的伪环境变量（`COMMON_ARG_PRESET_LOAD_ON_STARTUP`、`COMMON_ARG_PRESET_STOP_TIMEOUT`、`COMMON_ARG_PRESET_DEDUP_CACHE_MODELS`），仅用于 preset 内部逻辑，不作为普通 CLI 参数暴露。

### 3.5 全局行为环境变量
- `NO_COLOR`：遵循 no-color.org 规范，禁用终端彩色输出（`common/common.cpp`）。
- `TERM`：检测终端能力（同上）。
- `LLAMACPP_TEST_MODELFILE`：测试专用，指定测试模型路径（`common/common.cpp`）。

### 3.6 GGML 后端独立开关
不经过 common 参数系统，直接在后端初始化处 `getenv` 读取：
- `GGML_BACKEND_PATH`：动态后端库搜索路径（`ggml-backend-reg.cpp`）。
- `GGML_DISABLE_VULKAN`：禁用 Vulkan 后端注册（`ggml-backend-reg.cpp`）。
- `GGML_META_DEBUG`、`GGML_SCHED_DEBUG`、`GGML_SCHED_DEBUG_REALLOC`：调试开关（`ggml-backend-meta.cpp` / `ggml-backend.cpp`）。
- `GGML_CANN_NO_PINNED`、`GGML_OP_OFFLOAD_MIN_BATCH`：CANN 后端行为（`ggml-cann.cpp`）。
- `GGML_KLEIDIAI_SME`、`GGML_TOTAL_THREADS`、`GGML_KLEIDIAI_CHUNK_MULTIPLIER`：KleidiAI CPU 优化开关（`ggml-cpu/kleidiai/kleidiai.cpp`）。
- `SPACEMIT_CORE_ARCH`：Spacemit 后端（`ggml-cpu/spacemit/ime_env.cpp`）。

## 4. 约定与约束

- **命名约定**：用户态配置项统一使用 `LLAMA_ARG_<NAME>` 作为环境变量前缀（见 `common/arg.cpp` 中对 `LLAMA_ARG_NO_` 的兼容处理）；GGML 后端开关统一使用 `GGML_*` 前缀。
- **Negated 参数/env 兼容**：对于支持 `--no-xxx` 的参数，其 env 变量名除 `LLAMA_ARG_<NAME>` 外，还兼容 `LLAMA_ARG_NO_<NAME>`（`common_arg::get_value_from_env` 与 `has_value_from_env` 均做 `string_replace_all("LLAMA_ARG_", "LLAMA_ARG_NO_")` 后二次查找）。
- **Preset-only 参数隔离**：标记为 `is_preset_only` 的参数不会出现在 CLI 帮助中，也不会被当作普通 CLI 参数解析，仅能通过 preset 文件或 `__PRESET_*` 伪环境变量生效（`arg.h` 中 `COMMON_ARG_PRESET_LOAD_ON_STARTUP` 等常量）。
- **Unknown keys 策略**：`common_preset_context` 支持 `ignore_unknown_keys = true`，允许共享配置文件对不同二进制只暴露子集 key（`preset.h` 注释说明）。
- **IN I 语法约束**：由 `build_peg_parser` 定义的规则限定：key 必须匹配 `[a-zA-Z_] [a-zA-Z0-9_.-]*`，注释以 `;` 或 `#` 开头，值行以空白+可选注释结尾（`preset.cpp` 第 184–200 行）。
- **Server 启动流程强制调用 `common_params_parse`**：`tools/server/server.cpp` 中 `llama_server(int argc, char** argv)` 在初始化 backend 之前调用 `common_params_parse(argc, argv, params, LLAMA_EXAMPLE_SERVER)`，失败直接返回 1，这是 server 配置加载的唯一入口。
- **无集中式 secrets 管理**：仓库未发现专用的密钥/凭据管理系统；敏感值通常通过环境变量注入（例如 `HF_TOKEN` 由下载模块自行读取），不在代码中硬编码。
- **构建期配置**：版本/构建信息通过 CMake 模板 `src/llama-version.h.in` 与 `common/build-info.cpp.in` 生成，不属于运行期配置范畴。