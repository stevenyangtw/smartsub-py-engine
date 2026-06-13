# smartsub-py-engine

SmartSub 的 Python 推理 sidecar（faster-whisper），独立仓库构建与发布。

主应用 [buxuku/SmartSub](https://github.com/buxuku/SmartSub) 通过 `latest` Release 按需下载，不内置源码。

## Release

每次推送到 `main` 或手动触发 workflow 后，产物发布到：

**https://github.com/buxuku/smartsub-py-engine/releases/tag/latest**

资产：

```
smartsub-engine-macos-arm64.tar.gz
smartsub-engine-macos-x64.tar.gz
smartsub-engine-windows-x64.tar.gz
smartsub-engine-linux-x64.tar.gz
checksums.sha256
```

## 本地开发

```bash
python3 -m venv .venv
source .venv/bin/activate   # Windows: .venv\Scripts\activate
pip install -r requirements.txt pyinstaller

# 开发模式冒烟
python smoke_test.py

# 冻结构建
pyinstaller --clean --noconfirm smartsub-engine.spec
python smoke_test.py dist/smartsub-engine/smartsub-engine
```

SmartSub 主仓库开发时，可先在本仓库构建，再在 SmartSub 中设置：

```bash
export PYTHON_ENGINE_CMD="/path/to/dist/smartsub-engine/smartsub-engine"
```

或从 Resource Hub 下载安装到 `userData/py-engine/current/`。

## 协议

stdio JSON-lines，与 SmartSub `PythonRuntimeManager` 对应。详见 `main.py` 头部注释。
