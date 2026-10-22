# Cloudflare Tunnel 管理面板

一个单文件 Cloudflare Worker，提供网页管理面板，用于管理账户下的 Cloudflare Tunnel（原名 Argo Tunnel）：新建/编辑/删除隧道、查看连接 Token、增删域名路由、查看在线状态，并可选地把隧道上线/离线事件通过 Telegram 推送提醒。

整个项目只有 **一个文件**：`_worker.js`（Worker 代码 + 内嵌管理页面，无需构建、无外部依赖）。

## 功能一览

- **密码登录**：访问面板需输入密码，会话 Cookie 签名有效期 7 天，登录接口有防爆破延时。
- **隧道管理**：列表展示所有隧道及状态（在线/降级/离线/未启用）、搜索过滤、查看启动 Token、修改名称与本地服务地址、删除隧道。
- **新建隧道**：选择域名 (Zone)、子域名、可选路径、本地服务地址（如 `http://localhost:8080`、`tcp://localhost:22`），自动创建隧道并自动配置 DNS CNAME 记录。
- **多路由管理**：在已有隧道上增删"域名 + 路径 → 服务"的 ingress 规则（每条隧道最多 60 条），自动处理 DNS，冲突时拒绝写入。
- **Telegram 上下线提醒**：状态**发生变化**才推送（上线一次、离线一次），可按隧道逐个开关（**默认关闭**）；支持多个接收者（私聊/群组/频道）、暂停/恢复推送。
- **PWA**：可安装到桌面/手机主屏，带离线壳页面。

## 快速部署

1. 登录 [Cloudflare Dashboard](https://dash.cloudflare.com) → **Workers & Pages** → **Create** → 创建 Worker，把 `_worker.js` 内容全部粘贴进去，保存。
2. **【非必须】绑定 KV 命名空间**：Worker → Settings → Bindings → Add → KV Namespace，变量名填 `TUNNEL_STATE`（没有它无法保存提醒配置、接收者等信息）。
3. **【必须】配置环境变量**：Settings → Variables and Secrets，添加 `CF_API_TOKEN`、`CF_ACCOUNT_ID`、`PASSWORD` 其他变量非必须（详见下表）。
4. 部署后打开 `https://<Worker 域名>`，输入密码登录即可。

## 环境变量

| 变量 | 必填 | 说明 |
| --- | --- | --- |
| `CF_ACCOUNT_ID` | ✅ | Cloudflare 账户 ID,创建token时会显示 |
| `CF_API_TOKEN` | ✅ | API 令牌 (首页左侧菜单 → 管理账户 → 帐户 API 令牌); 权限：Account → Argo Tunnel → **Read & Edit**；Account DNS Settings → **Read & Edit**；DNS View → **Read & Edit**；Registrar Sandbox Domains → **Read & Admin** |
| `PASSWORD` | ✅ | 面板登录密码，至少 6 位 |
| `SESSION_SECRET` | 可选 | 会话签名密钥，一串随机字符。不设置时由密码派生（改密码会使所有人重新登录） |
| `TUNNEL_STATE` | 可选 | **KV 命名空间绑定**（Bindings，不是环境变量），存储登录外的全部状态与提醒配置 |
| `DEFAULT_SERVICE` | 可选 | 新建/编辑隧道时的默认本地服务地址，如 `http://localhost:8080` |
| `TG_BOT_TOKEN` | 可选 | Telegram 机器人 Token（环境变量兜底；也可在面板里保存到 KV） |
| `TG_CHAT_ID` | 可选 | 默认接收者 chat_id（环境变量兜底） |

## Telegram 提醒配置(可选)

1. 在 Telegram 找 [@BotFather](https://t.me/BotFather)，`/newbot` 创建机器人，拿到 **Bot Token**。
2. **想让谁收提醒，就让谁先给机器人发一条消息（或 `/start`）**——机器人无法主动私聊从未互动过的用户。
3. 获取接收者的 chat_id：
   - **个人私聊**：给 [@userinfobot](https://t.me/userinfobot) 发任意消息，它会回复你的**正数**数字 ID；
   - **群组**：把机器人拉进群，群的 ID 为负数（超级群组通常 `-100` 开头）；
   - **公开频道/群组**：直接填 `@用户名` 也可以，面板会解析成真实 chat_id 存入 KV。
4. 打开面板 → **Telegram 配置**：填入 chat_id（接收者，**不是机器人的 ID**）和 Bot Token，点"**测试发送**"确认可达，再点"**保存到 KV**"。
5. 回到 **Tunnel 管理**，把需要监控的隧道的"🔕 提醒关"点成"🔔 提醒开"。

> ⚠️ 常见误区：chat_id 是"谁来**接收**消息"（你的用户 ID / 群 ID），Bot Token 才是机器人的。填机器人自己的 ID 会提示"机器人不能给自己发消息"。

### 提醒机制

- **默认关闭**：每条隧道需手动开启，开启状态保存在 KV。
- 只在状态**变化**时发送一次（上线一次、离线一次），同一状态不重复提醒；首次记录到状态时不发送，部署后不会刷屏。
- 检查时机：每次打开/刷新面板时检查一次，面板连续刷新最少间隔 20 秒（防止把网络抖动看成多次上下线）。
- **无人值守检查**：给 Worker 添加 Cron 触发器（如 `*/5 * * * *`），代码中的 `scheduled` 处理会自动执行同样的检查，无需有人打开面板。
- **暂停推送**：面板可暂停（限时或无限期），期间状态照常记录，只是不发消息。

## API 接口概览

除 `/api/login` 外，所有接口需登录（Cookie 会话）。均为 `POST` + JSON：

| 路径 | 说明 |
| --- | --- |
| `/api/login` `/api/logout` | 登录 / 退出 |
| `/api/init` | 域名 (Zone) 列表、默认服务地址 |
| `/api/list` | 隧道列表 + 在线状态 + 提醒状态（同时触发一次检查） |
| `/api/create` `/api/update` `/api/delete` | 新建 / 编辑 / 删除隧道 |
| `/api/token` | 获取隧道启动 Token（cloudflared 用） |
| `/api/detail` `/api/route` | 查看 / 增删隧道 ingress 路由 |
| `/api/alert` | 隧道提醒开关 (toggle) / 吞掉一次变化 (skip) |
| `/api/tg` | Telegram 配置：保存、测试、增删接收者、暂停/恢复 |

## 常见问题

- **"KV 命名空间 TUNNEL_STATE"**：在 Worker → Settings → Bindings 里绑定 KV，变量名必须为 `TUNNEL_STATE`。
- **"找不到该 chat_id"**：私聊接收者没给机器人发过 `/start`；或把机器人 ID/正数当成了群 ID 填写。个人填正数用户 ID，群组填负数 ID。
- **"机器人 Token 无效"**：检查 Bot Token 是否复制完整、机器人是否被 BotFather 停用。
- **"隧道仍有活跃连接,请先停止 cloudflared"**：删除前需先在目标机器上停止 `cloudflared` 进程（Cloudflare 平台限制）。
- **"该域名已存在其他 DNS 记录"**：先到 Zone 的 DNS 页面删除冲突记录，或换个子域名。
- **改密码后所有人掉线**：未设置 `SESSION_SECRET` 时会话密钥由密码派生；设置独立的 `SESSION_SECRET` 可避免。
- **之前是"默认开提醒"的旧版本，现在想全部关**：新版已改为默认关，并带一次性迁移，打开面板后旧数据会自动清成关闭状态。

## 安全说明

- 所有 Cloudflare API 调用都在 Worker 服务端完成，Token 不会下发到浏览器。
- 隧道启动 Token（可冒充隧道接管流量）仅在登录后按需单独获取。
- 登录密码比较使用固定时间算法，防止时序侧信道；连续失败有延时惩罚。
- 建议同时设置 `SESSION_SECRET`，并给 Worker 域名加上 Cloudflare Access 作为第二层防护（可选）。
