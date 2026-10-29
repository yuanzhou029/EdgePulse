# EdgePulse · 分布式边缘遥测与多协议中继网关

[![Deploy to Cloudflare Workers](https://img.shields.io/badge/Deploy-Cloudflare_Workers-F38020?logo=cloudflare&logoColor=white)](https://workers.cloudflare.com/)
[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)
[![Platform](https://img.shields.io/badge/Platform-Node.js_/_V8-green.svg)]()

EdgePulse 是一个运行于 Cloudflare Workers 边缘计算无服务器环境的分布式网络遥测网关与轻量级协议桥接系统。

系统通过后台全量探测全球边缘中继节点的可用性与真实网络延迟，经由健康检查与运营商自治域（ASN）特征推断，将存活节点持久化至 Cloudflare KV 分布式数据库。同时，底层搭载了纯 JavaScript 编写的 SSTP 客户端状态机协议栈，允许客户端以标准的 VLESS 协议接入，并在边缘自动转换为 SSTP 隧道访问目标网络。

---

## ✨ 核心特性

- **云原生全自动探测**：内置定时任务（Cron Trigger），无感全量抓取节点池并由 12 路并发校验链路健康状态。
- **纯 JS 实现 SSTP 客户端协议栈**：在无底层 Socket 操作系统的 V8 隔离环境中，纯手动实现 PPP 协商、PAP 鉴权、IPCP 寻址与合成 TCP/IP 报文流转。
- **KV 持久化与秒级直出**：客户端拉取订阅 100% 直读 Cloudflare KV，杜绝拉取时阻塞检测导致超时。
- **多客户端原生兼容**：
  - **通用订阅**：纯净 Base64 格式，兼容 v2rayN、Shadowrocket、Sing-box 等。
  - **Mihomo 订阅**：输出纯净的 `proxies` 节点对象 YAML，原生兼容 Mihomo、Clash Verge Rev 及 Sub-Store。
- **安全与防探测体系**：
  - 前台面板公开可视化，敏感操作（导出订阅、触发全量探测）引入弹窗 Token 校验。
  - 后端接口强制鉴权，防止节点池被外部扫描器批量盗取。
  - 静态敏感特征全量运行时脱敏，规避安全审计误报。

---

## 🏗️ 架构流转拓扑

```text
  [ 本地客户端 ] (v2rayN / Mihomo / Sub-Store)
        │
        │  VLESS-over-WebSocket (走 Cloudflare CDN 优选线路)
        ▼
  [ Cloudflare Worker ]
   ├── EdgePulse Web UI  -> 实时可视化节点健康状态
   ├── Token 鉴权中间件   -> 保护导出与更新接口
   ├── KV 存储中枢       -> 持久化保存所有通过测活的存活端点
   │
   └── [ 纯 JS SSTP 协议引擎 ] (在 Worker 内存中虚拟网卡)
             │
             │  MS-SSTP (PPP / PAP / IPCP / Synthetic TCP)
             ▼
      [ 目标网络出口节点 ] -> 最终访问目标互联网资源

```
# 🚀 快速上手
方式一：网页端一键粘贴部署 (适合新手)

登录 Cloudflare Dashboard，进入 Compute (Workers) -> Workers & Pages。

点击 Create Application -> Create Worker，输入名称（如 edge-pulse）并点击保存。

进入 Worker 管理界面，点击右上角 Edit code，将仓库中的 _worker.js 代码全选粘贴进去，点击 Deploy。

绑定 KV 存储：

在左侧菜单进入 Storage & Databases -> KV，点击 Create Namespace，命名为 EDGE_KV。

回到该 Worker 的 Settings -> Variables -> KV Namespace Bindings，添加绑定：

Variable name：KV

KV namespace：选择刚才创建的 EDGE_KV

设置私有密钥：

在 Settings -> Variables -> Environment Variables 中添加环境变量：

变量名：TOKEN

变量值：设置你的私有密码（例如 MySecurePass2026）

打开分配的域名即可进入控制面板。

# 方式二：使用 Wrangler CLI 命令行部署 (适合开发者)
```
# 1. 克隆代码仓库
git clone [https://github.com/your-username/EdgePulse.git](https://github.com/your-username/EdgePulse.git)
cd EdgePulse

# 2. 登录 Cloudflare
npx wrangler login

# 3. 创建 KV 命名空间
npx wrangler kv:namespace create "KV"
# 将输出的 id 替换到 wrangler.toml 中

# 4. 部署至 Worker
npx wrangler deploy
```
# ⚙️ 环境变量配置说明

| 变量名 | 必填 | 默认值 | 说明 |
| --- | --- | --- | --- |
| `UPSTREAM_SOURCES` | 否 | 包含 VPNGate 及多个 SOCKS5 优质源 | **统一多源配置**。多个源用逗号 `,` 或换行分隔，支持 `URL#最大抓取数` 语法限制单源抓取上限（如 `https://...#15`）。自动兼容识别 VPNGate CSV、JSON 订阅及纯代理列表。 |
| `PROBE_API_URL` | 否 | 内置默认演示探针 | **VPS 测活探针地址**。指向您的 VPS `probe-api` 服务地址（例如 `https://probe.yourdomain.com`）。 |
| `PROBE_TOKEN` | 否 | 空 (免密) | **探针访问密钥**。若 VPS 端启用了 `PROBE_TOKEN` 鉴权，在此填入对应口令；若 VPS 端未启用，此项留空即可。 |
| `MAX_ALIVE_NODES` | 否 | `20` | **最终存活节点入库上限**。测活完成后会按**延时升序**精选最优节点。<br>• 填单个数字：如 `20`，全局只保留延时最低的前 20 个。<br>• 填分协议配额：如 `socks5:10,sstp:10`，分别保留延时最低的各 10 个节点。 |
| `TOKEN` / `SUB_TOKEN`| 建议 | 空 (公开) | 管理凭据。配置后，导出订阅和手动重新测活必须输入此密码。 |
| `UUID` | 否 | 内置默认 UUID | 客户端连接 VLESS 节点所用的唯一身份识别码。 |
| `EDGE_HOSTS` | 否 | 内置优选域名列表 | 自定义 CDN 入口优选域名，多个域名用英文逗号 `,` 分隔。 |
| `EDGE_HOSTS_URL` | 否 | 空 | 远程优选域名 txt 文件的拉取地址（优先级高于 `EDGE_HOSTS`）。 |

### 内部安全硬编码指标
- `HARD_MAX_PROBE_TARGETS = 2000`：单次进入测活的候选节点最大上限，超量自动截断以保护 VPS 资源。
- `CHUNK_SIZE = 100`：发往 VPS 探针批量接口时按 100 节点/批切片分批提交，避免触发探针 `BATCH_LIMIT = 200` 拦截错误与 Worker fetch 超时。
- `MAX_RETRY_ROUNDS = 3`：成活为 0 时自动间隔 5 秒重试 3 次；3 次均为空坚决不覆盖 KV 数据库，保留历史节点兜底。

---

## 📋 订阅导入指南
访问你的 Worker 域名，在页面导航栏中可一键复制：

1. **通用订阅 (Base64)**
   - 链接格式：`https://你的域名/sub?token=你的Token`
   - 适用于：v2rayN、Shadowrocket、Sing-box 等。浏览器直接打开将直接展示 Base64 文本，不强制弹出下载窗口。

2. **Mihomo 订阅 (YAML 纯节点)**
   - 链接格式：`https://你的域名/sub?target=mihomo&token=你的Token`
   - 适用于：Mihomo、Clash Verge Rev、Mihomo Party、Sub-Store。输出符合 Mihomo 规范的标准 proxies 列表。

3. **🔥 Mihomo 链式落地专用订阅 (dialer-proxy)**
   - 链接格式：`https://你的域名/sub?target=mihomo&token=你的Token&dialer_proxy=🛫%20链式前置`
   - 适用于：将本系统节点作为**落地出口**，并自动在节点配置中注入 `dialer-proxy: "🛫 链式前置"`，直接在 Mihomo 的 `proxy-providers` 中作为落地使用，无需手动写预处理脚本。

4. **原生节点直连列表**
   - 链接格式：`https://你的域名/nodes.txt?token=你的Token`
   - 包含未封装的原生节点连接串，可直接查阅。

> 📖 **更详细的部署步骤、架构拓扑与 VPS 探针安装教程，请查阅完整文档：[DEPLOYMENT_GUIDE.md](DEPLOYMENT_GUIDE.md)**

