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
