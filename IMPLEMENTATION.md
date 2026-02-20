# mcp-server-weread 实现说明

## 项目概述

本地运行的 MCP（Model Context Protocol）服务器，让 Claude 等 LLM 客户端能够读取微信读书的书架、笔记和划线数据。

- **版本**：0.2.2
- **SDK**：@modelcontextprotocol/sdk 1.26.0
- **运行方式**：stdio 传输，由 Claude Desktop 直接启动

## 项目结构

```
mcp-server-weread/
├── src/
│   ├── index.ts        # MCP 服务器主入口，工具注册与处理逻辑
│   └── WeReadApi.ts    # 微信读书 API 封装
├── build/              # 编译产物（tsc 输出）
│   ├── index.js
│   └── WeReadApi.js
├── .env                # 本地环境变量（不提交 git）
└── package.json
```

## Cookie 配置

认证通过 Cookie 实现，支持两种方式（优先级从高到低）：

1. **CookieCloud（推荐）**：自动从本地 CookieCloud 服务同步，无需手动更新
2. **直接配置**：在 `.env` 中写入 `WEREAD_COOKIE`

当前使用 CookieCloud，配置在 `.env` 和 Claude Desktop 的 `env` 字段中：

```
CC_URL=http://localhost:8088
CC_ID=rTEEmHfsFH11k4eFmqXHoF
CC_PASSWORD=kkQrVY49H5LMXNB3mZQET5
```

`WeReadApi` 在服务启动时初始化为单例，只调用一次 CookieCloud 获取 Cookie。

**Cookie 保活机制**：在 CookieCloud 浏览器扩展的「保活」选项中填入 `https://weread.qq.com`，扩展会定期访问该地址保持会话有效，防止 `wr_skey` 因长时间不活跃而过期。

**Cookie 自动刷新**：当 API 返回 `-2010`（会话失效）或 `-2012`（Cookie 过期）时，服务器会重置初始化状态，下一次 retry 自动重新从 CookieCloud 拉取最新 Cookie，无需手动重启服务。

## Claude Desktop 配置

配置文件路径：`~/Library/Application Support/Claude/claude_desktop_config.json`

```json
{
  "mcpServers": {
    "weread": {
      "command": "node",
      "args": ["/Users/zhangshuo/Projects/mcp-server-weread/build/index.js"],
      "env": {
        "CC_URL": "http://localhost:8088",
        "CC_ID": "rTEEmHfsFH11k4eFmqXHoF",
        "CC_PASSWORD": "kkQrVY49H5LMXNB3mZQET5"
      }
    }
  }
}
```

修改代码后需要执行 `npm run build`，然后重启 Claude Desktop 生效。

---

## 可用工具（5 个）

### 1. `get_bookshelf`

获取书架概览和书籍列表。

**参数：**
| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| limit | integer | 100 | 返回书籍数量上限 |

**返回：**
- `stats`：总书数、阅读状态（未读/在读/读完）、书籍来源、已购数量、有笔记数量、分类统计
- `booklistStats`：书单统计（总数、最大/最小书单）
- `booklists`：所有书单列表（名称、ID、书籍数量）
- `books`：书籍列表，按活跃度排序（笔记数+划线数+进度），每本包含：
  - `bookId`, `title`, `author`
  - `finishReading`（是否读完）
  - `progress`（百分比）
  - `readingTimeFormatted`（如"3小时20分钟"）
  - `last_read_time`（ISO 格式，可用于判断"最近一周"等时间筛选）
  - `noteCount`, `bookmarkCount`

> 注意：导入书籍（`bookId` 以 `CB_` 开头）的笔记无法通过微信读书 API 读取，需使用 `get_imported_book_highlights`。

---

### 2. `search_books`

在书架中按关键词搜索书籍。

**参数：**
| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| keyword | string | 必填 | 搜索关键词 |
| exact_match | boolean | false | 是否精确匹配 |
| include_details | boolean | true | 是否返回详细信息 |
| max_results | integer | 5 | 最大返回数量 |

**搜索范围**：书名、作者、译者、分类、书单名称。

**返回（include_details=true 时）**：
- 书籍基本信息（bookId、封面、出版信息、格式）
- 阅读状态（进度、阅读时长、开始时间、最后阅读时间）
- 书籍详情（字数、评分、简介、ISBN、出版社）
- 笔记/划线/想法数量

---

### 3. `get_book_notes_and_highlights`

获取指定书籍的所有划线和笔记，按章节组织。

**参数：**
| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| book_id | string | 必填 | 书籍 ID（从 get_bookshelf 或 search_books 获取） |
| include_chapters | boolean | true | 是否包含章节信息 |
| organize_by_chapter | boolean | true | 是否按章节组织 |
| highlight_style | integer\|null | null | 划线样式筛选（null=全部） |

**返回**：
- 书籍基本信息和阅读状态
- `chapters`：树状章节结构，每章包含 `highlights`（划线）和 `notes`（笔记）
- `uncategorized`：未归入章节的内容

> 注意：仅适用于微信读书原生书籍（非 `CB_` 前缀）。传入导入书籍 ID 时会直接返回错误并提示使用 `get_imported_book_highlights`。

---

### 4. `get_imported_book_highlights`

获取导入书籍（`CB_` 前缀）的划线和笔记，数据来源是本地 web-highlighter 服务（`http://localhost:3100`）。

**参数：**
| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| keyword | string | 可选 | 按书名关键词过滤，不填则返回全部 |

**返回**：
- `total_books`：匹配书籍数
- `total_highlights`：匹配划线总数
- `books`：按书名分组的划线列表，每条包含 `text`、`note`、`timestamp`

**依赖**：需要 web-highlighter 服务在 `localhost:3100` 运行。

---

### 5. `get_book_best_reviews`

获取指定书籍的热门书评。

**参数：**
| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| book_id | string | 必填 | 书籍 ID |
| count | integer | 10 | 返回书评数量 |
| max_idx | integer | 0 | 分页索引 |
| synckey | integer | 0 | 分页同步键 |

**返回**：书籍信息、书评总数、是否有更多、每条书评的内容/评分/点赞数/作者/是否剧透/是否置顶。

---

## 关键设计决策

### 响应体积控制

`get_bookshelf` 早期版本返回全量数据（498 本书约 305KB），超出 Claude 上下文处理能力导致回复失败。当前方案：
- 每本书只保留 8 个核心字段（去掉分类、书单、价格等）
- 默认最多返回 100 本，按活跃度排序

### WeReadApi 单例与 Cookie 自动刷新

`WeReadApi` 实例化于服务启动时，所有工具调用共享同一实例，避免每次请求都重新调用 CookieCloud。

当 API 返回 `-2010`/`-2012` 时，`handleErrcode` 将 `initialized` 重置为 `false`，`retry` 机制会触发 `ensureInitialized()` 重新从 CookieCloud 拉取最新 Cookie，整个过程对调用方透明。

### 错误返回格式

遵循 MCP SDK 1.x 规范，工具错误通过 `isError: true` 返回，而非旧版的 `{ error: { message } }`：

```typescript
return {
  isError: true,
  content: [{ type: "text", text: error.message }]
};
```

---

## 开发流程

```bash
# 安装依赖
npm install

# 编译（输出到 build/）
npm run build

# 启动 Inspector 调试（浏览器访问 http://localhost:6274）
npm run inspector
```

修改代码 → `npm run build` → 重启 Claude Desktop。
