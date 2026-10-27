# WB Gateway

Python 多账号模型网关，支持国内 / 国际账号、WebUI、余额与签到、模型倍率、调用记录和分区 API Key。不依赖原版桌面客户端或 Node.js 后端。

非官方项目，仅用于你有权使用的账号与接口。   

<img width="2538" height="1400" alt="02feabeb-ff93-44d6-8376-3c346d1cc2c9" src="https://github.com/user-attachments/assets/52ecf49a-479d-4a4d-a514-0d025919098a" />

一键运行，可视化web管理界面



## 运行

支持 Windows、macOS 和 Linux，需要 Python 3.12 或更高版本。

Windows（已在 Python 3.12 验证）：

```powershell
py -3.12 -m venv .venv
./.venv/Scripts/python.exe -m pip install -r requirements.txt
./.venv/Scripts/python.exe app.py
```

默认端口为 `8787`。在图形窗口打开控制台，复制管理密钥后登录。也可使用 `启动网关.pyw`。

无图形模式：

```powershell
./.venv/Scripts/python.exe app.py --headless --host 127.0.0.1 --port 8787
```

macOS / Linux（已在 macOS + Python 3.14 验证 headless 模式）：

```bash
python3 -m venv .venv
./.venv/bin/python -m pip install -r requirements.txt
./.venv/bin/python app.py --headless --host 127.0.0.1 --port 8787
```

不带 `--headless` 时会打开 Tk 图形窗口；若当前 Python 没有 tkinter（如 Homebrew Python 未安装 `python-tk`）或 Linux 没有图形会话，程序会自动以 headless 模式启动。

headless 模式的管理密钥可通过 `WB_ADMIN_KEY` 环境变量指定，未指定则自动生成并保存于私有 `data/keys.json`。

常用环境变量：

| 变量 | 默认值 | 说明 |
|---|---|---|
| `WB_HOST` / `WB_PORT` | `127.0.0.1` / `8787` | 监听地址与端口 |
| `WB_DATA_DIR` | `./data` | 账号、密钥与调用记录目录 |
| `WB_ADMIN_KEY` | 自动生成 | 管理密钥 |
| `WB_MAX_BODY_BYTES` | `104857600` | 请求体大小上限（默认 100MB，适合长上下文与图片） |
| `WB_PROXY_MODE` | `auto` | `direct` 表示不使用 `HTTPS_PROXY` / `HTTP_PROXY` 访问上游 |
| `WB_ALLOWED_ORIGINS` | 空 | 允许跨域访问的 Origin，逗号分隔 |

## 接入

- 控制台：`http://127.0.0.1:8787/ui/`
- OpenAI 风格地址：`http://127.0.0.1:8787/v1`
- Messages 风格地址：`http://127.0.0.1:8787`

在控制台添加账号、完成官方网页授权，再创建调用 Key。客户端使用调用 Key，不使用管理密钥。

通过 `GET /v1/models` 获取实际模型，保留 `cn/` 或 `gl/` 前缀。思考强度、输出上限和工具由客户端设置，并遵循模型实际能力。支持 Chat Completions、无状态 Responses、Messages 和 SSE。

## 思考过程（Reasoning）透传

推理模型的思考过程按原文实时透传为各协议的标准字段，先于正文输出，网关不注入提示词、不改写内容：

| 端点 | 流式 | 非流式 |
|---|---|---|
| OpenAI Chat Completions | `delta.reasoning_content` | `message.reasoning_content` |
| Anthropic Messages | `thinking` block + `thinking_delta` | `thinking` content block |
| OpenAI Responses | `reasoning` output item（`summary_text`） | `reasoning` output item |

思考强度用 `reasoning_effort`（Chat）、`reasoning.effort`（Responses）或 `output_config.effort` / `thinking`（Messages）设置。模型不思考时没有对应字段。流式响应在上游静默期间每 15 秒发送一次 SSE 注释心跳，避免客户端超时。

```python
stream = client.chat.completions.create(
    model="cn/deepseek-v4.1-flash",
    messages=[{"role": "user", "content": "9.11 和 9.9 哪个大？"}],
    stream=True,
)
for chunk in stream:
    if not chunk.choices:
        continue
    delta = chunk.choices[0].delta
    if getattr(delta, "reasoning_content", None):
        print(delta.reasoning_content, end="")
    if delta.content:
        print(delta.content, end="")
```

## 注意

- 默认仅监听本机；远程部署需自行配置 HTTPS 与访问控制。
- 工具由客户端执行；不支持的能力及损坏参数会明确拒绝，不保证所有 Agent 的复杂工作流。

许可证尚未指定，发布前请自行选择并补充 `LICENSE`。
