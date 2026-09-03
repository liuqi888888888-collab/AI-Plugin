# Codex Pentest Bridge

让 Codex CLI 直接驱动你的 Chrome 浏览器,对**已授权目标**做渗透测试。
请求从你真实浏览器的上下文发出,自动携带登录态(Cookie/Session),
适合测试需要认证的接口:越权(IDOR)、注入、认证缺陷、前端漏洞等。

## 架构

```
Codex CLI (终端里的 AI)
   |  node bridge/cli.mjs fetch https://target/api -X POST ...
   v
桥接服务 (Node 零依赖, 仅监听 127.0.0.1:8765)
   ^  WebSocket + token
   |
Chrome 扩展 (MV3 background service worker)
   |  chrome.tabs / scripting / cookies / debugger
   v
你的真实浏览器 (带登录态)
```

## 目录

```
codex-pentest-bridge/
|-- extension/          Chrome 扩展(MV3,加载"已解压的扩展程序"选这个目录)
|   |-- manifest.json
|   |-- background.js   核心:接收命令、在浏览器上下文执行
|   |-- popup.html/js   弹窗:连接状态 + 服务地址/Token 配置
|-- bridge/
|   |-- server.mjs      本地桥接服务(零依赖,自带 WebSocket 实现)
|   |-- cli.mjs         命令行客户端(Codex 通过它下发命令)
|-- test/
|   |-- run-test.mjs    端到端自测(模拟扩展客户端)
```

## 安装(一次性)

1. 需要 Node 18+(本机已验证 v24)。
2. 启动桥接服务:
   ```
   node bridge/server.mjs          # 可选 --port 8765 --token xxx
   ```
3. Chrome 打开 chrome://extensions -> 开启"开发者模式" -> "加载已解压的扩展程序" -> 选择 `extension/` 目录。
4. 点击扩展弹窗,显示"已连接"即就绪(默认地址/Token 与 server 一致,无需改)。

## Codex 联动

技能 `codex-pentest-bridge` 已写入 `~/.codex/skills/`。
以后你只要对 Codex 说"用浏览器测一下这个网站: <url>",
Codex 会自动检查桥接状态并通过 cli.mjs 驱动你的浏览器。

## 命令参考(输出均为 JSON)

| 命令 | 说明 |
| --- | --- |
| `status` | 桥接与扩展连接状态 |
| `tabs` / `open <url> [--bg]` / `close <tabId>` / `goto <tabId> <url>` | 标签页管理 |
| `fetch <url> [-X M] [-H 'K: V']... [-d body]` | 在同源标签页上下文发请求,自动带 Cookie;无同源标签页时自动开一个后台标签页完成请求 |
| `cookies <domain\|url>` | 读取 Cookie(含 HttpOnly) |
| `dom <tabId> [selector]` | 获取页面 HTML |
| `links <tabId>` | 提取链接/表单/内联脚本中的 API 路径 |
| `eval <tabId> <expression>` | 经 DevTools 协议执行任意 JS(可点击/输入/验证 DOM 型漏洞) |
| `shot <tabId> [file.png]` | 截图当前可见区域 |

示例:

```
node bridge/cli.mjs open https://target.example.com
node bridge/cli.mjs tabs
node bridge/cli.mjs links 123
node bridge/cli.mjs fetch https://target.example.com/api/user/1001
node bridge/cli.mjs fetch https://target.example.com/api/user/1002 -H 'Accept: application/json'
node bridge/cli.mjs eval 123 "document.querySelector('form').submit()"
```

## 安全须知

- 仅限测试你拥有授权的目标;未授权测试是违法行为。
- 桥接只监听 127.0.0.1,且 WebSocket 需 token(默认 `codex-pentest`,
  正式使用建议 `--token` 换成随机值,并在扩展弹窗里同步修改)。
- `eval` 可在页面执行任意 JS,`fetch` 可发出带凭据的任意请求——
  请像保管密码一样保管运行桥接的机器与浏览器。
- 破坏性操作(删除/修改数据)前请先三思或备份。

## 故障排查

- 弹窗显示"未连接":确认 `node bridge/server.mjs` 在跑、端口未被占用;
  查看终端日志是否有 `extension connected`。
- 扩展 Service Worker 休眠后会自动通过 alarm 重连(≤1 分钟)。
- `fetch` 报 executeScript failed:目标标签页是 chrome:// 等受限页面,先导航到普通页面。
- 自测:`node test/run-test.mjs`(不需要 Chrome)。
