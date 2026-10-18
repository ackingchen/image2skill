---
name: "image2"
description: "Generate images through a separately configured image relay and save them locally. Use when the user requests an image, illustration, poster, banner, icon, logo, or visual asset, including implicit requests such as '给我配张图'. Use this relay workflow when configured, including environments where built-in image generation is unavailable. Do not use for ordinary chat or code requests, and never change the chat model provider."
---

# image2：独立中转站生图技能（分享版）

这个技能只做一件事：**在用户明确要求生图时，调用用户自己的图片站接口出图并保存到本地。**

- 聊天、写代码：继续走 `~/.codex/config.toml` 里配置的 `model_provider`，**不要动它**。
- 生图：只读下面这个私有配置文件，走另一条链路。

## 配置文件

```
%USERPROFILE%\.codex\image2.env
```

格式（一行一个字段）：

```
BASE_URL=https://your-image-relay.example.com/v1
API_KEY=replace-with-your-image-relay-api-key
MODEL=gpt-image-1
```

可选字段：`CHAT_MODEL`、`RESPONSES_MODEL`、`OUT_DIR`、`MODE`。

首次安装时，将技能目录里的 `image2.env.example` 复制到上述位置，再填写自己的图片站地址、密钥和模型名。不要覆盖已有配置。
脚本需要 Windows PowerShell 5.1 或 PowerShell 7，以及 `curl.exe`。

**规则：永远不要在回复里显示 API_KEY，也不要把配置文件内容完整打印出来。**
**不要扫描、不要复用 `~/.codex/auth.json`、`config.toml`、ccswitch 里聊天用的 key 或 base_url。**
配置可被 `IMAGEGEN_*` 环境变量覆盖；此分享版不会使用 `OPENAI_API_KEY` 兜底。

## 生成命令

```powershell
powershell -NoProfile -ExecutionPolicy Bypass -File "$env:USERPROFILE\.codex\skills\image2\scripts\imagegen.ps1" `
  -Prompt "<提示词>" `
  -Size 1024x1024
```

常用参数：

| 参数 | 说明 |
| --- | --- |
| `-Prompt` | 提示词 |
| `-PromptFile` | 从 UTF-8 文件读提示词（长提示词推荐） |
| `-Size` | `1024x1024` 方图 / `1536x1024` 横版 / `1024x1536` 竖版 / `auto` |
| `-Quality` | `low` 草稿 / `medium` / `high`；不传则不发该字段 |
| `-N` | 出图张数，默认 1 |
| `-Mode` | `auto`（默认）/ `images` / `images-stream` / `responses` / `chat` |
| `-Out` / `-OutDir` | 输出文件 / 输出目录，默认 `<当前项目>\imagegen-output\` |
| `-ListModels` | 只列模型，不产生费用 |
| `-DryRun` | 只打印配置和请求体，不发请求 |
| `-Probe` | 查询模型并依次实测接口，到第一个成功为止；可能发生多次付费请求 |
| `-KeepRaw` | 保留原始响应，排错用 |
| `-Force` | 覆盖同名文件（默认自动改名 `-v2`） |
| `-EnvFile` | 指定独立配置文件；也可通过 `IMAGEGEN_ENV_FILE` 指定 |
| `-Model` | 覆盖 `MODEL`，用于 images / images-stream 模式 |
| `-TimeoutSec` | 请求超时秒数，默认 300 |

尺寸习惯：方图/头像/图标 → `1024x1024`；横版/banner/封面 → `1536x1024`；竖版/手机壁纸 → `1024x1536`。

**输出目录必须在当前项目里**。默认使用当前工作目录下的 `imagegen-output`；调用前确认当前目录是用户项目。若 `OUT_DIR` 或 `IMAGEGEN_OUT_DIR` 指向项目外，用 `-OutDir` 显式指定项目内目录。

本脚本当前只有文字提示词入口，不支持上传参考图、蒙版或直接修改已有图片。收到图片编辑请求时说明这一限制，不把文字生图称为已完成原图编辑。

## 接口形态排查

不同中转站支持的接口不一样，脚本按 `-Mode auto` 依次尝试，成功即停：

1. `images` —— `POST {BASE}/images/generations`（最常见）
2. `images-stream` —— 同上但 `stream=true`，用于规避 Cloudflare 524 超时
3. `responses` —— `POST {BASE}/responses` + `image_generation` 工具（建议配置该站支持的 `RESPONSES_MODEL`；脚本默认回退到 `CHAT_MODEL` 或 `gpt-5.1`）
4. `chat` —— `POST {BASE}/chat/completions`，读取回复里内联的图片（需要配 `CHAT_MODEL`，如 `gemini-2.5-flash-image`）

自动尝试和 `-Probe` 可能产生多次请求。确认接口后优先指定 `-Mode`，避免重复尝试；不要在失败后无限重试。`-N` 只会在 images / images-stream 请求中发送，responses / chat 不保证指定张数。

失败时的处置顺序：

1. `-ListModels` 看真实的图片模型名，改 `MODEL`
2. `-DryRun` 检查 BASE_URL / 请求体
3. `-Probe` 看哪种形态能通
4. 仍失败可用 `-KeepRaw` 保留原始响应排查，只给用户展示 HTTP 状态和必要的错误片段；先检查是否包含凭据、私人提示词或图片内容，不直接公开完整响应

## 回复格式（必须遵守）

生成成功后，在回复里给出：

1. 图片预览 —— **Markdown 路径必须用正斜杠**，否则 Codex 预览面板会报 Unable to load file：

```markdown
![image2 output](C:/projects/demo/imagegen-output/generated-image.png)
```

2. 保存的绝对路径列表
3. 实际使用的模式与模型名

失败时只说清三件事：哪个环节失败（key / URL / 模型名 / 接口形态 / 网络）、HTTP 状态、上游返回的错误片段。

