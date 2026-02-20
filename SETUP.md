# 微信读书 MCP Server — 安装与配置引导

按下面步骤操作即可完成本地安装和接入（如 Claude Desktop）。

---

## 一、环境要求

| 项目 | 要求 |
|------|------|
| Node.js | **16.x 或更高** |
| 微信读书 | 已注册账号 |
| （可选）Docker | 用于本地 CookieCloud 服务 |

检查 Node 版本：

```bash
node -v   # 应 >= 16
npm -v
```

若未安装或版本过低，请到 [Node.js 官网](https://nodejs.org/) 下载安装。

---

## 二、安装项目依赖

在项目根目录执行：

```bash
cd /Users/zhangshuo/Projects/mcp-server-weread
npm install
```

安装完成后构建：

```bash
npm run build
```

本地调试可运行：

```bash
npm start
# 或使用 MCP 检查工具
npm run inspector
```

---

## 三、Cookie 配置（二选一）

访问微信读书书架、笔记等需要有效 Cookie，推荐用 **CookieCloud**，备用方案是 **手动 Cookie**。

### 方案 A：CookieCloud（推荐）

**1. 安装浏览器插件**

- Edge：[CookieCloud for Edge](https://microsoftedge.microsoft.com/addons/detail/cookiecloud/bffenpfpjikaeocaihdonmgnjjdpjkeo)
- Chrome：[CookieCloud for Chrome](https://chromewebstore.google.com/detail/cookiecloud/ffjiejobkoibkjlhjnlgmcnnigeelbdl)

**2. 启动本地 CookieCloud 服务（Docker）**

```bash
docker run -d --name cookiecloud -p 8088:8088 easychen/cookiecloud:latest
```

插件里「服务器地址」填：`http://localhost:8088`

**3. 在插件中配置**

- 服务器地址：`http://localhost:8088`
- 点击「自动生成密码」并记住密码
- 「同步域名关键词」填：`weread`
- 保存后点击「手动同步」

**4. 配置保活（避免 Cookie 过期）**

在 CookieCloud 插件设置中：

- 找到「保活」
- 填入：`https://weread.qq.com`
- 保存

**5. 记下三个值，后面配置 MCP 要用**

- `CC_URL`：`http://localhost:8088`
- `CC_ID`：插件里显示的「用户 ID / UUID」
- `CC_PASSWORD`：上一步生成的密码

### 方案 B：手动 Cookie（备用）

1. 浏览器打开 [微信读书网页版](https://weread.qq.com/) 并登录
2. F12 → Network，刷新页面
3. 任选一个 `weread.qq.com` 的请求 → Headers → 复制完整 **Cookie**
4. 在 MCP 或 `.env` 里配置为 `WEREAD_COOKIE=复制的整段 Cookie`

---

## 四、配置 Claude Desktop（或其它 MCP 客户端）

### 使用 npx（推荐，无需全局安装）

在 Claude Desktop 的 MCP 配置里添加（JSON 片段）：

```json
{
  "mcpServers": {
    "mcp-server-weread": {
      "command": "npx",
      "args": ["-y", "mcp-server-weread"],
      "env": {
        "CC_URL": "http://localhost:8088",
        "CC_ID": "你的CookieCloud用户ID",
        "CC_PASSWORD": "你的CookieCloud密码"
      }
    }
  }
}
```

若用手动 Cookie，则把 `env` 改成：

```json
"env": {
  "WEREAD_COOKIE": "你复制的完整Cookie字符串"
}
```

### 使用本地开发目录

若你正在改代码、想用当前项目版本，可把 `command` 指向本地：

```json
{
  "mcpServers": {
    "mcp-server-weread": {
      "command": "node",
      "args": ["/Users/zhangshuo/Projects/mcp-server-weread/build/index.js"],
      "env": {
        "CC_URL": "http://localhost:8088",
        "CC_ID": "你的CC_ID",
        "CC_PASSWORD": "你的CC_PASSWORD"
      }
    }
  }
}
```

保存配置后重启 Claude Desktop，在对话里应能使用「微信读书」相关工具。

---

## 五、可选：本地 .env（不推荐给 Claude 用）

仅当你在本机用 `npm start` 或脚本调试、且不想在 MCP 里写 env 时，可复制示例并填写：

```bash
cp .env.example .env
# 编辑 .env，填写 CC_URL、CC_ID、CC_PASSWORD 或 WEREAD_COOKIE
```

注意：README 里已说明公共 CookieCloud 服务器 `https://cc.chenge.ink` 已过期，请用本地 Docker 的 `http://localhost:8088`。

---

## 六、验证

1. **Cookie 是否有效（用 CookieCloud 时）**

   README 中有 `check_weread_cookie.sh` 示例脚本，用 curl 请求 CookieCloud 的 `get` 接口，检查返回里是否包含 `wr_skey`。若无则需重新登录微信读书并再次在插件里「手动同步」。

2. **在 Claude 里验证**

   - 说：「帮我查看我的微信读书书架」
   - 若返回书籍列表，说明 MCP 与 Cookie 均正常。

---

## 七、常见问题

| 现象 | 处理 |
|------|------|
| 提示「用户不存在」或空数据 | 多为 `wr_skey` 缺失/过期：重新登录微信读书 → CookieCloud 手动同步 → 确认保活已填 `https://weread.qq.com` |
| 公共服务器连不上 | 不要用 `cc.chenge.ink`，改用本地 Docker `http://localhost:8088` |
| Node 版本不符 | 安装 Node 16+ 并确保 `node -v` 正确 |

---

## 八、项目脚本速查

| 命令 | 说明 |
|------|------|
| `npm install` | 安装依赖 |
| `npm run build` | 编译 TypeScript → `build/` |
| `npm start` | 运行 `build/index.js` |
| `npm run inspector` | 用 MCP Inspector 调试 |
| `npm run watch` | 监听源码变更并编译 |

按「二 → 三 → 四」顺序做完即可正常使用；若你卡在某一步，可以说出当前执行到哪、报错或现象，我可以按步骤帮你排查。
