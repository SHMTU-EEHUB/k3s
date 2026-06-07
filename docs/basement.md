# Basement

> 基础设施介绍。本文只说明 Longhorn、Tailscale 与 Mihomo / Clash 的作用、边界、使用原则和验收要点；当前硬件与运行状态以 `docs/machines.md` 为准，安装 / 恢复步骤以 `docs/init.md` 为准。当前 Tailscale 为主机级安装，不使用 Kubernetes Operator。

## 1. Longhorn

### 1.1 作用

Longhorn 是当前集群的 Kubernetes 分布式块存储方案，用于给工作负载提供 PVC。

它主要承担：

- 管理 HDD / NVMe 数据盘。
- 为服务提供持久化卷。
- 提供卷副本、重建、快照等能力。
- 在多节点上线后提供跨节点副本能力。

### 1.2 使用边界

- 当前只有单节点在线时，Longhorn 不具备跨节点容灾能力。
- 单节点阶段只应按单副本做功能验证。
- Worker 上线并加入集群后，再逐步提高关键卷副本数。
- Longhorn 副本不是应用级备份，重要数据仍需独立备份策略。
- 重要数据应具备外部备份与恢复验证。

### 1.3 存储分层

| 存储层      | 用途                                              |
| ----------- | ------------------------------------------------- |
| HDD 数据层  | 通用持久化数据、较大容量数据、常规 PVC            |
| NVMe 高速层 | 对 I/O 更敏感，且可接受较低副本或可重建的数据负载 |

实际磁盘来源、容量和在线状态见 `docs/machines.md`。

### 1.4 推荐 StorageClass

| StorageClass             | 副本 | 介质       | 用途                 |
| ------------------------ | ---: | ---------- | -------------------- |
| `longhorn-hdd-3replica`  |    3 | HDD        | 关键持久化服务       |
| `longhorn-hdd-2replica`  |    2 | HDD        | 大容量普通持久化数据 |
| `longhorn-fast-1replica` |    1 | NVMe / SSD | 缓存、高速临时数据   |

单节点阶段可先使用单副本验证，Worker 上线后再按服务重要性调整。

### 1.5 节点基础要求

每个参与 Longhorn 的 Alpine 节点需要满足：

| 项目           | 要求                                                |
| -------------- | --------------------------------------------------- |
| K3s 角色       | 以 K3s Server 或 Agent 方式加入当前集群             |
| 存储依赖       | 安装 iSCSI / NFS / 文件系统工具等 Longhorn 运行依赖 |
| iSCSI 服务     | `iscsid` 需要启用并运行                             |
| iSCSI 内核模块 | 支持 `iscsi_tcp` 等模块                             |
| 挂载传播       | 根挂载具备 Kubernetes CSI 所需的共享挂载传播能力    |
| 数据盘挂载     | HDD / NVMe 挂载到固定路径后再在 Longhorn 中添加     |
| 数据盘 shared  | Longhorn 数据盘目录本身不需要单独设置为 `shared`    |
| 数据盘配额     | Longhorn 不依赖底层 `prjquota` 来限制 PVC 容量      |

### 1.6 验收要点

- Longhorn UI 可访问。
- 节点磁盘状态 Online。
- StorageClass 能创建 PVC。
- 测试 Pod 能挂载 PVC 并读写。
- Worker 上线后，Longhorn 节点组件能自动派驻。
- 卷副本、重建和故障恢复行为符合当前阶段预期。

## 2. Mihomo / Clash

### 2.1 作用

Mihomo / Clash 是当前集群的集中式出站代理组件，用于给需要代理的工作负载提供受控外网访问。

它主要承担：

- 代理订阅与规则管理。
- HTTP / SOCKS / Mixed 出站代理。
- DNS 与 TUN 能力。
- 为指定 Pod 提供可控代理出口。

### 2.2 使用边界

- 当前方案运行在 K3s + Flannel 网络上。
- `proxy.enabled=true` 只是选择标签，不会自动让 Pod 走代理。
- Kubernetes NetworkPolicy 只能允许 / 拒绝流量，不能把流量改路由到 Mihomo。
- 如未来更换 CNI，需要重新验证代理路由方案。
- 多节点阶段应先验证跨节点透明路由可用性；不稳定时优先使用显式代理。

### 2.3 Kubernetes 形态

当前资源名称、镜像、端口和资源限制见 `docs/machines.md`。

总体形态：

| 资源                  | 作用                                                                            |
| --------------------- | ------------------------------------------------------------------------------- |
| SealedSecret / Secret | `secret-sealed.yaml` 解封为 `Secret/mihomo-config`，只读提供 `config.yaml`       |
| PersistentVolumeClaim | `mihomo-runtime` 挂载到 `/root/.config/mihomo`，持久化 provider、缓存与 geodata |
| Headless Service      | 暴露代理服务，供业务 Pod 发现 Mihomo Pod                                        |
| Deployment            | 运行单副本 Mihomo                                                               |

Mihomo Pod 需要 `NET_ADMIN` 与 `NET_RAW` capability。

`config.yaml` 保持 Secret / SealedSecret 管理，真实订阅链接和密钥不能提交到仓库。`proxy_provider/`、`rule_provider/`、`cache.db`、`geoip.dat`、`geoip.metadb`、`geosite.dat`，以及下载得到的 `.mrs` / `.mmdb` 运行数据由 `PersistentVolumeClaim/mihomo-runtime` 保存，不进入 Git。

`mihomo-runtime` 使用 Longhorn `longhorn-hdd-1replica`，容量 `2Gi`，访问模式 `ReadWriteOnce`。它能避免 Mihomo 重启后丢失 provider、规则、缓存和 geodata，但单副本 Longhorn 卷没有跨节点副本保护，节点或磁盘故障时仍有丢失风险。

### 2.4 业务接入方式

当前推荐两种接入方式：

| 方式                   | 适用场景                                            | 推荐程度 |
| ---------------------- | --------------------------------------------------- | -------- |
| 显式 HTTP / Mixed 代理 | 应用支持 `HTTP_PROXY` / `HTTPS_PROXY` / `ALL_PROXY` | 优先     |
| InitContainer 路由注入 | 应用不支持代理变量，且希望尽量透明接入              | 谨慎使用 |

业务 Pod 建议：

- 需要代理时添加 `proxy.enabled=true` 标签。
- 支持代理环境变量的应用优先使用显式代理。
- `NO_PROXY` 应包含本地回环、`.svc`、`.cluster.local`、Pod 网段、Service 网段和节点局域网网段。
- 不需要代理的普通 Pod 不添加代理配置，默认直连。

InitContainer 路由注入原则：

- InitContainer 需要 `NET_ADMIN` capability。
- 业务容器本身不需要 `NET_ADMIN`。
- InitContainer 解析集群内代理 Service，获取 Mihomo Pod IP。
- InitContainer 将当前 Pod 的默认路由替换为 Mihomo Pod IP。
- Mihomo Pod 侧需要启用 TUN 作为透明网关入口；TUN 只作用于 Mihomo Pod 的网络命名空间，不应使用 `hostNetwork`。
- 如果出现 `invalid gateway`、`network unreachable` 或代理不生效，改用显式代理、Sidecar，或宿主机策略路由方案。

### 2.5 Web UI / 控制器

Mihomo 控制器只应通过可信网络访问，必须设置控制器密钥。当前端口和资源名称见 `docs/machines.md`。

| 方式                   | 用途     | 说明                                     |
| ---------------------- | -------- | ---------------------------------------- |
| `kubectl port-forward` | 临时管理 | 最小暴露面，适合单人运维                 |
| Tailscale 内网访问     | 日常管理 | 只在可信 Tailnet 内暴露                  |
| NodePort               | 日常管理 | 仅限可信 LAN / Tailscale，不直接公网暴露 |

### 2.6 敏感配置

以下内容不得提交到仓库：

- 代理订阅链接。
- 代理节点密码、UUID、Token、私钥。
- Mihomo 控制器密钥。
- 临时 Bootstrap 外部代理地址。

应使用 Kubernetes Secret、私有配置或外部密码管理器管理。

### 2.7 验收要点

- Mihomo Pod 正常运行。
- 代理 Service Endpoints 能解析到 Mihomo Pod IP。
- 支持显式代理的测试 Pod 出口符合预期。
- 使用 InitContainer 路由注入的测试 Pod 默认路由符合预期。
- 普通 Pod 不受代理影响。
- 临时停止 Mihomo 后，需要代理的测试 Pod 不应静默直连外网。
- Tailscale 流量不被 Mihomo 误劫持。
- 常用外部目标按规则访问。

## 3. Tailscale

### 3.1 作用

Tailscale 是当前集群的可信远程访问与运维通道。

当前采用**主机级安装**方式：直接在 `master1` 上安装 Tailscale、广告 K3s 子网路由、作为所有服务管理入口和其它节点的运维跳板。

它主要承担：

- 对外暗露一个稳定的 Tailnet 节点，供可信设备 SSH 到 `master1`。
- 广告 K3s Pod / Service CIDR，让 Tailnet 内设备可直接访问集群内部网段。
- 作为 worker 节点的运维跳板，不需要每台 worker 单独安装 Tailscale。
- NodePort 所对应的管理服务由 Tailnet 进入。

### 3.2 使用边界

- Tailscale 只作为可信 Tailnet 内访问入口，不替代公网暴露、TLS 证书和应用鉴权。
- `master1` 上的 NodePort 不会自动变成“只允许 Tailscale”；如需严格限制，还需要配合主机防火墙规则。
- Tailnet route approve、Auth Key 不提交到仓库。
- `master1` 是当前唯一 Tailnet 节点，若 `master1` 异常，则暂时失去远程运维能力，需保留局域网应急访问方式。
- 多节点阶段可为每台 worker 也安装 Tailscale，现阶段只需 `master1` 一台即可满足运维目标。

### 3.3 部署形态

当前节点信息与访问口径见 `docs/machines.md`。

总体形态：

| 资源                              | 作用                                   |
| --------------------------------- | -------------------------------------- |
| Alpine `apk`                      | 安装 `tailscale`、`tailscale-openrc`   |
| OpenRC `tailscale` service        | 开机自启，常驻运行                     |
| `tailscale up --advertise-routes` | 广告 K3s Pod / Service CIDR 到 Tailnet |

### 3.4 访问方式

| 方式                     | 用途     | 说明                                                   |
| ------------------------ | -------- | ------------------------------------------------------ |
| SSH `root@100.100.1.2`   | 节点管理 | 直接登录 `master1`，再跳到 worker                      |
| `100.100.1.2:<NodePort>` | 管理服务 | Aether / Mihomo 控制器与 Web UI 等允许的 NodePort 服务 |
| `172.25.x.x:<port>`      | 服务直连 | 经子网路由访问 ClusterIP                               |
| `172.24.x.x:<port>`      | Pod 直连 | 调试排障用，Pod IP 会重建后变化                        |

### 3.5 权限原则

- Tailnet ACL 决定谁能连接 `master1`。
- SSH 登录 `master1` 后再通过 `kubectl` 控制 K3s，是当前最直接的控制方式。
- NodePort 管理服务需配合应用层自身鉴权（如 Aether 管理认证、Mihomo 控制器密钥）。

### 3.6 敏感配置

以下内容不得提交到仓库：

- Tailscale Auth Key。
- Tailnet ACL 私有策略、真实用户邮箱、真实设备名和 Tailnet 名称。
- kubeconfig、K3s token、临时访问凭据。

### 3.7 验收要点

- `master1` 上 Tailscale 服务正常运行，节点在线。
- 已广告的子网路由在 Tailscale Admin Console 已批准。
- Tailnet 客户端能直接 SSH 到 `master1`。
- Tailnet 客户端能访问 ClusterIP（`172.25.x.x`）与 Pod IP（`172.24.x.x`）。
- Tailnet 客户端能通过 `master1` NodePort 访问管理面板。
- Mihomo 不劫持 Tailscale 相关接口与地址段。
