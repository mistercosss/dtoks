/**
 * Cloudflare Worker - Microsoft Edge TTS 服务代理
 *
 * @version 2.4.0 (稳定版)
 * @description 实现了内部自动批处理机制，优雅地处理 Cloudflare 的子请求限制。
 * API 现在可以处理任何长度的文本，不会因为"子请求过多"而失败。
 * 这是最终的生产就绪版本。
 * 
 * @features
 * - 支持流式和非流式 TTS 输出
 * - 自动文本清理和分块处理
 * - 智能批处理避免 Cloudflare 限制
 * - 兼容 OpenAI TTS API 格式
 * - 支持多种中英文语音
 */

// =================================================================================
// 配置参数
// =================================================================================

// API 密钥配置
const API_KEY = globalThis.API_KEY;

// 批处理配置 - 控制并发请求数量以避免 Cloudflare 限制
const DEFAULT_CONCURRENCY = 10; // 现在作为批处理大小使用
const DEFAULT_CHUNK_SIZE = 300; // 默认文本分块大小

// OpenAI 语音映射到 Microsoft 语音
const OPENAI_VOICE_MAP = {
  "shimmer": "zh-CN-XiaoxiaoNeural",    // 温柔女声 -> 晓晓
  "alloy": "zh-CN-YunyangNeural",       // 专业男声 -> 云扬  
  "fable": "zh-CN-YunjianNeural",       // 激情男声 -> 云健
  "onyx": "zh-CN-XiaoyiNeural",         // 活泼女声 -> 晓伊
  "nova": "zh-CN-YunxiNeural",          // 阳光男声 -> 云希
  "echo": "zh-CN-liaoning-XiaobeiNeural" // 东北女声 -> 晓北
};

const htmlContent = getHtmlContent();

// =================================================================================
// 主事件监听器
// =================================================================================

addEventListener("fetch", event => {
  event.respondWith(handleRequest(event));
});

/**
 * 处理所有传入的 HTTP 请求
 * @param {FetchEvent} event - Cloudflare Worker 事件对象
 * @returns {Promise<Response>} HTTP 响应
 */
async function handleRequest(event) {
  const request = event.request;

  // 处理 CORS 预检请求
  if (request.method === "OPTIONS") return handleOptions(request);

  const url = new URL(request.url);
  // 处理HTML页面请求
  if (url.pathname === '/' || url.pathname === '/index.html') {
    return new Response(htmlContent, {
      headers: {
        "Content-Type": "text/html;charset=UTF-8",
        "Cache-Control": "public, max-age=86400" // 缓存1d
      }
    });
  }

  // API 密钥验证
  if (API_KEY) {
    const authHeader = request.headers.get("authorization");
    if (!authHeader || !authHeader.startsWith("Bearer ") || authHeader.slice(7) !== API_KEY) {
      return errorResponse("无效的 API 密钥", 401, "invalid_api_key");
    }
  }


  try {
    // 路由分发
    if (url.pathname === "/v1/audio/speech") return await handleSpeechRequest(request);
    if (url.pathname === "/v1/models") return handleModelsRequest();
  } catch (err) {
    console.error("请求处理器错误:", err);
    return errorResponse(err.message, 500, "internal_server_error");
  }

  return errorResponse("未找到", 404, "not_found");
}


// =================================================================================
// 路由处理器
// =================================================================================

/**
 * 处理 CORS 预检请求
 * @param {Request} request - HTTP 请求对象
 * @returns {Response} CORS 响应
 */
function handleOptions(request) {
  const headers = makeCORSHeaders(request.headers.get("Access-Control-Request-Headers"));
  return new Response(null, { status: 204, headers });
}

/**
 * 处理语音合成请求
 * @param {Request} request - HTTP 请求对象
 * @returns {Promise<Response>} 语音数据响应
 */
async function handleSpeechRequest(request) {
  if (request.method !== "POST") {
    return errorResponse("不允许的方法", 405, "method_not_allowed");
  }

  const requestBody = await request.json();
  if (!requestBody.input) {
    return errorResponse("'input' 是必需参数", 400, "invalid_request_error");
  }

  // 解析请求参数并设置默认值
  const {
    model = "tts-1",                    // 模型名称
    input,                              // 输入文本
    voice = "shimmer",                  // 语音
    speed = 1.0,                        // 语速 (0.25-2.0)
    pitch = 1.0,                        // 音调 (0.5-1.5)
    style = "general",                  // 语音风格
    stream = false,                     // 是否流式输出
    concurrency = DEFAULT_CONCURRENCY, // 并发数
    chunk_size = DEFAULT_CHUNK_SIZE,    // 分块大小
    cleaning_options = {}               // 文本清理选项
  } = requestBody;

  // 合并默认清理选项
  const finalCleaningOptions = {
    remove_markdown: true,      // 移除 Markdown
    remove_emoji: true,         // 移除 Emoji
    remove_urls: true,          // 移除 URL
    remove_line_breaks: true,   // 移除换行符
    remove_citation_numbers: true, // 移除引用数字
    custom_keywords: "",        // 自定义关键词
    ...cleaning_options
  };

  // 清理输入文本
  const cleanedInput = cleanText(input, finalCleaningOptions);

  // 语音映射处理
  const modelVoice = !voice ? OPENAI_VOICE_MAP[model.replace('tts-1-', '')] : null;
  const finalVoice = modelVoice || voice;
  if (!finalVoice) {
    return errorResponse("无效的语音模型", 400, "invalid_request_error");
  }

  // 参数转换为 Microsoft TTS 格式
  const rate = ((speed - 1) * 100).toFixed(0);        // 语速转换
  const finalPitch = ((pitch - 1) * 100).toFixed(0);  // 音调转换
  const outputFormat = "audio-24khz-48kbitrate-mono-mp3"; // 输出格式

  // 智能文本分块
  const textChunks = smartChunkText(cleanedInput, chunk_size);
  const ttsArgs = [finalVoice, rate, finalPitch, style, outputFormat];

  // 根据是否流式选择处理方式
  if (stream) {
    return await streamVoice(textChunks, concurrency, ...ttsArgs);
  } else {
    return await getVoice(textChunks, concurrency, ...ttsArgs);
  }
}

/**
 * 处理模型列表请求
 * @returns {Response} 可用模型列表
 */
function handleModelsRequest() {
  const models = [
    { id: 'tts-1', object: 'model', created: Date.now(), owned_by: 'openai' },
    { id: 'tts-1-hd', object: 'model', created: Date.now(), owned_by: 'openai' },
    ...Object.keys(OPENAI_VOICE_MAP).map(v => ({
      id: `tts-1-${v}`,
      object: 'model',
      created: Date.now(),
      owned_by: 'openai'
    }))
  ];
  return new Response(JSON.stringify({ object: "list", data: models }), {
    headers: { "Content-Type": "application/json", ...makeCORSHeaders() }
  });
}

// =================================================================================
// 核心 TTS 逻辑 (自动批处理机制)
// =================================================================================

/**
 * 流式语音生成
 * @param {string[]} textChunks - 文本块数组
 * @param {number} concurrency - 并发数
 * @param {...any} ttsArgs - TTS 参数
 * @returns {Promise<Response>} 流式音频响应
 */
async function streamVoice(textChunks, concurrency, ...ttsArgs) {
  const { readable, writable } = new TransformStream();
  try {
    // 等待流式管道完成以便捕获错误
    await pipeChunksToStream(writable.getWriter(), textChunks, concurrency, ...ttsArgs);
    return new Response(readable, {
      headers: { "Content-Type": "audio/mpeg", ...makeCORSHeaders() }
    });
  } catch (error) {
    console.error("流式 TTS 失败:", error);
    return errorResponse(error.message, 500, "tts_generation_error");
  }
}

/**
 * 将文本块流式传输到响应流
 * @param {WritableStreamDefaultWriter} writer - 写入器
 * @param {string[]} chunks - 文本块
 * @param {number} concurrency - 并发数
 * @param {...any} ttsArgs - TTS 参数
 */
async function pipeChunksToStream(writer, chunks, concurrency, ...ttsArgs) {
  try {
    // 分批处理文本块以避免超出 Cloudflare 子请求限制
    for (let i = 0; i < chunks.length; i += concurrency) {
      const batch = chunks.slice(i, i + concurrency);
      const audioPromises = batch.map(chunk => getAudioChunk(chunk, ...ttsArgs));

      // 仅等待当前批次完成
      const audioBlobs = await Promise.all(audioPromises);

      // 将音频数据写入流
      for (const blob of audioBlobs) {
        const arrayBuffer = await blob.arrayBuffer();
        writer.write(new Uint8Array(arrayBuffer));
      }
    }
  } catch (error) {
    console.error("流式 TTS 失败:", error);
    writer.abort(error);
    throw error;
  } finally {
    writer.close();
  }
}

/**
 * 非流式语音生成
 * @param {string[]} textChunks - 文本块数组
 * @param {number} concurrency - 并发数
 * @param {...any} ttsArgs - TTS 参数
 * @returns {Promise<Response>} 完整音频响应
 */
async function getVoice(textChunks, concurrency, ...ttsArgs) {
  const allAudioBlobs = [];
  try {
    // 非流式模式也使用批处理
    for (let i = 0; i < textChunks.length; i += concurrency) {
      const batch = textChunks.slice(i, i + concurrency);
      const audioPromises = batch.map(chunk => getAudioChunk(chunk, ...ttsArgs));

      // 等待当前批次并收集结果
      const audioBlobs = await Promise.all(audioPromises);
      allAudioBlobs.push(...audioBlobs);
    }

    // 合并所有音频数据
    const concatenatedAudio = new Blob(allAudioBlobs, { type: 'audio/mpeg' });
    return new Response(concatenatedAudio, {
      headers: { "Content-Type": "audio/mpeg", ...makeCORSHeaders() }
    });
  } catch (error) {
    console.error("非流式 TTS 失败:", error);
    return errorResponse(error.message, 500, "tts_generation_error");
  }
}

/**
 * 获取单个文本块的音频数据
 * @param {string} text - 文本内容
 * @param {string} voiceName - 语音名称
 * @param {string} rate - 语速
 * @param {string} pitch - 音调
 * @param {string} style - 语音风格
 * @param {string} outputFormat - 输出格式
 * @returns {Promise<Blob>} 音频 Blob
 */
async function getAudioChunk(text, voiceName, rate, pitch, style, outputFormat) {
  const endpoint = await getEndpoint();
  const url = `https://${endpoint.r}.tts.speech.microsoft.com/cognitiveservices/v1`;
  const ssml = getSsml(text, voiceName, rate, pitch, style);

  const response = await fetch(url, {
    method: "POST",
    headers: {
      "Authorization": endpoint.t,
      "Content-Type": "application/ssml+xml",
      "User-Agent": "okhttp/4.5.0",
      "X-Microsoft-OutputFormat": outputFormat
    },
    body: ssml
  });

  if (!response.ok) {
    const errorText = await response.text();
    throw new Error(`Edge TTS API 错误: ${response.status} ${response.statusText} - ${errorText}`);
  }

  return response.blob();
}


// =================================================================================
// 稳定的身份验证与辅助函数
// =================================================================================

// Token 缓存信息
let tokenInfo = { endpoint: null, token: null, expiredAt: null };
const TOKEN_REFRESH_BEFORE_EXPIRY = 5 * 60; // 提前 5 分钟刷新 Token

/**
 * 获取 Microsoft TTS 服务端点和 Token
 * @returns {Promise<Object>} 端点信息对象
 */
async function getEndpoint() {
  const now = Date.now() / 1000;

  // 检查 Token 是否仍然有效
  if (tokenInfo.token && tokenInfo.expiredAt &&
    now < tokenInfo.expiredAt - TOKEN_REFRESH_BEFORE_EXPIRY) {
    return tokenInfo.endpoint;
  }

  const endpointUrl = "https://dev.microsofttranslator.com/apps/endpoint?api-version=1.0";
  const clientId = crypto.randomUUID().replace(/-/g, "");

  try {
    const response = await fetch(endpointUrl, {
      method: "POST",
      headers: {
        "Accept-Language": "zh-Hans",
        "X-ClientVersion": "4.0.530a 5fe1dc6c",
        "X-UserId": "0f04d16a175c411e",
        "X-HomeGeographicRegion": "zh-Hans-CN",
        "X-ClientTraceId": clientId,
        "X-MT-Signature": await sign(endpointUrl),
        "User-Agent": "okhttp/4.5.0",
        "Content-Type": "application/json; charset=utf-8",
        "Content-Length": "0",
        "Accept-Encoding": "gzip"
      }
    });

    if (!response.ok) {
      throw new Error(`获取端点失败: ${response.status}`);
    }

    const data = await response.json();

    // 解析 JWT Token 获取过期时间
    const jwt = data.t.split(".")[1];
    const decodedJwt = JSON.parse(atob(jwt));

    // 更新 Token 缓存
    tokenInfo = {
      endpoint: data,
      token: data.t,
      expiredAt: decodedJwt.exp
    };

    console.log(`成功获取新 Token，有效期 ${((decodedJwt.exp - now) / 60).toFixed(1)} 分钟`);
    return data;
  } catch (error) {
    console.error("获取端点失败:", error);

    // 如果有缓存的 Token，使用过期的 Token 作为备用
    if (tokenInfo.token) {
      console.log("使用过期的缓存 Token 作为备用");
      return tokenInfo.endpoint;
    }

    throw error;
  }
}

/**
 * 生成 Microsoft Translator 签名
 * @param {string} urlStr - 要签名的 URL
 * @returns {Promise<string>} 签名字符串
 */
async function sign(urlStr) {
  const url = urlStr.split("://")[1];
  const encodedUrl = encodeURIComponent(url);
  const uuidStr = crypto.randomUUID().replace(/-/g, "");
  const formattedDate = (new Date()).toUTCString().replace(/GMT/, "").trim() + " GMT";

  // 构建待签名字符串
  const bytesToSign = `MSTranslatorAndroidApp${encodedUrl}${formattedDate}${uuidStr}`.toLowerCase();

  // 解码密钥并生成 HMAC 签名
  const decode = await base64ToBytes("oik6PdDdMnOXemTbwvMn9de/h9lFnfBaCWbGMMZqqoSaQaqUOqjVGm5NqsmjcBI1x+sS9ugjB55HEJWRiFXYFw==");
  const signData = await hmacSha256(decode, bytesToSign);
  const signBase64 = await bytesToBase64(signData);

  return `MSTranslatorAndroidApp::${signBase64}::${formattedDate}::${uuidStr}`;
}

/**
 * HMAC-SHA256 签名
 * @param {Uint8Array} key - 密钥
 * @param {string} data - 待签名数据
 * @returns {Promise<Uint8Array>} 签名结果
 */
async function hmacSha256(key, data) {
  const cryptoKey = await crypto.subtle.importKey(
    "raw",
    key,
    { name: "HMAC", hash: { name: "SHA-256" } },
    false,
    ["sign"]
  );
  const signature = await crypto.subtle.sign("HMAC", cryptoKey, new TextEncoder().encode(data));
  return new Uint8Array(signature);
}

/**
 * Base64 字符串转字节数组
 * @param {string} base64 - Base64 字符串
 * @returns {Promise<Uint8Array>} 字节数组
 */
async function base64ToBytes(base64) {
  const binaryString = atob(base64);
  const bytes = new Uint8Array(binaryString.length);
  for (let i = 0; i < binaryString.length; i++) {
    bytes[i] = binaryString.charCodeAt(i);
  }
  return bytes;
}

/**
 * 字节数组转 Base64 字符串
 * @param {Uint8Array} bytes - 字节数组
 * @returns {Promise<string>} Base64 字符串
 */
async function bytesToBase64(bytes) {
  return btoa(String.fromCharCode.apply(null, bytes));
}


// =================================================================================
// 通用工具函数
// =================================================================================

/**
 * 生成 SSML (Speech Synthesis Markup Language) 文档
 * @param {string} text - 文本内容
 * @param {string} voiceName - 语音名称
 * @param {string} rate - 语速百分比
 * @param {string} pitch - 音调百分比
 * @param {string} style - 语音风格
 * @returns {string} SSML 文档
 */
function getSsml(text, voiceName, rate, pitch, style) {
  // 先保护 break 标签
  const breakTagRegex = /<break\s+time="[^"]*"\s*\/?>|<break\s*\/?>|<break\s+time='[^']*'\s*\/?>/gi;
  const breakTags = [];
  let processedText = text.replace(breakTagRegex, (match) => {
    const placeholder = `__BREAK_TAG_${breakTags.length}__`;
    breakTags.push(match);
    return placeholder;
  });

  // 转义其他 XML 特殊字符
  const sanitizedText = processedText
    .replace(/&/g, '&amp;')
    .replace(/</g, '&lt;')
    .replace(/>/g, '&gt;');

  // 恢复 break 标签
  let finalText = sanitizedText;
  breakTags.forEach((tag, index) => {
    finalText = finalText.replace(`__BREAK_TAG_${index}__`, tag);
  });

  return `<speak xmlns="http://www.w3.org/2001/10/synthesis" xmlns:mstts="http://www.w3.org/2001/mstts" version="1.0" xml:lang="en-US">
    <voice name="${voiceName}">
      <mstts:express-as style="${style}">
        <prosody rate="${rate}%" pitch="${pitch}%">${finalText}</prosody>
      </mstts:express-as>
    </voice>
  </speak>`;
}

/**
 * 智能文本分块 - 按句子边界分割文本
 * @param {string} text - 输入文本
 * @param {number} maxChunkLength - 最大分块长度
 * @returns {string[]} 文本块数组
 */
function smartChunkText(text, maxChunkLength) {
  if (!text) return [];

  const chunks = [];
  let currentChunk = "";

  // 按句子分隔符分割（支持中英文标点）
  const sentences = text.split(/([.?!,;:\n。？！，；：\r]+)/g);

  for (const part of sentences) {
    // 如果当前块加上新部分不超过限制，则添加
    if (currentChunk.length + part.length <= maxChunkLength) {
      currentChunk += part;
    } else {
      // 保存当前块并开始新块
      if (currentChunk.trim()) {
        chunks.push(currentChunk.trim());
      }
      currentChunk = part;
    }
  }

  // 添加最后一个块
  if (currentChunk.trim()) {
    chunks.push(currentChunk.trim());
  }

  // 如果没有分块成功且文本不为空，强制按长度分割
  if (chunks.length === 0 && text.length > 0) {
    for (let i = 0; i < text.length; i += maxChunkLength) {
      chunks.push(text.substring(i, i + maxChunkLength));
    }
  }

  return chunks.filter(chunk => chunk.length > 0);
}

/**
 * 多阶段文本清理函数
 * @param {string} text - 输入文本
 * @param {Object} options - 清理选项
 * @returns {string} 清理后的文本
 */
function cleanText(text, options) {
  let cleanedText = text;

  // 阶段 1: 结构化内容移除
  if (options.remove_urls) {
    cleanedText = cleanedText.replace(/(https?:\/\/[^\s]+)/g, '');
  }

  if (options.remove_markdown) {
    // 移除图片链接
    cleanedText = cleanedText.replace(/!\[.*?\]\(.*?\)/g, '');
    // 移除普通链接，保留链接文本
    cleanedText = cleanedText.replace(/\[(.*?)\]\(.*?\)/g, '$1');
    // 移除粗体和斜体
    cleanedText = cleanedText.replace(/(\*\*|__)(.*?)\1/g, '$2');
    cleanedText = cleanedText.replace(/(\*|_)(.*?)\1/g, '$2');
    // 移除代码块
    cleanedText = cleanedText.replace(/`{1,3}(.*?)`{1,3}/g, '$1');
    // 移除标题标记
    cleanedText = cleanedText.replace(/#{1,6}\s/g, '');
  }

  // 阶段 2: 自定义内容移除
  if (options.custom_keywords) {
    const keywords = options.custom_keywords
      .split(',')
      .map(k => k.trim())
      .filter(k => k);

    if (keywords.length > 0) {
      // 转义正则表达式特殊字符
      const escapedKeywords = keywords.map(k =>
        k.replace(/[-\/\\^$*+?.()|[\]{}]/g, '\\$&')
      );
      const regex = new RegExp(escapedKeywords.join('|'), 'g');
      cleanedText = cleanedText.replace(regex, '');
    }
  }

  // 阶段 3: 字符移除
  if (options.remove_emoji) {
    // 移除 Emoji 表情符号
    cleanedText = cleanedText.replace(/\p{Emoji_Presentation}/gu, '');
  }

  // 阶段 4: 上下文感知格式清理
  if (options.remove_citation_numbers) {
    // 移除引用数字（如文末的 [1], [2] 等）
    cleanedText = cleanedText.replace(/\s\d{1,2}(?=[.。，,;；:：]|$)/g, '');
  }

  // 阶段 5: 通用格式清理
  if (options.remove_line_breaks) {
    // 移除所有多余的空白字符
    cleanedText = cleanedText.replace(/\s+/g, ' ');
  }

  // 阶段 6: 最终清理
  return cleanedText.trim();
}

/**
 * 生成错误响应
 * @param {string} message - 错误消息
 * @param {number} status - HTTP 状态码
 * @param {string} code - 错误代码
 * @param {string} type - 错误类型
 * @returns {Response} 错误响应对象
 */
function errorResponse(message, status, code, type = "api_error") {
  return new Response(
    JSON.stringify({
      error: { message, type, param: null, code }
    }),
    {
      status,
      headers: { "Content-Type": "application/json", ...makeCORSHeaders() }
    }
  );
}

/**
 * 生成 CORS 响应头
 * @param {string} extraHeaders - 额外的允许头部
 * @returns {Object} CORS 头部对象
 */
function makeCORSHeaders(extraHeaders = "Content-Type, Authorization") {
  return {
    "Access-Control-Allow-Origin": "*",
    "Access-Control-Allow-Methods": "GET, POST, OPTIONS",
    "Access-Control-Allow-Headers": extraHeaders,
    "Access-Control-Max-Age": "86400"
  };
}

/**
 * 获取 HTML 内容
 * @returns {string} HTML 页面内容
 */
function getHtmlContent() {
  return `
<!DOCTYPE html>
<html lang="zh-Hans">

<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0, user-scalable=no" />
  <title>🎙️ TTS 服务终极测试页面 (v3.0 - Vue3重构版)</title>
  <style>
    :root {
      --primary-color: #4f46e5;
      --success-color: #22c55e;
      --error-color: #ef4444;
      --warning-color: #f59e0b;
      --light-gray: #f8fafc;
      --gray: #64748b;
      --border-color: #e2e8f0;
      --text-color: #1e293b;
      --mint-start: #f0fdfa;
      --mint-middle: #e6fffa;
      --mint-end: #fdf2f8;
      --mint-accent: #6ee7b7;
    }

    * {
      box-sizing: border-box;
    }

    html {
      padding: 0;
      margin: 0;
    }

    body {
      font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto,
        "Helvetica Neue", Arial, sans-serif;
      background: linear-gradient(135deg, var(--mint-start) 0%, var(--mint-middle) 50%, var(--mint-end) 100%);
      min-height: 100vh;
      color: var(--text-color);
      line-height: 1.6;
      padding: 0;
      margin: 0;
    }

    [v-cloak] {
      display: none;
    }

    .app-container {
      display: flex;
      justify-content: center;
      align-items: flex-start;
      min-height: 100vh;
      padding: 2rem 0 2rem;
    }

    .container {
      max-width: 800px;
      width: 100%;
      background-color: rgba(255, 255, 255, 0.95);
      backdrop-filter: blur(10px);
      padding: 2rem;
      border-radius: 16px;
      box-shadow: 0 20px 40px rgba(0, 0, 0, 0.1), 0 8px 16px rgba(0, 0, 0, 0.06);
      border: 1px solid rgba(255, 255, 255, 0.2);
    }

    h1 {
      text-align: center;
      color: var(--text-color);
      margin-bottom: 2rem;
      font-weight: 700;
      font-size: 1.8rem;
      background: linear-gradient(135deg, var(--primary-color), var(--mint-accent));
      -webkit-background-clip: text;
      -webkit-text-fill-color: transparent;
      background-clip: text;
    }

    .form-group {
      margin-bottom: 1.5rem;
    }

    label {
      display: block;
      font-weight: 600;
      margin-bottom: 0.5rem;
      color: var(--text-color);
    }

    input[type="text"],
    input[type="password"],
    select,
    textarea {
      width: 100%;
      padding: 0.8rem 1rem;
      border: 1px solid var(--border-color);
      border-radius: 8px;
      font-size: 1rem;
      background-color: white;
      transition: border-color 0.2s, box-shadow 0.2s;
    }

    input[type="text"]:focus,
    input[type="password"]:focus,
    select:focus,
    textarea:focus {
      outline: none;
      border-color: var(--primary-color);
      box-shadow: 0 0 0 3px rgba(79, 70, 229, 0.15);
    }

    textarea {
      resize: vertical;
      min-height: 120px;
    }

    .textarea-footer {
      display: flex;
      justify-content: space-between;
      align-items: center;
      font-size: 0.85rem;
      color: var(--gray);
      margin-top: 0.5rem;
    }

    .clear-btn {
      background: none;
      border: none;
      color: var(--primary-color);
      cursor: pointer;
      padding: 0.2rem 0.5rem;
      border-radius: 4px;
      transition: background-color 0.2s;
    }

    .clear-btn:hover {
      background-color: rgba(79, 70, 229, 0.1);
    }

    .label-with-controls {
      display: flex;
      justify-content: space-between;
      align-items: center;
      margin-bottom: 0.5rem;
    }

    .pause-controls {
      display: flex;
      align-items: center;
      gap: 0.5rem;
    }

    .pause-input {
      width: 6.5em;
      padding: 0.4rem 0.6rem;
      border: 1px solid var(--border-color);
      border-radius: 6px;
      font-size: 0.9rem;
      text-align: center;
    }

    .pause-input:focus {
      outline: none;
      border-color: var(--primary-color);
      box-shadow: 0 0 0 2px rgba(79, 70, 229, 0.15);
    }

    .btn-insert-pause {
      background: linear-gradient(135deg, var(--mint-accent), #10b981);
      color: white;
      padding: 0.4rem 0.8rem;
      border: none;
      border-radius: 6px;
      font-size: 0.85rem;
      font-weight: 500;
      cursor: pointer;
      transition: all 0.2s;
      white-space: nowrap;
    }

    .btn-insert-pause:hover {
      transform: translateY(-1px);
      box-shadow: 0 4px 8px rgba(16, 185, 129, 0.3);
    }

    .btn-insert-pause:active {
      transform: scale(0.95);
    }

    .grid-layout {
      display: grid;
      grid-template-columns: repeat(auto-fit, minmax(200px, 1fr));
      gap: 1.5rem;
    }

    .slider-group {
      display: flex;
      align-items: center;
      gap: 1rem;
    }

    .slider-group input[type="range"] {
      flex-grow: 1;
      height: 6px;
      border-radius: 3px;
      background: var(--border-color);
      outline: none;
      -webkit-appearance: none;
      appearance: none;
    }

    .slider-group input[type="range"]::-webkit-slider-thumb {
      -webkit-appearance: none;
      appearance: none;
      width: 20px;
      height: 20px;
      border-radius: 50%;
      background: var(--primary-color);
      cursor: pointer;
    }

    .slider-group input[type="range"]::-moz-range-thumb {
      width: 20px;
      height: 20px;
      border-radius: 50%;
      background: var(--primary-color);
      cursor: pointer;
      border: none;
    }

    .slider-group span {
      font-weight: 500;
      min-width: 50px;
      text-align: right;
      color: var(--primary-color);
      font-size: 0.9rem;
    }

    .button-group {
      display: grid;
      grid-template-columns: 1fr 1fr;
      gap: 1rem;
      margin-top: 2rem;
    }

    button {
      padding: 0.9rem 1rem;
      border: none;
      border-radius: 8px;
      font-size: 1rem;
      font-weight: 600;
      cursor: pointer;
      transition: all 0.2s;
      position: relative;
      overflow: hidden;
    }

    button:active {
      transform: scale(0.97);
    }

    .btn-generate {
      background: linear-gradient(135deg, var(--gray), #475569);
      color: white;
    }

    .btn-stream {
      background: linear-gradient(135deg, var(--success-color), #16a34a);
      color: white;
    }

    button:hover {
      transform: translateY(-2px);
      box-shadow: 0 8px 16px rgba(0, 0, 0, 0.15);
    }

    button:disabled {
      opacity: 0.6;
      cursor: not-allowed;
      transform: none;
      box-shadow: none;
    }

    .status {
      margin-top: 1.5rem;
      padding: 1rem;
      border-radius: 8px;
      text-align: center;
      font-weight: 500;
      display: none;
    }

    .status.show {
      display: block;
    }

    .status-info {
      background-color: #dbeafe;
      color: #1d4ed8;
      border: 1px solid #93c5fd;
    }

    .status-success {
      background-color: #dcfce7;
      color: #166534;
      border: 1px solid #86efac;
    }

    .status-error {
      background-color: #fee2e2;
      color: #dc2626;
      border: 1px solid #fca5a5;
    }

    audio {
      width: 100%;
      margin-top: 1.5rem;
      border-radius: 8px;
    }

    .download-section {
      margin-top: 1rem;
      text-align: center;
    }

    .btn-download {
      background: linear-gradient(135deg, var(--warning-color), #d97706);
      color: white;
      padding: 0.8rem 1.5rem;
      border: none;
      border-radius: 8px;
      font-size: 0.9rem;
      font-weight: 600;
      cursor: pointer;
      transition: all 0.2s;
      display: inline-flex;
      align-items: center;
      gap: 0.5rem;
    }

    .btn-download:hover {
      transform: translateY(-2px);
      box-shadow: 0 6px 12px rgba(245, 158, 11, 0.3);
    }

    .btn-download:active {
      transform: scale(0.97);
    }

    details {
      border: 1px solid var(--border-color);
      border-radius: 8px;
      padding: 1rem;
      margin-bottom: 1.5rem;
      background-color: rgba(248, 250, 252, 0.8);
    }

    summary {
      font-weight: 600;
      cursor: pointer;
      color: var(--text-color);
      padding: 0.5rem 0;
    }

    summary:hover {
      color: var(--primary-color);
    }

    .checkbox-grid {
      margin-top: 1rem;
      display: grid;
      grid-template-columns: repeat(auto-fit, minmax(180px, 1fr));
      gap: 0.8rem;
    }

    .checkbox-item {
      display: flex;
      align-items: center;
      gap: 0.5rem;
    }

    .checkbox-item input[type="checkbox"] {
      width: auto;
      margin: 0;
    }

    /* 移动端适配 */
    @media (max-width: 768px) {
      body {
        padding: 0.5rem;
      }

      .app-container {
        padding: 0 0 1rem;
      }

      .container {
        padding: 1.5rem;
        border-radius: 12px;
      }

      h1 {
        font-size: 1.5rem;
        margin-bottom: 1.5rem;
      }

      .grid-layout {
        grid-template-columns: 1fr;
        gap: 1rem;
      }

      .button-group {
        grid-template-columns: 1fr;
      }

      .checkbox-grid {
        grid-template-columns: 1fr;
      }

      .slider-group span {
        min-width: 45px;
        font-size: 0.85rem;
      }

      textarea {
        min-height: 100px;
      }

      .label-with-controls {
        flex-direction: column;
        align-items: flex-start;
        gap: 0.5rem;
      }

      .pause-controls {
        align-self: flex-end;
      }

      .pause-input {
        font-size: 0.85rem;
      }

      .btn-insert-pause {
        font-size: 0.8rem;
        padding: 0.35rem 0.7rem;
      }
    }

    @media (max-width: 480px) {
      .container {
        padding: 1rem;
        margin: 0.5rem;
      }

      .form-group {
        margin-bottom: 1rem;
      }

      input[type="text"],
      input[type="password"],
      select,
      textarea {
        padding: 0.7rem;
        font-size: 16px;
        /* 防止iOS缩放 */
      }

      .slider-group {
        gap: 0.5rem;
      }
    }

    /* 加载动画 */
    .loading {
      display: inline-block;
      width: 20px;
      height: 20px;
      border: 3px solid rgba(255, 255, 255, 0.3);
      border-radius: 50%;
      border-top-color: white;
      animation: spin 1s ease-in-out infinite;
    }

    @keyframes spin {
      to {
        transform: rotate(360deg);
      }
    }
  </style>
</head>

<body>
  <div id="app" class="app-container">
    <main class="container">
      <h1 v-cloak>{{ title }}</h1>

      <details>
        <summary>API 配置</summary>
        <div class="form-group" style="margin-top: 1rem">
          <label for="baseUrl">API Base URL</label>
          <input type="text" id="baseUrl" v-model="config.baseUrl" @input="saveConfig" placeholder="https://你的域名" />
        </div>
        <div class="form-group" style="margin-bottom: 0">
          <label for="apiKey">API Key</label>
          <input type="password" id="apiKey" v-model="config.apiKey" @input="saveConfig" placeholder="你的密钥" />
        </div>
      </details>

      <div class="form-group">
        <div class="label-with-controls">
          <label for="inputText">输入文本</label>
          <div class="pause-controls">
            <input type="number" v-model.number="pauseTime" min="0.01" max="100" step="0.01" placeholder="停顿时长"
              class="pause-input" />
            <button type="button" @click="insertPause" class="btn-insert-pause" title="在光标位置插入停顿">
              插入停顿
            </button>
          </div>
        </div>
        <textarea id="inputText" ref="textareaRef" v-model="form.inputText" @input="saveForm"
          placeholder="请在这里输入文本，目前尽可能不要超过1点5万字每次，不然会报错。音色映射可以自行修改workers的配置"></textarea>
        <div class="textarea-footer">
          <span v-cloak>{{ charCount }} 字符</span>
          <button class="clear-btn" @click="clearText">清除</button>
        </div>
      </div>

      <div class="grid-layout">
        <div class="form-group">
          <label for="voice">选择音色 (Model)</label>
          <select id="voice" v-model="form.voice" @change="saveForm">
            <option value="zh-CN-XiaoxiaoNeural">中文女声 (晓晓)</option>
            <option value="zh-CN-YunxiNeural">中文男声 (云希)</option>
            <option value="zh-CN-YunyangNeural">中文男声 (云扬)</option>
            <option value="zh-CN-XiaoyiNeural">中文女声 (晓伊)</option>
            <option value="zh-CN-YunjianNeural">中文男声 (云健)</option>
            <option value="zh-CN-XiaochenNeural">中文女声 (晓辰)</option>
            <option value="zh-CN-XiaohanNeural">中文女声 (晓涵)</option>
            <option value="zh-CN-XiaomengNeural">中文女声 (晓梦)</option>
            <option value="zh-CN-XiaomoNeural">中文女声 (晓墨)</option>
            <option value="zh-CN-XiaoqiuNeural">中文女声 (晓秋)</option>
            <option value="zh-CN-XiaoruiNeural">中文女声 (晓睿)</option>
            <option value="zh-CN-XiaoshuangNeural">中文女声 (晓双)</option>
            <option value="zh-CN-XiaoxuanNeural">中文女声 (晓萱)</option>
            <option value="zh-CN-XiaoyanNeural">中文女声 (晓颜)</option>
            <option value="zh-CN-XiaoyouNeural">中文女声 (晓悠)</option>
            <option value="zh-CN-XiaozhenNeural">中文女声 (晓甄)</option>
            <option value="zh-CN-YunfengNeural">中文男声 (云枫)</option>
            <option value="zh-CN-YunhaoNeural">中文男声 (云皓)</option>
            <option value="zh-CN-YunxiaNeural">中文男声 (云夏)</option>
            <option value="zh-CN-YunyeNeural">中文男声 (云野)</option>
            <option value="zh-CN-YunzeNeural">中文男声 (云泽)</option>
            <option value="en-US-JennyNeural">英文女声 (Jenny)</option>
            <option value="en-US-GuyNeural">英文男声 (Guy)</option>
            <option value="en-US-AriaNeural">英文女声 (Aria)</option>
            <option value="en-US-DavisNeural">英文男声 (Davis)</option>
            <option value="en-US-AmberNeural">英文女声 (Amber)</option>
            <option value="en-US-AnaNeural">英文女声 (Ana)</option>
            <option value="en-US-AshleyNeural">英文女声 (Ashley)</option>
            <option value="en-US-BrandonNeural">英文男声 (Brandon)</option>
            <option value="en-US-ChristopherNeural">英文男声 (Christopher)</option>
            <option value="en-US-CoraNeural">英文女声 (Cora)</option>
            <option value="en-US-ElizabethNeural">英文女声 (Elizabeth)</option>
            <option value="en-US-EricNeural">英文男声 (Eric)</option>
            <option value="en-US-JacobNeural">英文男声 (Jacob)</option>
            <option value="en-US-JaneNeural">英文女声 (Jane)</option>
            <option value="en-US-JasonNeural">英文男声 (Jason)</option>
            <option value="en-US-MichelleNeural">英文女声 (Michelle)</option>
            <option value="en-US-MonicaNeural">英文女声 (Monica)</option>
            <option value="en-US-NancyNeural">英文女声 (Nancy)</option>
            <option value="en-US-RogerNeural">英文男声 (Roger)</option>
            <option value="en-US-SaraNeural">英文女声 (Sara)</option>
            <option value="en-US-SteffanNeural">英文男声 (Steffan)</option>
            <option value="en-US-TonyNeural">英文男声 (Tony)</option>
          </select>
        </div>
        <div class="form-group">
          <label>语速</label>
          <div class="slider-group">
            <input type="range" v-model.number="form.speed" @input="saveForm" min="0.25" max="2.0" step="0.05" />
            <span v-cloak>{{ speedDisplay }}</span>
          </div>
        </div>
        <div class="form-group">
          <label>音调</label>
          <div class="slider-group">
            <input type="range" v-model.number="form.pitch" @input="saveForm" min="0.5" max="1.5" step="0.05" />
            <span v-cloak>{{ pitchDisplay }}</span>
          </div>
        </div>
      </div>

      <details>
        <summary>高级文本清理选项</summary>
        <div class="checkbox-grid">
          <label class="checkbox-item">
            <input type="checkbox" v-model="form.cleaning.removeMarkdown" @change="saveForm" />
            移除 Markdown
          </label>
          <label class="checkbox-item">
            <input type="checkbox" v-model="form.cleaning.removeEmoji" @change="saveForm" />
            移除 Emoji
          </label>
          <label class="checkbox-item">
            <input type="checkbox" v-model="form.cleaning.removeUrls" @change="saveForm" />
            移除 URL
          </label>
          <label class="checkbox-item">
            <input type="checkbox" v-model="form.cleaning.removeLineBreaks" @change="saveForm" />
            移除所有空白/换行
          </label>
          <label class="checkbox-item">
            <input type="checkbox" v-model="form.cleaning.removeCitation" @change="saveForm" />
            移除引用标记数字
          </label>
        </div>
        <div class="form-group" style="margin-top: 1rem; margin-bottom: 0">
          <label for="customKeywords">自定义移除关键词 (逗号分隔)</label>
          <input type="text" id="customKeywords" v-model="form.cleaning.customKeywords" @input="saveForm"
            placeholder="例如: ABC,XYZ" />
        </div>
      </details>

      <div class="button-group">
        <button class="btn-generate" v-cloak :disabled="isLoading" @click="generateSpeech(false)">
          <span v-if="isLoading && !isStreaming" class="loading"></span>
          {{ isLoading && !isStreaming ? '生成中...' : '生成语音 (标准)' }}
        </button>
        <button class="btn-stream" v-cloak :disabled="isLoading" @click="generateSpeech(true)">
          <span v-if="isLoading && isStreaming" class="loading"></span>
          {{ isLoading && isStreaming ? '流式播放中...' : '生成语音 (流式)' }}
        </button>
      </div>

      <div class="status" :class="['status-' + status.type, { show: status.show }]" v-cloak>
        {{ status.message }}
      </div>

      <audio ref="audioPlayer" controls v-show="audioSrc" v-cloak :src="audioSrc" @loadstart="onAudioLoadStart"
        @canplay="onAudioCanPlay"></audio>

      <!-- 下载按钮 -->
      <div v-if="showDownloadBtn" class="download-section" v-cloak>
        <button class="btn-download" @click="downloadAudio">
          <span>📥</span> 下载音频文件
        </button>
      </div>
    </main>
  </div>

  <!-- Vue 3 CDN -->
  <script src="https://unpkg.com/vue@3/dist/vue.global.js"></script>

  <script>
    const { createApp } = Vue;

    createApp({
      data() {
        return {
          title: 'TTS 服务终极测试页面 (v3.0 - Vue3重构版)',
          isLoading: false,
          isStreaming: false,
          audioSrc: '',
          downloadUrl: '', // 添加下载链接
          showDownloadBtn: false, // 控制下载按钮显示
          pauseTime: 1.0, // 停顿时间
          config: {
            baseUrl: 'https://你的域名',
            apiKey: '你的密钥'
          },
          form: {
            inputText: '请在这里输入文本，目前尽可能不要超过1点5万字每次，不然会报错。音色映射可以自行修改workers的配置',
            voice: 'zh-CN-XiaoxiaoNeural',
            speed: 1.0,
            pitch: 1.0,
            cleaning: {
              removeMarkdown: true,
              removeEmoji: true,
              removeUrls: true,
              removeLineBreaks: true,
              removeCitation: true,
              customKeywords: ''
            }
          },
          status: {
            show: false,
            message: '',
            type: 'info'
          }
        }
      },
      computed: {
        charCount() {
          return this.form.inputText.length;
        },
        speedDisplay() {
          return this.form.speed.toFixed(2);
        },
        pitchDisplay() {
          return this.form.pitch.toFixed(2);
        }
      },
      methods: {
        loadConfig() {
          try {
            const saved = localStorage.getItem('tts_config');
            if (saved) {
              this.config = { ...this.config, ...JSON.parse(saved) };
              if (this.config.baseUrl.endsWith('/')) {
                this.config.baseUrl = this.config.baseUrl.slice(0, -1); // 去除末尾的斜杠
              }
            }
          } catch (e) {
            console.warn('Failed to load config from localStorage:', e);
          }
        },
        saveConfig() {
          try {
            localStorage.setItem('tts_config', JSON.stringify(this.config));
          } catch (e) {
            console.warn('Failed to save config to localStorage:', e);
          }
        },
        loadForm() {
          try {
            const saved = localStorage.getItem('tts_form');
            if (saved) {
              this.form = { ...this.form, ...JSON.parse(saved) };
            }
          } catch (e) {
            console.warn('Failed to load form from localStorage:', e);
          }
        },
        saveForm() {
          try {
            localStorage.setItem('tts_form', JSON.stringify(this.form));
          } catch (e) {
            console.warn('Failed to save form to localStorage:', e);
          }
        },
        clearText() {
          this.form.inputText = '';
          this.saveForm();
        },
        downloadAudio() {
          if (this.downloadUrl) {
            const link = document.createElement('a');
            link.href = this.downloadUrl;
            let timeString = (new Date().toLocaleString() + '-').replace(/[\/\:]/g, '-').replace(/\s/g, '_').replace(/[-_](\d)[-_]/g, '-0$1-').slice(0, -1);
            link.download = 'tts-audio-' + timeString + '.mp3';
            document.body.appendChild(link);
            link.click();
            document.body.removeChild(link);
          }
        },
        updateStatus(message, type = 'info') {
          this.status = {
            show: true,
            message,
            type
          };
        },
        hideStatus() {
          this.status.show = false;
        },
        getRequestBody() {
          return {
            voice: this.form.voice,
            input: this.form.inputText.trim(),
            speed: this.form.speed,
            pitch: this.form.pitch,
            cleaning_options: {
              remove_markdown: this.form.cleaning.removeMarkdown,
              remove_emoji: this.form.cleaning.removeEmoji,
              remove_urls: this.form.cleaning.removeUrls,
              remove_line_breaks: this.form.cleaning.removeLineBreaks,
              remove_citation_numbers: this.form.cleaning.removeCitation,
              custom_keywords: this.form.cleaning.customKeywords,
            },
          };
        },
        async generateSpeech(isStream) {
          const baseUrl = this.config.baseUrl.trim(); // 去除末尾的斜杠
          const apiKey = this.config.apiKey.trim();
          const text = this.form.inputText.trim();

          if (!baseUrl || !apiKey || !text) {
            this.updateStatus('请填写 API 配置和输入文本', 'error');
            return;
          }

          const requestBody = this.getRequestBody();
          requestBody.stream = isStream;

          this.isLoading = true;
          this.isStreaming = isStream;
          this.audioSrc = '';
          this.showDownloadBtn = false; // 重置下载按钮状态
          if (this.downloadUrl) {
            URL.revokeObjectURL(this.downloadUrl); // 清理之前的下载链接
            this.downloadUrl = '';
          }
          this.updateStatus('正在连接服务器...', 'info');

          try {
            if (isStream) {
              await this.playStreamWithMSE(baseUrl, apiKey, requestBody);
            } else {
              await this.playStandard(baseUrl, apiKey, requestBody);
            }
          } catch (error) {
            console.error('Error generating speech:', error);
            this.updateStatus('错误: ' + error.message, 'error');
          } finally {
            this.isLoading = false;
            this.isStreaming = false;
          }
        },
        async playStandard(baseUrl, apiKey, body) {
          const response = await fetch(baseUrl + '/v1/audio/speech', {
            method: 'POST',
            headers: {
              'Authorization': 'Bearer ' + apiKey,
              'Content-Type': 'application/json',
            },
            body: JSON.stringify(body),
          });

          if (!response.ok) {
            const errorData = await response.json();
            throw new Error(
              errorData.error?.message ||
              'HTTP error! status: ' + response.status
            );
          }

          const blob = await response.blob();
          this.audioSrc = URL.createObjectURL(blob);
          this.downloadUrl = this.audioSrc; // 非流式模式直接使用相同的URL
          this.showDownloadBtn = true;
          this.updateStatus('播放中...', 'success');

          // 自动播放
          this.$nextTick(() => {
            this.$refs.audioPlayer.play().catch(e =>
              console.warn('Autoplay was prevented:', e)
            );
          });
        },
        async playStreamWithMSE(baseUrl, apiKey, body) {
          const mediaSource = new MediaSource();
          this.audioSrc = URL.createObjectURL(mediaSource);

          // 用于收集音频数据的数组
          const audioChunks = [];

          return new Promise((resolve, reject) => {
            mediaSource.addEventListener('sourceopen', async () => {
              URL.revokeObjectURL(this.audioSrc);
              const sourceBuffer = mediaSource.addSourceBuffer('audio/mpeg');

              try {
                const response = await fetch(baseUrl + '/v1/audio/speech', {
                  method: 'POST',
                  headers: {
                    'Authorization': 'Bearer ' + apiKey,
                    'Content-Type': 'application/json',
                  },
                  body: JSON.stringify(body),
                });

                if (!response.ok) {
                  const errorData = await response.json();
                  throw new Error(
                    errorData.error?.message ||
                    'HTTP error! status: ' + response.status
                  );
                }

                this.updateStatus('已连接，接收数据中...', 'info');

                // 自动播放
                this.$nextTick(() => {
                  this.$refs.audioPlayer.play().catch(e =>
                    console.warn('Autoplay was prevented:', e)
                  );
                });

                const reader = response.body.getReader();

                const pump = async () => {
                  const { done, value } = await reader.read();

                  if (done) {
                    if (mediaSource.readyState === 'open' && !sourceBuffer.updating) {
                      mediaSource.endOfStream();
                    }

                    // 创建完整的音频文件用于下载
                    const completeAudioBlob = new Blob(audioChunks, { type: 'audio/mpeg' });
                    this.downloadUrl = URL.createObjectURL(completeAudioBlob);
                    this.showDownloadBtn = true;

                    this.updateStatus('播放完毕！可点击下载按钮保存音频', 'success');
                    resolve();
                    return;
                  }

                  // 收集音频数据块
                  audioChunks.push(value.slice()); // 使用slice()创建副本

                  if (sourceBuffer.updating) {
                    await new Promise(resolve =>
                      sourceBuffer.addEventListener('updateend', resolve, { once: true })
                    );
                  }

                  sourceBuffer.appendBuffer(value);
                  this.updateStatus('正在流式播放...', 'success');
                };

                sourceBuffer.addEventListener('updateend', pump);
                await pump();
              } catch (error) {
                console.error('Error in MSE streaming:', error);
                this.updateStatus('错误: ' + error.message, 'error');
                if (mediaSource.readyState === 'open') {
                  try {
                    mediaSource.endOfStream();
                  } catch (e) { }
                }
                reject(error);
              }
            }, { once: true });
          });
        },
        onAudioLoadStart() {
          console.log('Audio loading started');
        },
        onAudioCanPlay() {
          console.log('Audio can play');
        },
        // 插入停顿标签
        insertPause() {
          const textarea = this.$refs.textareaRef;
          if (!textarea) return;
          if (!this.pauseTime || this.pauseTime <= 0 || this.pauseTime > 100) {
            window.alert('停顿时间必须在 0.01 到 100 秒之间', 'error');
            return;
          }

          const start = textarea.selectionStart;
          const end = textarea.selectionEnd;
          const breakTag = '<break time="' + this.pauseTime + 's"/>';

          const newText = this.form.inputText.slice(0, start) +
            breakTag +
            this.form.inputText.slice(end);

          this.form.inputText = newText;

          // 保持光标位置
          this.$nextTick(() => {
            const newPos = start + breakTag.length;
            textarea.focus();
            textarea.setSelectionRange(newPos, newPos);
          });
        }
      },
      mounted() {
        this.loadConfig();
        this.loadForm();
      },
      beforeUnmount() {
        // 清理URL对象，避免内存泄漏
        if (this.audioSrc) {
          URL.revokeObjectURL(this.audioSrc);
        }
        if (this.downloadUrl && this.downloadUrl !== this.audioSrc) {
          URL.revokeObjectURL(this.downloadUrl);
        }
      }
    }).mount('#app');
  </script>
</body>

</html>
  `;
}
