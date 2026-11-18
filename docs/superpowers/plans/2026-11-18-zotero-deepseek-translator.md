# Zotero DeepSeek PDF Translator Implementation Plan

> **For agentic workers:** REQUIRED: Use superpowers:subagent-driven-development (if subagents available) or superpowers:executing-plans to implement this plan. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build and package an independent Zotero 7–10 plugin that translates selected PDF text through the DeepSeek API.

**Architecture:** A thin `bootstrap.js` lifecycle layer loads native JavaScript modules into one Zotero plugin sandbox. Pure prompt and DeepSeek request modules are testable under Node; a Reader adapter injects a DeepSeek section into Zotero's existing text-selection popup.

**Tech Stack:** Zotero 7+ bootstrap extension API, XHTML/Fluent preferences, native JavaScript, Node `node:test`, PowerShell ZIP packaging.

---

## File Map

- `manifest.json`: plugin identity and Zotero compatibility range.
- `bootstrap.js`: plugin lifecycle, Chrome resource registration, sandbox loading.
- `prefs.js`: default preference values.
- `chrome/content/scripts/prompt.js`: source normalization, prompt construction, result cleanup.
- `chrome/content/scripts/deepseek.js`: endpoint construction, request transport, HTTP/error normalization.
- `chrome/content/scripts/selectionPopup.js`: Reader selection-popup DOM and state machine.
- `chrome/content/scripts/index.js`: Zotero API integration, preferences, cache, connection test.
- `chrome/content/preferences.xhtml`: native Zotero preference pane.
- `chrome/content/styles/reader.css`: popup styling.
- `chrome/content/styles/preferences.css`: settings layout.
- `locale/zh-CN/deepseek-pdf-translator-*.ftl`: Chinese strings.
- `locale/en-US/deepseek-pdf-translator-*.ftl`: English strings.
- `test/prompt.test.js`, `test/deepseek.test.js`, `test/setup.test.js`: automated verification.
- `build.ps1`: deterministic XPI packaging.
- `README.md`: installation and usage.

## Chunk 1: Translation Core

### Task 1: Prompt and Result Processing

**Files:**
- Create: `chrome/content/scripts/prompt.js`
- Test: `test/prompt.test.js`

- [ ] **Step 1: Write the failing prompt tests**

```javascript
const test = require("node:test");
const assert = require("node:assert/strict");
const prompt = require("../chrome/content/scripts/prompt.js");

test("normalizes PDF selection whitespace", () => {
  assert.equal(
    prompt.normalizeSourceText("  first\r\n\r\nsecond\u0000  "),
    "first\n\nsecond"
  );
});

test("rejects empty and oversized selections", () => {
  assert.throws(() => prompt.validateSourceText("  "), /没有可翻译/);
  assert.throws(
    () => prompt.validateSourceText("x".repeat(prompt.MAX_SOURCE_LENGTH + 1)),
    /文本过长/
  );
});

test("builds an academic translation prompt", () => {
  const messages = prompt.buildMessages({
    sourceText: "Protein folding is essential.",
    targetLanguage: "简体中文",
  });
  assert.equal(messages.length, 2);
  assert.equal(messages[0].role, "system");
  assert.match(messages[0].content, /简体中文/);
  assert.match(messages[0].content, /只返回译文/);
  assert.deepEqual(messages[1], {
    role: "user",
    content: "Protein folding is essential.",
  });
});

test("supports a custom prompt and target placeholder", () => {
  const messages = prompt.buildMessages({
    sourceText: "sample",
    targetLanguage: "日本語",
    customPrompt: "Translate to ${targetLanguage}.",
  });
  assert.equal(messages[0].content, "Translate to 日本語.");
});

test("extracts content and removes a complete Markdown fence", () => {
  const response = {
    choices: [{ message: { content: "```markdown\n译文 **bold**\n```" } }],
  };
  assert.equal(prompt.extractTranslation(response), "译文 **bold**");
});

test("rejects malformed API responses", () => {
  assert.throws(() => prompt.extractTranslation({}), /响应格式异常/);
});
```

- [ ] **Step 2: Run the prompt tests and observe failure**

Run: `node --test test/prompt.test.js`

Expected: FAIL because `prompt.js` does not exist.

- [ ] **Step 3: Implement the prompt module**

```javascript
(function (root, factory) {
  const api = factory();
  if (typeof module !== "undefined" && module.exports) module.exports = api;
  if (root) {
    root.DeepSeekPDFTranslator = root.DeepSeekPDFTranslator || {};
    root.DeepSeekPDFTranslator.prompt = api;
  }
})(typeof globalThis !== "undefined" ? globalThis : this, function () {
  "use strict";

  const MAX_SOURCE_LENGTH = 20000;
  const DEFAULT_SYSTEM_PROMPT = [
    "你是一名严谨的学术论文翻译专家。",
    "请将用户提供的论文片段完整、准确地翻译为${targetLanguage}。",
    "保持原文的段落结构、术语、数字、单位、基因名、化学式、公式、引用标记和缩写；",
    "不要总结、扩写、解释或添加原文没有的信息。",
    "只返回译文，不要使用代码围栏，也不要添加“译文：”等前后缀。",
  ].join("");

  function normalizeSourceText(value) {
    if (typeof value !== "string") return "";
    return value
      .replace(/\r\n?/g, "\n")
      .replace(/\u0000/g, "")
      .trim();
  }

  function validateSourceText(value) {
    const text = normalizeSourceText(value);
    if (!text) throw new Error("没有可翻译的文本");
    if (text.length > MAX_SOURCE_LENGTH) {
      throw new Error(`文本过长（${text.length} 字符），请分段翻译`);
    }
    return text;
  }

  function renderPrompt(template, targetLanguage) {
    return String(template).replaceAll("${targetLanguage}", targetLanguage);
  }

  function buildMessages({ sourceText, targetLanguage, customPrompt }) {
    const text = validateSourceText(sourceText);
    const target = String(targetLanguage || "简体中文").trim();
    const template = String(customPrompt || "").trim() || DEFAULT_SYSTEM_PROMPT;
    return [
      { role: "system", content: renderPrompt(template, target) },
      { role: "user", content: text },
    ];
  }

  function stripOuterCodeFence(value) {
    const text = String(value || "").trim();
    const match = text.match(
      /^```(?:markdown|md|text)?\s*\n?([\s\S]*?)\n?```$/i
    );
    return match ? match[1].trim() : text;
  }

  function extractTranslation(response) {
    const content = response?.choices?.[0]?.message?.content;
    if (typeof content !== "string" || !content.trim()) {
      throw new Error("DeepSeek 响应格式异常：缺少译文");
    }
    return stripOuterCodeFence(content);
  }

  return {
    DEFAULT_SYSTEM_PROMPT,
    MAX_SOURCE_LENGTH,
    normalizeSourceText,
    validateSourceText,
    buildMessages,
    stripOuterCodeFence,
    extractTranslation,
  };
});
```

- [ ] **Step 4: Run the prompt tests**

Run: `node --test test/prompt.test.js`

Expected: 6 tests pass.

- [ ] **Step 5: Commit**

```powershell
git add chrome/content/scripts/prompt.js test/prompt.test.js
git commit -m "feat: add translation prompt core"
```

### Task 2: DeepSeek Request and Error Handling

**Files:**
- Create: `chrome/content/scripts/deepseek.js`
- Test: `test/deepseek.test.js`

- [ ] **Step 1: Write the failing DeepSeek tests**

```javascript
const test = require("node:test");
const assert = require("node:assert/strict");
const deepseek = require("../chrome/content/scripts/deepseek.js");
const prompt = require("../chrome/content/scripts/prompt.js");

const base = {
  apiKey: "sk-test",
  apiEndpoint: "https://api.deepseek.com",
  model: "deepseek-chat",
  temperature: 0.2,
  timeoutSeconds: 60,
};

test("normalizes endpoint variants", () => {
  assert.equal(
    deepseek.normalizeEndpoint("https://api.deepseek.com/v1"),
    "https://api.deepseek.com/v1/chat/completions"
  );
  assert.equal(
    deepseek.normalizeEndpoint("https://proxy.example/v1/chat/completions/"),
    "https://proxy.example/v1/chat/completions"
  );
});

test("builds an authenticated OpenAI-compatible request", () => {
  const messages = prompt.buildMessages({
    sourceText: "cell",
    targetLanguage: "简体中文",
  });
  const request = deepseek.buildChatRequest({ ...base, messages });
  assert.equal(request.url, "https://api.deepseek.com/chat/completions");
  assert.equal(request.headers.Authorization, "Bearer sk-test");
  assert.equal(JSON.parse(request.body).model, "deepseek-chat");
});

test("returns translated text and usage", async () => {
  let captured;
  const result = await deepseek.requestTranslation({
    config: base,
    messages: [{ role: "user", content: "cell" }],
    transport: async (request) => {
      captured = request;
      return {
        status: 200,
        data: {
          model: "deepseek-chat",
          choices: [{ message: { content: "细胞" } }],
          usage: { total_tokens: 12 },
        },
      };
    },
  });
  assert.equal(captured.url, "https://api.deepseek.com/chat/completions");
  assert.equal(result.text, "细胞");
  assert.equal(result.usage.total_tokens, 12);
});

test("maps status codes to stable error codes", async () => {
  for (const [status, code] of [
    [401, "auth"],
    [402, "quota"],
    [429, "rate_limit"],
    [500, "server"],
  ]) {
    await assert.rejects(
      deepseek.requestTranslation({
        config: base,
        messages: [{ role: "user", content: "x" }],
        transport: async () => ({
          status,
          data: { error: { message: "provider detail" } },
        }),
      }),
      (error) => error.code === code
    );
  }
});

test("maps timeout and cancellation", async () => {
  await assert.rejects(
    deepseek.requestTranslation({
      config: base,
      messages: [{ role: "user", content: "x" }],
      transport: async () => {
        const error = new Error("timed out");
        error.name = "TimeoutError";
        throw error;
      },
    }),
    (error) => error.code === "timeout"
  );

  const controller = new AbortController();
  controller.abort();
  await assert.rejects(
    deepseek.requestTranslation({
      config: base,
      messages: [{ role: "user", content: "x" }],
      signal: controller.signal,
      transport: async () => {
        const error = new Error("cancelled");
        error.name = "AbortError";
        throw error;
      },
    }),
    (error) => error.code === "cancelled"
  );
});
```

- [ ] **Step 2: Run the DeepSeek tests and observe failure**

Run: `node --test test/deepseek.test.js`

Expected: FAIL because `deepseek.js` does not exist.

- [ ] **Step 3: Implement the DeepSeek module**

```javascript
(function (root, factory) {
  const api = factory(root);
  if (typeof module !== "undefined" && module.exports) module.exports = api;
  if (root) {
    root.DeepSeekPDFTranslator = root.DeepSeekPDFTranslator || {};
    root.DeepSeekPDFTranslator.deepseek = api;
  }
})(typeof globalThis !== "undefined" ? globalThis : this, function (root) {
  "use strict";

  class DeepSeekError extends Error {
    constructor(code, message, details = {}) {
      super(message);
      this.name = "DeepSeekError";
      this.code = code;
      this.status = details.status;
      this.providerMessage = details.providerMessage;
    }
  }

  function normalizeEndpoint(value) {
    const raw = String(value || "https://api.deepseek.com").trim();
    let url;
    try {
      url = new URL(raw);
    } catch (_error) {
      throw new DeepSeekError("config", "API 地址格式无效");
    }
    if (!["http:", "https:"].includes(url.protocol)) {
      throw new DeepSeekError("config", "API 地址必须使用 HTTP 或 HTTPS");
    }
    url.hash = "";
    url.search = "";
    let path = url.pathname.replace(/\/+$/, "");
    if (!path.endsWith("/chat/completions")) {
      path += "/chat/completions";
    }
    url.pathname = path;
    return url.toString();
  }

  function buildChatRequest({
    apiKey,
    apiEndpoint,
    model,
    messages,
    temperature,
    maxTokens,
    timeoutSeconds,
  }) {
    const key = String(apiKey || "").trim();
    if (!key) throw new DeepSeekError("missing_key", "请先配置 DeepSeek API Key");
    if (!Array.isArray(messages) || messages.length === 0) {
      throw new DeepSeekError("config", "请求消息为空");
    }
    const payload = {
      model: String(model || "deepseek-chat").trim(),
      messages,
      temperature: Math.min(2, Math.max(0, Number(temperature) || 0)),
      stream: false,
    };
    if (Number.isFinite(Number(maxTokens)) && Number(maxTokens) > 0) {
      payload.max_tokens = Math.floor(Number(maxTokens));
    }
    return {
      url: normalizeEndpoint(apiEndpoint),
      headers: {
        "Content-Type": "application/json",
        Authorization: `Bearer ${key}`,
      },
      body: JSON.stringify(payload),
      timeoutMs: Math.max(5000, Number(timeoutSeconds || 60) * 1000),
    };
  }

  function providerMessage(body) {
    const message = body?.error?.message || body?.message || "";
    return String(message).replace(/\s+/g, " ").slice(0, 500);
  }

  function statusError(status, body) {
    const detail = providerMessage(body);
    if (status === 401 || status === 403) {
      return new DeepSeekError(
        "auth",
        "DeepSeek 鉴权失败，请检查 API Key",
        { status, providerMessage: detail }
      );
    }
    if (status === 402) {
      return new DeepSeekError(
        "quota",
        "DeepSeek 账户余额或额度不足",
        { status, providerMessage: detail }
      );
    }
    if (status === 429) {
      return new DeepSeekError(
        "rate_limit",
        "DeepSeek 请求过于频繁，请稍后重试",
        { status, providerMessage: detail }
      );
    }
    if (status >= 500) {
      return new DeepSeekError(
        "server",
        "DeepSeek 服务暂时不可用",
        { status, providerMessage: detail }
      );
    }
    return new DeepSeekError(
      "http",
      `DeepSeek 请求失败（HTTP ${status || "未知"}）`,
      { status, providerMessage: detail }
    );
  }

  function transportError(error, signal) {
    if (signal?.aborted || error?.name === "AbortError" ||
        /cancel(?:led|ed)/i.test(String(error?.message || ""))) {
      return new DeepSeekError("cancelled", "翻译已取消");
    }
    if (error?.name === "TimeoutError" ||
        /time(?:d)? ?out|timeout/i.test(String(error?.message || ""))) {
      return new DeepSeekError("timeout", "DeepSeek 请求超时");
    }
    return new DeepSeekError("network", "无法连接 DeepSeek API");
  }

  async function requestTranslation({
    config,
    messages,
    transport,
    signal,
    maxTokens,
  }) {
    if (signal?.aborted) {
      throw new DeepSeekError("cancelled", "翻译已取消");
    }
    const request = buildChatRequest({
      ...config,
      messages,
      maxTokens,
    });
    let response;
    try {
      response = await transport({ ...request, signal });
    } catch (error) {
      if (error instanceof DeepSeekError) throw error;
      throw transportError(error, signal);
    }
    if (response.status < 200 || response.status >= 300) {
      throw statusError(response.status, response.data);
    }
    const data = response.data;
    const promptApi = root?.DeepSeekPDFTranslator?.prompt;
    const text = promptApi?.extractTranslation(data);
    if (!text) throw new DeepSeekError("response", "DeepSeek 响应格式异常");
    return { text, model: data.model, usage: data.usage || null };
  }

  function createZoteroTransport(Zotero) {
    return function zoteroTransport(request) {
      return new Promise((resolve, reject) => {
        let cancelRequest = null;
        const abort = () => {
          if (cancelRequest) cancelRequest();
        };
        if (request.signal?.aborted) {
          reject(new DeepSeekError("cancelled", "翻译已取消"));
          return;
        }
        request.signal?.addEventListener("abort", abort, { once: true });
        Zotero.HTTP.request("POST", request.url, {
          headers: request.headers,
          body: request.body,
          responseType: "json",
          timeout: request.timeoutMs,
          successCodes: false,
          errorDelayMax: 0,
          cancellerReceiver(canceller) {
            cancelRequest = canceller;
          },
        }).then((xhr) => {
          resolve({ status: xhr.status, data: xhr.response });
        }).catch((error) => {
          if (error instanceof Zotero.HTTP.CancelledException) {
            error.name = "AbortError";
          } else if (error instanceof Zotero.HTTP.TimeoutException) {
            error.name = "TimeoutError";
          }
          reject(error);
        });
      });
    };
  }

  return {
    DeepSeekError,
    normalizeEndpoint,
    buildChatRequest,
    requestTranslation,
    createZoteroTransport,
  };
});
```

- [ ] **Step 4: Run both core test suites**

Run: `node --test test/prompt.test.js test/deepseek.test.js`

Expected: all tests pass.

- [ ] **Step 5: Commit**

```powershell
git add chrome/content/scripts/deepseek.js test/deepseek.test.js
git commit -m "feat: add DeepSeek API client"
```

## Chunk 2: Zotero Reader Integration

### Task 3: Plugin Skeleton and Selection Popup

**Files:**
- Create: `manifest.json`
- Create: `bootstrap.js`
- Create: `prefs.js`
- Create: `package.json`
- Create: `.gitignore`
- Create: `chrome/content/scripts/selectionPopup.js`
- Create: `chrome/content/styles/reader.css`
- Create: `chrome/content/icons/icon.svg`

- [ ] **Step 1: Add failing setup checks**

Create `test/setup.test.js`:

```javascript
const test = require("node:test");
const assert = require("node:assert/strict");
const fs = require("node:fs");
const path = require("node:path");

const root = path.resolve(__dirname, "..");

test("manifest targets Zotero 7 through 10", () => {
  const manifest = JSON.parse(
    fs.readFileSync(path.join(root, "manifest.json"), "utf8")
  );
  assert.equal(manifest.applications.zotero.strict_min_version, "7.9.9");
  assert.equal(manifest.applications.zotero.strict_max_version, "10.9.9");
});

test("required plugin entry files exist", () => {
  for (const file of ["bootstrap.js", "prefs.js", "manifest.json"]) {
    assert.equal(fs.existsSync(path.join(root, file)), true, file);
  }
});

test("preferences default to manual translation", () => {
  const prefs = fs.readFileSync(path.join(root, "prefs.js"), "utf8");
  assert.match(prefs, /autoTranslate", false/);
  assert.match(prefs, /deepseek-chat/);
});
```

Run: `node --test test/setup.test.js`

Expected: FAIL because the plugin skeleton does not exist.

- [ ] **Step 2: Create the manifest, lifecycle, defaults, and package metadata**

`manifest.json`:

```json
{
  "manifest_version": 2,
  "name": "DeepSeek PDF Translator",
  "version": "1.0.0",
  "description": "Translate selected PDF text with the DeepSeek API.",
  "author": "DeepSeek PDF Translator contributors",
  "icons": {
    "48": "chrome/content/icons/icon.svg",
    "96": "chrome/content/icons/icon.svg"
  },
  "applications": {
    "zotero": {
      "id": "deepseek-pdf-translator@codex.local",
      "strict_min_version": "7.9.9",
      "strict_max_version": "10.9.9"
    }
  }
}
```

`bootstrap.js`:

```javascript
var chromeHandle;
var pluginInstance;

async function startup({ id, version, resourceURI, rootURI }, _reason) {
  await Zotero.initializationPromise;
  rootURI = rootURI || resourceURI.spec;

  const aomStartup = Components.classes[
    "@mozilla.org/addons/addon-manager-startup;1"
  ].getService(Components.interfaces.amIAddonManagerStartup);
  const manifestURI = Services.io.newURI(rootURI + "manifest.json");
  chromeHandle = aomStartup.registerChrome(manifestURI, [
    ["content", "deepseek-pdf-translator", rootURI + "chrome/content/"],
  ]);

  const ctx = { id, version, rootURI };
  ctx._globalThis = ctx;
  for (const file of [
    "prompt.js",
    "deepseek.js",
    "selectionPopup.js",
    "index.js",
  ]) {
    Services.scriptloader.loadSubScript(
      `${rootURI}chrome/content/scripts/${file}`,
      ctx
    );
  }
  pluginInstance = ctx.DeepSeekPDFTranslator;
  await pluginInstance.startup({ id, version, rootURI });
}

function shutdown(_data, reason) {
  if (reason === APP_SHUTDOWN) return;
  try {
    pluginInstance?.shutdown();
  } catch (error) {
    Zotero.logError(error);
  }
  if (chromeHandle) {
    chromeHandle.destruct();
    chromeHandle = null;
  }
  Cc["@mozilla.org/intl/stringbundle;1"]
    .getService(Components.interfaces.nsIStringBundleService)
    .flushBundles();
  pluginInstance = null;
}

function install() {}
function uninstall() {}
function onMainWindowLoad() {}
function onMainWindowUnload() {}
```

`prefs.js`:

```javascript
pref("extensions.zotero.deepseekPDFTranslator.apiEndpoint", "https://api.deepseek.com/chat/completions");
pref("extensions.zotero.deepseekPDFTranslator.apiKey", "");
pref("extensions.zotero.deepseekPDFTranslator.model", "deepseek-chat");
pref("extensions.zotero.deepseekPDFTranslator.targetLanguage", "简体中文");
pref("extensions.zotero.deepseekPDFTranslator.customPrompt", "");
pref("extensions.zotero.deepseekPDFTranslator.autoTranslate", false);
pref("extensions.zotero.deepseekPDFTranslator.timeoutSeconds", "60");
pref("extensions.zotero.deepseekPDFTranslator.temperature", "0.2");
pref("extensions.zotero.deepseekPDFTranslator.maxResultWidth", "520");
pref("extensions.zotero.deepseekPDFTranslator.fontSize", "13");
```

`package.json`:

```json
{
  "name": "zotero-deepseek-pdf-translator",
  "version": "1.0.0",
  "private": true,
  "scripts": {
    "test": "node --test test/*.test.js",
    "check": "node --check bootstrap.js && node --check chrome/content/scripts/index.js && node --check chrome/content/scripts/selectionPopup.js"
  }
}
```

`.gitignore`:

```text
dist/
*.xpi
.DS_Store
Thumbs.db
```

- [ ] **Step 3: Implement the selection popup state machine**

Create `chrome/content/scripts/selectionPopup.js` with one exported function,
`createSelectionPopup(event, options)`. It must:

- Read `event.params.annotation.text`.
- Append one `div#deepseek-pdf-translator-popup-<readerInstanceID>` through
  `event.append`.
- Build controls with `doc.createElement`; never assign translated text to
  `innerHTML`.
- Use `AbortController` and expose `translate`, `cancel`, `retry`, `copy`.
- Render `idle`, `loading`, `success`, and `error` states.
- Start automatically only when `options.autoTranslate === true`.
- Return `{ root, destroy }`.

Use compact DOM helpers:

```javascript
function el(doc, tag, props = {}, children = []) {
  const node = doc.createElement(tag);
  for (const [key, value] of Object.entries(props)) {
    if (key === "className") node.className = value;
    else if (key === "text") node.textContent = value;
    else if (key === "dataset") Object.assign(node.dataset, value);
    else node.setAttribute(key, value);
  }
  for (const child of children) node.append(child);
  return node;
}
```

The main result element must be a read-only textarea. The translation callback
receives `{ sourceText, signal, force }`; the copy callback receives the result
text. All button labels come from `options.strings`.

- [ ] **Step 4: Add popup CSS and icon**

`chrome/content/styles/reader.css` must define
`.deepseek-pdf-translator-section`, `.deepseek-pdf-translator-actions`,
`.deepseek-pdf-translator-result`, and `.deepseek-pdf-translator-status`.
Use Zotero CSS variables such as `--fill-secondary`, `--fill-quaternary`,
`--text-primary`, `--accent-blue`, with concrete fallback colors.

Create a static SVG icon with a document outline and `DS` mark. The icon must
not reference remote resources.

- [ ] **Step 5: Run setup tests and JavaScript syntax checks**

Run: `node --test test/setup.test.js`

Expected: 3 tests pass.

Run: `node --check chrome/content/scripts/selectionPopup.js`

Expected: exit code 0, no syntax errors.

- [ ] **Step 6: Commit**

```powershell
git add manifest.json bootstrap.js prefs.js package.json .gitignore chrome/content test/setup.test.js
git commit -m "feat: add Zotero plugin skeleton and selection popup"
```

### Task 4: Zotero Integration, Cache, and Settings Bridge

**Files:**
- Create: `chrome/content/scripts/index.js`
- Create: `chrome/content/preferences.xhtml`
- Create: `chrome/content/styles/preferences.css`
- Create: `locale/zh-CN/deepseek-pdf-translator-addon.ftl`
- Create: `locale/zh-CN/deepseek-pdf-translator-preferences.ftl`
- Create: `locale/en-US/deepseek-pdf-translator-addon.ftl`
- Create: `locale/en-US/deepseek-pdf-translator-preferences.ftl`

- [ ] **Step 1: Implement `index.js` integration**

The module attaches `globalThis.DeepSeekPDFTranslator` and exposes `startup`,
`shutdown`, and `onPrefsLoad`. It must:

- Wait for `Zotero.initializationPromise`, `unlockPromise`, and
  `uiReadyPromise`.
- Register `renderTextSelectionPopup` with plugin ID.
- Keep the handler function so shutdown can call
  `Zotero.Reader.unregisterEventListener(type, handler)`.
- Build settings from `Zotero.Prefs.get(key, true)`.
- Use `deepseek.createZoteroTransport(Zotero)`.
- Cache successful translations in a `Map` keyed by
  `model + "\n" + targetLanguage + "\n" + sourceText`.
- Limit cache to 100 entries, deleting the oldest key.
- Copy through `Zotero.Utilities.Internal.copyTextToClipboard`.
- Map error codes to localized strings.
- Register the preference pane through `Zotero.PreferencePanes.register`.
- Expose `Zotero.DeepSeekPDFTranslator.onPrefsLoad`.

The Reader handler must use:

```javascript
const text = promptApi.normalizeSourceText(
  event.params?.annotation?.text
);
if (!text) return;
```

The translation function must pass a cache hit directly to the popup and only
force a network request when `force === true`.

- [ ] **Step 2: Implement the connection test**

In `onPrefsLoad(event)`, attach one click listener to
`#deepseek-pdf-translator-test-connection`. On click:

1. Disable the button and show `strings.testing`.
2. Send a minimal request with system message `Reply with exactly: OK` and
   `maxTokens: 4`.
3. Show `strings.testSuccess` on a valid response.
4. Show the localized error message on failure.
5. Restore the button in a `finally` block.

Do not store API keys in `Zotero.debug`, error messages, or DOM data attributes.

- [ ] **Step 3: Build the preference pane**

Use native `preference="extensions.zotero.deepseekPDFTranslator.<key>"`
attributes on password, text, number, and checkbox controls. Include:

- Endpoint, API key, model.
- Connection test button and status label.
- Target language and custom prompt textarea.
- Auto-translate checkbox.
- Timeout, temperature, maximum width, font size.
- A local-only security note.

Root element:

```xml
<vbox
  id="deepseek-pdf-translator-prefpane"
  onload="Zotero.DeepSeekPDFTranslator.onPrefsLoad(event)"
>
```

Include both localization and stylesheet links.

- [ ] **Step 4: Add Chinese and English Fluent strings**

The addon FTL files define localized popup labels, status messages, and errors
with IDs beginning `deepseek-pdf-translator-`. The preferences FTL files define
all `data-l10n-id` values used by `preferences.xhtml`, including the pane title.

- [ ] **Step 5: Verify syntax and startup references**

Run: `node --check chrome/content/scripts/index.js`

Expected: exit code 0.

Run: `node --test test/*.test.js`

Expected: all core and setup tests pass.

- [ ] **Step 6: Commit**

```powershell
git add chrome/content locale
git commit -m "feat: connect DeepSeek translation to Zotero reader"
```

## Chunk 3: Packaging, Documentation, and End-to-End Verification

### Task 5: XPI Build and Documentation

**Files:**
- Create: `build.ps1`
- Create: `README.md`
- Modify: `test/setup.test.js`

- [ ] **Step 1: Add packaging assertions**

Extend `test/setup.test.js` with assertions that:

- `README.md` documents the DeepSeek API endpoint and `deepseek-chat`.
- `build.ps1` includes `manifest.json`, `bootstrap.js`, `prefs.js`,
  `chrome/content`, and `locale`.
- `dist/` and `*.xpi` are ignored by Git.

Run: `node --test test/setup.test.js`

Expected: FAIL until documentation and build script exist.

- [ ] **Step 2: Implement `build.ps1`**

The script must:

- Use `$PSScriptRoot` as the project root.
- Require Node 18 or newer.
- Run `node --test test/*.test.js`.
- Run syntax checks.
- Create a unique staging directory under `dist`.
- Copy only runtime files.
- Verify runtime file presence.
- Create `dist/deepseek-pdf-translator-1.0.0.xpi` with manifest at ZIP root.
- Verify the XPI contains `manifest.json`.
- Remove only the staging directory after successful packaging.
- Print the absolute XPI path and file size.

Use `Compress-Archive -Path (Join-Path $stage "*")` so the XPI has no extra
wrapper directory.

- [ ] **Step 3: Write the README**

Include:

- Feature summary and supported Zotero range.
- Build command `powershell -ExecutionPolicy Bypass -File .\build.ps1`.
- Installation through Zotero Tools → Add-ons → Install Add-on From File.
- DeepSeek key creation and endpoint guidance.
- Default endpoint, model, language, timeout, and privacy note.
- Manual and automatic translation usage.
- Error troubleshooting for 401, 402, 429, timeout, and proxy.
- Development test and temporary-profile smoke-test instructions.
- Uninstall and preference-reset instructions.

- [ ] **Step 4: Run tests and commit**

Run: `node --test test/*.test.js`

Expected: all tests pass.

```powershell
git add build.ps1 README.md test/setup.test.js
git commit -m "docs: add build and installation guide"
```

### Task 6: Build and Smoke-Test the XPI

**Files:**
- Verify: `dist/deepseek-pdf-translator-1.0.0.xpi`

- [ ] **Step 1: Run the build**

Run: `powershell -ExecutionPolicy Bypass -File .\build.ps1`

Expected: tests pass, syntax checks pass, and the script prints the absolute XPI
path.

- [ ] **Step 2: Inspect XPI contents**

Run:

```powershell
$zip = [System.IO.Compression.ZipFile]::OpenRead(
  (Resolve-Path .\dist\deepseek-pdf-translator-1.0.0.xpi)
)
$zip.Entries.FullName
$zip.Dispose()
```

Expected: `manifest.json`, `bootstrap.js`, `prefs.js`, all runtime scripts,
styles, locale files, and the icon are listed. No `test/`, `docs/`, `.git/`, or
local preference files are present.

- [ ] **Step 3: Verify Zotero 10.0.8 installation in an isolated profile**

Create a temporary profile directory and launch Zotero with
`-profile <temporary-path> -no-remote`. Install the XPI manually. Confirm:

- Add-on appears without a manifest compatibility error.
- Preference pane opens.
- Missing-key error appears when translating without a key.
- With a valid user-provided key, selected PDF text returns a translation.
- Cancel stops an in-flight request.
- Automatic translation remains off by default.

Do not copy `prefs.js`, `storage`, or library data into the temporary profile.

- [ ] **Step 4: Record final verification**

Run: `git status --short`

Expected: no unintended generated files. `dist/` stays ignored.

- [ ] **Step 5: Commit any verification-only changes**

If verification required a code fix:

```powershell
git add <fixed-files>
git commit -m "fix: address smoke-test findings"
```

If no code changed, do not create an empty commit.
