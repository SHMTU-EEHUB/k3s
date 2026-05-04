# Basement

> 基础设施介绍。本文只说明 Longhorn、Tailscale 与 Mihomo / Clash 的作用、边界、使用原则和验收要点；当前硬件与运行状态以 `docs/machines.md` 为准，安装 / 恢复步骤以 `docs/init.md` 为准。

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

| 资源             | 作用                                                   |
| ---------------- | ------------------------------------------------------ |
| ConfigMap        | 提供 Mihomo 配置文件；真实订阅链接和密钥不能提交到仓库 |
| Headless Service | 暴露代理服务，供业务 Pod 发现 Mihomo Pod               |
| Deployment       | 运行单副本 Mihomo                                      |

Mihomo Pod 需要 `NET_ADMIN` 与 `NET_RAW` capability。

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

Tailscale 是当前集群的可信远程访问与运维通道，主要用于 Tailnet 内访问 K3s 控制面、Pod 网段和 Service 网段。

它主要承担：

- 通过 Kubernetes Operator 管理集群内 Tailnet 节点。
- 通过 `Connector` 广告 K3s Pod / Service CIDR。
- 通过 `ProxyGroup` 暴露 K3s API Server proxy。
- 通过 Tailnet policy 和 Kubernetes RBAC 组合控制运维权限。

### 3.2 使用边界

- Tailscale 只作为可信 Tailnet 内访问入口，不替代公网暴露、TLS 证书和应用鉴权。
- Tailnet route approve、ACL grants、OAuth Client Secret 不提交到仓库。
- 允许访问 Pod / Service CIDR 不等于允许控制 K3s；控制 K3s 仍通过 API Server proxy 与 RBAC 控制。
- API Server proxy 运行在集群内；若集群自身不可调度，可能失去该控制入口，因此仍需保留本地 / 物理网络应急访问方式。
- 多节点阶段应验证跨节点 Pod / Service 网段经 Tailscale 访问是否稳定。
- Linux Tailnet 客户端需要显式接受子网路由；其他平台以 Tailscale 客户端实际行为为准。

### 3.3 Kubernetes 形态

当前资源名称和网段见 `docs/machines.md`。

总体形态：

| 资源                 | 作用                                                  |
| -------------------- | ----------------------------------------------------- |
| Helm chart           | 安装 Tailscale Kubernetes Operator、CRD 与 Controller |
| `ProxyClass`         | 统一约束 Operator 创建的代理 Pod 调度与资源限制       |
| `Connector`          | 创建 Tailscale 子网路由节点，广告 Pod / Service CIDR  |
| `ProxyGroup`         | 创建 K3s API Server proxy 节点                        |
| `ClusterRoleBinding` | 将 Tailnet impersonation group 绑定到 Kubernetes 权限 |

### 3.4 访问方式

| 方式                             | 用途                  | 说明                                                |
| -------------------------------- | --------------------- | --------------------------------------------------- |
| 子网路由                         | 访问 Pod / Service IP | 适合运维、诊断和少量可信内部访问                    |
| API Server proxy                 | 控制 K3s              | 用 `tailscale configure kubeconfig` 生成 kubeconfig |
| Tailscale Ingress / LoadBalancer | 暴露长期服务          | 适合需要稳定 MagicDNS 名称的服务                    |
| Traefik Ingress + Tailnet/LAN    | 暴露 HTTP 服务        | 适合已有域名入口，但需要确保入口只在可信网络可达    |

### 3.5 权限原则

- Tailnet ACL 决定谁能连接 Tailscale 节点与 API Server proxy。
- API Server proxy 的 `auth` 模式会把 Tailnet 身份 impersonate 成 Kubernetes 用户或组。
- Kubernetes RBAC 决定被 impersonate 后能操作哪些资源。
- 默认只保留 `tailnet-k8s-admins` 与 `tailnet-k8s-readers` 两类组，真实用户组在 Tailnet policy 中映射。
- 如只需要日常查看，应映射到 reader 组，避免默认 cluster-admin。

### 3.6 敏感配置

以下内容不得提交到仓库：

- Tailscale OAuth Client Secret。
- Tailscale Auth Key。
- Tailnet ACL 私有策略、真实用户邮箱、真实设备名和 Tailnet 名称。
- kubeconfig、K3s token、临时访问凭据。

应使用 Sealed Secrets、本地私有 values 或外部密码管理器管理。

### 3.7 验收要点

- Tailscale Operator Pod 正常运行。
- `Connector` 创建的子网路由节点在线，Pod / Service CIDR 已批准。
- Tailnet 客户端接受路由后可以访问测试 ClusterIP / Pod IP。
- `ProxyGroup` 显示 API Server proxy URL。
- Tailnet 管理员可通过 API Server proxy 执行 `kubectl get nodes`。
- Tailnet 只读用户只能读取允许范围内的 Kubernetes 资源。
- Mihomo 不劫持 Tailscale 相关接口与地址段。
