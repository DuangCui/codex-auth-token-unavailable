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

## 参考

- openai/codex issue #21118: Browser Use actions fail with "Codex auth token is unavailable" in API-key mode
- openai/codex issue #21710: Chrome Browser Use remote navigation depends on ChatGPT auth and breaks in API-key/custom-provider mode
- openai/codex issue #24457: Codex Desktop mixes local custom provider base_url with remote auth.json API key
- openai/codex issue #19187: Windows Codex app 内 `node.exe` 无法启动导致 Browser Use 失败(另一类根因)
- CC Switch PR #5469 / #5828 与 v3.20.1 发布说明:`requires_openai_auth` 与第三方 live 路由的投影规则
- CC Switch 官方指南:Codex 官方认证保留(`docs/guides/codex-official-auth-preservation-guide`)
