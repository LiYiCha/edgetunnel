# edgetunnel 项目分析记录

> 分析时间：2026-02-18  
> 项目地址：https://github.com/cmliu/edgetunnel  
> 作者：cmliu

---

## 一、项目概述

**edgetunnel** 是一个运行在 **Cloudflare Workers / Pages** 边缘计算平台上的网络隧道代理工具。它利用 CF 的全球边缘节点作为中转，将客户端的流量通过 WebSocket 协议传输，实现代理穿透。

核心文件只有一个：`_worker.js`（约 2376 行），部署到 CF Workers 或 Pages 后即可运行，无需服务器。

---

## 二、核心原理

### 2.1 整体架构

```
客户端 (v2rayN / Clash / Sing-box 等)
    │
    │  VLESS / Trojan over WebSocket (TLS)
    ▼
Cloudflare Edge Node（_worker.js 运行于此）
    │
    │  TCP 直连 或 通过 ProxyIP / SOCKS5 / HTTP 代理转发
    ▼
目标服务器（任意互联网地址）
```

### 2.2 请求处理流程

Worker 的入口是 `fetch(request, env, ctx)` 函数，根据请求类型分两条路：

#### 路径 A：普通 HTTP 请求（管理/订阅）
- 无 `Upgrade: websocket` 头时进入此路径
- 路由逻辑（按 URL 路径分发）：
  - `/login` → 登录页面（密码验证，设置 Cookie）
  - `/admin` → 管理后台（需 Cookie 认证）
  - `/sub` → 订阅内容生成（需 Token 验证）
  - `/logout` → 清除 Cookie
  - `/locations` → 返回 CF 数据中心列表
  - 其他 → 返回伪装页面（nginx 假页 或 反代指定 URL）

#### 路径 B：WebSocket 请求（实际代理流量）
- 有 `Upgrade: websocket` 头时进入此路径
- 调用 `处理WS请求()` 函数建立 WebSocket 隧道

### 2.3 WebSocket 隧道核心流程

```
1. 建立 WebSocket 连接（客户端 ↔ Worker）
2. 读取客户端发来的第一个数据包
3. 协议识别：
   - 检测是否为 Trojan 协议（判断第56、57字节是否为 0x0D 0x0A）
   - 否则按 VLESS 协议解析
4. 解析目标地址（hostname + port）
5. 建立到目标的 TCP 连接（直连 or 反代）
6. 双向数据转发（客户端 ↔ Worker ↔ 目标服务器）
```

---

## 三、支持的协议

| 协议 | 说明 |
|------|------|
| **VLESS** | 主要协议，解析函数 `解析魏烈思请求()` |
| **Trojan** | 备用协议，解析函数 `解析木马请求()`，用 SHA-224 验证密码 |

两种协议都通过 WebSocket 传输（`ws` 或 `wss`），外层 TLS 由 Cloudflare 提供。

---

## 四、身份认证机制

### UUID / 密码生成
- 管理员密码来自环境变量 `ADMIN`（或 `PASSWORD`、`TOKEN` 等）
- UUID 生成逻辑：
  ```
  userIDMD5 = MD5MD5(管理员密码 + 加密秘钥KEY)
  UUID = 从 MD5 结果中截取并拼接成标准 UUIDv4 格式
  ```
- 若环境变量中直接设置了 `UUID`，则优先使用

### 管理后台认证
- 登录时验证密码，成功后设置 Cookie：
  ```
  auth = MD5MD5(User-Agent + KEY + 管理员密码)
  ```
- Cookie 有效期 24 小时，绑定 UA，防止 Cookie 被盗用

### 订阅 Token
- 订阅路径 `/sub` 需要 Token 参数：
  ```
  订阅TOKEN = MD5MD5(hostname + userID)
  ```

---

## 五、TCP 转发与反代机制

### 直连模式
直接用 CF 的 `connect()` API 连接目标服务器（仅限非 CF 托管的 IP）。

### ProxyIP 反代模式（默认）
- CF Workers 无法直接连接 CF 自己的 IP（如 Cloudflare CDN 节点）
- 通过 `PROXYIP` 环境变量指定一个中转 IP（ProxyIP）
- 未配置时默认使用 `{CF数据中心代码}.PrOxYIp.CmLiUsSsS.nEt` 动态解析
- 支持多个 ProxyIP，随机选择并有轮询兜底机制

### SOCKS5 / HTTP 代理模式
- 通过路径参数或环境变量配置
- 支持带认证的 SOCKS5 和 HTTP CONNECT 代理
- 可配置白名单（`GO2SOCKS5`），只对特定域名走 SOCKS5

### 连接优先级
```
1. 若目标在 SOCKS5 白名单 或 启用全局 SOCKS5 → 走 SOCKS5/HTTP 代理
2. 否则先尝试直连目标
3. 直连失败（目标是 CF IP）→ 自动切换到 ProxyIP 反代
```

---

## 六、订阅系统

### 订阅类型自动识别
根据请求的 `User-Agent` 或 URL 参数自动判断：
- `clash` / `meta` / `mihomo` → Clash 格式
- `singbox` / `sing-box` → Sing-box 格式
- `surge` → Surge 格式
- `quantumult` → QuanX 格式
- 其他 → mixed（Base64 编码的通用格式）

### 订阅内容生成
- **本地生成（默认）**：从 KV 存储的 `ADD.txt` 或随机生成 CF IP，拼接成 VLESS/Trojan 链接
- **优选订阅生成器**：从外部 API 获取优选 IP 列表
- **订阅转换**：调用外部 SubConverter API 转换为 Clash/Surge 等格式

### 订阅热补丁
对从外部获取的订阅内容进行修改：
- `Clash订阅配置文件热补丁()`：修改 DNS 配置、添加 ECH 支持、替换 UUID
- `Singbox订阅配置文件热补丁()`：修改出站配置、处理规则集
- `Surge订阅配置文件热补丁()`：修改 Surge 配置格式

---

## 七、ECH（加密客户端 Hello）支持

- 通过 DoH（DNS over HTTPS）查询目标域名的 HTTPS 记录获取 ECH 配置
- 支持 Clash Meta 和 Sing-box 的 ECH 参数注入
- 可增强抗检测能力，隐藏 SNI 信息

---

## 八、KV 存储结构

| Key | 内容 |
|-----|------|
| `config.json` | 主配置（UUID、HOST、传输协议、订阅设置等） |
| `cf.json` | Cloudflare API 凭证（用于查询用量） |
| `tg.json` | Telegram Bot 配置（用于访问日志推送） |
| `ADD.txt` | 自定义优选 IP 列表 |
| `log.json` | 访问日志记录 |

---

## 九、环境变量

| 变量名 | 必填 | 说明 |
|--------|:----:|------|
| `ADMIN` | ✅ | 管理后台登录密码（同时用于生成 UUID） |
| `KEY` | ❌ | 加密密钥 + 快速订阅路径 |
| `UUID` | ❌ | 强制指定 UUID（标准 UUIDv4 格式） |
| `PROXYIP` | ❌ | 自定义反代 IP（支持多个，逗号分隔） |
| `URL` | ❌ | 伪装主页地址（填 `nginx` 或 `1101` 显示假页） |
| `GO2SOCKS5` | ❌ | 强制走 SOCKS5 的域名白名单 |

---

## 十、部署方式

### CF Workers
1. 创建 Worker，粘贴 `_worker.js` 内容
2. 添加环境变量 `ADMIN`（密码）
3. 绑定 KV 命名空间（变量名 `KV`）
4. 绑定自定义域名（需在 CF 托管的域名）
5. 访问 `https://你的域名/admin` 登录后台

### CF Pages（推荐）
1. 上传 `main.zip`（包含 `_worker.js`）到 CF Pages
2. 设置环境变量 `ADMIN`
3. 绑定 KV 命名空间
4. 设置 CNAME 自定义域名

---

## 十一、安全机制

- **伪装**：未配置 ADMIN 时返回 404 静态页，不暴露功能
- **防扫描**：`/robots.txt` 返回 `Disallow: /`
- **速度测试拦截**：屏蔽 `speed.cloudflare.com` 的代理请求，防止被 CF 检测
- **Cookie 绑定 UA**：防止 Cookie 被其他设备复用
- **代码混淆**：函数名使用中文，关键字符串用 `atob()` 编码，注释中大量插入多语言"无害声明"以干扰 AI 代码审查

---

## 十二、关键函数索引

| 函数名 | 功能 |
|--------|------|
| `fetch()` | 主入口，路由分发 |
| `处理WS请求()` | WebSocket 隧道建立 |
| `解析魏烈思请求()` | 解析 VLESS 协议头 |
| `解析木马请求()` | 解析 Trojan 协议头 |
| `forwardataTCP()` | TCP 流量转发（含反代逻辑） |
| `forwardataudp()` | UDP/DNS 转发（转发到 8.8.4.4:53） |
| `connectStreams()` | 双向数据流桥接 |
| `socks5Connect()` | SOCKS5 代理连接 |
| `httpConnect()` | HTTP CONNECT 代理连接 |
| `读取config_JSON()` | 从 KV 读取/初始化配置 |
| `Clash订阅配置文件热补丁()` | Clash 订阅内容修改 |
| `Singbox订阅配置文件热补丁()` | Sing-box 订阅内容修改 |
| `请求优选API()` | 从外部 API 获取优选 IP |
| `DoH查询()` | DNS over HTTPS 查询 |
| `getECH()` | 获取 ECH 配置 |
| `MD5MD5()` | 双重 MD5 哈希（用于认证） |
| `sha224()` | SHA-224 哈希（Trojan 密码验证） |
