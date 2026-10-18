# image2

通过独立配置的图片中转站生成图片，并保存到当前项目。技能说明见 [SKILL.md](SKILL.md)，执行脚本为 [scripts/imagegen.ps1](scripts/imagegen.ps1)。

聊天和代码任务继续使用原有配置；本技能只在用户要求生图时使用独立图片接口。不会修改聊天模型提供商，也不会读取聊天凭据作为备用密钥。

## 环境要求

- Windows PowerShell 5.1 或 PowerShell 7
- `curl.exe`（脚本实际使用的 HTTP 客户端）
- 安装时需要 Git；生图时需要可用的图片中转站及其密钥

## 安装

在 PowerShell 中执行：

```powershell
git clone https://github.com/ackingchen/image2skill.git "$env:USERPROFILE\.codex\skills\image2"
```

如果目标目录已经存在，先检查已有技能，避免覆盖本地修改。安装后在新的会话中使用该技能。

复制配置模板，保留已有配置：

```powershell
$configPath = Join-Path $env:USERPROFILE '.codex\image2.env'
if (-not (Test-Path -LiteralPath $configPath)) {
    Copy-Item -LiteralPath "$env:USERPROFILE\.codex\skills\image2\image2.env.example" -Destination $configPath
}
notepad $configPath
```

填入你自己的配置：

```dotenv
BASE_URL=https://your-image-relay.example.com/v1
API_KEY=replace-with-your-image-relay-api-key
MODEL=your-supported-image-model
MODE=images
```

模板中的地址、密钥和模型仅为示例。请以你的中转站实际支持情况为准。私有配置放在技能仓库之外，不应上传。

## 使用

先切换到需要保存图片的项目目录，然后执行：

```powershell
$scriptPath = Join-Path $env:USERPROFILE '.codex\skills\image2\scripts\imagegen.ps1'

# 查看可用模型；会访问中转站
& $scriptPath -ListModels

# 仅检查请求构造，不发送生成请求
& $scriptPath -Prompt '水墨风格的远山与松树' -Mode images -DryRun

# 实际生成；可能产生费用
& $scriptPath -Prompt '水墨风格的远山与松树' -Mode images -Size 1024x1024
```

默认输出到当前项目的 `imagegen-output`。可用 `-OutDir` 指定项目内其他位置，用 `-PromptFile` 读取 UTF-8 长提示词。

支持 `images`、`images-stream`、`responses`、`chat` 四种接口模式。`auto` 会依次尝试；`-Probe` 也可能发出多次请求。确认支持的接口后，请固定 `-Mode`，避免重复计费请求。

脚本只有文字生图入口，不支持上传参考图、蒙版或直接编辑已有图片。尺寸、质量、透明背景和输出数量的实际支持取决于中转站和所选模型；本仓库不保证所有模型支持同一组参数。

## 隐私与分享

- 只发布配置模板；`.gitignore` 排除真实 `.env` 配置、凭据文件、图片输出、日志和临时文件。
- `-DryRun` 隐藏 API 密钥，但会显示接口地址、提示词和本地路径，分享输出前请自行脱敏。
- `-KeepRaw` 会保留原始响应，可能包含私人提示词、图片或带签名的下载地址，不应公开。
- 访问返回的图片下载地址时使用 `-SkipAuthorization`，不向图片托管站发送中转站密钥。
- `.gitignore` 不会移除已经提交的敏感信息；提交前仍应检查暂存内容。若密钥已泄露，应立即撤销并更换。

## 许可证

沿用本仓库已有的 [GNU GPL v3 许可证](LICENSE)。
