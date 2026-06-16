# smartsub-py-engine

SmartSub 的 Python 推理 sidecar（多引擎：faster-whisper / funasr），独立仓库构建与发布。

主应用 [buxuku/SmartSub](https://github.com/buxuku/SmartSub) 通过 `latest` Release 按需下载，不内置源码。

引擎与底层库：

| engineId | 底层库 | 说明 |
| --- | --- | --- |
| `faster-whisper` | `faster-whisper`（ctranslate2） | 通用多语种 Whisper |
| `funasr` | `sherpa-onnx` + `numpy` | SenseVoice-Small（ONNX，torch-free），中文/粤语/日/韩高准确度，自带 silero VAD |

## Release

每次推送到 `main` 或手动触发 workflow 后，CI 以 `engine_id × 平台` 矩阵（2 引擎 × 4 平台 = 8 个包）发布到：

**https://github.com/buxuku/smartsub-py-engine/releases/tag/latest**

资产（产物名 `smartsub-<engineId>-<suffix>.tar.gz`）：

```
smartsub-faster-whisper-{macos-arm64,macos-x64,windows-x64,linux-x64}.tar.gz
smartsub-funasr-{macos-arm64,macos-x64,windows-x64,linux-x64}.tar.gz
checksums.sha256
manifest.json   # engines[] + 顶层 artifacts(=faster-whisper, 兼容) + enginePackages{<engineId>:{sidecar,artifacts}}
```

## 本地开发

依赖 [`uv`](https://docs.astral.sh/uv/)（锁定 Python，见 `.python-version`）。产物是
**可重定位引擎包**（`main.py` + `site-packages/`），由 SmartSub 内置的
python-build-standalone 基座经 `PYTHONPATH` 加载，不再使用 PyInstaller 冻结。

```bash
# 开发模式冒烟（用 uv 环境直接跑 ./main.py）
uv run --python "$(cat .python-version)" -- python smoke_test.py

# 构建可重定位引擎包到 dist/<engineId>/（main.py + engines/ + site-packages/）
# 第二个参数是 engineId：faster-whisper | funasr，读取 requirements-<engineId>.txt
uv run --python "$(cat .python-version)" -- python build_engine_package.py dist/faster-whisper faster-whisper
uv run --python "$(cat .python-version)" -- python build_engine_package.py dist/funasr funasr

# 包模式冒烟：基座解释器 + PYTHONPATH=site-packages 跑 dist/<engineId>/main.py
PY="$(uv python find "$(cat .python-version)")"
"$PY" smoke_test.py --package dist/funasr "$PY"

# funasr 真实模型转写冒烟（可选，需先下载 SenseVoice int8 + silero_vad.onnx）：
# SMARTSUB_FUNASR_ASR_MODEL / _TOKENS / _VAD_MODEL / _WAV 配齐后，上面的 --package 会顺带跑一次转写。
```

SmartSub 主仓库开发时，把 `dist/<engineId>/` 拷到 `userData/py-engines/<engineId>/`，
或从 Resource Hub 下载安装；App 用内置基座加载该包。

## 协议

stdio JSON-lines，与 SmartSub `PythonRuntimeManager` 对应。详见 `main.py` 头部注释。
