---

categories: [network]
layout: single
title: Kubernetes 外部流量接入实战：MetalLB + Envoy Gateway + Gateway API
published: true
series: 云原生网络专栏
series_id: cloud-native-network
series_order: 4

---

# Kubernetes 外部流量接入实战：MetalLB + Envoy Gateway + Gateway API

## 前言

在 Kubernetes 中，如何让集群外部的流量访问到集群内的服务，是一个核心问题。

**云环境（AWS、阿里云、腾讯云）**：创建 LoadBalancer 类型的 Service，云厂商自动创建 LB 并分配公网 IP，开箱即用。

**裸金属 / 自建集群（包括 Kind）**：没有云厂商的 LB 支持，`type: LoadBalancer` 的 Service 永远卡在 Pending。需要自己搭建流量入口。

而这个需求，恰好引出了 Kubernetes 生态中两个重要组件：
- **MetalLB** — 裸金属负载均衡器，为 LoadBalancer Service 分配 IP
- **Gateway API** — Kubernetes 官方推出的下一代流量管理 API（Ingress 的继任者）

本文将完整记录如何搭建一套生产可用的流量接入方案：

1. 部署 MetalLB，让 LoadBalancer 类型 Service 能分配到 IP
2. 部署 Envoy Gateway（Gateway API 的官方参考实现）
3. 通过 Gateway API 的 HTTPRoute 将流量路由到后端服务
4. 宿主机端口转发，打通外部访问

> **这套方案不只是 Kind 玩具**，裸金属 K8s 集群、边缘计算、本地机房，都在用同样的架构。

---

## 环境说明

| 项目 | 配置 |
|---|---|
| 服务器 | 腾讯云轻量云，40GB 磁盘 |
| OS | OpenCloudOS 9.4 |
| Kind 集群 | 5 节点（1 control-plane + 4 worker→后缩减为 2 节点） |
| K8s 版本 | v1.33.1 |
| Kind 版本 | v0.29.0 |
| Docker 网络 | 172.21.0.0/16 |

---

## 第一章：裸金属集群的网络困境

### 云环境 vs 裸金属

在云环境（AWS EKS、阿里云 ACK、腾讯云 TKE）中，创建一个 LoadBalancer Service 时：

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-app
spec:
  type: LoadBalancer  # 云厂商自动创建 LB
```

云厂商的控制器会自动创建负载均衡器、分配公网 IP、配置健康检查，一切自动完成。

但在**裸金属或自建集群**中，没有云厂商 API 可以用。`type: LoadBalancer` 的 Service 永远卡在 `<pending>` 状态。

### 网络隔离问题

以 Kind 集群为例，实际生产裸金属集群的原理类似：

```
外部机器（笔记本）
    │
    ├─ 宿主机网卡 (81.69.8.26) ← 真实物理 IP
    │   │
    │   ├─ 容器/VM 网络: 172.21.0.0/16
    │   │   ├─ K8s 节点 (172.21.0.2 ~ 172.21.0.6)
    │   │   │   ├─ Pod IP 段: 10.244.0.0/16
    │   │   │   │   └─ 应用 Pod (10.244.1.2:8080)
    │   │   │   └─ Service ClusterIP: 10.96.0.0/16
    │   │   └─ 这些 IP 段外部无法直接访问
    │   │
    │   └─ 对外 IP
```

有三个隔离的 IP 段：

| IP 段 | 谁在用 | 谁能访问 |
|---|---|---|
| 10.244.0.0/16 | Pod | 仅集群内部 |
| 10.96.0.0/16 | Service ClusterIP | 仅集群内部 |
| 172.21.0.0/16 | 节点 / MetalLB | 宿主机 + 内部网络 |
| 81.69.8.26 | 宿主机网卡 | 外部互联网 |

> **裸金属生产环境也是类似的**：服务器有物理网卡 IP，Pod 在独立的网络命名空间里，外部不能直接访问。

### 解决方案架构

打通外部到 Pod 的访问，需要四层组件：

```
外部请求
  ↓
① 宿主机端口转发（Nginx / iptables DNAT / 物理 LB）
  ↓  宿主机 80 端口 → MetalLB IP
② MetalLB（裸金属 LB）
  ↓  将 IP 分配给 LoadBalancer Service
③ Envoy Gateway（Gateway API 控制器）
  ↓  解析 HTTPRoute 路由规则
④ 后端 Service（ClusterIP）
  ↓
⑤ Pod
```

---

## 第二章：部署 MetalLB

### MetalLB 是什么？

MetalLB 是一个裸金属 Kubernetes 的负载均衡器实现。它让你在没有云厂商 LB 的环境下，也能给 Service 分配 External IP。

它有两种工作模式：

- **Layer2 模式**：用 ARP/NDP 协议宣告 IP，节点间故障转移
- **BGP 模式**：通过 BGP 协议与路由器交换路由

本文将使用 Layer2 模式。

### 安装 MetalLB

```bash
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.14.9/config/manifests/metallb-native.yaml
```

**⚠️ 国内网络问题：** 如果拉取 quay.io 镜像慢，可以先把镜像拉到宿主机，再用 `kind load docker-image` 导入到集群节点：

```bash
docker pull quay.io/metallb/controller:v0.14.9
docker pull quay.io/metallb/speaker:v0.14.9
kind --name my-5-node-cluster load docker-image quay.io/metallb/controller:v0.14.9 quay.io/metallb/speaker:v0.14.9
```

### 配置 IP 地址池

MetalLB 需要知道哪些 IP 可以分配。这些 IP 必须在 Docker 桥接网络所在的子网内。

先查看 Docker 网络信息：

```bash
docker network inspect kind --format '{{range .IPAM.Config}}{{.Subnet}} - {{.Gateway}}{{end}}'
```

输出：`172.21.0.0/16 - 172.21.0.1`

选择一个不与其他容器冲突的地址段，创建 IPAddressPool 和 L2Advertisement：

```yaml
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: my-pool
  namespace: metallb-system
spec:
  addresses:
  - 172.21.0.100-172.21.0.200
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: my-l2-advertisement
  namespace: metallb-system
spec:
  ipAddressPools:
  - my-pool
```

### 验证 MetalLB 安装

```bash
kubectl get pods -n metallb-system
```

所有 Pod 状态为 Running 即表示安装成功。

### 给 Service 分配 IP

将一个已有的 Service 改为 LoadBalancer 类型：

```bash
kubectl patch svc test-service -p '{"spec":{"type":"LoadBalancer"}}'
```

查看 External-IP：

```bash
kubectl get svc test-service
# NAME           TYPE           CLUSTER-IP    EXTERNAL-IP
# test-service   LoadBalancer   10.96.79.78   172.21.0.100
```

MetalLB 从池中自动分配了 `172.21.0.100` 给这个 Service。

---

## 第三章：部署 Envoy Gateway

### 为什么选 Envoy Gateway？

Kubernetes 官方已宣布 Ingress 进入冻结状态，不再增加新功能。**Gateway API** 是其正式继任者，提供了：

- 更精细的路由规则（按 Header、权重等）
- 跨命名空间路由
- 面向角色设计的资源模型（基础设施提供者、集群运维、应用开发者）

Envoy Gateway 是 **Gateway API 的官方参考实现**，由 Envoy 核心团队维护。

### 安装 Gateway API CRD

```bash
curl -sL -o /tmp/gateway-crd.yaml https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.2.1/standard-install.yaml
kubectl apply -f /tmp/gateway-crd.yaml
```

安装了以下 CRD：

- `gatewayclasses.gateway.networking.k8s.io`
- `gateways.gateway.networking.k8s.io`
- `httproutes.gateway.networking.k8s.io`
- `grpcroutes.gateway.networking.k8s.io`
- `referencegrants.gateway.networking.k8s.io`

### 安装 Envoy Gateway

```bash
curl -sL -o /tmp/envoy-gateway-install.yaml https://github.com/envoyproxy/gateway/releases/download/v1.3.0/install.yaml
kubectl apply -f /tmp/envoy-gateway-install.yaml
```

**⚠️ 镜像拉取问题：** 同样需要手动拉镜像并导入 Kind 节点：

```bash
docker pull envoyproxy/gateway:v1.3.0
docker pull envoyproxy/envoy:distroless-v1.33.0
kind --name my-5-node-cluster load docker-image envoyproxy/gateway:v1.3.0
kind --name my-5-node-cluster load docker-image envoyproxy/envoy:distroless-v1.33.0
```

### 了解 Gateway API 三层模型

Gateway API 引入了三个核心角色：

1. **GatewayClass** — 定义网关的类型（类似 StorageClass），由基础设施提供者创建
2. **Gateway** — 网关实例的声明，指定监听端口和协议，由集群运维创建
3. **HTTPRoute** — 定义路由规则，将流量导向后端 Service，由应用开发者创建

---

## 第四章：创建 Gateway API 资源

### 创建 GatewayClass

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
  name: eg
spec:
  controllerName: gateway.envoyproxy.io/gatewayclass-controller
```

`controllerName` 告诉 Kubernetes 哪个控制器应当处理这个 GatewayClass 的实例。

### 创建 Gateway

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: my-gateway
  namespace: default
spec:
  gatewayClassName: eg
  listeners:
  - name: http
    protocol: HTTP
    port: 80
    allowedRoutes:
      namespaces:
        from: All
```

创建 Gateway 后，Envoy Gateway 会自动做两件事：

1. **创建 Envoy Proxy Pod** — 一个运行 Envoy 的数据面代理
2. **创建 LoadBalancer Service** — MetalLB 分配 External IP

查看结果：

```bash
kubectl get gateway my-gateway -o wide
# NAME         CLASS   ADDRESS        PROGRAMMED   AGE
# my-gateway   eg      172.21.0.101   True         25m

kubectl get svc -n envoy-gateway-system
# NAME                                TYPE           CLUSTER-IP     EXTERNAL-IP
# envoy-default-my-gateway-xxxxx      LoadBalancer   10.96.245.18   172.21.0.101
```

### 创建 HTTPRoute

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: test-app-route
  namespace: default
spec:
  parentRefs:
  - name: my-gateway
  rules:
  - backendRefs:
    - name: test-service
      port: 80
```

**关键字段说明：**

- `parentRefs` — 关联到哪个 Gateway
- `backendRefs` — 转发到哪个 Service（不需要 LoadBalancer 类型，ClusterIP 即可）

### 测试内部连通性

在宿主机上 curl MetalLB 分配的 IP：

```bash
curl http://172.21.0.101
# <html><body><h1>tenten</h1></body></html>
```

---

## 第五章：宿主机反向代理（打通外部访问）

### 为什么还需要这一步？

MetalLB 分配的 IP（如 `172.21.0.101`）在 Docker 内部网络，外部机器无法直接访问。我们需要在宿主机上运行 Nginx，将宿主机端口转发到 Gateway 的 IP。

用 Docker 运行 Nginx：

```bash
docker run -d --name nginx-gateway --restart=always --network kind -p 80:80 nginx:alpine
```

配置 Nginx 反向代理：

```nginx
server {
    listen 80;
    location / {
        proxy_pass http://172.21.0.101:80;
    }
}
```

> **为什么 Nginx 容器要用 `--network kind`？** 这样才能直接访问 Docker 桥接网络内的 IP（即 MetalLB 分配的地址）。

### 开放防火墙

腾讯云安全组和宿主机防火墙都需要开放对应端口：

```bash
firewall-cmd --add-port=80/tcp --permanent
firewall-cmd --reload
```

然后在腾讯云控制台安全组加一条入站规则：TCP 80 端口。

### 最终验证

```bash
# 从宿主机测试
curl http://127.0.0.1:80
# <html><body><h1>tenten</h1></body></html>

# 从外部机器访问
curl http://81.69.8.26:80
# <html><body><h1>tenten</h1></body></html>
```

---

## 第六章：完整架构图

```
                         ┌─────────────────────────┐
                         │  你的电脑                 │
                         │  http://81.69.8.26       │
                         └─────────────┬───────────┘
                                       │
                         ┌─────────────▼───────────┐
                         │  腾讯云安全组（80 端口）  │
                         └─────────────┬───────────┘
                                       │
                         ┌─────────────▼───────────┐
                         │  宿主机 firewalld（80）   │
                         └─────────────┬───────────┘
                                       │
                         ┌─────────────▼───────────┐
                         │  Nginx 容器 (port 80)    │
                         │  proxy_pass ↓           │
                         └─────────────┬───────────┘
                                       │
                         ┌─────────────▼───────────┐
                         │  Envoy Gateway           │
                         │  172.21.0.101:80         │
                         │  (MetalLB 分配)          │
                         └─────────────┬───────────┘
                                       │
                    ┌──────────────────┼──────────────────┐
                    │                  │                  │
         ┌──────────▼─────────┐  ┌────▼─────────┐  ┌────▼─────────┐
         │  HTTPRoute         │  │  HTTPRoute   │  │  HTTPRoute   │
         │  test-app-route    │  │  my-app-route│  │  (你的新服务) │
         └──────────┬─────────┘  └──────┬───────┘  └──────┬───────┘
                    │                   │                  │
         ┌──────────▼─────────┐  ┌──────▼───────┐  ┌──────▼───────┐
         │  test-service      │  │  my-app-svc  │  │  新 Service   │
         │  (ClusterIP)       │  │  (ClusterIP) │  │  (ClusterIP)  │
         └────────────────────┘  └──────────────┘  └──────────────┘
```

---

## 第七章：如何添加新服务

以后每加一个新应用，只需要三步：

**步骤 1：部署应用**
```bash
kubectl create deployment my-app --image=my-image
kubectl expose deployment my-app --port=80 --name=my-app-svc
```

> 注意：不需要 `--type=LoadBalancer`，默认 ClusterIP 即可。

**步骤 2：创建 HTTPRoute**
```bash
cat <<EOF | kubectl apply -f -
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: my-app-route
spec:
  parentRefs:
  - name: my-gateway
  rules:
  - backendRefs:
    - name: my-app-svc
      port: 80
EOF
```

**步骤 3：验证**
```bash
curl http://81.69.8.26
```

不需要修改 Nginx，不需要改防火墙，不需要碰 MetalLB 配置 —— Envoy Gateway 会自动热加载路由规则。

---

## 踩坑记录

### 1. 国内网络慢

所有 Docker 镜像（quay.io, Docker Hub, GitHub raw content）在国内下载都很慢。解决方式：

- 使用中科大 USTC 镜像源
- 先用 `docker pull` 拉到宿主机，再用 `kind load docker-image` 导入集群
- 部分容器设置 `imagePullPolicy: IfNotPresent`

### 2. 磁盘空间不足

40GB 磁盘 + 5 节点 Kind 集群非常紧张。Kind 节点使用 Docker overlayfs，每个节点都有独立的镜像存储。

解决方案：

- 缩减 worker 节点数量（5→2）
- 定期清理 Docker 未使用的镜像和卷
- 清理历史日志

### 3. Firewalld 重置 iptables 规则

`firewall-cmd --reload` 会清空手工添加的 iptables DNAT 规则。替代方案：

- 使用 firewalld 的富规则或端口转发功能
- 或直接用 Nginx/Docker 暴露端口（推荐）

### 4. Docker 重启需要恢复网络

重启 Docker 守护进程后：

- Kind 节点容器需要重新启动
- 需要等待 Kubernetes 组件恢复（约 30 秒）
- Nginx 容器可能需要重新启动

---

## 附录：常用命令速查

```bash
# MetalLB 相关
kubectl get pods -n metallb-system
kubectl get ipaddresspool -n metallb-system
kubectl get l2advertisement -n metallb-system

# Gateway API 相关
kubectl get gatewayclass
kubectl get gateway
kubectl get httproute
kubectl get svc -n envoy-gateway-system

# 调试
kubectl logs -n envoy-gateway-system -l gateway.envoyproxy.io/owning-gateway-namespace=default
curl -v http://172.21.0.101

# 宿主机 Nginx
docker logs nginx-gateway
docker exec nginx-gateway nginx -s reload
```

---

## 总结

### 架构总览

```
外部请求 → 宿主机端口转发 → MetalLB → Envoy Gateway → HTTPRoute → Service → Pod
                                                      ↑
                                              Gateway API 标准
```

### 各组件定位

| 组件 | 作用 | 生产替代方案 |
|---|---|---|
| MetalLB | 裸金属 LB，分配 External IP | 同方案，或物理 F5 / HAProxy |
| Envoy Gateway | Gateway API 控制器，路由流量 | Istio、Contour、Kong 均可 |
| Gateway API CRDs | 声明式路由规则（GatewayClass → Gateway → HTTPRoute）| 标准 API，厂商无关 |
| Nginx | 宿主机端口转发 | 物理 LB、iptables DNAT |

### 这套方案的优势

- **标准化** — Gateway API 是 Kubernetes 官方标准，Ingress 的正式继任者
- **厂商无关** — 不管用 Envoy Gateway、Istio 还是 Contour，上层的 HTTPRoute 资源写法完全一样
- **声明式** — 加新服务只需 `kubectl apply -f httproute.yaml`，不改任何基础设施
- **无侵入** — 所有流量管理在集群内完成，宿主机只需一个端口转发
- **多维度路由** — 支持按域名、路径前缀、Header、权重等多维度分发

### 这套方案用在哪些地方

这套架构不是 Kind 的玩具方案，而是**生产环境的通用模式**：

- **裸金属机房** — 物理服务器上自建 K8s，用 MetalLB + Gateway API 做流量入口
- **边缘计算** — 无云环境的边缘节点，用同样的方案暴露服务
- **混合云** — 部分在云上、部分在本地，统一使用 Gateway API 管理路由
- **开发测试** — Kind 集群完全模拟生产环境的流量路径，本地验证后再上线
