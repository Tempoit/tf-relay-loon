# TestFlight Relay — Loon 适配版

把 [ScriptingApp/Community-Scripts](https://github.com/ScriptingApp/Community-Scripts) 的 **TestFlight 中转脚本**（原本是 Surge 模块）适配到 **Loon** 的分发仓库。

- 脚本本体：`TestFlight-Relay.js`（上游 `module/scripts/TestFlight-Relay.js` + 两处平台补丁）
- Loon 插件：`TestFlight-Relay.plugin`

> Surge 用户请直接用上游官方 `.sgmodule`，**不需要**这个仓库。

## 它能做什么

TestFlight 的「多账号 + 中转」方案：插件拦截 TestFlight 发往 `http://tf-relay.test/...` 的请求并代为应答，同时给 Scripting 里的「TestFlight 魔法攻击」脚本提供账号控制接口。

- 给 Scripting 脚本：`/api/v1/store-accounts`、`/api/v1/control/initialize|reset|status`
- 给 TestFlight：`/v1/session/authenticate`、`/v1/appCategories`、`/v4/accounts/{id}/apps` 等

账号上下文（Cookie / AMD / kbSync）**只保存在本机 Loon 的持久化存储**（`tf.accounts` / `tf.apps` / `tf.details` / `tf.tasks`），不会上传到任何服务器，也不会进入本仓库。

## 安装

1. Loon 里**先停用**其他 TestFlight Relay 插件（避免两条规则同时命中）；
2. 导入插件：

   ```
   https://cdn.jsdelivr.net/gh/Tempoit/tf-relay-loon@main/TestFlight-Relay.plugin
   ```

3. 启用插件，确认脚本已下载；
4. 后台划掉 TestFlight，回到脚本重新点「启动 TestFlight」。

需要 Loon **≥ 3.5.1 (983)**（插件用的是新版 Script 语法：`request if ${url} ~= /regex/i then script(...) with ...`）。

## 为什么需要补丁

Loon 的脚本运行时和 Surge/QX 有两处差异，命中后表现为**TestFlight 打开就报「TestFlight 不可用 / 网络连接已断开」**，而账号页、拉取列表都正常。

### 补丁 1：转发分支（主要）

```diff
 const forwardRequest = async req => {
   const request = { url: "https://testflight.apple.com" + req.path + req.url.search, ... }
-  if (!globalThis.$task) return request;
+  if (!globalThis.$task && !globalThis.$httpClient) return request;
   const response = await http[req.method.toLowerCase()]({ ... })
   return { response }
```

Loon 没有 `$task`，原逻辑会把改写后的请求（`http://tf-relay.test/...` → `https://testflight.apple.com/...`）交回代理"续发"。Surge/QX 支持这种跨域 + http→https 续发，Loon 不支持，会直接掐断连接。补丁让 Loon 走下方的 `$httpClient` 代发分支：由脚本自己请求 Apple，再把响应交回。

### 补丁 2：`this.$request` 判定

```diff
 const main = async () => {
+  const hasRequest = typeof $request !== "undefined" && !!$request;
   try {
-    const req = this.$request ? createRequestContext() : null;
+    const req = hasRequest ? createRequestContext() : null;
 ...
-    finish(this.$request ? { response: createErrorResponse(error) } : undefined);
+    finish(hasRequest ? { response: createErrorResponse(error) } : undefined);
```

在模块作用域运行时 `this` 可能是空对象/undefined，脚本会把每个 HTTP 请求误判成"定时任务"，直接 `$done()` 放行，同样表现为连接中断。改为访问全局 `$request`（Surge / Loon / QX 都提供），Surge/QX 行为不变。

脚本里搜 `[Loon 补丁]` 可定位这两处。

## 自查（不上 TestFlight 也能验证）

在设备上（Loon 开启）访问：

| 地址 | 期望 |
| --- | --- |
| `/api/v1/store-accounts` | 200 + 账号列表 JSON |
| `/api/v1/control/status` | 200 + 记录的客户端协议版本，如 `4.4.1 (697.1)` |
| `/api/v1/nothing` | 404 + `{"error":{"code":"not_found"}}` |
| `/v1/appCategories`、`/v4/accounts/<id>/apps` | 补丁生效后返回 JSON，而不是"网络连接已中断" |

> 别用根路径 `http://tf-relay.test/` 判断健康度：脚本对 `/` 没有路由，会走转发分支，看到的"打不开"是假线索。

也可以用 Scripting 的 `scripting-ts eval` + `fetch` 在设备上直接测（走同一个 VPN）。

## 常见问题

**导入后 TestFlight 依然断网**
先确认插件已启用、脚本已下载成功；再自查上表。仍不行请看下一节回滚。

**改脚本后怎么生效**
直接改本仓库的 `TestFlight-Relay.js`，插件指向 `@main` 无需改动。jsDelivr 有缓存，急用把链接换成 `@<commit-sha>`。

**上游更新了怎么办**
重新取上游 `module/scripts/TestFlight-Relay.js`，重新打上面两处补丁（搜 `$task` / `this.$request` 即可定位）。

**想恢复原状**
停用本插件，重新启用上游模块/转换版插件即可；中转里的账号数据不受影响。要清空账号用脚本里的「⋯ → 重置登录」。

**能不能把中转放到别的机器**
可以，但需要那份服务端（Surge 脚本形态或自建 Node 服务），并把脚本里的中转地址改成它的地址即可；本仓库只解决 Loon 侧的适配。

## 来源与致谢

- 脚本与协议实现：[@ScriptingApp/Community-Scripts](https://github.com/ScriptingApp/Community-Scripts) 的 `module/scripts/TestFlight-Relay.js`、`module/TestFlight-Relay.sgmodule`，版权归原作者。
- 本仓库仅做 Loon 兼容性适配（上列两处），未改变业务逻辑。
- 上游仓库当前未附带 LICENSE 文件；如原作者有任何异议，请提 issue，会立即调整或下线。

## 免责声明

仅供个人学习与自用。使用中转/多账号方案可能违反 Apple 的服务条款，由此产生的账号风险由使用者自行承担。请勿在仓库或 issue 中粘贴 Cookie、AMD、kbSync、密码等任何凭据。

## 许可

- 本仓库的**适配部分**（两处 `[Loon 补丁]`、`.plugin`、README、LICENSE）采用 **MIT**，见 [LICENSE](LICENSE)。
- `TestFlight-Relay.js` 的**脚本主体**来自 [Community-Scripts](https://github.com/ScriptingApp/Community-Scripts)，版权归原作者，上游当前未附带 LICENSE；MIT 不改变也不主张该部分版权。
- 原作者如有异议，提 issue 即调整或下线。

---

## English

**TestFlight Relay — Loon build.** A Loon-compatible distribution of the TestFlight relay script from [ScriptingApp/Community-Scripts](https://github.com/ScriptingApp/Community-Scripts) (originally a Surge module).

- `TestFlight-Relay.js` — upstream `module/scripts/TestFlight-Relay.js` plus two Loon compatibility patches
- `TestFlight-Relay.plugin` — importable Loon plugin
- Surge users should keep using the upstream `.sgmodule`; this repo is not needed there.

**What it does.** The plugin intercepts TestFlight's requests to `http://tf-relay.test/...` and answers them locally, while also exposing the control API used by the “TestFlight 魔法攻击” Scripting script (`/api/v1/store-accounts`, `/api/v1/control/*`). Account material (cookies, AMD, kbSync) stays in the local Loon persistent store (`tf.accounts` / `tf.apps` / `tf.details` / `tf.tasks`) and is never uploaded anywhere.

**Install.**

1. Disable any other TestFlight Relay plugin first (avoid two matching rules).
2. Import: `https://cdn.jsdelivr.net/gh/Tempoit/tf-relay-loon@main/TestFlight-Relay.plugin`
3. Enable it and make sure the script downloaded successfully.
4. Force-quit TestFlight, then hit “Start TestFlight” again in the script.

Requires Loon **≥ 3.5.1 (983)** (the plugin uses the new Script syntax: `request if ${url} ~= /regex/i then script(...) with ...`).

**Why patches are needed.** Two runtime differences from Surge/QX, both ending in TestFlight showing “TestFlight 不可用 / 网络连接已断开” while the account page still works:

1. *Forwarding branch.* Loon has no `$task`, so the upstream code hands a rewritten request back to the proxy for continuation (`http://tf-relay.test/...` → `https://testflight.apple.com/...`). Surge/QX support that cross-origin + http→https continuation; Loon kills the connection. Patch: `if (!globalThis.$task && !globalThis.$httpClient) return request;` so Loon uses the script-side `$httpClient` fetch instead.
2. *`this.$request` detection.* In module scope `this` may be empty, so every request is misclassified as a cron run and simply passed through. Patch: check the global `$request` instead (`const hasRequest = typeof $request !== "undefined" && !!$request;`).

Search `[Loon 补丁]` in the script to locate both.

**Self-check endpoints** (no need to launch TestFlight):

| Endpoint | Expected |
| --- | --- |
| `/api/v1/store-accounts` | 200 + account list JSON |
| `/api/v1/control/status` | 200 + recorded client version, e.g. `4.4.1 (697.1)` |
| `/api/v1/nothing` | 404 + `{"error":{"code":"not_found"}}` |
| `/v1/appCategories`, `/v4/accounts/<id>/apps` | JSON after patching (previously “connection interrupted”) |

Do not use the root path `http://tf-relay.test/` as a health check — the script has no route for `/` and falls through to the forwarding branch.

**Updating.** Edit `TestFlight-Relay.js` in this repo; the plugin points at `@main`, so no plugin change is needed. jsDelivr caches, so pin `@<commit-sha>` when you need it immediately. When upstream changes, re-apply the two patches (search `$task` / `this.$request`).

**Credits.** Script and protocol work by the [Community-Scripts](https://github.com/ScriptingApp/Community-Scripts) authors; this repo only adds Loon compatibility. See [LICENSE](LICENSE).

**Disclaimer.** For personal study and use only. Relay/multi-account setups may violate Apple's terms of service; you bear the account risk. Never paste cookies, AMD, kbSync, passwords or any credentials into this repo or its issues.

