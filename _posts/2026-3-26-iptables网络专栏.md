---

categories: [network]
layout: single
title: iptables 网络专栏：从基础规则到 kube-proxy 与 envoy 实战
published: true
---

# iptables 网络专栏：从基础规则到 kube-proxy 与 envoy 实战

> 面向 Linux / 云原生工程师，建立从「规则语法」到「生产链路」的完整认知

## 1. 什么是 iptables？

`iptables` 是 Linux 内核 `netfilter` 框架在用户态的管理工具。  
可以把它理解为：**用命令行给内核网络包处理流程编程**。

从实现上说：

- **netfilter**：在内核网络栈的关键路径上提供多个 hook 点。
- **iptables**：把规则写入这些 hook 点对应的表和链中。
- **规则（rule）**：由“匹配条件 + 动作（target）”组成。

一个最小规则结构通常是：

```bash
iptables -A INPUT -p tcp --dport 22 -j ACCEPT
```

含义是：在 `INPUT` 链尾追加规则，匹配 TCP 22 端口，命中后执行 `ACCEPT`。

---

## 2. iptables 的作用是什么？

`iptables` 的核心价值可以归纳为 4 类：

### 2.1 访问控制（防火墙）

控制哪些流量可以进、可以出、可以转发。例如：

- 允许运维网段 SSH
- 拒绝异常扫描端口
- 对特定服务白名单放行

### 2.2 地址转换（NAT）

最常见是：

- **SNAT/MASQUERADE**：内网地址改写后访问外网
- **DNAT**：把访问某个地址/端口的流量转发到后端服务

### 2.3 流量治理与隔离

结合连接跟踪（`conntrack`）做状态防火墙：

- 允许已建立连接返回流量
- 对新连接做更严格限制
- 按网段、端口、协议隔离业务

### 2.4 云原生基础能力

容器网络、Kubernetes Service、节点端口转发等能力，底层大量依赖 `iptables` 规则实现。

---

## 3. Linux 中 iptables 命令怎么使用（创建 / 修改 / 删除）

这一节用一组可直接复用的命令，覆盖日常高频操作。

## 3.1 基本概念：表（table）与链（chain）

常见表：

- `filter`：默认表，用于放行/拒绝（`INPUT`、`FORWARD`、`OUTPUT`）
- `nat`：做地址转换（`PREROUTING`、`POSTROUTING`、`OUTPUT`）
- `mangle`：修改包标记、TOS 等

查看规则建议先用带行号和详细信息的方式：

```bash
iptables -L -n -v --line-numbers
iptables -t nat -L -n -v --line-numbers
```

---

## 3.2 创建规则（新增）

### 示例 A：允许 SSH（22 端口）

```bash
iptables -A INPUT -p tcp --dport 22 -j ACCEPT
```

### 示例 B：允许已建立连接返回流量（非常重要）

```bash
iptables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
```

### 示例 C：拒绝其他未明确放行的入站流量

```bash
iptables -P INPUT DROP
```

> 建议顺序：先放行必要流量，再设置默认拒绝策略，避免把自己锁在机器外。

### 示例 D：做 SNAT（服务器单网卡常见）

```bash
iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
```

---

## 3.3 修改规则

`iptables` 没有“就地编辑文本”这种体验，常用方法是 **按位置替换（-R）** 或 **删除后重建**。

先看行号：

```bash
iptables -L INPUT -n -v --line-numbers
```

把 `INPUT` 链第 3 条替换为只允许办公网段 SSH：

```bash
iptables -R INPUT 3 -p tcp -s 10.10.0.0/16 --dport 22 -j ACCEPT
```

---

## 3.4 删除规则

### 按“完整规则”删除

```bash
iptables -D INPUT -p tcp --dport 22 -j ACCEPT
```

### 按“行号”删除（更常用）

```bash
iptables -L INPUT -n --line-numbers
iptables -D INPUT 3
```

### 清空整条链

```bash
iptables -F INPUT
```

---

## 3.5 插入、查看、保存、恢复

### 在链头插入（优先级更高）

```bash
iptables -I INPUT 1 -p tcp --dport 443 -j ACCEPT
```

### 仅查看当前已生效规则

```bash
iptables -S
iptables -t nat -S
```

### 持久化规则（发行版相关）

```bash
# Debian/Ubuntu 常见
iptables-save > /etc/iptables/rules.v4

# 恢复
iptables-restore < /etc/iptables/rules.v4
```

> 在 systemd 环境中，建议配合发行版的持久化服务（如 `netfilter-persistent`）统一管理。

---

## 3.6 一份可落地的最小防火墙模板

```bash
# 1) 清空旧规则（生产环境执行前务必远程会话保活）
iptables -F
iptables -t nat -F
iptables -X

# 2) 默认策略
iptables -P INPUT DROP
iptables -P FORWARD DROP
iptables -P OUTPUT ACCEPT

# 3) 放行回环与已建立连接
iptables -A INPUT -i lo -j ACCEPT
iptables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT

# 4) 放行必要端口（示例：22/80/443）
iptables -A INPUT -p tcp --dport 22 -j ACCEPT
iptables -A INPUT -p tcp --dport 80 -j ACCEPT
iptables -A INPUT -p tcp --dport 443 -j ACCEPT

# 5) 可选：记录并拒绝其他流量
iptables -A INPUT -j REJECT --reject-with icmp-host-prohibited
```

---

## 4. iptables 在 kube-proxy、envoy 中的应用

这部分是很多工程师“知道命令但不理解生产行为”的分水岭。

## 4.1 kube-proxy（iptables 模式）

在 Kubernetes 中，`Service` 的虚拟 IP（ClusterIP）并不是真实网卡 IP。  
`kube-proxy` 在 `iptables` 模式下会生成大量链和规则，实现：

- ClusterIP 到后端 Pod 的转发（DNAT）
- NodePort 的入站转发
- 部分场景的会话保持与负载分发

你在节点上常能看到类似链名：

- `KUBE-SERVICES`
- `KUBE-NODEPORTS`
- `KUBE-SEP-*`

可通过命令观察：

```bash
iptables -t nat -L -n | grep KUBE
```

工作机制（简化）：

1. 客户端访问 `Service IP:Port`
2. 在 `PREROUTING/OUTPUT` 等路径命中 `KUBE-SERVICES`
3. 跳转到某个后端 endpoint 链（`KUBE-SEP-*`）
4. 执行 DNAT 到 Pod IP:Port

这就是“Service 看起来是一个固定地址，但实际落到不同 Pod”的关键。

---

## 4.2 envoy 与 iptables（Service Mesh 侧车拦截）

在 Istio 等 Service Mesh 中，`envoy` 是用户态代理；  
而把业务流量“导入 envoy”这一步，通常靠 `iptables` 完成。

典型做法：

- 在 Pod 网络命名空间中写入 `nat` 规则
- 通过 `REDIRECT`/`TPROXY` 把入站与出站流量重定向到 `envoy` 监听端口（如 `15001`、`15006`）
- `envoy` 再执行路由、mTLS、熔断、重试、可观测性上报等 L7 能力

常见链名（不同版本/实现略有差异）：

- `ISTIO_INBOUND`
- `ISTIO_OUTPUT`
- `ISTIO_REDIRECT`

为什么这套组合重要：

- `iptables` 擅长 L3/L4 透明拦截与转发
- `envoy` 擅长 L7 策略与治理
- 二者结合，形成“无侵入接管流量 + 高级治理能力”的架构

---

## 5. 生产实践建议

1. **先观测再改动**：改规则前先 `iptables-save` 备份。
2. **变更可回滚**：保持一条可用 SSH 会话，避免误封禁。
3. **规则顺序即语义**：`iptables` 首条命中即停止，顺序错误常导致“看起来规则都在却不生效”。
4. **关注 conntrack 容量**：高并发场景下连接跟踪表满会引发隐蔽故障。
5. **云原生环境优先看自动生成链**：Kubernetes/Istio 的问题，先排查 `KUBE-*` 与 `ISTIO_*` 链。

---

## 6. 总结

`iptables` 不只是“老防火墙命令”，而是 Linux 网络包处理的基础控制面。  
理解它，你才能真正看懂：

- 一条流量为什么被放行/拒绝
- Kubernetes Service 为什么能工作
- Service Mesh 为什么能透明劫持业务流量

如果你后续准备深入网络专栏，建议下一篇可以写：

- `conntrack` 原理与排障
- `nftables` 与 `iptables` 关系
- `kube-proxy` 的 `iptables` 模式 vs `ipvs` 模式

