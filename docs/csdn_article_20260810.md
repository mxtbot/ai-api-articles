# 一个 URL 接入 127 个 MCP 工具：Cursor、Claude、Dify 三分钟配好，Agent 从此真的会"干活"

## 写在前面

你有没有遇到过这种情况：

让 AI 写代码，它写得头头是道——但让它"把仓库里所有 open 的 Issue 列出来"，它说"我做不到"；
让它总结新闻，它说"我的知识截止到训练时"；
让它读一个 PDF、查一张图片里的文字、拉一下今天的股价——统统摇头。

**AI 很聪明，但它没有手。**

MCP（Model Context Protocol，模型上下文协议）就是给 AI 安上双手的协议。2024 年底 Anthropic 开源，如今已经是事实上的行业标准——Cursor、Claude Desktop、VS Code、JetBrains、Dify、Coze、LangChain 全部原生支持。

但真正用起来，你会发现一个尴尬的现实：

> **每一个工具都是一个 MCP Server，你得一个个找、一个个配、一个个维护。**

GitHub 一个、文件系统一个、数据库一个、搜索一个……配五个工具就要装五个 Server，换台电脑重来一遍。更别提很多好用的工具压根没有现成的 MCP Server，得自己写。

我这半年在做 AI 算力平台（模型通），顺手把这件事彻底解决掉了：

**我把 127 个 MCP 工具聚合到了一个 URL 上。**

Cursor 里填一个地址，127 个工具全部出现在你的 Agent 面前，开箱即用，零配置。今天把整个方案和接入教程分享出来。

## 一、为什么需要"工具聚合"而不是"工具堆砌"

先看一张图，理解 MCP 的两种用法：

```
❌ 传统方式：一个工具一个 Server
Cursor ──→ GitHub Server（要配 Token）
       ──→ 文件系统 Server（要配目录）
       ──→ 数据库 Server（要配连接串）
       ──→ 搜索 Server（要配 API Key）
       ──→ …… 配到崩溃

✅ 聚合方式：一个地址全接入
Cursor ──→ https://mxt-api.cn/mcp/gateway/（一个 URL）
              ├── GitHub（44 个工具）
              ├── 文件系统（14 个）
              ├── 数据库（3 个）
              ├── 搜索 / 热榜 / OCR / 办公 / 视频……
              └── 共 127 个工具，23 大上游
```

聚合网关的价值，一句话：**把"配环境"的成本从用户身上拿走。**

- 用户侧：填一个 URL + 一个 Token，完事
- 平台侧：所有 Server 托管在云端，更新、扩容、加新工具都在一处
- 生态侧：接入一个新 Agent 客户端 = 配一次，127 个工具全部生效

## 二、127 个工具都有什么

先上清单，按用途分类（前缀即工具命名空间，一眼看出是哪个上游）：

| 分类 | 工具示例 | 能干什么 |
|:-----|:---------|:---------|
| 🐙 GitHub（44个） | gh_issue_read / gh_create_pull_request / gh_search_code / gh_push_files | 仓库、Issue、PR、代码搜索、甚至直接推文件 |
| 📁 文件系统（14个） | fs_read_file / fs_write_file / fs_search_files | 读写文件、目录操作、搜索 |
| 📈 市场数据（4个） | mkt_get_stock_quote / mkt_get_crypto_price / mkt_get_exchange_rate / mkt_get_weather | 股票、加密货币、汇率、天气，真实数据源 |
| 📊 中文热榜（3个） | hot_get_baidu / hot_get_bilibili / hot_get_all | 百度热搜、B站热门，一键聚合 |
| 🔤 OCR 识别（1个） | ocr_recognize | 图片文字提取，中英文，本地跑不花钱 |
| 📝 办公文档（4个） | office_excel_read / office_excel_write / office_docx_read / office_pptx_read | Excel 读写、Word/PPT 内容提取 |
| 🎬 视频音频（4个） | video_info / video_extract_audio / video_thumbnail / video_convert | 视频信息、抽音频、取封面、转格式 |
| 🌍 国际资讯（3个） | global_hackernews / global_reddit / global_arxiv | HN 头条、Reddit 热帖、arXiv 论文搜索 |
| ✈️ 物流航班（2个） | logi_flights / logi_couriers | 实时航班（OpenSky 免费源）、快递公司代码库 |
| 📰 新闻 RSS（2个） | news_get_news / news_hot_topics | IT之家、cnBeta、少数派、阮一峰、V2EX 头条聚合 |
| 🧰 实用工具（4个） | util_translate / util_ip_info / util_shorten_url / util_make_qrcode | 翻译、IP 归属地、短链、二维码 |
| 🗄️ 数据库（3个） | db_sqlite_query / db_mysql_query | 只读 SQL，写操作拦截，Agent 直连数据 |
| 🌐 浏览器（2个） | browser_fetch_page / browser_screenshot | 网页抓取转文本、无头浏览器截图 |
| 🖼️ 图片处理（3个） | img_image_info / img_image_resize / img_image_convert | 图片信息、缩放压缩、格式转换 |
| 📄 PDF 处理（3个） | pdf_extract_text / pdf_merge / pdf_info | 文本提取、多 PDF 合并、页数信息 |
| 🔎 联网搜索（1个） | search_search_web | 实时检索 + AI 摘要，让 Agent 拥有"当下" |
| 🧠 思维链（1个） | think_sequentialthinking | 让 Agent 分步推理，复杂问题更可靠 |
| 🗂️ 记忆图谱（9个） | mem_create_entities / mem_search_nodes | 知识图谱记忆，跨会话记住事实 |
| 🧪 测试套件（13个） | demo_echo / demo_get-sum | 协议测试与演示 |
| ⏱️ 其他自研（6个） | time_* / rand_* / saas_* | 时间、随机数、SaaS 接入演示 |

**说几个真实场景，感受一下"有手"的 Agent：**

1. **代码审查自动化**：让 Agent 列出仓库所有 Open PR → 逐个读 diff → 调用思维链推理 → 直接在 GitHub 上发评论。全程对话驱动。
2. **日报自动化**：Agent 拉新闻头条 + 查行情 + 读昨天的日报文档 → 生成今天的日报 → 写到文件系统。你只管看结果。
3. **文档流水线**：发一张截图 → OCR 提取文字 → 转成 Word 文档 → 存到指定目录。办公场景直接闭环。
4. **数据查询**：连上自己的 MySQL（只读），问 Agent"上个月订单量前 10 的商品是什么"，它自己写 SQL 自己查。

## 三、三分钟接入教程（Cursor 为例）

**第一步：准备 Token**

网关采用 Token 鉴权（安全考虑，无 Token 一律拒绝）。向网关提供方申请获取内测 Token，审核通过后发放，形如 `mxt-mcp-xxxxxxxxxxxxxxxx`。

**第二步：Cursor 里配置 MCP**

1. 打开 Cursor → Settings → 搜索 "MCP"
2. 点击 **Add new MCP server**
3. Type 选 **Streamable HTTP**
4. URL 填：
   ```
   https://mxt-api.cn/mcp/gateway/?token=你的内测Token
   ```
5. 保存，回到对话界面，点开工具列表——**127 个工具已经全部躺在里面了**

VS Code、JetBrains、Claude Desktop、Dify、Coze 的配置方式完全一样，只是入口位置不同（教程页有 7 个客户端的详细图文步骤）。

**第三步：验证**

不会配？直接一行 curl 验证网关是否活着：

```bash
curl -X POST "https://mxt-api.cn/mcp/gateway/" \
  -H "Content-Type: application/json" \
  -H "Accept: application/json, text/event-stream" \
  -H "Authorization: Bearer 你的内测Token" \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/list"}'
```

返回 127 个工具的定义，说明一切就绪。

**开发者专属**：网关完全兼容 MCP 标准协议（JSON-RPC 2.0），用任何语言的 MCP SDK 都能直接调用，不局限于特定客户端。

## 四、安全与成本：为什么敢免费开放

有人会问：聚合 127 个工具，安全吗？免费吗？

**安全方面做了三道闸：**

1. **Token 鉴权**：网关层强制校验，无 Token 直接 401，不暴露任何能力
2. **写操作隔离**：数据库只读、写操作拦截；GitHub 等敏感上游由独立机器账号承载，与个人账号隔离
3. **按需开放**：内测期逐批发放 Token，可控可回收

**成本方面：**

大部分工具跑在本地或免费接口上（OCR、办公、图片、PDF、热榜、航班……），零边际成本；GitHub 官方 MCP 在免费额度内；唯一有成本的是联网搜索，也有免费额度兜底。所以内测阶段工具全部免费开放，没有隐藏收费。

## 五、接入与体验

如果你也受够了"AI 没手"的日子，可以这样开始：

- **网关地址**：https://mxt-api.cn/mcp/gateway/（我的部署实例，供读者体验）
- **接入教程**：MCP 页有完整 7 客户端接入教程（Cursor / VS Code / JetBrains / Claude / Dify / Coze / LangChain）
- **工具投票**：MCP 页上有"工具需求投票"区——下一批接什么工具，用户说了算，呼声最高的优先接入

> 说明：聚合网关是标准 MCP 协议实现，你也可以基于开源组件自建一套（技术细节见另一篇文章）。

## 写在最后

做这个聚合网关的初衷很简单：**工具应该像水电一样即开即用，而不是每个都要自己拉管线。**

127 个只是起点。按用户的真实需求投票，这个数字会一直涨。

如果你有特别想要而清单里没有的工具，来投票区提——说不定下一批就有它。

---

*本文作者：独立开发者，MCP 工具聚合实践分享。*
