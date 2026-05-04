# Machines

> 硬件配置架构与当前状态。本文记录机器、磁盘、网络、K3s、Longhorn、Tailscale、Mihomo 的当前事实；安装 / 恢复步骤见 `docs/init.md`，组件原则见 `docs/basement.md`。

## 1. 架构概览

| 项目              | 当前口径                                                |
| ----------------- | ------------------------------------------------------- |
| 集群形态          | 1 台 Master + 4 台 Worker                               |
| 当前在线节点      | `master1.eehub.mingz.top`                               |
| 下一台待接入节点  | `worker1.eehub.mingz.top`                               |
| 操作系统          | Alpine Linux                                            |
| Kubernetes 发行版 | K3s                                                     |
| CNI               | Flannel VXLAN                                           |
| 控制平面入口      | `master1.eehub.mingz.top`                               |
| Tailnet 访问      | Tailscale Operator 管理 API Server Proxy 与子网路由节点 |
| 存储基础设施      | Longhorn，当前单节点运行，Worker 上线后再启用跨节点副本 |
| 出站代理基础设施  | Mihomo / Clash，作为集群内集中式代理网关                |

## 2. 状态口径

| 状态     | 含义                                            |
| -------- | ----------------------------------------------- |
| 已实装   | 已安装系统，已加入当前 K3s 集群，并且当前可观测 |
| 等待实装 | 已规划为下一台安装 / 接入的节点，但当前尚未上线 |
| 未实装   | 已规划硬件规格，但当前尚未安装 / 接入集群       |

## 3. 节点与硬件清单

未实装节点的 ZRAM、磁盘和角色为目标规划值。

| 节点                      | 角色   | 内存 | ZRAM Swap | 系统盘         | 额外 SSD / NVMe | HDD                     | 当前状态 | 说明                             |
| ------------------------- | ------ | ---: | --------: | -------------- | --------------- | ----------------------- | -------- | -------------------------------- |
| `master1.eehub.mingz.top` | Master | 16GB |       8GB | 128GB SATA SSD | -               | 2 × 1TB HDD，总量约 2TB | 已实装   | 当前唯一已上线节点；控制平面节点 |
| `worker1.eehub.mingz.top` | Worker | 16GB |       8GB | 64GB NVMe SSD  | 512GB NVMe SSD  | 2TB HDD，尚未实装       | 等待实装 | 下一台待安装 / 接入节点          |
| `worker2.eehub.mingz.top` | Worker |  8GB |       4GB | 64GB SSD       | -               | 2TB HDD                 | 未实装   | 普通 Worker 节点                 |
| `worker3.eehub.mingz.top` | Worker |  8GB |       4GB | 64GB SSD       | -               | 2TB HDD                 | 未实装   | 普通 Worker 节点                 |
| `worker4.eehub.mingz.top` | Worker |  8GB |       4GB | 64GB SSD       | -               | 2TB HDD                 | 未实装   | 普通 Worker 节点                 |

## 4. 资源汇总

| 资源             | 当前在线 | 待上线 | 完整规划 |
| ---------------- | -------: | -----: | -------: |
| 节点数           |        1 |      4 |        5 |
| 内存             |     16GB |   40GB |     56GB |
| ZRAM Swap        |      8GB |   20GB |     28GB |
| HDD 原始容量     |   约 2TB | 约 8TB |  约 10TB |
| 额外 NVMe 高速盘 |      0GB |  512GB |    512GB |

## 5. K3s 集群状态

| 项目                        | 当前配置                                                   |
| --------------------------- | ---------------------------------------------------------- |
| Kubernetes 发行版           | K3s                                                        |
| 当前控制平面节点            | `master1.eehub.mingz.top`                                  |
| 当前已上线 Worker           | 无                                                         |
| 当前 CNI                    | Flannel                                                    |
| Flannel backend             | VXLAN                                                      |
| Datastore                   | Embedded etcd，当前单 Server                               |
| 网络策略能力                | K3s 默认网络策略控制器                                     |
| 节点域名后缀                | `eehub.mingz.top`                                          |
| Master 节点名               | `master1`                                                  |
| Master 域名                 | `master1.eehub.mingz.top`                                  |
| K3s API TLS SAN             | `master1.eehub.mingz.top`                                  |
| Tailnet 控制入口            | `ProxyGroup/eehub-k3s-api`，MagicDNS 名称由 Tailscale 分配 |
| Pod 总网段 / Cluster CIDR   | IPv4：`172.24.0.0/16`；IPv6：`fd00:42::/56`                |
| Service 网段 / Service CIDR | IPv4：`172.25.0.0/16`；IPv6：`fd00:43::/112`               |
| 当前 `master1` Pod CIDR     | IPv4：`172.24.0.0/24`；IPv6：`fd00:42::/64`                |
| 网段规划原则                | 避开上游物理网络使用的 `10.x.x.x` 网段                     |
| 已禁用内置组件              | `traefik`、`servicelb`                                     |

## 6. 物理网络状态

| 项目          | 当前情况                                                    |
| ------------- | ----------------------------------------------------------- |
| 节点网卡      | `eth0`                                                      |
| IPv4 获取方式 | `dhcpcd` 管理 `eth0`，通过 DHCP 获取地址                    |
| IPv6 管理方式 | `dhcpcd` 管理 `eth0` 的 IPv6 地址与 RA 路由                 |
| IPv6 获取方式 | `dhcpcd` / DHCPv6 获取 `/128` 单地址                        |
| IPv6 默认路由 | 由 `dhcpcd` 接收上游 RA 后安装默认路由                      |
| IPv6 RA 接收  | 当前以 `dhcpcd` 行为为准；若默认路由恢复异常，再显式配置 RA |
| 网络环境      | 一级路由器 / 认证网络环境                                   |
| 具体地址记录  | 不在本文记录                                                |

## 7. 已实装节点运行状态

### `master1.eehub.mingz.top`

| 项目            | 当前值                            |
| --------------- | --------------------------------- |
| 当前状态        | 已实装                            |
| 操作系统        | Alpine Linux v3.23                |
| Kernel          | `6.18.25-0-lts`                   |
| K3s 版本        | `v1.35.4+k3s1`                    |
| containerd 版本 | `2.2.3-k3s1`                      |
| 系统盘实际识别  | `/dev/sda`，约 111.8GiB           |
| HDD 1           | 约 1TB，挂载到 `/mnt/storage/sdb` |
| HDD 2           | 约 1TB，挂载到 `/mnt/storage/sdc` |

## 8. Longhorn 当前状态

| 项目                       | 当前情况                                          |
| -------------------------- | ------------------------------------------------- |
| Longhorn 部署状态          | 已在当前单节点 K3s 集群中部署                     |
| 当前 Longhorn 所在节点     | `master1.eehub.mingz.top`                         |
| 当前可用存储节点数         | 1                                                 |
| 当前可用 Worker 存储节点数 | 0                                                 |
| 当前 HDD 数据盘            | `/mnt/storage/sdb`、`/mnt/storage/sdc`            |
| 当前阶段容灾状态           | 当前只有单节点，不具备跨节点容灾条件              |
| 当前阶段副本策略           | 单节点阶段按单副本使用；Worker 上线后再调整副本数 |
| Longhorn UI                | 已部署在 `longhorn-system` 命名空间中             |
| Longhorn 版本              | `v1.11.1`                                         |

### 8.1 当前在线磁盘

| 节点                      | 挂载路径           | 文件系统 | 当前状态                     | 说明       |
| ------------------------- | ------------------ | -------- | ---------------------------- | ---------- |
| `master1.eehub.mingz.top` | `/mnt/storage/sdb` | ext4     | 已挂载 / Longhorn HDD 数据盘 | HDD 数据层 |
| `master1.eehub.mingz.top` | `/mnt/storage/sdc` | ext4     | 已挂载 / Longhorn HDD 数据盘 | HDD 数据层 |

### 8.2 待上线磁盘

| 节点                      | HDD               | 高速盘         | 当前状态 | 说明                                  |
| ------------------------- | ----------------- | -------------- | -------- | ------------------------------------- |
| `worker1.eehub.mingz.top` | 2TB HDD，尚未实装 | 512GB NVMe SSD | 等待实装 | 512GB NVMe 可作为 Longhorn 高速盘候选 |
| `worker2.eehub.mingz.top` | 2TB HDD           | -              | 未实装   | 节点未上线，磁盘当前不在线            |
| `worker3.eehub.mingz.top` | 2TB HDD           | -              | 未实装   | 节点未上线，磁盘当前不在线            |
| `worker4.eehub.mingz.top` | 2TB HDD           | -              | 未实装   | 节点未上线，磁盘当前不在线            |

### 8.3 存储分层容量

| 存储层      | 磁盘来源                                         | 当前在线容量 | 待上线容量 | 完整规划容量 | 当前状态          |
| ----------- | ------------------------------------------------ | -----------: | ---------: | -----------: | ----------------- |
| HDD 数据层  | `master1` 的 2 × 1TB HDD，`worker1-4` 的 2TB HDD |       约 2TB |     约 8TB |      约 10TB | 仅 `master1` 在线 |
| NVMe 高速层 | `worker1` 的 512GB NVMe SSD                      |          0GB |      512GB |        512GB | 待 `worker1` 实装 |

## 9. Mihomo / Clash 当前状态

当前代理方案：K3s 使用 Flannel，Mihomo 作为集群内集中式代理网关。需要代理的 Pod 通过显式代理环境变量或路由注入接入；普通 Pod 默认不走代理。

### 9.1 Kubernetes 资源形态

| 项目               | 当前配置                            |
| ------------------ | ----------------------------------- |
| 配置资源           | `ConfigMap`：`clash-config`         |
| 集群内代理 Service | `clash-proxy`                       |
| Service 类型       | Headless Service，`clusterIP: None` |
| 代理 Workload      | `Deployment`：`mihomo`              |
| 代理副本数         | 1                                   |
| Web UI Workload    | `Deployment`：`metacubexd-ui`       |
| 命名空间           | `default`                           |
| 容器镜像           | `metacubex/mihomo:latest`           |
| 镜像拉取策略       | `IfNotPresent`                      |
| Linux capabilities | `NET_ADMIN`、`NET_RAW`              |
| 设备挂载           | `/dev/net/tun`                      |
| 资源 requests      | 128Mi 内存、100m CPU                |
| 资源 limits        | 256Mi 内存、200m CPU                |

### 9.2 暴露端口与 Service

| 用途                   | Kubernetes 资源           | 集群内端口 | 节点端口 | 说明                                           |
| ---------------------- | ------------------------- | ---------: | -------: | ---------------------------------------------- |
| 集群内代理发现         | `clash-proxy`             |          - |        - | Headless Service，解析到 Pod IP                |
| 宿主机 / Worker 拉镜像 | `mihomo-proxy-nodeport`   |     `7897` |  `30789` | Mixed 代理入口                                 |
| Clean 节点专用代理     | `mihomo-clean-provider-1` |     `7896` |        - | ClusterIP，独立 Mixed 入口，仅供集群内容器使用 |
| 控制器 API             | `mihomo-api-ui`           |     `9097` |  `30910` | 仅可信网络访问，必须设置密钥                   |
| Web UI                 | `metacubexd-ui-svc`       |       `80` |  `30911` | 面板前端，连接控制器 API                       |

### 9.3 监听与网络配置

| 项目                  | 当前配置                                            |
| --------------------- | --------------------------------------------------- |
| 运行模式              | `rule`                                              |
| IPv6                  | 启用                                                |
| UDP                   | 启用                                                |
| LAN 访问              | 启用，绑定所有地址                                  |
| Mixed 端口            | `7897`                                              |
| Clean 节点专用端口    | `7896`；独立 `listeners` 入口                       |
| SOCKS 端口            | `7898`                                              |
| HTTP 端口             | `7899`                                              |
| 外部控制器            | `9097`；仅应通过可信网络访问                        |
| 控制器密钥            | 使用私有配置提供；本文不记录明文                    |
| DNS                   | 启用                                                |
| DNS 增强模式          | `fake-ip`                                           |
| DNS 劫持              | TUN 中启用 DNS hijack                               |
| TUN                   | 启用                                                |
| TUN stack             | `system`                                            |
| TUN 设备名            | `Mihomo`                                            |
| auto-route            | 启用                                                |
| strict-route          | 关闭                                                |
| auto-detect-interface | 启用                                                |
| MTU                   | `9000`                                              |
| 排除接口              | `tailscale0`                                        |
| 排除地址              | Pod / Service 网段、常见内网网段与 Tailscale 地址段 |

### 9.4 敏感配置记录原则

本文不记录 Mihomo 订阅链接、代理节点密码、UUID、Token、私钥、控制器密钥和临时 Bootstrap 外部代理地址；相关原则见 `docs/basement.md`。

## 10. Tailscale 当前状态

当前 Tailscale 方案为**主机级安装**：直接在 `master1` 上通过 `apk` 安装 Tailscale，以 OpenRC 服务方式运行，不使用 Kubernetes Operator。

### 10.1 节点信息

| 项目           | 当前值                                                       |
| -------------- | ------------------------------------------------------------ |
| 安装方式       | Alpine apk：`tailscale tailscale-openrc`                     |
| 服务管理       | OpenRC，`tailscale`，已加入 `default` runlevel               |
| Tailnet 节点名 | `k3s-master`                                                 |
| Tailnet IPv4   | `100.100.1.2`                                                |
| Tailnet IPv6   | `fd7a:115c:a1e0::239:2363`                                   |
| 子网路由广告   | `172.24.0.0/16`（Pod CIDR）、`172.25.0.0/16`（Service CIDR） |
| 路由批准状态   | 已在 Tailscale Admin Console 批准                            |
| SSH 入口       | `ssh root@100.100.1.2`                                       |

### 10.2 Tailnet 访问入口

#### 节点管理

| 用途             | 地址                                    |
| ---------------- | --------------------------------------- |
| SSH 登录 master1 | `root@100.100.1.2` 或 `root@k3s-master` |

#### 管理服务 NodePort

| 服务               | 地址                | 用途                 |
| ------------------ | ------------------- | -------------------- |
| Mihomo API         | `100.100.1.2:30910` | Mihomo 控制 API      |
| Mihomo UI          | `100.100.1.2:30911` | Mihomo Web UI        |
| Mihomo Mixed Proxy | `100.100.1.2:30789` | HTTP/HTTPS 代理入口  |
| ArgoCD HTTP        | `100.100.1.2:30930` | ArgoCD 管理 UI       |
| ArgoCD HTTPS       | `100.100.1.2:30443` | ArgoCD HTTPS 管理 UI |

#### 服务直连（ClusterIP，经子网路由可达）

| 服务              | ClusterIP        | 端口   |
| ----------------- | ---------------- | ------ |
| Metapi            | `172.25.202.34`  | `4000` |
| AxonHub           | `172.25.189.130` | `8090` |
| Aether            | `172.25.165.45`  | `8084` |
| CLI Proxy API     | `172.25.68.30`   | `8317` |
| Kiro RS           | `172.25.14.63`   | `8990` |
| Longhorn Frontend | `172.25.241.144` | `80`   |

### 10.3 敏感配置记录原则

本文不记录 Tailscale Auth Key、Tailnet 名称、Tailnet ACL 私有规则与设备地址；相关原则见 `docs/basement.md`。
