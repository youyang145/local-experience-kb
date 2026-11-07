# 本机接口

面板（`127.0.0.1:7788`）用的 HTTP 接口。**这是本机 UI 的接口，不是对外 REST 标准**——只监听本地；写接口要配对授权（见下）。

统一返回：

```json
{ "ok": true, "data": { } }
{ "ok": false, "error": "人话错误信息" }
```

## 写接口的配对授权

写接口（`POST` / `PATCH` / `DELETE`）不再是"随便调"：

1. 面板「MCP → 配对授权」显示 6 位配对码；
2. 第三方 `POST /api/pair {"code":"123456","name":"WorkBuddy"}` → 拿到 `token` 和 `tokenId`；
3. 之后每个写请求带 `x-kb-token: <token>`；
4. 浏览器里的同源面板按 `Origin` 放行，不用 token；
5. 服务重启 → 配对码和所有 token 一起失效；连错 5 次锁定 10 分钟并换码。

| 方法 | 路径 | 说明 |
|---|---|---|
| GET | `/api/pair/code` | 当前配对码（面板显示用），锁定后返回 `locked` |
| POST | `/api/pair` | `{ code, name? }` → `{ token, tokenId, at }` |
| GET | `/api/auth` | 已配对的 token 列表（编号、名称、配对时间、最近使用、调用次数） |
| POST | `/api/auth/revoke` | `{ tokenId }` 撤销单个 token |

留痕写在 `_mcp/api-auth.jsonl`，每次配对、失败、锁定、写操作都带 `at`（ISO 日期）。

**边界**：本机恶意进程仍能读内存或伪造 `Origin`；这套机制挡的是跨站请求和未授权的工具，不是本机提权。面板和服务的监听仍只绑 `127.0.0.1`。

## 读

| 方法 | 路径 | 说明 |
|---|---|---|
| GET | `/api/overview` | 统计、分区平表与两级树、设置、库路径、孤儿数、待拍板数 |
| GET | `/api/blocks?q=&type=&domain=&limit=` | 块列表；`domain` 按前缀命中（选「工具」含「工具/GG修改器」） |
| GET | `/api/blocks/:id` | 单块详情，含 `outline`（目录）、`tags` |
| GET | `/api/search?q=&limit=&includeRefuted=` | 检索；默认不含被推翻的经验 |
| GET | `/api/experiences?tab=` | 经验列表（主库 / 小经验区 / 反例） |
| GET | `/api/proposals` | 待拍板的提议 |
| GET | `/api/probation` | 小经验区条目 |
| GET | `/api/candidates` | 候选队列 |
| GET | `/api/settings` | 三档模式与阈值 |

## 写

| 方法 | 路径 | Body |
|---|---|---|
| POST | `/api/blocks` | `{ title, body, summary?, source?, domain?, kind?, tags? }` |
| PATCH | `/api/blocks/:id` | 同上任意字段 |
| DELETE | `/api/blocks/:id` | 逻辑删除，进 `_trash/` |
| POST | `/api/experiences` | `{ claim, kind?, scope?, limits?, domain? }` |
| POST | `/api/experiences/:id/action` | `{ action: "apply" \| "hit" \| "refine" \| "vote" \| "pin" \| "promote" \| "refute", ... }` |
| POST | `/api/merge` | `{ a, b, claim, scope?, domain? }` 融合两条 |
| POST | `/api/transplant` | `{ id, domain, claim? }` 跨域派生 |
| POST | `/api/proposals/:seq/decide` | `{ accept: true\|false, claim? }` |
| POST | `/api/settings` | 局部更新 |
| POST | `/api/repair` | 重扫文件、修映射 |

## 例子

检索：

```
GET /api/search?q=分词&limit=5
```

套用一条经验（成功）：

```
POST /api/experiences/k261101jxwoua/action
{ "action": "apply", "result": "success" }
```

返回里带 `priority` 与 `warnings`（套用前的反例提示）。

## 与插件的关系

插件 `dsh-plugin-local-kb` 不直接调这些 HTTP 接口，而是 import `src/` 里的库函数（`consult.mjs` 等），少一层网络。接口只服务面板。
