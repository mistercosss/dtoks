# 高性能 Edge TTS Cloudflare Worker 代理

这是一个部署在 Cloudflare Workers 上的高性能文本转语音（TTS）代理服务。它巧妙地将微软 Edge 强大且自然的语音合成服务，封装成了一个兼容 OpenAI API 格式的接口。这使得开发者可以无缝地将各种现有应用对接到这个免费、高质量的 TTS 服务上。

项目包含两个核心文件：

1.  `workers.js`: 部署在 Cloudflare 上的核心服务脚本。
2.  `webui.html`: 一个功能完备的本地网页，用于方便地测试和调用服务。

---

## ✨ 功能亮点

- **🚀 OpenAI 兼容**: 完全模拟 OpenAI 的 `/v1/audio/speech` 接口，可被官方的 OpenAI SDK 或任何现有工具直接调用。
- **🗣️ 高质量音色**: 利用微软 Edge TTS 提供的多种自然、流畅的神经网络语音。
- ** STREAMING**: 支持**流式**和**标准**（非流式）两种响应模式，流式响应可极大降低长文本的首次播放延迟。
- **🧠 智能文本清理**: 内置强大的“文本清理流水线”，可自动处理从 PDF 或网页复制的杂乱文本：
  - 移除 Markdown 格式。
  - 移除 Emoji 表情符号。
  - 移除 URL 链接。
  - 移除论文或文档中的引用标记数字（如 `...文本 1.`），同时保留有意义的数字（如年份、数量）。
  - 移除所有多余的空格和硬换行，确保中文听感自然连贯。
  - 支持自定义关键词过滤。
- **🎛️ 灵活的参数配置**: 支持通过 API 请求动态调整所有核心参数，包括音色、语速、音调、分块大小、并发数以及所有清理选项。
- **🌐 零依赖部署**: 脚本完全自包含，无需配置 KV、队列等任何外部服务，部署过程极其简单。
- **💻 便捷的测试工具**: 提供一个功能丰富的 `webui.html`，让用户无需编写任何代码即可测试所有功能。

---

## 🚀 它是如何工作的？

这个 Worker 脚本扮演了一个智能“中间人”的角色。

1.  **接收请求**: 它接收一个格式与 OpenAI TTS API 完全相同的 `POST` 请求。
2.  **文本预处理**: 它会立即运行强大的文本清理流水线，对输入的文本进行净化，以达到最佳的听感。
3.  **智能决策**: 它会快速预估处理这段文本所需的资源。
    - **短文本**: 如果文本长度在安全范围内，它会采用最高效的“直接同步模式”进行处理。
    - **超长文本**: 如果文本长度可能触发 Cloudflare 的平台限制，它会自动启用“自我调节机制”，重新计算并调整分块策略，以保证任务 100%成功。
4.  **分块与并发**: 它将处理好的文本分割成多个小块，并根据您指定的并发数，分批次地向微软 Edge TTS 服务器发起请求。
5.  **返回结果**:
    - **标准模式**: 等待所有音频块返回，拼接成一个完整的 MP3 文件后一次性返回。
    - **流式模式**: 在收到第一个音频块时就立即开始向客户端发送数据，实现低延迟播放。

---

## 部署指南

部署这个服务非常简单，只需要几分钟。

### 准备工作

- 一个 Cloudflare 账户（免费版即可）。

### 步骤一：创建 Cloudflare Worker

1.  登录到您的 Cloudflare 仪表板。
2.  在左侧菜单中，找到并点击 **Workers 和 Pages**。
3.  点击 **创建应用程序** -> **创建 Worker**。
4.  为您的 Worker 指定一个独特的名称（例如 `my-edge-tts-proxy`），这将成为您 URL 的一部分。
5.  点击 **部署**。

### 步骤二：配置 Worker

1.  Worker 创建成功后，点击 **配置 Worker** 按钮。
2.  **粘贴代码**: 在左侧的编辑器中，删除所有现有代码，然后将 `workers.js` 文件的**全部内容**复制并粘贴进去。
3.  **添加密钥**:
    - 在右侧，选择 **设置** -> **变量**。
    - 在“环境变量”下的 **机密变量** 部分，点击 **添加机密变量**。
    - **变量名称**: 输入 `API_KEY`
    - **机密值**: 输入一个您自己设定的、足够复杂的密码（例如 `sk-my-super-secret-key-12345`）。这个值将是您调用 API 时的凭证。
    - 点击 **加密**。

### 步骤三：部署

1.  完成以上配置后，返回代码编辑器页面。
2.  点击右上角的 **保存并部署** 按钮。

恭喜！您的 TTS 代理服务现在已经全球可用了。您的 API 地址就是 Worker 页面上显示的 `https://<您的Worker名称>.<您的子域>.workers.dev`。

### 步骤四：使用 Web UI

~~1.  将 `webui.html` 文件保存到您的本地电脑上。~~
~~2.  用您的浏览器（如 Chrome, Safari）直接打开这个 HTML 文件。~~
1.  用浏览器访问上方的`https://<您的Worker名称>.<您的子域>.workers.dev`，即可使用webui了（本仓库中的`webui.html`没有实际用途，仅用于归档，因为它的全部内容实际已经在`worker.js`里有一份了）。
2.  在页面的“API 配置”部分，填入您的 Worker URL 和您刚才设置的 API Key。
3.  现在，您可以尽情测试所有功能了！

---

## 🛠️ API 使用指南

### 端点

`POST https://<您的Worker名称>.<您的子域>.workers.dev/v1/audio/speech`

### 认证

使用 `Bearer Token` 认证方式。将您的 API Key 放在 `Authorization` 请求头中。

`Authorization: Bearer YOUR_API_KEY`

### 请求体参数 (`JSON`)

| 参数 (Parameter)            | 类型 (Type) | 默认值 (Default)         | 描述 (Description)                                                    |
| --------------------------- | ----------- | ------------------------ | --------------------------------------------------------------------- |
| `model`                     | `string`    | `"tts-1"`                | 模型 ID。支持 `tts-1`, `tts-1-hd`，或映射的音色如 `tts-1-alloy`。     |
| `input`                     | `string`    | **必需**                 | 需要转换为语音的文本。**支持任意长度**。                              |
| `voice`                     | `string`    | `"zh-CN-XiaoxiaoNeural"` | 直接指定微软的音色名称。当 `model` 参数未被映射时生效。               |
| `speed`                     | `number`    | `1.0`                    | 语速。范围从 0.25 到 2.0。                                            |
| `pitch`                     | `number`    | `1.0`                    | 音调。                                                                |
| `stream`                    | `boolean`   | `false`                  | 是否使用流式响应。设为 `true` 可极大降低长文本的首次延迟。            |
| `concurrency`               | `number`    | `10`                     | 并发请求数。控制同时向微软服务器发送多少个文本块请求。                |
| `chunk_size`                | `number`    | `300`                    | 文本分块大小（字符数）。Worker 会根据平台限制自动调整此值以确保成功。 |
| `cleaning_options`          | `object`    | `{...}`                  | 一个包含文本清理开关的对象。                                          |
| `├ remove_markdown`         | `boolean`   | `true`                   | 是否移除 Markdown 格式。                                              |
| `├ remove_emoji`            | `boolean`   | `true`                   | 是否移除 Emoji。                                                      |
| `├ remove_urls`             | `boolean`   | `true`                   | 是否移除 URL。                                                        |
| `├ remove_line_breaks`      | `boolean`   | `true`                   | 是否移除所有空格和换行符。                                            |
| `├ remove_citation_numbers` | `boolean`   | `true`                   | 是否智能移除论文引用标记。                                            |
| `├ custom_keywords`         | `string`    | `""`                     | 自定义要移除的关键词，以逗号分隔。                                    |

### cURL 示例

#### 1. 标准请求

```bash
curl --location 'https://<YOUR_WORKER_URL>/v1/audio/speech' \
--header 'Authorization: Bearer YOUR_API_KEY' \
--header 'Content-Type: application/json' \
--data '{
    "model": "tts-1-alloy",
    "input": "你好，这是一个标准的语音合成请求。"
}' \
--output standard.mp3
```

#### 2. 流式请求 (用于长文本)

```bash
curl --location 'https://<YOUR_WORKER_URL>/v1/audio/speech' \
--header 'Authorization: Bearer YOUR_API_KEY' \
--header 'Content-Type: application/json' \
--data '{
    "model": "tts-1-nova",
    "input": "这是一个流式请求的示例，对于较长的文本，你能更快地听到声音的开头部分。",
    "stream": true
}' \
--output streaming.mp3
```

#### 3. 使用高级过滤选项

```bash
curl --location 'https://<YOUR_WORKER_URL>/v1/audio/speech' \
--header 'Authorization: Bearer YOUR_API_KEY' \
--header 'Content-Type: application/json' \
--data '{
    "model": "tts-1-shimmer",
    "input": "这段文本包含 **Markdown** 1、一个链接 https://example.com 和一个表情 😂。",
    "cleaning_options": {
        "remove_markdown": true,
        "remove_urls": true,
        "remove_emoji": true,
        "remove_citation_numbers": true
    }
}' \
--output cleaned.mp3
```

---

## ⚠️ 重要限制

- **字符数限制**: 此版本的脚本设计为在 **Cloudflare 免费套餐** 上稳定运行。为了保证这一点，对于单次请求的文本长度有一个隐性的上限，大约在 **12 万字符** 左右 (`~50 * 2500`)。对于绝大多数应用场景（包括转化整篇长文）都已经完全足够。超过这个长度，API 会返回一个清晰的错误提示。

---

## 📄 项目文件说明

- **`workers.js`**: 核心服务逻辑。包含了 API 路由、认证、文本清理、自我调节机制、以及与微软 TTS 服务的交互。这是您需要部署到 Cloudflare 的唯一文件。
- **`webui.html`**: 一个独立的、功能强大的 HTML 测试页面。它无需任何服务器，可以直接在本地用浏览器打开。它封装了所有 API 参数，并提供了一个直观的界面来与您的 Worker 服务进行交互。

---
