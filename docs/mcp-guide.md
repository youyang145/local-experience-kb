# MCP 第三方接入指南

给别的 Agent 看的操作说明。规范本身在 [MCP 工具契约](spec/mcp-tool-contract.md)，这份只讲"怎么连、怎么调、哪里会错"。

## 0. 前置

- 面板在跑：`dsh-kb serve` 或 DSH 插件里打开面板；
- 面板 → MCP → 「启动 MCP」（默认关闭，关着时 POST 返回 403）；
- 参考实现默认只监听 `127.0.0.1:7788`。局域网/公网怎么开、怎么鉴权，属于宿主自己的部署，见《宿主接入》D-040。

## 1. 地址

```
POST http://127.0.0.1:7788/mcp
content-type: application/json
```

- 只支持 POST；GET 返回 405；
- JSON-RPC 2.0，协议版本 `2024-11-05`，`specVersion: "1.0"`。

## 2. 三步连上

### 第一步：initialize，拿会话号

```powershell
curl.exe -i -X POST http://127.0.0.1:7788/mcp `
  -H "content-type: application/json" `
  -d "{\"jsonrpc\":\"2.0\",\"id\":1,\"method\":\"initialize\",\"params\":{\"clientInfo\":{\"name\":\"my-agent\",\"version\":\"1.0\"}}}"
```

响应头里有：

```
Mcp-Session-Id: mcp-xxxxxxxx
```

响应体里有：

```json
{
  "result": {
    "protocolVersion": "2024-11-05",
    "serverInfo": { "name": "local-experience-kb", "version": "0.15.x" },
    "specVersion": "1.0",
    "readOnly": false,
    "writeScope": "只写「待分区」收件箱与「待拍板」队列，不写主库",
    "instructions": "……你的连接编号：E-001……库的目录：……"
  }
}
```

**把 `Mcp-Session-Id` 存下来**，后面每个请求都要带。

> PowerShell 的 `Invoke-WebRequest` 取头要转字符串：`$sid = [string]$r.Headers["Mcp-Session-Id"]`；发请求时头名用小写 `mcp-session-id` 更稳。

### 第二步：看工具清单

```powershell
curl.exe -X POST http://127.0.0.1:7788/mcp `
  -H "content-type: application/json" `
  -H "Mcp-Session-Id: mcp-xxxxxxxx" `
  -d "{\"jsonrpc\":\"2.0\",\"id\":2,\"method\":\"tools/list\"}"
```

### 第三步：调工具

```powershell
curl.exe -X POST http://127.0.0.1:7788/mcp `
  -H "content-type: application/json" `
  -H "Mcp-Session-Id: mcp-xxxxxxxx" `
  -d "{\"jsonrpc\":\"2.0\",\"id\":3,\"method\":\"tools/call\",\"params\":{\"name\":\"kb_consult\",\"arguments\":{\"query\":\"BOM 启动失败\",\"limit\":5}}}"
```

返回：

```json
{ "result": { "content": [ { "type": "text", "text": "判定：查库\n……" } ] } }
```

## 3. Node 例子

```js
const endpoint = 'http://127.0.0.1:7788/mcp'
let sessionId = null

async function rpc(method, params) {
  const res = await fetch(endpoint, {
    method: 'POST',
    headers: {
      'content-type': 'application/json',
      ...(sessionId ? { 'Mcp-Session-Id': sessionId } : {}),
    },
    body: JSON.stringify({ jsonrpc: '2.0', id: Date.now(), method, params }),
  })
  sessionId ??= res.headers.get('mcp-session-id')
  const json = await res.json()
  if (json.error) throw new Error(`${json.error.code}: ${json.error.message}`)
  return json.result
}

await rpc('initialize', { clientInfo: { name: 'my-agent', version: '1.0' } })
const index = await rpc('tools/call', { name: 'kb_index', arguments: {} })
const hit = await rpc('tools/call', { name: 'kb_consult', arguments: { query: '插件加载顺序', limit: 5 } })
console.log(hit.content?.[0]?.text)
```

## 4. 四个工具

| 工具 | 参数 | 写不写库 |
|---|---|---|
| `kb_consult` | `{ query, limit? }` | 不写 |
| `kb_index` | `{}` | 不写 |
| `kb_stage` | `{ title, text }`，一次最多 5 块 | 只写「待分区」收件箱 |
| `kb_recommend` | `{ id, action: "promote"\|"refute", reason }` | 只进「待拍板」 |

主库的增删改、转正、推翻都只能由人在面板做。`kb_recommend` 只是提名，AI 不执行。

> 提醒：`kb_stage` 单块投稿时用你传的 `title`；多块按正文一级标题分块。

## 5. 会遇到的错误

| 现象 | 原因 | 处理 |
|---|---|---|
| HTTP 403 | MCP 开关没开 | 面板 → MCP → 启动 |
| HTTP 405 | 用了 GET | 只用 POST |
| `-32602`「先 initialize」 | 没带 `Mcp-Session-Id` | 重做第一步并保存会话号 |
| 查不到东西 | 库是空的 | 面板首页看块数；先入库 |
| 重启后会话失效 | 会话存在内存 | 重启后重新 `initialize`，编号会递增 |

## 6. 留痕

每次调用写 `_mcp/activity.jsonl`：时间、连接编号（`E-001` 起）、客户端名、工具、参数摘要。
写接口（`/api/*`）另有配对授权，和 MCP 是两套：MCP 不开主库写，所以不要求配对码。
