# Codex Desktop 报 "Codex auth token is unavailable" 的完整排查与修复

适用于 Windows + Codex Desktop + CC Switch(cc-switch)组合,模型走第三方中转 API 的场景。

## 一句话结论

`auth.json` 只负责官方登录态(浏览器通道、插件市场、远程控制、用量),`config.toml` 只负责模型流量走哪个 provider。两者必须并存,不能互相覆盖。本次故障是 `auth.json` 被清空,导致 Computer Use 的浏览器通道拿不到官方 ChatGPT token。

## 症状

在 Codex 里让模型操作浏览器或读取页面时,工具调用返回:

```json
{"apps":[],"browsers":[],"errors":["Browsers: Error: Codex auth token is unavailable"]}
```

同时 Electron 侧日志反复出现:

```
warning [electron-fetch-wrapper] desktop_fetch_auth_401 hadToken=false skipRetryReason=no_token_attached
  target="GET https://chatgpt.com/backend-api/wham/usage"
warning [electron-message-handler] sa_server_request_failed ... routePattern=/settings/user status=401
```

注意这条报错与模型 provider 无关。即使模型走第三方中转一切正常,只要没有官方登录,浏览器通道依然报这句话。

## 涉及的两个文件

| 文件 | 作用 | 谁维护 |
| --- | --- | --- |
| `~/.codex/auth.json` | 官方 ChatGPT 登录缓存:`auth_mode`、`tokens.access_token/refresh_token/id_token/account_id`、`last_refresh` | Codex 登录流程 |
| `~/.codex/config.toml` | `model_provider`、`[model_providers.*]`(base_url、wire_api、token、requires_openai_auth)、沙箱与桌面设置 | CC Switch 切换供应商时重写 |

关键规则(Codex CLI >= 0.144):provider 表里的 `requires_openai_auth = true` 含义是"用保留下来的官方登录去认证",不是"这个 provider 需要 API key"。第三方中转应当写成:

```toml
model_provider = "custom"
model = "gpt-5.6-terra"

[model_providers.custom]
name = "第三方中转"
base_url = "https://your-relay.example.com"
wire_api = "responses"
requires_openai_auth = false
experimental_bearer_token = "sk-..."
```

## 排查过程

1. 定位报错来源:在会话记录 `~/.codex/sessions/YYYY/MM/DD/rollout-*.jsonl` 里检索 `item_completed` 里含 `auth token is unavailable` 的条目,可以看到失败的是 MCP server `cua_repl`(插件 `unified-computer-use@openai-bundled`)的 `cua.getState()`。

2. 确认官方登录状态:检查 `~/.codex/auth.json` 是否存在。本次它不存在。

3. 找它消失的原因:核心日志 `~/.codex/logs_2.sqlite`(表 `logs`,`ts` 为微秒)里检索 `oauth/revoke`,命中:

```
app_server.request ... rpc.method="account/logout"
  Request completed method=POST url=https://auth.openai.com/oauth/revoke
```

即一次应用内退出登录撤销了 token,随后 `auth.json` 被删除,之后请求报 `"code": "token_revoked"`。

4. 反证:登录成功后同一条 `cua.getState()` 立刻成功返回浏览器与标签页列表,证明该通道只依赖官方登录。

## 修复步骤

1. 先设置 CC Switch:`设置 -> 通用 -> Codex 应用增强 -> 切换第三方供应商时保留官方登录` 打开。对应 `~/.cc-switch/settings.json` 的 `preserveCodexOfficialAuthOnSwitch: true`(默认关闭)。修改该文件时先退出 CC Switch,改完再启动,并回读确认。

2. 在 Codex 里登录官方 ChatGPT 账号一次。登录成功后 `~/.codex/auth.json` 会重新生成,`auth_mode` 为 `chatgpt`。官方文档说明 Free 订阅即可满足这一步。

3. 在 CC Switch 里选择要用的第三方供应商,确认 `config.toml` 里该 provider 带 `experimental_bearer_token` 且 `requires_openai_auth = false`,然后重启 Codex(配置在启动时读取)。

4. 验证:

```bash
# 中转本身是否可用
curl -s -o /dev/null -w '%{http_code}\n' --max-time 25 -X POST "https://your-relay.example.com/responses" \
  -H "Authorization: Bearer $KEY" -H "Content-Type: application/json" \
  -d '{"model":"gpt-5.6-terra","input":"hi","max_output_tokens":16}'

# 登录态是否齐全
python -c "import json;d=json.load(open(r'%USERPROFILE%\.codex\auth.json'));print(d['auth_mode'], bool(d['tokens']['access_token']), d.get('last_refresh'))"
```

## 不要做的事情

- 不要在 Codex 里先退出登录再切换供应商。那是 CC Switch 3.16.1 之前的旧行为,会把第三方 key 写进 `auth.json`,覆盖官方登录缓存,正是浏览器通道失效的根源。
- 不要把 `auth.json` 的内容复制到任何地方,里面有可用的 access token 与 refresh token。
- 不要在开启本地路由接管时切到官方供应商,通过代理访问官方接口存在账号风险。

## 2026-10-12 续一:重启后掉登录(token 自然到期)

### 症状

- 重启电脑后 Codex Desktop 停在登录页,不再自动进入;
- 中转与 CC Switch 一切正常:当天用量日志仍在增长,本地路由 `127.0.0.1:15721` 正常监听,`config.toml` 未被改动。

### 根因

官方登录 token 有自然寿命(约 30 天),平时由应用自动续期。本次 `auth.json` 的 `last_refresh` 停在 2026-09-10(即上次事故后重新登录的时间),到 2026-10-12 已 32 天,token 过期且续期未成功。应用常驻运行时不会主动重验,重启迫使它做一次校验,于是退回登录页——重启只是恰好撞上到期,不是它弄坏的。

### 排查要点

`auth.json` 存在、四件套齐全时,只看一个字段就能定位:

```bash
python -c "import json;d=json.load(open(r'%USERPROFILE%\.codex\auth.json'));print(d['last_refresh'])"
```

距今超过 30 天即为到期。同时确认 15721 在监听(`netstat -ano | findstr 15721`)、`~/.cc-switch/settings.json` 的 `preserveCodexOfficialAuthOnSwitch` 仍为 `true`、历史库 `thread_history_1.sqlite` 完好。

### 修复

登录页点"继续登录",用同一个 OpenAI 账号登录一次,`last_refresh` 刷新为当天即成功。历史记录不受影响——会话存于本地 `sessions/` 与 `thread_history_1.sqlite`,登录只换 token,不碰数据。

## 2026-10-12 续二:重新登录后,模型选择器里没有 API 模型

### 症状

登录成功后,模型选择器只剩官方模型(gpt-6-astra 一类),第三方供应商的模型全部消失;`config.toml` 的 `model` 被重置为官方默认,并新增了 `model_reasoning_effort = "ultra"`。

### 根因

登录动作会从官方账号同步一份官方模型目录,并把默认模型重置为官方款;同时 CC Switch 侧的"模型目录接管"被摘除——`~/.cc-switch/codex-client-ack.json` 的 `catalog` 字段变为 `"none"`,`codex-catalog-history.json` 里指纹在 `none` 与哈希之间翻转。目录未注入时,Codex 回落到官方模型列表。本地路由、provider、登录态本身都完好,缺的只是目录接管这一层。

### 修复

1. CC Switch → Codex → 重新点击一次要用的供应商卡片(重新应用):它会重写 `config.toml`(恢复该供应商的默认模型)并重新注入模型目录(`model_catalog_json = "cc-switch-model-catalog.json"` + `~/.codex/cc-switch-model-catalog.json`);
2. 完全退出 Codex 再启动(托盘右键退出,不是关窗口)——`config.toml` 与模型目录都在启动时读取;
3. 配了多个供应商时,注意点真正要用的那张卡片:每家注入的模型目录不同(例如 DeepSeek 直连供应商注入 deepseek-v4-*,中转供应商注入自己的模型列表)。

`preserveCodexOfficialAuthOnSwitch = true` 会保住刚登录好的官方 token,此步不会把它覆盖。

### 周期性提醒

官方 token 约 30 天到期属正常现象。完整恢复链:登录一次 → CC Switch 重切供应商 → 完全重启 Codex,三步即可。重新登录后不要直接用官方默认模型跑——此时默认模型是官方目录里的名字,中转不一定认。

## 参考

- openai/codex issue #21118: Browser Use actions fail with "Codex auth token is unavailable" in API-key mode
- openai/codex issue #21710: Chrome Browser Use remote navigation depends on ChatGPT auth and breaks in API-key/custom-provider mode
- openai/codex issue #24457: Codex Desktop mixes local custom provider base_url with remote auth.json API key
- openai/codex issue #19187: Windows Codex app 内 `node.exe` 无法启动导致 Browser Use 失败(另一类根因)
- CC Switch PR #5469 / #5828 与 v3.20.1 发布说明:`requires_openai_auth` 与第三方 live 路由的投影规则
- CC Switch 官方指南:Codex 官方认证保留(`docs/guides/codex-official-auth-preservation-guide`)
