# smartsub-py-engine

SmartSub 的 Python 推理 sidecar（faster-whisper），独立仓库构建与发布。

主应用 [buxuku/SmartSub](https://github.com/buxuku/SmartSub) 通过 `latest` Release 按需下载，不内置源码。

## Release

每次推送到 `main` 或手动触发 workflow 后，产物发布到：

**https://github.com/buxuku/smartsub-py-engine/releases/tag/latest**

资产（产物名按引擎区分，便于后续 funasr/qwen 多引擎共存同一 release）：

```
smartsub-faster-whisper-macos-arm64.tar.gz
smartsub-faster-whisper-macos-x64.tar.gz
smartsub-faster-whisper-windows-x64.tar.gz
smartsub-faster-whisper-linux-x64.tar.gz
checksums.sha256
manifest.json
```

## 本地开发

依赖 [`uv`](https://docs.astral.sh/uv/)（锁定 Python，见 `.python-version`）。产物是
**可重定位引擎包**（`main.py` + `site-packages/`），由 SmartSub 内置的
python-build-standalone 基座经 `PYTHONPATH` 加载，不再使用 PyInstaller 冻结。

```bash
# 开发模式冒烟（用 uv 环境直接跑 ./main.py）
uv run --python "$(cat .python-version)" -- python smoke_test.py

# 构建可重定位引擎包到 dist/package/（main.py + site-packages/）
uv run --python "$(cat .python-version)" -- python build_engine_package.py dist/package

# 包模式冒烟：基座解释器 + PYTHONPATH=site-packages 跑 dist/package/main.py
PY="$(uv python find "$(cat .python-version)")"
"$PY" smoke_test.py --package dist/package "$PY"
```

SmartSub 主仓库开发时，把 `dist/package/` 拷到 `userData/py-engines/faster-whisper/`，
或从 Resource Hub 下载安装；App 用内置基座加载该包。

## 协议

stdio JSON-lines，与 SmartSub `PythonRuntimeManager` 对应。详见 `main.py` 头部注释。
