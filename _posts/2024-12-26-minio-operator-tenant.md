---
category: k8s
layout: post_layout
title: minio-operator
time: 2024年12月26日 星期四
location: 杭州
pulished: true
excerpt_separator: "##"
---

## 1、自定义minio-operator的Tenant生成的pool名称

[自定义名称](https://github.com/minio/operator/blob/master/docs/custom-name-templates.md)

[Minio Tenant 扩容](https://github.com/minio/operator/blob/master/docs/expansion.md)

[Minio Operator 中kes的配置](https://github.com/minio/operator/blob/master/docs/kes.md)

[Minio Tenant的yaml文件具体字段的介绍](https://github.com/minio/operator/blob/master/docs/operator-fields.md)

部署之后只会生成sts的证书，这个证书是做什么用的完全不知道

[Minio Operator自定义证书](https://github.com/minio/operator/blob/master/docs/operator-tls.md)

[官方推荐的Tenant部署方式](https://github.com/minio/operator/blob/master/docs/tenant-creation.md)

[在虚拟环境中部署minio-operator的最佳实践](https://blog.min.io/best-practices-minio-virtualized/)

[Kubernetes对象存储的最佳实践](https://blog.min.io/best-practices-for-kubernetes-object-storage/)

[minio 利用kubernetes的CSR进行证书管理](https://blog.min.io/minio-operator-certificate-kubernetes-csr/)

[minio 添加手动平衡](https://blog.min.io/minio-adds-manual-rebalancing/)

[minio disk 损坏，更换的最佳实践](https://blog.min.io/troubleshooting-disk-failures/)

在这里面提到了erasure set的使用，以及数据的恢复

1、当部分磁盘损坏的时候，支持从剩余磁盘中可靠的恢复数据。

    - 使用的是RS(Reed-Solomon)，中文名叫：里德-所罗门类纠错码

2、这里面提到了一个叫做bit rot protection 的东西。使用的是 [HighwayHash算法](https://github.com/minio/highwayhash)来优化的。

[minio erasure set官方中文文档](https://www.minio.org.cn/docs/minio/linux/operations/concepts/erasure-coding.html)

暂时没有理解为什么 在一个节点，16个1TB驱动器组成的mino集群里面，不同纠错码。回影响到总存储空间，存储比例。

| 奇偶校验 | 总存储空间 | 存储比例  | 用于读取操作的最小驱动器 | 用于写操作的最小驱动数量 |
|:----:|:-----:|:-----:|:------------:|:------------:|
| EC:4 | 12TB  | 0.75  |      12      |      12      |
| EC:6 | 10TB  | 0.625 |      10      |      10      |
| EC:8 |  8TB  | 0.50  |      8       |      9       |


当集群中只有一个 erasure set，一共有16个Drivers的时候。配置 EC:4 的时候

存储数据的基本流程：

1. 将 object 分成 12 个数据块

2. 然后针对这12个数据块，生成4个校验块。

3. 最后将这16个块，尽量均分的分布在16个drivers上面。为什么说是尽量呢。因为当存储的对象变的多了，
数据可能就不是均匀存储的了。

#### 一次性运行多少个磁盘故障，还能恢复数据
1. 根据上面的信息知道。当设置EC:2的时候，最多可以同时有2个磁盘故障的情况下面。minio可以根据剩下的数据将
丢失的数据进行恢复

2. 当设置EC:4的时候，最多可以同时有4个磁盘故障的情况下面。minio可以根据剩下的数据将丢失的数据进行恢复

3. 当设置EC:6的时候，最多可以同时有6个磁盘故障的情况下面。minio可以根据剩下的数据将丢失的数据进行恢复

4. 当设置EC:8的时候，最多可以同时有8个磁盘故障的情况下面。minio可以根据剩下的数据将丢失的数据进行恢复。
但是在这情况下面，虽然数据可以恢复。但是不能在写入数据了。因为为了防止脑裂情况。


### 现在配置文件中配置的东西

1. MINIO_STORAGE_CLASS_STANDARD 在 start.yaml 中指定为 EC:2。就是指定纠错码数量为2.
2. 