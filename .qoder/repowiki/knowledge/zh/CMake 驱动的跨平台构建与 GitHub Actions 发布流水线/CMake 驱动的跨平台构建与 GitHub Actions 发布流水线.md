---
kind: build_system
name: CMake 驱动的跨平台构建与 GitHub Actions 发布流水线
category: build_system
scope:
    - '**'
source_files:
    - CMakeLists.txt
    - CMakePresets.json
    - cmake/common.cmake
    - cmake/x64-windows-llvm.cmake
    - cmake/arm64-windows-llvm.cmake
    - cmake/arm64-apple-clang.cmake
    - .github/workflows/release.yml
    - .github/workflows/build-cpu.yml
    - .github/workflows/docker.yml
    - .github/workflows/ui-build.yml
    - .github/workflows/server.yml
    - .devops/cpu.Dockerfile
    - .devops/cuda.Dockerfile
    - .devops/vulkan.Dockerfile
    - .devops/openvino.Dockerfile
    - scripts/release.sh
    - scripts/make-release-desc.sh
    - scripts/check-release-apiabi.sh
---

## 1. 构建系统总览

llama.cpp 仓库以 **CMake ≥ 3.14** 为唯一官方构建入口，顶层 `Makefile` 仅抛出一个错误提示用户转向 CMake（见 `Makefile:6-9`）。所有子项目（`ggml/`、`src/`、`common/`、`tools/`、`examples/`、`tests/`、`app/`）通过 `add_subdirectory()` 在根 `CMakeLists.txt` 中条件性装配。

版本由 `CMakeLists.txt:5-21` 集中声明：
- `LLAMA_VERSION_MAJOR/MINOR/PATCH` 三段式；
- `LLAMA_BUILD_IS_DEV=ON` 时追加 `-dev` 后缀；
- 开发/nightly 与 release tag 两种模式通过该选项切换。

## 2. 关键文件与目录

- 根构建入口：`CMakeLists.txt`（定义选项、子目录、install/pkg-config）、`CMakePresets.json`（Ninja + MSVC/SYCL/Vulkan 预设）。
- 工具链配置：`cmake/*.cmake`（`x64-windows-llvm.cmake`、`arm64-windows-llvm.cmake`、`arm64-apple-clang.cmake`、`riscv64-spacemit-linux-gnu-gcc.cmake`、`build-info.cmake`、`common.cmake`、`license.cmake`、`git-vars.cmake`）。
- 公共编译开关：`cmake/common.cmake` 提供 `llama_add_compile_flags()`，统一注入 `-Wall -Wextra -Wpedantic`、`-Werror`（当 `LLAMA_FATAL_WARNINGS=ON`）以及 Thread/Address/Undef sanitizer。
- 安装与包导出：生成 `llama-config.cmake`、`llama-config-version.cmake`、`llama.pc`（pkg-config），并安装 `include/llama.h`、`include/llama-cpp.h`。
- Docker 镜像：`.devops/*.Dockerfile`（cpu/cuda/musa/vulkan/openvino/intel/s390x/cann/rocm/zendnn）。
- CI 工作流：`.github/workflows/*.yml`（CPU/CUDA/Vulkan/OpenVINO/SYCL/WebGPU/WASM/Android/RISC-V/MSYS/Apple/Snapdragon 等矩阵）。
- 发布脚本：`scripts/release.sh`、`scripts/make-release-desc.sh`、`scripts/check-release-apiabi.sh`。

## 3. 架构与约定

### 3.1 选项体系
根 `CMakeLists.txt` 暴露一组 `LLAMA_*` 选项控制产物：
- `LLAMA_BUILD_COMMON / LLAMA_BUILD_TESTS / LLAMA_BUILD_TOOLS / LLAMA_BUILD_EXAMPLES / LLAMA_BUILD_SERVER / LLAMA_BUILD_APP / LLAMA_BUILD_UI / LLAMA_BUILD_MTMD`。
- 硬件后端通过 `GGML_*` 选项启用（CUDA、METAL、VULKAN、SYCL、OPENCL、CANN、ROCM、WEBGPU、ET、RPC、BLAS、ZEN_DNN 等），由 `ggml/` 子树处理。
- 旧版 `LLAMA_CUDA`、`LLAMA_METAL` 等通过 `llama_option_depr()` 重定向到 `GGML_*`，实现向后兼容。

默认行为：非 Xcode/MSVC 且未指定 `CMAKE_BUILD_TYPE` 时强制 `Release`；Windows 下默认静态库，其他平台默认动态库；iOS/Android/Emscripten 禁用 subprocess。

### 3.2 子项目装配顺序
1. `ggml/`（或外部 `find_package(ggml)`，由 `LLAMA_USE_SYSTEM_GGML` 控制）→
2. `src/`（核心运行时）→
3. `vendor/` →
4. `common/`（可选）→
5. `tests/`（需 common+tests 开启）→
6. `examples/` + `pocs/` →
7. `tools/` →
8. `app/`（可选）→
9. install/pkg-config 生成。

### 3.3 预置构建
`CMakePresets.json` 提供可复用的 Ninja/MSVC/SYCL/Vulkan 组合，如 `x64-linux-gcc-release`、`arm64-apple-clang+static-release`、`x64-windows-sycl-release-f16`，并通过 `base` 预设统一设置 `CMAKE_EXPORT_COMPILE_COMMANDS=ON` 和 `CMAKE_INSTALL_RPATH=$ORIGIN;$ORIGIN/..`。

### 3.4 多后端分发
每个 GGML 后端位于 `ggml/src/ggml-<backend>/`，通过统一的 `ggml-backend.h` 抽象与动态加载机制接入。CI 按后端拆分工作流（`build-cuda.yml`、`build-vulkan.yml`、`build-openvino.yml`、`build-sycl.yml`、`build-wasm.yml`、`build-webgpu.yml`、`build-android.yml`、`build-riscv.yml`、`build-cann.yml`、`build-virtgpu.yml`、`build-opencl.yml`、`build-msys.yml`、`build-cross.yml`、`build-apple.yml`、`build-ibm.yml`）。

### 3.5 发布流水线
`.github/workflows/release.yml` 是主发布入口：
- `check-release` job 根据 commit message 是否包含 `[no release]` 决定是否打 tag；
- 并行构建 macOS (arm64/x64)、Ubuntu (x64/arm64/s390x)、Windows (x64/arm64) CPU 二进制；
- 单独构建 CUDA (12.8/13.4, x64/arm64)、Vulkan、OpenVINO、ROCm/HIP、Android (arm64)、Snapdragon (arm64 Android/Linux) 等变体；
- 使用 `ccache-action` 缓存构建产物，通过 HuggingFace Bucket (`ggml-org/cache`) 共享 ccache；
- 打包产物命名形如 `llama-<tag>-bin-<os>-<variant>.tar.gz`，CUDA 变体额外附带 `cudart-*.tar.gz`；
- OpenVINO 构建将 runtime 库与 TBB 一并打入 RPATH=`$ORIGIN` 的 tarball，无需 `setupvars.sh`。

辅助流程：`release-publish.yml` 负责最终上传 Release/GitHub Releases；`make-release.yml` 触发 release；`docker.yml` 构建并发布 Docker 镜像；`ui-build.yml` / `ui-publish.yml` 独立构建 Web UI 作为 artifact 供各 release job 下载。

### 3.6 Python 生态
- `convert_hf_to_gguf.py` 等转换器依赖 `requirements/requirements-convert_hf_to_gguf.txt`；
- `gguf-py/` 是独立的 Python 包（`pyproject.toml`），提供 GGUF 读写 API；
- `requirements-all.txt` 聚合全部 Python 依赖。

## 4. 约定与约束

- **禁止直接使用 Makefile 构建**：顶层 `Makefile` 直接 `$(error ...)` 报错，强制迁移至 CMake（来源：`Makefile:6-9`）。
- **版本必须三端一致**：`CMakeLists.txt` 中的 `LLAMA_VERSION_*` 与 git tag `vX.Y.Z` 对应，release 分支上应关闭 `LLAMA_BUILD_IS_DEV`（来源：`CMakeLists.txt:11-21` 注释）。
- **警告即错误**：CI 在所有构建中传递 `-DLLAMA_FATAL_WARNINGS=ON`，使 `-Werror` 生效（来源：`cmake/common.cmake:4-10` 及 `.github/workflows/build-cpu.yml:112`、`release.yml` 多处）。
- **RPATH 固定**：所有 release 构建均设置 `CMAKE_INSTALL_RPATH='$ORIGIN'`（macOS 用 `@loader_path`）并开启 `CMAKE_BUILD_WITH_INSTALL_RPATH=ON`，确保二进制自包含（来源：`release.yml` 中各 job 的 cmake 调用）。
- **后端启用统一走 GGML_* 选项**：旧 `LLAMA_*` 选项经 `llama_option_depr()` 重定向，新增后端应遵循此模式（来源：`CMakeLists.txt:173-195`）。
- **测试运行器**：通过 `ctest -L 'main|python'` 执行，测试按标签分类（来源：`.github/workflows/build-cpu.yml:132`）。
- **CCache 策略**：CI 使用 `ggml-org/ccache-action`，master push 时保存到 HF Bucket `ggml-org/cache`，PR 只读不写（来源：`.github/workflows/build-cpu.yml:98-126`）。
- **Docker 镜像按后端拆分**：`.devops/` 下每个 `<backend>.Dockerfile` 对应一个发行镜像，由 `docker.yml` 统一触发（来源：`.devops/` 目录结构）。
- **Python 类型检查**：通过 `pyrightconfig.json`、`mypy.ini`、`ty.toml`、`.flake8`、`.pre-commit-config.yaml` 在 CI 中校验（来源：`.github/workflows/python-lint.yml`、`python-type-check.yml`）。
- **API ABI 检查**：`scripts/check-release-apiabi.sh` 与 `scripts/check-apiabi-compat.sh` 在 release 前校验 C API 兼容性（来源：`scripts/`）。
- **文档同步**：`scripts/create_ops_docs.py` 从源码生成 `docs/ops/*.csv` 算子支持矩阵（来源：`scripts/`）。

## 5. 适用性判断

本仓库存在完整的 CMake 构建系统、数十个 GitHub Actions 工作流、Docker 镜像、发布脚本与跨平台工具链配置，属于高度成熟的构建与制品管理体系，因此本类别完全适用。