# 从零搭建 MCP 工具聚合网关：把 127 个工具塞进一个 URL 的技术笔记

## 写在前面

上周我干了一件事：把一个 MCP 聚合网关部署上线，把 127 个工具（GitHub、文件系统、数据库、OCR、办公、视频、搜索、热榜……）聚合到一个 URL，任何 MCP 客户端填一个地址就能全部使用。

这篇文章是完整的技术复盘。不吹不黑，从架构设计到踩坑记录，全部分享。如果你也在做 Agent 工具相关的开发，应该用得上。

## 一、为什么做聚合，而不是各用各的

MCP 生态的现状是：**工具很多，但很散**。

- 官方工具（filesystem、memory、sequential-thinking）要一个个装
- 第三方工具（GitHub、Tavily 搜索）要单独配 Token
- 自研工具要自己写 Server

用户配 5 个工具，就要装 5 个 Server、记 5 套配置。体验极差。

聚合网关的定位：**把"装工具"变成"填地址"**。

```
用户客户端（Cursor/Claude/Dify...）
        │
        ▼
┌─────────────────────────────┐
│   聚合网关 (MCP Gateway)     │
│  · 统一鉴权 (Token)          │
│  · 工具注册表 (127 tools)    │
│  · 请求路由 (前缀分发)        │
└─────────────────────────────┘
   │      │      │      │
   ▼      ▼      ▼      ▼
 GitHub  文件   数据库  搜索...
 (HTTP) (stdio) (stdio) (stdio)
```

## 二、核心架构设计

### 1. 双通道上游

上游分两类，各有适用场景：

**HTTP 上游**（远程服务）：
```javascript
{
  prefix: 'gh',
  name: 'GitHub 官方 MCP',
  type: 'http',
  url: 'https://api.githubcopilot.com/mcp/',
  headers: { Authorization: `Bearer ${process.env.GITHUB_TOKEN}` }
}
```
适用：已经提供 HTTP 端点的服务（GitHub 官方 MCP、自研 SaaS）。

**stdio 上游**（本地进程）：
```javascript
{
  prefix: 'fs',
  name: 'filesystem',
  type: 'stdio',
  command: 'node',
  args: ['.../server-filesystem/dist/index.js', '/tmp']
}
```
适用：Node 生态的官方 Server、自研工具。聚合器启动时拉起子进程，通过 stdin/stdout 走 JSON-RPC。

### 2. 前缀命名空间

每个上游一个前缀，工具名统一为 `前缀_工具名`：

```
gh_create_pull_request   ← GitHub 的 create_pull_request
fs_read_file             ← filesystem 的 read_file
mkt_get_stock_quote      ← 市场数据的 get_stock_quote
```

好处：
- 用户一眼看出工具归属
- 工具重名不会冲突
- 路由分发简单（按前缀找上游）

### 3. 鉴权与安全

网关层强制 Token 鉴权，双通道支持：

```
① Header: Authorization: Bearer <token>
② URL:    ?token=<token>
```

无 Token 一律 401，不泄露任何工具列表。敏感上游（GitHub）用独立机器账号承载写操作，与个人账号隔离。

## 三、踩过的坑（重点）

### 坑 1：MCP SDK 新版强制 Zod schema

早期写自研上游时：

```javascript
server.tool('get_news', '获取新闻', { source: 'string' }, handler)
// 报错：expected a Zod schema
```

新版 MCP SDK 要求参数 schema 必须是 Zod 对象：

```javascript
import { z } from 'zod'

server.tool('get_news', '获取新闻',
  { source: z.string().optional().describe('新闻源') },
  async ({ source }) => { ... }
)
```

**教训**：写 MCP Server 必须装 zod，参数定义用 `z.xxx().describe()`。

### 坑 2：慢上游拖死全局

最初所有上游同步连接，遇到 GitHub 远程慢（跨境网络 20 秒超时），整个聚合器启动卡死。

解决：每个上游连接 + listTools 加超时保护：

```javascript
const withTimeout = (p, ms) =>
  Promise.race([p, new Promise((_, rej) =>
    setTimeout(() => rej(new Error('timeout')), ms))])

const tools = await withTimeout(upstream.listTools(), 20000)
  .catch(() => [])  // 超时跳过，不拖死全局
```

**教训**：聚合器里没有哪个上游是不可或缺的。超时就跳过，保底可用性。

### 坑 3：stdio 子进程的生命周期

stdio 上游是聚合器的子进程，聚合器退出时必须回收：

```javascript
process.on('exit', () => {
  children.forEach(c => c.kill())
})
```

同时注意：**一个端口只能一个 Server**。初期把图片和 PDF 处理放同一个 HTTP Server，端口冲突导致重复注册——拆成两个独立 Server 解决。

### 坑 4：协议兼容性（Accept 头）

MCP 新版走 Streamable HTTP 协议，客户端必须声明接受两种格式：

```
Accept: application/json, text/event-stream
```

curl 测试时不加这个头，服务端会拒绝（返回 406）。排查半天才发现是测试姿势问题，不是服务问题。

### 坑 5：工具数量变更后的全页面同步

127 个工具不是一次到位的。每次加工具，页面上的"工具数"要全站同步——标题、副标题、卡片、FAQ，漏一处就出现"上一秒 110 下一秒 127"的诡异现象。

**教训**：改完数字，`grep -oE "[0-9]+ 个工具"` 全站扫一遍。

## 四、性能与监控

- **启动时间**：约 20 秒（主要是 GitHub 远程连接）
- **内存占用**：聚合器本体 ~12MB，20 个 stdio 子进程各 ~5-50MB
- **健康检查**：`tools/list` 返回工具数，异常时告警
- **pm2 管理**：进程守护 + 自动重启 + 日志轮转

## 五、后续规划

1. **按用户鉴权**：现在一个内测 Token 管所有，正式版要做用户级 Token（对应 One API 的子 Key 体系）
2. **用量计费**：工具调用按算力计费（罐结算体系），高级工具扣费、免费工具不扣
3. **工具市场**：开放第三方开发者接入，平台抽成，用户按需订阅
4. **智能路由**：同功能多上游（如多个搜索源）自动选优

## 写在最后

聚合网关本质上是一层"翻译+路由+鉴权"的薄壳，技术难度不高，但工程细节很多——超时、生命周期、协议兼容、命名空间，每个都是坑。

但它带来的体验提升是巨大的：**用户不再关心工具从哪来，只关心"AI 能不能帮我干活"。**

工具即插即用，Agent 的能力边界，从"会聊天"扩展到"会干活"。这层薄壳，值得做。

---

*本文作者：独立开发者，网关实测环境：Node 22 + MCP SDK + pm2。*
