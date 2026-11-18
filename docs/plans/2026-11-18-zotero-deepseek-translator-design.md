# Zotero DeepSeek PDF 划词翻译插件设计

## 目标

构建一个独立的 Zotero 插件，在 PDF 阅读器中选择文本后，通过 DeepSeek API 翻译为简体中文。插件不依赖 `zotero-pdf-translate`，默认不自动发起请求，用户可主动点击翻译，也可在设置中开启自动翻译。

## 范围

- 支持 Zotero 7.9.9 至 10.9.9，优先在本机 Zotero 10.0.8 验证。
- 在 PDF 选区弹窗中提供翻译、取消、复制和重试操作。
- 提供 DeepSeek API 地址、API Key、模型、语言、提示词和超时配置。
- 提供连接测试、错误提示和会话内翻译缓存。
- 提供中英文界面、自动化测试、XPI 构建脚本和安装文档。

不在首版范围内：

- 全篇 PDF 自动翻译。
- 标题、摘要和批注的批量翻译。
- 独立侧栏或独立阅读器窗口。
- 多翻译服务切换。
- 云同步 API Key。

## 架构

```text
zotero-deepseek-translator/
├─ manifest.json
├─ bootstrap.js
├─ prefs.js
├─ chrome/content/
│  ├─ scripts/
│  │  ├─ index.js
│  │  ├─ deepseek.js
│  │  ├─ prompt.js
│  │  └─ selectionPopup.js
│  ├─ preferences.xhtml
│  └─ styles/preferences.css
├─ locale/
│  ├─ zh-CN/
│  └─ en-US/
├─ test/
├─ build.ps1
└─ README.md
```

`bootstrap.js` 仅管理插件生命周期、Chrome 注册和清理。`index.js` 注册 Zotero Reader 事件与偏好页。`deepseek.js` 负责网络请求和错误归一化。`prompt.js` 负责提示词组装与结果清理。`selectionPopup.js` 负责向 PDF 选区弹窗追加界面。

插件使用原生 JavaScript，不引入运行时依赖。测试使用 Node 内置 `node:test`，构建使用 PowerShell。

## 数据流

1. Zotero Reader 触发 `renderTextSelectionPopup`，插件从 `params.annotation.text` 读取选区。
2. 插件向弹窗追加 DeepSeek 翻译控件，但不替换 Zotero 原有按钮。
3. 默认仅在用户点击“DeepSeek 翻译”时发送请求；自动翻译开启时，弹窗出现即发送。
4. 请求采用 OpenAI 兼容格式，默认发送至 `https://api.deepseek.com/chat/completions`，默认模型为 `deepseek-chat`。
5. 默认提示词要求完成学术翻译，保留数字、公式、引文和专业术语，只返回译文。
6. 请求成功后读取 `choices[0].message.content`，以纯文本显示。
7. 同一 Reader 发起新请求时取消旧请求，避免旧结果覆盖新选区。
8. 会话内缓存以模型、目标语言和原文为键，避免重复翻译同一段落。

## 配置

偏好键统一放在 `extensions.zotero.deepseekPDFTranslator.*` 下。

- `apiEndpoint`：默认 `https://api.deepseek.com/chat/completions`。
- `apiKey`：本地保存，使用密码输入框。
- `model`：默认 `deepseek-chat`。
- `targetLanguage`：默认简体中文。
- `customPrompt`：为空时使用内置学术翻译提示词。
- `autoTranslate`：默认 `false`。
- `timeoutSeconds`：默认 `60`。
- `temperature`：默认 `0.2`。
- `maxResultWidth`：结果区域最大宽度。
- `fontSize`：结果字号。

API Key 不写日志、不放入提示词、不发送到 DeepSeek 以外的地址。若用户修改 API 地址，插件只向该用户指定地址发送请求，并在设置页显示说明。

## 界面

选区弹窗新增独立区块：

- 空闲状态显示“DeepSeek 翻译”。
- 加载状态显示进度提示和“取消”。
- 成功状态显示只读译文，并提供“复制”和“重试”。
- 失败状态显示人类可读错误，并提供“重试”。

结果按纯文本渲染，不执行模型返回的 HTML。长段落、公式字符和中英文混排必须可查看、选择和复制。

## 错误处理

- 未配置 API Key：提示打开设置。
- HTTP 401/403：提示检查密钥和权限。
- HTTP 429：提示请求频率或账户额度限制。
- HTTP 5xx：提示 DeepSeek 服务端异常。
- 网络错误、超时和主动取消使用不同提示。
- JSON 或响应结构异常：显示“响应格式异常”，并记录不含密钥的诊断摘要。
- API 输出若被 Markdown 代码围栏包裹，去除外层围栏；不删除正常 Markdown 内容。

## 安全

- 不加载远程脚本。
- 不使用 `eval` 或动态执行 API 响应。
- 使用纯文本节点展示模型输出。
- API Key 仅保存在 Zotero 本地偏好中，文档明确其属于本地明文配置。
- 构建产物不包含开发密钥、测试凭据和本地配置。

## 测试与交付

Node 测试覆盖：

- DeepSeek 请求构造和认证头。
- API 响应提取、异常结构处理和代码围栏清理。
- 401、429、5xx、超时、网络错误和取消的错误归一化。
- 默认与自定义提示词组装。
- 选区文本空白和超长文本处理。

构建验证：

- 校验 manifest 版本范围和必需文件。
- 校验 XPI 路径结构。
- 运行全部 Node 测试和 JavaScript 语法检查。
- 提供专用临时 Zotero 配置的启动测试说明，不修改日常配置。

交付物包括源码、README、测试、`build.ps1` 和可安装 XPI。
