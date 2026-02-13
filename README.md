# 微信读书 MCP Server

一个为微信读书提供MCP（Model Context Protocol）服务的工具，支持将微信读书的书籍、笔记和划线数据提供给支持MCP的大语言模型客户端，如Claude Desktop。

## 功能特点

- 从微信读书获取书架信息
- 搜索书架中的图书
- 获取图书的笔记和划线
- 获取图书的热门书评
- 支持按章节组织笔记和划线
- 与支持MCP协议的LLM客户端无缝集成

## 主要工具

1. **get_bookshelf** - 获取用户书架上所有书籍
   - 返回书籍基本信息，包括书名、作者、译者和分类等

2. **search_books** - 通过关键词检索用户书架上的书籍
   - 支持模糊匹配和精确匹配
   - 可选是否包含详细信息
   - 可设置最大结果数量

3. **get_book_notes_and_highlights** - 获取指定书籍的所有划线和笔记
   - 支持按章节组织结果
   - 支持筛选划线样式
   - 返回结构化的数据以便于LLM理解

4. **get_book_best_reviews** - 获取指定书籍的热门书评
   - 支持设置返回数量
   - 支持分页浏览
   - 包含评分、点赞数和评论者信息

## 安装与使用

### 先决条件

- Node.js 16.x 或更高版本
- 微信读书账号和有效的Cookie

### 安装教程

详见：[Weread MCP Server 使用指南](https://chenge.ink/article/post20250505)

### 与Claude Desktop集成

有多种方式可以与Claude Desktop集成：

#### 方式一：通过 npx 使用（最简单，推荐）
1. 打开Claude Desktop
2. 进入设置 -> MCP配置
3. 添加工具，使用以下JSON配置：
   ```json
   {
     "mcpServers": {
       "mcp-server-weread": {
         "command": "npx",
         "args": ["-y", "mcp-server-weread"],
         "env": {
           // 使用本地 CookieCloud（推荐）
           "CC_URL": "http://localhost:8088",
           "CC_ID": "您的ID",
           "CC_PASSWORD": "您的密码"
         }
       }
     }
   }
   ```

#### 方式二：全局安装后使用

1. 全局安装包：
   ```bash
   npm install -g mcp-server-weread
   ```

2. 在Claude配置中使用：
   ```json
   {
     "mcpServers": {
       "mcp-server-weread": {
         "command": "mcp-server-weread",
         "env": {
           // 同上方式配置环境变量
         }
       }
     }
   }
   ```

> 提示：直接在Claude配置中提供环境变量的方式更加方便，无需设置.env文件，推荐使用。

## CookieCloud 配置说明

为了解决 Cookie 频繁过期，需要重新获取并更新环境变量的问题。本项目支持 [CookieCloud](https://github.com/easychen/CookieCloud) 服务来自动同步和更新 Cookie。CookieCloud 是一个开源的跨浏览器 Cookie 同步工具，支持自建服务器。

### ⚠️ 重要：关于 CookieCloud 服务器

**公共服务器 `https://cc.chenge.ink` 已过期，请使用本地 Docker 部署。**

### 配置步骤：

#### 第一步：安装浏览器插件
- Edge商店：[CookieCloud for Edge](https://microsoftedge.microsoft.com/addons/detail/cookiecloud/bffenpfpjikaeocaihdonmgnjjdpjkeo)
- Chrome商店：[CookieCloud for Chrome](https://chromewebstore.google.com/detail/cookiecloud/ffjiejobkoibkjlhjnlgmcnnigeelbdl)

#### 第二步：启动本地 CookieCloud 服务（Docker）

```bash
# 启动本地 CookieCloud 服务（映射到 8088 端口）
docker run -d --name cookiecloud -p 8088:8088 easychen/cookiecloud:latest
```

插件中服务器地址填：`http://localhost:8088`

#### 第三步：配置 CookieCloud 插件

1. **服务器地址**：`http://localhost:8088`（本地 Docker）
2. **点击「自动生成密码」** 生成密码
3. **同步域名关键词**中填入 `weread`
4. **点击「保存」**，然后点击「手动同步」确保配置生效

#### 第四步：配置保活功能（重要！）

⚠️ **必须配置保活功能，否则 Cookie 会过期！**

在 CookieCloud 插件设置中：
1. 找到 **「保活」** 选项
2. 填入：`https://weread.qq.com`
3. 保存设置

**保活的作用**：插件会自动定期访问微信读书，保持 Cookie 有效，避免 `wr_skey` 过期。

### 关于 wr_skey

`wr_skey` 是微信读书的 **会话密钥 (Session Key)**，用于验证登录身份。它是访问私密数据（书架、笔记等）的关键 Cookie。

**如果缺少 `wr_skey`**：
- API 会返回错误：`errCode: -2010, errMsg: 用户不存在`
- 解决方法：重新登录微信读书后，手动同步 CookieCloud

### Cookie 检测脚本

你可以使用以下脚本检测 Cookie 是否包含 `wr_skey`：

```bash
#!/bin/bash
# 保存为 check_weread_cookie.sh

CC_URL="http://localhost:8088"
CC_ID="您的ID"
CC_PASSWORD="您的密码"

echo "=== 微信读书 Cookie 检测 ==="
echo "时间: $(date "+%Y-%m-%d %H:%M")"

RESULT=$(curl -s -X POST "${CC_URL}/get/${CC_ID}" \
  -H "Content-Type: application/json" \
  -d "{\"password\": \"${CC_PASSWORD}\"}" 2>/dev/null)

HAS_SKEY=$(echo "$RESULT" | grep -o '"wr_skey"' | head -1)

if [ -n "$HAS_SKEY" ]; then
    echo "✅ wr_skey 存在，Cookie 正常"
else
    echo "❌ wr_skey 缺失！"
    echo "解决方法："
    echo "1. 打开浏览器访问 https://weread.qq.com"
    echo "2. 退出登录后重新扫码登录"
    echo "3. 点击 CookieCloud 扩展手动同步"
    echo "4. 确保在 CookieCloud 插件中配置了保活地址"
fi
```

### 在 MCP 配置中填写 CookieCloud 变量

| 变量 | 说明 | 示例 |
|------|------|------|
| `CC_URL` | CookieCloud 服务器地址 | `http://localhost:8088` |
| `CC_ID` | CookieCloud 用户 UUID | `ftx64pJ1i8fAoj6mTLTmYb` |
| `CC_PASSWORD` | CookieCloud 密码 | `your_password` |

注意：配置 CookieCloud 后，系统会优先使用 CookieCloud 获取 Cookie，获取失败才会使用 `WEREAD_COOKIE` 环境变量的值。

## Cookie 获取方法（备用方案）

如果不想使用 CookieCloud，可以手动获取 Cookie：

1. 使用 Chrome 浏览器登录 [微信读书网页版](https://weread.qq.com/)
2. 按 F12 打开开发者工具，切换到 Network 标签页
3. 刷新页面，在请求列表中找到 weread.qq.com 的请求
4. 在 Headers 中找到 Cookie 字段，复制其完整内容
5. 将复制的内容添加到 MCP Json 的 `WEREAD_COOKIE` 环境变量中

## 使用示例

以下是一些与 LLM 对话的示例：

**获取书架信息**
```
用户: 帮我查看我的微信读书书架
LLM: 我将获取您的微信读书书架信息。
[LLM调用get_bookshelf工具]
我从您的微信读书书架获取到了208本书籍。以下是部分书籍：
1. 《隐藏的自我》- 大卫·伊格曼
2. 《思考，快与慢》- 丹尼尔·卡尼曼
3. 《人类简史》- 尤瓦尔·赫拉利
...
```

**搜索特定书籍**
```
用户: 帮我找关于心理学的书
LLM: 我将搜索您书架上与心理学相关的书籍。
[LLM调用search_books工具，参数为keyword="心理学"]
在您的书架上，我找到了3本与心理学相关的书籍：
1. 《思考，快与慢》- 丹尼尔·卡尼曼（进度：75%）
2. 《乌合之众》- 古斯塔夫·勒庞（进度：50%）
3. 《心理学与生活》- 理查德·格里格（进度：30%）
您想了解哪一本书的更多信息？
```

**获取笔记和划线**
```
用户: 帮我整理《思考，快与慢》这本书的笔记
LLM: 我将获取《思考，快与慢》的笔记和划线。
[LLM调用search_books工具，找到书籍ID]
[LLM调用get_book_notes_and_highlights工具]
以下是您在《思考，快与慢》中的笔记和划线（按章节组织）：

第一章：系统1和系统2
- 划线：「系统1的运行是无意识且快速的，不怎么费脑力，没有感觉，完全处于自主控制状态。」
- 您的笔记：「系统1负责直觉和快速判断，系统2负责缓慢的理性思考。」

第三章：锚定效应
- 划线：「锚定效应指的是在不确定情境下进行数字估计时，会过度受到事先接触的不相干数字的影响。」
...
```

**获取热门书评**
```
用户: 这本《原则》有哪些热门书评？
LLM: 我会为您查找《原则》的热门书评。
[LLM调用search_books工具，找到书籍ID]
[LLM调用get_book_best_reviews工具]
以下是《原则》的热门书评：

1. 评论者"思考者"（评分：5星）：
   "这是一本关于如何思考的书。达利欧将自己多年的经验总结为可操作的原则，帮助我们做出更好的决策。特别喜欢他关于'痛苦+反思=进步'的观点，非常实用。"
   👍 182 | 💬 23

2. 评论者"投资学习者"（评分：4星）：
   "桥水基金创始人的思想精华，值得反复阅读。书中的工作原则部分对管理者特别有帮助，建议先读生活原则，再读工作原则。"
   👍 94 | 💬 12
...
```

## 常见问题

### Q: 为什么提示「用户不存在」或返回空数据？

A: 通常是 `wr_skey` Cookie 缺失或过期。解决方法：
1. 重新登录微信读书网页版
2. 手动点击 CookieCloud 扩展同步
3. 确保配置了保活功能

### Q: CookieCloud 保活功能在哪里？

A: 在 CookieCloud 浏览器扩展的设置页面，找到「保活」选项，填入 `https://weread.qq.com`

### Q: 公共服务器为什么不能用？

A: `https://cc.chenge.ink` 公共服务器已过期，请使用本地 Docker 部署：
```bash
docker run -d --name cookiecloud -p 8088:8088 easychen/cookiecloud:latest
```

## 许可证

MIT

## 贡献

欢迎提交 Pull Request 或 Issue 来改进此项目。
