# EdgePulse · 分布式边缘遥测与多协议中继网关 (全新架构手册)

[![Deploy to Cloudflare Workers](https://img.shields.io/badge/Deploy-Cloudflare_Workers-F38020?logo=cloudflare&logoColor=white)](https://workers.cloudflare.com/)
[![Node.js](https://img.shields.io/badge/Runtime-Node.js_18+-green.svg)]()
[![Platform](https://img.shields.io/badge/Platform-Cloudflare_Pages_/_Workers-blue.svg)]()
[![Mihomo](https://img.shields.io/badge/Support-Mihomo_Dialer--Proxy-purple.svg)]()

EdgePulse 是一套运行于 **Cloudflare Workers / Pages** 边缘计算环境的分布式网络遥测网关与中继系统。

本系统采用了**控制面与测活数据面解耦**的现代化架构：
1. **Cloudflare Worker (控制中枢与调度器)**：负责多上游源聚合拉取、规则清洗与去重、分片调度、低延迟节点优选入库，以及提供 Web 仪表盘和多协议订阅分发（支持通用 Base64、Mihomo YAML 以及 Mihomo 链式代理落地专用订阅）。
2. **VPS 独立探针 (测活数据面)**：通过纯 Node.js 标准库编写的独立探测引擎（`probe-api`），对上游候选节点执行高精度真实出网体检（解析真实出口 IP、国家代码、ASN 归属、家宽/机房属性及真实握手延迟），彻底卸载 Worker 内部繁重的 TCP/TLS 握手，杜绝 CPU 超限与超时。

---

## 🏗️ 架构拓扑图

```text
                  [ 客户端 (Mihomo / Clash / v2rayN / Sing-box) ]
                                      │
            ┌─────────────────────────┴────────────────────────┐
            │ (标准 VLESS-over-WebSocket / 链式 dialer-proxy)  │
            ▼                                                  ▼
   [ Cloudflare Worker ]                            [ Mihomo 链式落地出口 ]
   ├── Web 仪表盘 UI                                  (绑定: 🛫 链式前置)
   ├── /sub 多格式订阅引擎
   │     ├─ Base64 通用订阅
   │     ├─ Mihomo proxies 列表
   │     └─ 链式落地专用订阅 (dialer-proxy)
   │
   ├── 多源抓取引擎 (UPSTREAM_SOURCES)
   │     ├─ VPNGate CSV 源
   │     ├─ 用户私有订阅源
   │     └─ 公共开放 SOCKS5 代理源
   │
   ├── 大数据切片分批调度 (每批 100 节点)
   │     │  POST /check/batch (切片流转)
   │     ▼
   │  [ VPS 探针引擎 (probe-api) ]
   │     ├── SOCKS5 隧道验证 (带/无认证)
   │     ├── HTTP / HTTPS CONNECT 验证
   │     ├── SSTP 原生握手与合成 TCP 验证
   │     └── 真实出口探测 (iplocate/ipify)
   │
   └── 优选入库与数据保护 (Cloudflare KV)
         ├─ 延时升序排序 (Low Latency First)
         ├─ 优选截断 (MAX_ALIVE_NODES)
         └─ 3次重试保护与历史数据兜底
```

---

## ⚙️ 环境变量与参数全览

所有配置项均可在 Cloudflare Dashboard 的 **Settings -> Variables and Secrets**（Workers）或 **Settings -> Environment variables**（Pages）中进行配置：

| 变量名 | 类型 | 必填 | 默认值 | 详细说明 |
|---|:---:|:---:|---|---|
| `UPSTREAM_SOURCES` | 文本 | 否 | 包含 VPNGate 及多个 SOCKS5 优质源 | **统一多源配置**。多个源用逗号 `,` 或换行分隔，支持 `URL#最大抓取数` 语法限制单源抓取上限（如 `https://...#15`）。自动兼容识别 VPNGate CSV、JSON 订阅及纯代理列表。 |
| `PROBE_API_URL` | 文本 | 否 | 内置默认演示探针 | **VPS 测活探针地址**。指向您的 VPS `probe-api` 服务地址（例如 `https://probe.yourdomain.com` 或带端口的 IP）。 |
| `PROBE_TOKEN` | 文本 | 否 | 空 (免密) | **探针访问密钥**。若 VPS 端启用了 `PROBE_TOKEN` 鉴权，在此填入对应口令；若 VPS 端未启用，此项留空即可。 |
| `MAX_ALIVE_NODES` | 文本/数值 | 否 | `20` | **最终存活节点入库上限**。测活完成后会按**延时升序**精选最优节点。<br>• 填单个数字：如 `20`，全局只保留延时最低的前 20 个。<br>• 填分协议配额：如 `socks5:10,sstp:10`，分别保留延时最低的各 10 个节点。 |
| `TOKEN` / `SUB_TOKEN`| 文本 | 建议 | 空 (公开) | **控制面板与订阅管理密码**。配置后，打开面板的敏感操作、导出订阅、更新接口强制校验此 Token。 |
| `UUID` | 文本 | 否 | `d342d11e...` | 客户端连接 VLESS 节点所用的全局唯一用户 ID。建议设置为自己的专属 UUID。 |
| `EDGE_HOSTS` | 文本 | 否 | 内置 8 个优选 CDN 域名 | **优选 CDN 接入点列表**。多个域名/IP 用逗号隔开，客户端订阅将通过这些入口分发加速。 |
| `EDGE_HOSTS_URL` | 文本 | 否 | 空 | 远程优选域名/IP 的 txt 拉取地址（配置后优先级高于 `EDGE_HOSTS`）。 |

### 内部安全硬编码指标（系统自律守护）
- `HARD_MAX_PROBE_TARGETS = 2000`：无论上游抓取到多少候选节点，进入测活前单次最多只取 2000 个，超量自动截断，防止大列表挤垮 VPS 资源。
- `CHUNK_SIZE = 100`：向 VPS 探针投递任务时，严格按 100 个节点为一组进行切片分批提交，避免触发探针 `BATCH_LIMIT = 200` 错误，防止 Cloudflare Worker 外部子请求超时。
- `MAX_RETRY_ROUNDS = 3`：若测活返回存活为 0，系统自动间隔 5 秒重试，上限 3 次；若 3 次均为空，坚决不覆盖 KV 数据库，保留历史节点兜底。

---

## 🚀 完整部署实战指南

系统分为 **Cloudflare Worker 边缘端** 与 **VPS 测活探针端** 两部分。

### 第一部分：部署 Cloudflare Worker (控制端)

#### 推荐方式：Cloudflare Pages 部署 (最稳定)
1. 登录 [Cloudflare Dashboard](https://dash.cloudflare.com/)，在左侧导航栏点击 **Workers & Pages** -> **Create application** -> **Pages**。
2. 选择 **Upload assets**（或绑定 GitHub 仓库自动化部署）：
   - 将本项目 `kky/EdgePulse/public/` 下的 `_worker.js` 准备好。
   - 上传或推送到代码仓库的根目录。
3. **创建与绑定 KV 数据库**：
   - 进入 **Storage & Databases** -> **KV**，点击 **Create namespace**，命名为 `EDGE_KV`。
   - 回到刚才创建的 Pages 项目 -> **Settings** -> **Functions** -> **KV namespace bindings**。
   - 点击 **Add binding**：
     - **Variable name**：填 `KV`
     - **KV namespace**：选择刚才创建的 `EDGE_KV`
4. **添加环境变量**：
   - 进入 **Settings** -> **Environment variables**，点击 **Add variable**，按需填入上文表格中的配置（如 `TOKEN`、`PROBE_API_URL`、`MAX_ALIVE_NODES` 等）。
5. 保存并重新部署，访问分配给您的 Pages 域名即可进入仪表盘。

---

### 第二部分：部署 VPS 测活探针 (probe-api)

> 探针基于纯 Node.js 标准库实现，零第三方 `npm` 依赖，极低内存消耗（运行仅占用约 20MB 内存）。

#### 1. VPS 环境准备
- 操作系统：Debian 11/12/13、Ubuntu 20.04/22.04+、CentOS/Rocky 8+
- Node.js 版本：`>= 18.0.0`
- 检查命令：
  ```bash
  node -v
  command -v node # 记录 node 的实际绝对路径，例如 /usr/local/bin/node 或 /usr/bin/node
  ```

#### 2. 解压与放置服务物料
```bash
# 1. 创建服务专用目录
sudo mkdir -p /opt/probe-api

# 2. 上传并解压 probe-api.zip 到目标目录
sudo unzip -q -o probe-api.zip -d /opt/probe-api

# 3. 查看物料结构 (应该直接包含 src/、deploy/、package.json 等)
ls -la /opt/probe-api
```

#### 3. 配置系统服务与环境变量
```bash
# 1. 复制环境变量配置文件
sudo cp /opt/probe-api/deploy/probe-api.env.example /etc/probe-api.env
sudo chmod 600 /etc/probe-api.env

# 2. 编辑配置文件 (设置端口与口令)
sudo nano /etc/probe-api.env
```
`/etc/probe-api.env` 内容示例：
```ini
PORT=8787
HOST=0.0.0.0
# 若要启用访问口令，填写下面的口令；若留空则为免密公开模式
PROBE_TOKEN=your_secure_password_here
# 并发线程数与超时控制
CONCURRENCY=20
CHECK_TIMEOUT_MS=12000
```

```bash
# 3. 注册 systemd 守护进程
sudo cp /opt/probe-api/deploy/probe-api.service /etc/systemd/system/

# 如果您的 node 路径不是 /usr/bin/node (如 /usr/local/bin/node)，请修改 service 文件中的 ExecStart 路径：
sudo nano /etc/systemd/system/probe-api.service

# 4. 启动并设置开机自启
sudo systemctl daemon-reload
sudo systemctl enable --now probe-api

# 5. 检查运行状态
sudo systemctl status probe-api
```

#### 4. 探针连通性自测
在 VPS 本地或任意机器上执行验证：
```bash
curl -i http://127.0.0.1:8787/health
# 正常应返回: {"ok":true,"service":"probe-api","version":"1.0.0",...}
```

---

## 📋 客户端订阅接入指南

访问您的 Worker / Pages 域名，在页面导航栏中即可直观获取各类订阅链接：

### 1. 通用 Base64 订阅 (支持 v2rayN / Shadowrocket / Sing-box)
- **URL 格式**：
  ```text
  https://你的域名/sub?token=你的Token
  ```
- **特点**：直接返回经过安全 Base64 编码的标准 VLESS 节点列表，浏览器直接访问不弹窗下载，粘贴至各类客户端即可直接更新。

---

### 2. Mihomo / Clash Verge 订阅 (标准直连模式)
- **URL 格式**：
  ```text
  https://你的域名/sub?target=mihomo&token=你的Token
  ```
- **特点**：输出纯净的 `proxies` 节点对象 YAML 格式，原生兼容 Mihomo、Clash Verge Rev、ShellCrash 等。

---

### 3. 🔥 Mihomo 链式代理专用落地订阅 (dialer-proxy 模式)
> **应用场景**：当您需要将本系统输出的全球住宅/中继节点作为**落地出口**，并通过您自己的高速节点/策略组（如香港、新加坡专线）作为前置跳板时使用。

- **URL 格式**：
  ```text
  https://你的域名/sub?target=mihomo&token=你的Token&dialer_proxy=🛫%20链式前置
  ```
- **使用方法**：
  1. 在 EdgePulse 面板首页点击 **`🔗 复制落地订阅 (Mihomo 链式)`** 按钮。
  2. 弹窗中输入您在 Mihomo 配置中所定义的前置代理名称（默认预填 `🛫 链式前置`）。
  3. 系统将自动生成带 `&dialer_proxy=...` 的专属订阅链接并复制到剪贴板。
- **Mihomo 客户端输出效果 (自动附加 dialer-proxy)**：
  ```yaml
  proxies:
  - {"type":"vless","name":"日本-专线-01","server":"cf-in.com","port":443,...,"dialer-proxy":"🛫 链式前置"}
  - {"type":"vless","name":"韩国-边缘-02","server":"cf-in.com","port":443,...,"dialer-proxy":"🛫 链式前置"}
  ```
- **在 Mihomo 配置中的配置示例**：
  ```yaml
  # 1. 定义前置节点/策略组
  proxy-groups:
    - name: "🛫 链式前置"
      type: select
      proxies:
        - "香港高速专线 01"
        - "新加坡专线 02"

    - name: "🎯 落地出口选择"
      type: select
      use:
        - edgepulse_nodes # 自动使用下方拉取下来的落地节点

  # 2. 引入 EdgePulse 链式订阅
  proxy-providers:
    edgepulse_nodes:
      type: http
      url: "https://你的域名/sub?target=mihomo&token=你的Token&dialer_proxy=🛫%20链式前置"
      path: ./profiles/proxies/edgepulse.yaml
      interval: 3600
      health-check:
        enable: true
        url: http://www.gstatic.com/generate_204
        interval: 300
  ```

---

## 🛠️ 运维与更新接口

| 接口路径 | 方法 | 说明 |
|---|:---:|---|
| `/` | `GET` | 节点监控仪表盘，可视化展示各节点国家、延迟、运营商类型及出口 IP。 |
| `/sub` | `GET` | 多格式订阅导出接口（支持 `target=mihomo`、`dialer_proxy=...`）。 |
| `/update` | `GET` / `POST` | 手动触发全量重新拉取与 VPS 测活（需携带 `?token=...`）。 |
| `/health` | `GET` | Worker 存活性健康检测，返回当前配置概览与版本状态。 |
| `/nodes.txt` | `GET` | 导出原始纯文本代理节点（SSTP/SOCKS5 直连格式）。 |

---

## 🛡️ 安全合规与最佳实践

1. **强密码建议**：请务必在 Worker 环境变量中设置 `TOKEN`，避免您的边缘中继被网络爬虫扫描与未授权消耗。
2. **VPS 探针安全**：建议在 VPS 探针设置 `PROBE_TOKEN`，并通过云厂商安全组仅放行必要端口，或通过反向代理（如 Nginx / Cloudflare Tunnel / InstaBox）为探针配置 HTTPS 加密与访问控制。
3. **低延迟节点配额**：建议保持 `MAX_ALIVE_NODES=20` 左右，既能保证订阅内节点的高可用与低延迟，又能大幅提升客户端规则加载与分流测速的性能。
