---
kind: dependency_management
name: llama.cpp 依赖管理：CMake + Python 双栈与 vendor 内联策略
category: dependency_management
scope:
    - '**'
source_files:
    - scripts/sync_vendor.py
    - scripts/sync-ggml.sh
    - requirements.txt
    - requirements/requirements-all.txt
    - requirements/requirements-convert_hf_to_gguf.txt
    - pyproject.toml
    - gguf-py/pyproject.toml
    - common/CMakeLists.txt
    - tools/mtmd/CMakeLists.txt
    - tools/server/CMakeLists.txt
    - tests/CMakeLists.txt
    - CMakeLists.txt
    - .gitmodules
---

## 1. 总体方案

llama.cpp 仓库采用 **多语言、分层式** 的依赖管理策略，核心思路是：
- C/C++ 第三方库通过 `scripts/sync_vendor.py` 拉取并内联到 `vendor/` 目录（无 git submodule）。
- Python 脚本依赖通过 `requirements/*.txt` 分工具拆分，顶层 `requirements.txt` 聚合；同时提供 `pyproject.toml`（Poetry / uv 兼容）作为可发布包的元数据。
- ggml 子项目以“同步副本”方式维护（`scripts/sync-ggml.sh`），而非 git submodule。
- CMake 是唯一的构建入口，所有第三方依赖最终通过 CMake target（如 `vendor::nlohmann`、`cpp-httplib`、`vendor::hash`、`vendor::miniaudio`、`vendor::stb`、`vendor::sheredom`）暴露给上层目标。

## 2. C/C++ 第三方依赖（vendor 内联）

### 2.1 拉取机制
`scripts/sync_vendor.py` 是唯一的权威来源，集中声明每个上游 URL → 本地路径映射（见 `vendor` dict），并支持两类补丁：
- 文本替换补丁（`patches` dict）：用于修复编译期问题（如 Windows SDK 缺 `stdalign.h`、sha1 符号冲突、未初始化警告）。
- `git apply` 补丁（`patch_files` list）：用于较大改动（如 nlohmann/json 的深嵌套回退修复）。

被 vendored 的库包括：
| 上游 | 本地路径 | 版本锁定方式 |
|---|---|---|
| nlohmann/json | `vendor/nlohmann/json.hpp`, `json_fwd.hpp` | `releases/latest/download`（注释说明避免使用最新 tag 以避免 #17179） |
| stb (stb_image.h) | `vendor/stb/stb_image.h` | `refs/heads/master` |
| miniaudio | `vendor/miniaudio/miniaudio.h` | 固定 commit `9634bedb5b...`（注释引用 PR #17179 讨论） |
| cpp-httplib | `vendor/cpp-httplib/*` + `httplib.h`/`split.py` | 固定 tag `v0.61.0`（变量 `HTTPLIB_VERSION`） |
| sheredom/subprocess.h | `vendor/sheredom/subprocess.h` | 固定 commit `0dccaa9a...` |
| Cyan4973/xxHash | `vendor/hash/xxhash/*` | 固定 commit `9f465f1e...` |
| clibs/sha1 | `vendor/hash/sha1/*` | 固定 commit `e1e2536f...` |
| jb55/sha256.c | `vendor/hash/sha256/*` | 固定 commit `5e637272...` |
| jb55/rotate-bits.h | `vendor/hash/rotate-bits/*` | 固定 commit `27e78494...` |

### 2.2 构建集成
- `common/CMakeLists.txt` 将 `vendor::nlohmann`、`vendor::sheredom` 作为公共依赖链接，并将 `cpp-httplib` 作为私有依赖。
- `tools/mtmd/CMakeLists.txt` 链接 `vendor::hash`、`vendor::miniaudio`、`vendor::stb`、`vendor::sheredom`。
- `tests/CMakeLists.txt` 条件性链接 `cpp-httplib`，并在启用 OpenSSL 时检查 `CPPHTTPLIB_OPENSSL_SUPPORT`。
- `tools/server/CMakeLists.txt` 直接链接 `cpp-httplib`。

### 2.3 关键约束
- `vendor/` 目录为空（`list_dir` 返回空），由 `sync_vendor.py` 生成——该脚本会因上游源码变更而显式报错退出（"cannot apply patch on ... upstream code has changed"），强制开发者在升级前审查差异。
- `.gitmodules` 为空文件，确认仓库不使用 git submodule。
- ggml 子项目通过 `scripts/sync-ggml.sh` 从外部 `../ggml` 目录复制源码，不是 submodule，也不是 FetchContent。

## 3. Python 依赖管理

### 3.1 requirements 体系
根级 `requirements.txt` 仅做聚合：
```
-r ./requirements/requirements-convert_legacy_llama.txt
-r ./requirements/requirements-convert_hf_to_gguf.txt
-r ./requirements/requirements-convert_hf_to_gguf_update.txt
-r ./requirements/requirements-convert_llama_ggml_to_gguf.txt
-r ./requirements/requirements-convert_lora_to_gguf.txt
-r ./requirements/requirements-tool_bench.txt
```
另有 `requirements/requirements-all.txt` 聚合更多工具链（mtmd、server bench/tests、compare-llama-bench、server-bench、pydantic、tokenizer-random、gguf-editor-gui、examples/model-conversion）。

各 `requirements-*.txt` 文件按工具域拆分，例如 `requirements-convert_hf_to_gguf.txt` 额外指定 PyTorch CPU wheel 源：
```
--extra-index-url https://download.pytorch.org/whl/cpu
torch==2.11.0
```

### 3.2 pyproject.toml（可发布包）
根 `pyproject.toml` 定义包名 `llama-cpp-scripts`，Python 版本要求 `>=3.10,<3.15`，运行时依赖为：
- `numpy (>=2.2.6,<3.0.0)`
- `sentencepiece (>=0.1.98,<0.3.0)`
- `transformers (==4.57.6)`（精确版本）
- `protobuf (>=4.21.0,<5.0.0)`
- `torch (>=2.6.0,<3.0.0)`
- `gguf @ ./gguf-py`（本地路径依赖，指向仓库内的 `gguf-py` 子包）

PyTorch 通过 Poetry 和 uv 双重配置从专用索引安装：
```toml
[[tool.poetry.source]]
name = "pytorch"
url = "https://download.pytorch.org/whl/cpu"
priority = "explicit"

[[tool.uv.index]]
name = "pytorch"
url = "https://download.pytorch.org/whl/cpu"
explicit = true
```

### 3.3 gguf-py 子包
`gguf-py/pyproject.toml` 定义独立发布的 `gguf` 包（版本 `0.19.0`），依赖 `numpy`、`tqdm`、`pyyaml`、`requests`，可选 GUI 依赖 `PySide6 (>=6.9,<7.0)`（限定 Python 3.9–3.14）。构建后端为 `poetry.core.masonry.api`。

## 4. 构建系统层面的依赖发现

- CMake 最低版本要求 `3.14...3.28`（根 `CMakeLists.txt`）。
- 通过 `find_package(Git)` 获取 Git 信息用于版本号注入（`cmake/build-info.cmake`、`cmake/git-vars.cmake`、`ggml/cmake/GitVars.cmake`）。
- NCCL 通过自定义 `FindNCCL.cmake` 查找。
- 没有使用 `FetchContent`、`ExternalProject_Add` 或 Conan/vcpkg 等包管理器。
- 所有第三方 C/C++ 依赖均以 vendored 源码形式存在，不依赖系统包管理器。

## 5. 约定与约束（规则）

1. **C/C++ 第三方库必须通过 `scripts/sync_vendor.py` 引入**：该脚本是唯一的上游 URL 清单，任何新增 vendored 库需在此处声明 URL 与本地路径映射，否则无法被构建系统识别。
2. **禁止使用 git submodule**：`.gitmodules` 为空，且 `sync-ggml.sh` 明确采用复制而非 submodule 的方式同步 ggml 子项目。
3. **vendor 源码不可手动编辑**：`sync_vendor.py` 对补丁应用失败会 `sys.exit(1)`，提示 "upstream code has changed"，强制开发者重新运行脚本。
4. **Python 顶层依赖不得直接写入根 `requirements.txt`**：根文件注释明确要求 "Avoid adding packages here directly"，应放入对应工具的 `requirements-*.txt` 中。
5. **Python 包发布依赖 `pyproject.toml`**：根包与 `gguf-py` 均使用 Poetry build backend（`poetry.core.masonry.api`），并通过 `requires-python` 限制解释器范围。
6. **PyTorch 必须从专用 CPU wheel 索引安装**：通过 Poetry `[[tool.poetry.source]]` 与 uv `[[tool.uv.index]]` 双重声明 `https://download.pytorch.org/whl/cpu`，标记为 `explicit`，防止默认 PyPI 解析到错误的 wheel。
7. **ggml 子项目通过 `scripts/sync-ggml.sh` 同步**：该脚本从外部 `../ggml` 目录复制 CMakeLists、src、include、tests、LICENSE 等，保持 llama.cpp 仓库内有一份完整副本。