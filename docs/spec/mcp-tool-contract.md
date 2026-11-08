# MCP 工具契约（v1.0）

其他 Agent 通过 MCP 使用本地经验库时，按这份契约实现或对接。
本文是**接口规范**，不是实现说明；换语言重写也以这份为准。
连不上时先看[第三方接入指南](mcp-guide.md)：三步、例子、错误表。

## 传输

| 项 | 值 |
|---|---|
| 端点 | `POST http://127.0.0.1:7788/mcp` |
| 编码 | JSON-RPC 2.0，`content-type: application/json` |
| 协议版本 | MCP `2024-11-05` |
| 规范版本 | `specVersion: "1.0"`（随本文档走） |
| 网络 | 参考实现默认只监听 `127.0.0.1`；局域网/公网由宿主自行部署与鉴权（D-040） |

## 会话

1. 客户端发 `initialize`，`params.clientInfo` 上报 `{name, version}`；
2. 服务端在响应头返回 `Mcp-Session-Id`，并分配连接编号 `E-001` 起；
3. 之后所有请求都要带 `Mcp-Session-Id`，否则 `tools/call` 返回错误。

`ping` 可用于探活：`{"jsonrpc":"2.0","id":1,"method":"ping"}` → 空 result。

MCP 开关默认关闭；关闭时 **POST 请求返回 403**。非 POST 请求会先被方法检查拦成 405（405 优先级高于 403）。

## 工具

### `kb_consult`

查库。只看该不该查、有哪些命中（简介），不读全文。

```json
{ "name": "kb_consult", "arguments": { "query": "BOM 启动失败", "limit": 5 } }
```

返回文本：判定（查库/不查）+ 索引摘要 + 命中列表。
零命中时给两种提示之一：

- 有相近分区：`没有命中。最接近的分区是「…」，可能只是换了措辞…`
- 没覆盖：`没有命中，而且本库没有覆盖这个话题（现有分区：…）。可以用你自己的知识回答，但要说明不是来自本库。`

### `kb_index`

看目录：分区、条数、最近标题。无参数。

### `kb_stage`

把材料放进「待分区」收件箱，一次最多 5 块。

```json
{ "name": "kb_stage", "arguments": { "title": "材料标题", "text": "Markdown 正文" } }
```

**只写收件箱，不写主库**；人再决定挪到哪个分区。

### `kb_recommend`

给某条经验提建议，必须带理由。

```json
{ "name": "kb_recommend", "arguments": { "id": "k...", "action": "promote", "reason": "为什么" } }
```

`action` 只能是 `promote`（转正）或 `refute`（推翻）。
只进「待拍板」，由人一票决定；**AI 不执行**。

## 错误码

| code | 含义 |
|---|---|
| `-32600` | 请求不是合法 JSON-RPC |
| `-32601` | 方法不支持 |
| `-32602` | 参数错 / 没有先 initialize / 未知工具 |
| `-32603` | 服务端执行失败 |
| HTTP 403 | MCP 开关关闭 |
| HTTP 405 | 用了 GET；只支持 POST |

## 边界

- **读**：consult / index 随时可用；
- **写**：只有 stage（收件箱）和 recommend（待拍板），主库的增删改必须由人在面板拍板；
- **留痕**：每次调用写 `_mcp/activity.jsonl`，含时间、连接编号、客户端、工具、参数摘要；
- **版本**：工具增删或参数语义变化 → `specVersion` 升次版本；破坏性改动升主版本。
