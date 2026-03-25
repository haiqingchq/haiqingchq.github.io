---

categories: [network]
layout: single
title: kube-proxy 深度解析：从 Service 转发到 iptables 与 ipvs 选型
published: true
---

# kube-proxy 深度解析：从 Service 转发到 iptables 与 ipvs 选型

> 面向 Kubernetes 实战，建立“概念 -> 数据路径 -> 模式差异 -> 排障”的完整认知

如果你已经学过 `iptables` 和 `conntrack`，那下一步理解 `kube-proxy` 会非常顺畅。  
因为 Kubernetes Service 的“稳定虚拟 IP + 负载转发”，底层核心执行者之一就是 `kube-proxy`。

这篇文章会回答 5 个关键问题：

1. `kube-proxy` 到底做什么？
2. Service 为什么能通过一个虚拟 IP 访问到后端 Pod？
3. `iptables` 模式和 `ipvs` 模式有什么差异？
4. 遇到 Service 不通、NodePort 异常，该怎么排？
5. 生产环境怎么选型和调优？

---

## 1. kube-proxy 是什么？

`kube-proxy` 是运行在每个 Kubernetes 节点上的网络代理组件。  
它的主要职责是：**把 Service 的抽象（ClusterIP/NodePort）落实为节点可执行的转发规则**。

换句话说：

- 控制面（API Server）定义了“期望路由关系”
- `kube-proxy` 监听这些变化（Service/Endpoints/EndpointSlice）
- 在节点写入转发规则（`iptables` 或 `ipvs`）
- 数据包进来时，按规则被转发到某个后端 Pod

所以它不是四层七层业务代理（那是 Envoy/Nginx 方向），而是偏内核网络路径编排。

---

## 2. Service 流量是如何被转发的？

以 ClusterIP 为例，简化链路如下：

1. Pod A 访问 `ServiceIP:Port`
2. 包到达节点网络栈
3. 命中 `kube-proxy` 下发的规则
4. 选择一个后端 Pod（Endpoint）
5. 执行 DNAT 到 `PodIP:TargetPort`
6. 回包通过 `conntrack` 维护的映射正确返回

关键点：

- `ClusterIP` 通常不是真实网卡地址，而是“虚拟服务地址”
- 真正让它“可达”的是节点上的转发表项
- NAT 状态由 `conntrack` 维护，所以两者必须一起理解

---

## 3. kube-proxy 的两种主流模式

Kubernetes 里常见两种模式：`iptables` 与 `ipvs`。

## 3.1 iptables 模式

`kube-proxy` 会在节点创建大量 `KUBE-*` 规则链，常见如：

- `KUBE-SERVICES`
- `KUBE-NODEPORTS`
- `KUBE-SEP-*`

你可以这样观察：

```bash
iptables -t nat -L -n | grep KUBE
```

特点：

- 实现直观，生态成熟
- 对中小规模集群足够稳定
- 规则数量大时，管理和排障复杂度会上升

## 3.2 ipvs 模式

`ipvs` 是 Linux 内核的四层负载均衡框架。  
`kube-proxy` 在该模式下使用 `ipvs` 虚拟服务表维护后端映射。

查看方式：

```bash
ipvsadm -Ln
```

特点：

- 大规模 Service/Endpoint 场景下性能通常更优
- 转发表结构更适合负载均衡语义
- 仍然依赖 `iptables` 处理部分包路径（并非“完全不需要 iptables”）

---

## 4. iptables vs ipvs：怎么选？

没有绝对答案，建议按场景决策：

### 适合优先考虑 iptables 的场景

- 集群规模中小，Service 数量可控
- 运维团队对 iptables 更熟悉
- 追求简单稳定、减少模式切换成本

### 适合优先考虑 ipvs 的场景

- Service/Endpoint 数量较大
- 高频更新、转发压力较高
- 需要更清晰的四层负载分发观测

### 共同注意点

- 两种模式都受 `conntrack` 影响
- 排障都离不开节点网络命名空间、NAT 与路由检查
- 模式切换前要做灰度验证，不要全量一次切换

---

## 5. 常见流量入口：ClusterIP、NodePort、ExternalTrafficPolicy

## 5.1 ClusterIP

集群内部访问最常见。  
`kube-proxy` 负责把 `ServiceIP:Port` 转到后端 Pod。

## 5.2 NodePort

外部流量访问 `NodeIP:NodePort`，再被转发到 Service 后端。  
常见问题是“节点可达但服务超时”，需要同时检查：

- 节点安全组/防火墙
- `KUBE-NODEPORTS` 规则
- 后端 Pod 就绪状态

## 5.3 ExternalTrafficPolicy

- `Cluster`：可跨节点转发，来源 IP 可能被改写
- `Local`：仅转发本地节点后端，常用于保留客户端源 IP

如果你发现源 IP 丢失，首先检查这里。

---

## 6. 常用排障命令（可直接复用）

## 6.1 看 kube-proxy 配置与日志

```bash
kubectl -n kube-system get ds kube-proxy -o wide
kubectl -n kube-system logs -l k8s-app=kube-proxy --tail=200
```

## 6.2 看 Service 与后端端点

```bash
kubectl get svc -A
kubectl get endpoints -A
kubectl get endpointslice -A
```

## 6.3 看节点规则

```bash
iptables -t nat -L -n --line-numbers | grep KUBE
ipvsadm -Ln
```

## 6.4 看连接跟踪容量

```bash
cat /proc/sys/net/netfilter/nf_conntrack_count
cat /proc/sys/net/netfilter/nf_conntrack_max
```

## 6.5 从 Pod 内验证链路

```bash
kubectl exec -it <pod> -- sh
# 在 Pod 内:
nslookup <service-name>
curl -v http://<service-name>:<port>
```

---

## 7. 一个实战排障案例（Service 偶发超时）

症状：

- 同一 Service 偶发超时
- 重试后成功
- 应用日志无明显异常

排查路径：

1. **确认后端就绪**
   - 检查 Pod readiness、Endpoints 是否完整
2. **确认 kube-proxy 正常**
   - 检查 DaemonSet、日志是否报规则同步失败
3. **确认节点转发规则存在**
   - `iptables/ipvs` 是否存在对应 Service 项
4. **确认 conntrack 没有触顶**
   - `nf_conntrack_count/max` 使用率是否过高
5. **确认节点网络与安全策略**
   - 安全组、NetworkPolicy、主机防火墙是否误拦截

高频根因通常是：

- 连接跟踪资源紧张
- 后端端点频繁抖动
- 节点局部网络异常（只在部分节点触发）

---

## 8. 生产实践建议

1. **监控先行**：纳入 Service 延迟、5xx、连接失败率、`conntrack` 使用率。
2. **模式选择要基于基线**：压测对比后再定 `iptables/ipvs`，不要凭印象。
3. **节点差异要可视化**：同一 Service 在不同节点表现不同，往往是关键线索。
4. **EndpointSlice 变化频率要关注**：频繁抖动会放大数据面不稳定。
5. **变更分层灰度**：先单节点、再节点池、最后全量，避免大面积故障。

---

## 9. 与 Envoy / Service Mesh 的边界

很多团队会混淆：

- `kube-proxy`：负责 Service 级别四层转发可达性
- `Envoy`：负责七层治理（路由、熔断、重试、mTLS、观测）

在 Service Mesh 场景中，两者会同时存在：

- `kube-proxy` 解决“流量怎么到达目标工作负载”
- `Envoy + xDS` 解决“流量到达后如何治理”

理解这个边界，排障效率会大幅提升。

---

## 10. 总结

`kube-proxy` 是 Kubernetes 网络里的“基础转发执行层”。  
它把 Service 抽象落地为真实可执行路径，是你理解集群网络行为的关键组件。

一句话记住：

**Service 是声明，kube-proxy 是落地，conntrack 是记忆。**

