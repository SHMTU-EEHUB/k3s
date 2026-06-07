# Init

> 从 0 安装 / 恢复 DevOps K3s 集群的路线图和验收清单。当前硬件与运行事实以 `docs/machines.md` 为准；Longhorn、Tailscale、Mihomo 的组件原则见 `docs/basement.md`。

## 1. 目标状态

初始化完成后，集群应达到以下状态：

- `master1.eehub.mingz.top` 作为 K3s Server / 控制平面节点运行。
- `worker1.eehub.mingz.top` 至 `worker4.eehub.mingz.top` 作为 K3s Agent / Worker 加入集群。
- 所有节点运行 Alpine Linux。
- 所有节点禁用传统 Swap，并按 `docs/machines.md` 的内存规划启用 ZRAM：16GB 节点使用 8GB，8GB 节点使用 4GB。
- K3s 使用 Flannel VXLAN 作为 CNI。
- 控制平面入口、节点域名和 TLS SAN 使用 `eehub.mingz.top` 下的规划域名。
- Tailscale 可用于可信外部访问和运维，并能访问 K3s Pod / Service 网段与控制 K3s。
- Longhorn 可识别 HDD 数据层，并在 `worker1` 上线后识别 512GB NVMe 高速层。
- Mihomo / Clash 作为集群内集中式出站代理运行。

## 2. 初始化前准备

### 2.1 硬件与命名确认

安装前先按 `docs/machines.md` 核对硬件、磁盘、内存和节点状态。目标节点命名如下：

| 节点     | 主机名                    | 角色   |
| -------- | ------------------------- | ------ |
| Master   | `master1.eehub.mingz.top` | Server |
| Worker-1 | `worker1.eehub.mingz.top` | Agent  |
| Worker-2 | `worker2.eehub.mingz.top` | Agent  |
| Worker-3 | `worker3.eehub.mingz.top` | Agent  |
| Worker-4 | `worker4.eehub.mingz.top` | Agent  |

安装前需要确认：

- 系统盘、HDD 数据盘、`worker1` 的 NVMe 高速盘与规划一致。
- 网线、交换机、上游网络和认证网络可用。
- BIOS 中关闭不必要的深度休眠策略。
- 如有需要，开启虚拟化支持。

### 2.2 私有信息准备

以下信息只应放在 Secret、私有配置或外部密码管理器中，不应提交到仓库：

- Cloudflare Zone ID 与 API Token。
- Tailscale Auth Key 与 Tailnet policy 私有内容。
- 校园网认证凭据。
- K3s Worker 加入 token。
- Mihomo 订阅链接、控制器密钥、节点密码、UUID、私钥。
- 上层工作负载凭据和备份凭据。
- Longhorn 备份目标凭据。

## 3. 从 0 安装流程

### 3.1 阶段一：安装操作系统

所有节点执行：

1. 将 Alpine Linux 安装到系统盘。
2. 设置主机名为规划域名中的节点名。
3. 配置软件源、时区和 NTP。
4. 升级基础系统包。
5. 使用 `dhcpcd` 管理 `eth0`，确认 IPv4 DHCP、IPv6 DHCPv6 / RA 路由均能恢复。
6. 安装常用运维工具和 Longhorn 依赖工具。

磁盘原则：

- 系统盘只承载 OS、K3s、containerd 和基础组件。
- HDD 数据盘预留给 Longhorn，不作为普通应用目录使用。
- `worker1` 的 512GB NVMe 预留给 Longhorn 高速层。

验收：

- 节点可正常重启。
- 主机名符合规划。
- 节点之间网络连通。
- 时间同步正常。

### 3.2 阶段二：内核、网络与内存配置

所有节点执行：

1. 加载 Kubernetes / Longhorn 所需内核模块。
2. 写入 Kubernetes 网络相关 sysctl。
3. 开启 cgroups。
4. 禁用传统 Swap。
5. 按物理内存一半启用 ZRAM。
6. 启用 iSCSI / NFS / 文件系统相关依赖。
7. 为 Mihomo 路由注入 fallback 准备 TUN：加载 `tun` 模块，并确保宿主机存在 `/dev/net/tun` 字符设备。
8. IPv6 由 `dhcpcd` 管理：确认 `dhcpcd` 能在开启转发后恢复 IPv6 `/128` 地址与 RA 默认路由；若重启或断网恢复后默认路由丢失，再针对 `eth0` 显式配置 RA 接收策略。

验收：

- 传统 Swap 未启用。
- ZRAM 重启后仍然生效。
- cgroups 可用。
- iSCSI 相关服务可用。
- 系统重启后 `dhcpcd` 自动恢复 IPv4 地址、IPv6 地址和 IPv6 默认路由。

### 3.3 阶段三：初始化 Master

`master1` 执行：

1. 确认当前网络、DNS 与 DDNS 记录准备就绪。
2. 安装 K3s Server。
3. 使用 Flannel VXLAN，并配置已规划的 Pod / Service 网段。
4. 禁用内置 `traefik` 与 `servicelb`。
5. 使用 `master1.eehub.mingz.top` 作为控制平面入口和 TLS SAN。
6. 保存 Worker 加入 token 到私有位置。
7. 为 Master 打控制平面标签。
8. 确认 Flannel、CoreDNS、metrics-server 等基础组件正常。
9. 部署 Cloudflare DDNS Agent。
10. 在 `master1` 上安装 Tailscale，广告 K3s 子网路由并完成节点接入。

验收：

- K3s Server 正常运行。
- `kubectl get nodes` 能看到 `master1`。
- Flannel 与 CoreDNS 正常。
- 控制平面域名解析正确。
- 通过控制平面域名可以访问 K3s API Server。
- Tailscale 服务在 `master1` 上正常运行，节点在线。

### 3.4 阶段四：接入 Worker

每台 Worker 执行：

1. 确认主机名与规划一致。
2. 确认能解析并访问 `master1.eehub.mingz.top`。
3. 使用私有保存的 token 安装 K3s Agent。
4. 等待节点加入集群。
5. 检查该节点的 Flannel Pod 网络。
6. 检查本地 K3s Agent 服务状态。
7. 确认 Longhorn 数据盘未被系统或其他服务占用。

`worker1` 额外执行：

1. 准备 512GB NVMe 高速盘。
2. 挂载到固定路径。
3. 确认重启后自动挂载。

验收：

- `kubectl get nodes -o wide` 能看到 5 台节点。
- 4 台 Worker 均为 Ready。
- Pod 网络正常。
- Worker 可以访问 Service 网段。
- `worker1` 的高速盘挂载正常。

### 3.5 阶段五：节点标签与资源边界

建议配置以下标签：

| 节点        | 标签                                         |
| ----------- | -------------------------------------------- |
| Master      | `node-role.kubernetes.io/control-plane=true` |
| Master      | `node-role.kubernetes.io/etcd=true`          |
| 所有 Worker | `workload-type=heavy-duty`                   |
| Worker-1    | `fast-node=true`                             |

资源边界原则：

- 长期服务必须设置 requests / limits。
- 临时工作负载必须设置资源上限和清理策略。
- 避免 BestEffort Pod 长期运行。
- 代理、存储、监控等基础组件优先保证稳定性。

### 3.6 阶段六：部署 Longhorn

部署步骤：

1. 安装 Longhorn。
2. 确认所有节点满足 Longhorn 依赖。
3. 将 `docs/machines.md` 中规划的 HDD 数据盘添加为 Longhorn 磁盘。
4. `worker1` 上线后，将 512GB NVMe 添加为高速层磁盘。
5. 为 HDD 数据层与 NVMe 高速层设置不同磁盘标签。
6. 按 `docs/basement.md` 中的 StorageClass 原则创建或规划存储类。

当前只有 `master1` 在线时，Longhorn 不具备跨节点容灾能力，只应按单副本做功能验证；Worker 上线后再调整副本数。

验收：

- Longhorn UI 可访问。
- 在线磁盘状态正常。
- 可以创建测试 PVC。
- 测试 Pod 能挂载 PVC 并读写。
- Worker 上线后，副本调度和重建行为符合预期。

### 3.7 阶段七：安装主机级 Tailscale

在 `master1` 上作为主机服务安装，不使用 Kubernetes Operator。

部署步骤：

1. 准备 Tailscale Auth Key （或登录 URL 授权），不提交到仓库。
2. 在 `master1` 上执行：
   ```sh
   apk add --no-cache tailscale tailscale-openrc
   rc-update add tailscale default
   rc-service tailscale start
   tailscale up --accept-routes --advertise-routes=172.24.0.0/16,172.25.0.0/16
   ```
3. 如使用登录 URL 方式，将输出的 URL 在浏览器中打开并完成授权。
4. 在 Tailscale Admin Console 确认 `master1` 节点在线。
5. 在 Tailscale Admin Console 批准 `172.24.0.0/16` 与 `172.25.0.0/16` 子网路由。
6. 在 Tailnet 客户端上确认路由已生效（Linux 客户端需要显式接受）。

验收：

- `master1` Tailscale 服务正常运行。
- Tailnet Admin Console 中能看到 `master1` 节点。
- 子网路由已批准。
- Tailnet 客户端能直接 SSH 到 `master1`。
- Tailnet 客户端能访问 ClusterIP（`172.25.x.x`）与 Pod IP（`172.24.x.x`）。
- Tailnet 客户端能通过 `master1` NodePort 访问管理面板。
- Mihomo 不劫持 Tailscale 接口、地址段。

### 3.8 阶段八：部署 Mihomo / Clash

部署步骤：

1. 准备私有 Mihomo 配置，生成 `Secret/mihomo-config`。
2. 在无法直接拉取镜像时，可临时借用可信局域网代理完成 Bootstrap，但临时代理地址不得写入仓库。
3. 拉取或预热 `metacubex/mihomo:v1.19.27` 镜像。
4. 部署 GitOps 资源：`Secret/mihomo-config`、`PersistentVolumeClaim/mihomo-runtime`、`clash-proxy` 与 `Deployment/mihomo`。
5. `mihomo-runtime` 持久化 `/root/.config/mihomo` 运行时文件，如 providers、rules、cache、geodata；私有配置仍由 Secret 提供。
6. 按 `docs/machines.md` 的当前形态部署代理 ClusterIP、控制器 API NodePort 和 Web UI NodePort。
7. 确认 `clash-proxy` Endpoints 指向 Mihomo Pod。
8. 验证 Mixed、SOCKS、HTTP、控制器 API 和 Web UI 端口与 `docs/machines.md` 一致。
9. 清理临时 Bootstrap 代理配置。
10. 如需让宿主机 / Worker 拉镜像走代理，需另行配置受控入口；默认不再暴露 Mihomo Mixed 代理 NodePort。
11. 为需要代理的业务配置显式代理环境变量或路由注入。

验收：

- Mihomo Pod 正常运行。
- 代理订阅能正常拉取，但订阅链接不出现在仓库中。
- 控制器只通过可信网络访问，并且设置了密钥。
- 支持显式代理的测试 Pod 出口符合预期。
- 使用路由注入的测试 Pod 默认路由符合预期。
- 普通 Pod 不受代理影响。
- 临时停止 Mihomo 后，需要代理的测试 Pod 不会静默直连外网。
- Tailscale 流量不被 Mihomo 误劫持。

## 4. 恢复流程

### 4.1 恢复前置材料

恢复前应确认以下材料可用，且敏感内容不提交到仓库：

- K3s datastore / snapshot 备份或确认可重建。
- Longhorn 备份目标与访问凭据。
- Kubernetes Secret、私有配置和上层工作负载凭据。
- Mihomo 私有配置、订阅来源和控制器密钥。
- Cloudflare DDNS 与 Tailscale Auth Key / Tailnet policy 所需凭据。
- 集群 manifests、Helm values 或其他部署配置的私有来源。

### 4.2 Master 重建 / 恢复

适用场景：`master1` 系统盘损坏、系统重装或控制平面需要重建。

恢复顺序：

1. 按 `docs/machines.md` 核对硬件和数据盘，不要误格式化 Longhorn 数据盘。
2. 重装 Alpine Linux 并恢复主机名、网络、ZRAM、cgroups、iSCSI 等基础配置。
3. 如有 K3s Server 数据备份，优先按备份恢复控制平面。
4. 如无可用控制平面备份，则按从 0 安装流程重建 K3s Server，并重新接入 Worker。
5. 恢复 Cloudflare DDNS，重新安装 Tailscale 并接入 Tailnet。
6. 恢复 Longhorn 与 Mihomo。
7. 执行最终验收清单。

### 4.3 Worker 重建 / 恢复

适用场景：Worker 系统盘损坏、系统重装或节点需要替换。

恢复顺序：

1. 保留并核对原 Longhorn 数据盘，除非明确要清空该节点数据。
2. 重装 Alpine Linux。
3. 使用相同主机名接入网络。
4. 恢复 ZRAM、cgroups、iSCSI 等基础配置。
5. 使用私有 token 重新加入 K3s 集群。
6. 恢复节点标签。
7. 让 Longhorn 重新识别磁盘，并观察副本重建。
8. 对 `worker1` 额外恢复 NVMe 高速层挂载。

### 4.4 Longhorn 数据恢复

恢复原则：

- 单节点阶段没有跨节点容灾能力，关键数据必须依赖外部备份。
- 多节点阶段优先让 Longhorn 完成副本重建，再恢复业务 Pod。
- 有应用级一致性要求的工作负载不能只依赖 Longhorn 副本，仍需应用级备份。
- 恢复前确认目标卷、备份时间点和业务停机窗口。

### 4.5 Mihomo 恢复

恢复顺序：

1. 恢复私有配置，生成 `Secret/mihomo-config`。
2. 重新部署 GitOps 资源：`Secret/mihomo-config`、`PersistentVolumeClaim/mihomo-runtime`、`clash-proxy` 与 `Deployment/mihomo`。
3. 如没有 Longhorn 备份，`mihomo-runtime` PVC 可从空卷启动，Mihomo 会重新下载或生成 providers、cache、geodata 等运行时文件。
4. 验证 Service、Endpoints、端口和控制器密钥。
5. 验证需要代理的测试 Pod 出口。
6. 验证普通 Pod 不受代理影响。

## 5. 最终验收清单

### 节点

- [ ] 5 台节点均 Ready。
- [ ] 所有节点传统 Swap 关闭。
- [ ] 所有节点 ZRAM 生效。
- [ ] 所有节点时间同步正常。

### 网络

- [ ] 控制平面域名解析正确。
- [ ] Worker 通过域名访问 Master。
- [ ] Flannel 正常。
- [ ] Tailscale 可从可信外部网络访问。
- [ ] Tailnet 客户端可直接 SSH 到 `master1`。
- [ ] Tailnet 客户端可访问 K3s Pod / Service 网段。
- [ ] 认证网络断线后能恢复。

### 存储

- [ ] Longhorn 在线磁盘状态正常。
- [ ] HDD StorageClass 可创建 PVC。
- [ ] NVMe / SSD StorageClass 可创建 PVC。
- [ ] 测试卷可挂载、读写、重建。

### 代理

- [ ] Mihomo Pod 正常运行。
- [ ] `clash-proxy` Endpoints 正常。
- [ ] 需要代理的 Pod 出口符合预期。
- [ ] 普通 Pod 默认不走代理。
- [ ] Tailscale 流量未被误劫持。

## 6. 初始化与恢复原则

- 先保证节点与网络，再部署存储。
- 先部署基础设施，再部署上层工作负载。
- 不要把 DHCP IP 写死到长期配置中。
- 不要把 Secret、Token、密码、订阅链接、UUID、私钥提交到仓库。
- 不要在 Longhorn 稳定前放入关键数据。
- 不要默认所有服务都走代理，只让需要的 Pod 走代理。
