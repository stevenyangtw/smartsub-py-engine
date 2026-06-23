# smartsub-py-engine

SmartSub 的 Python 推理 sidecar（**faster-whisper**），独立仓库构建与发布。

主应用 [buxuku/SmartSub](https://github.com/buxuku/SmartSub) 通过 `latest` Release 按需下载，不内置源码。

faster-whisper 是 SmartSub 唯一的 Python 引擎；funasr / qwen / firered 已改用 App 内的
**node sherpa-onnx addon**（原生库随应用内置），不再走本 Python sidecar。

## 产物形态：单个自包含运行时

每个 `(os, arch)` 发布**一个**自包含运行时 `smartsub-faster-whisper-runtime-<suffix>.tar.gz`，
内含「内嵌 python-build-standalone(PBS) CPython + `site-packages/` + `main.py`」：

```
runtime/
  bin/python3 (unix) | python.exe (win)   # PBS 解释器（可重定位）
  lib/...                                   # PBS 标准库
  site-packages/<deps>                      # uv 解析的官方平台 wheel
  main.py, _version.py, engines/            # sidecar 源码
```

SmartSub 下载匹配本机 `(os,arch)` 的产物后，以 `PYTHONHOME=<runtime>`、
`PYTHONPATH=<runtime>/site-packages` 启动 `<runtime>` 内嵌解释器跑 `main.py`。
**不再有可单独下载的 Python 基座**（基座已并入运行时），也**不使用 PyInstaller**——
faster-whisper 拖出的原生 wheel（ctranslate2 / av / tokenizers / onnxruntime）由
`uv` 安装正确的逐平台 wheel（原生依赖经上游 delocate/auditwheel/delvewheel 预捆绑），
跨平台远比 PyInstaller hook 收集稳健。

## Release

每次推送到 `main` 或手动触发 workflow 后，CI 以 `(os,arch)` 矩阵（4 平台）在各自原生 runner
（host == target）构建并发布到：

**https://github.com/buxuku/smartsub-py-engine/releases/tag/latest**

资产：

```
# CPU 包（默认，全平台）
smartsub-faster-whisper-runtime-{macos-arm64,macos-x64,windows-x64,linux-x64}.tar.gz
# Full GPU(CUDA12) 包（仅 windows-x64 / linux-x64；捆绑 NVIDIA cuBLAS/cuDNN，~1.4GB）
smartsub-faster-whisper-runtime-{windows-x64,linux-x64}-cuda.tar.gz
checksums.sha256
manifest.json   # engineVersion / protocolVersion / pythonVersion + runtime.artifacts(per-suffix sha256/size)
```

### CPU / GPU 变体

- **CPU 包**（原包名，默认）：仅 `faster-whisper`，体积小，全平台可用。
- **GPU 包**（`-cuda` 后缀）：在 CPU 基础上**捆绑** `nvidia-cublas-cu12` / `nvidia-cudnn-cu12`(9.x) /
  `nvidia-cuda-runtime-cu12`，N 卡 + 较新驱动（CUDA 12.3+）开箱即用，无需用户自装 CUDA Toolkit/cuDNN。
  仅 `windows-x64` / `linux-x64` 有官方 nvidia wheel；macOS 无 GPU，不产 GPU 包。
- `main.py` 启动时会在首个原生 import 前定位 CUDA 库：优先包内 `site-packages/nvidia/*`，
  其次系统 CUDA；Windows 用 `os.add_dll_directory`+`PATH`，Linux 用 `LD_LIBRARY_PATH` 后 re-exec 一次。
  CPU 包无 `nvidia/` 目录时该步为 no-op。
- App 侧默认拉取 CPU 包；是否下载 GPU 包由主仓库 SmartSub 按显卡/驱动判定与引导（本仓库只负责产出二者）。

## 本地开发

依赖 [`uv`](https://docs.astral.sh/uv/)（锁定 Python，见 `.python-version`）。

```bash
# 开发模式冒烟（用 uv 环境直接跑 ./main.py）
uv run --python "$(cat .python-version)" -- python smoke_test.py

# 构建单自包含运行时到 dist/runtime/（PBS 解释器 + site-packages + main.py）
uv run --python "$(cat .python-version)" -- python build_runtime_package.py dist/runtime

# 构建 Full GPU(CUDA12) 变体到 dist/runtime-cuda/（仅 windows-x64 / linux-x64；捆绑 cuBLAS/cuDNN）
uv run --python "$(cat .python-version)" -- python build_runtime_package.py dist/runtime-cuda --variant cuda

# 包模式冒烟：用运行时内嵌解释器 + PYTHONPATH=site-packages 跑 dist/runtime/main.py
#   unix:  dist/runtime/bin/python3 smoke_test.py --package dist/runtime dist/runtime/bin/python3
#   win:   dist/runtime/python.exe  smoke_test.py --package dist/runtime dist/runtime/python.exe
```

SmartSub 主仓库开发时，把 `dist/runtime/` 拷到 `userData/py-engines/faster-whisper/`，
或从「引擎与模型」页下载安装；App 直接用该运行时内嵌解释器加载。

构建加固（跨平台确定性）：`build_runtime_package.py` 内 `uv pip install --only-binary=:all:`
禁止任何 sdist 在 runner 源码编译（故 runner CPU 的 `-march` 绝不会烘进产物）；PBS 为预构建
baseline；重原生库（ct2/onnxruntime/numpy/av）运行时按 ISA 派发，老 CPU 自动走 baseline。

## 协议

stdio JSON-lines，与 SmartSub `PythonRuntimeManager` 对应。详见 `main.py` 头部注释。
