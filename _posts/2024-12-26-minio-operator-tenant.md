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

官方计算器：https://min.io/product/erasure-code-calculator

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



## 官方文档

1. [minio erasure code](https://min.io/docs/minio/linux/operations/concepts/erasure-coding.html)

2. [minio object healing](https://min.io/docs/minio/linux/operations/concepts/healing.html)

- 在这个文档中，很明显的提到了，需要恢复数据，需要最少一个校验分片。来恢复剩下的所有分片，包括损坏或者丢失的校验分片。
但是如果所有校验分片都丢失了的话，数据将无法恢复。这个是需要注意的。因此在设置EC值的时候还是合理。

- 在还有足够校验分片的情况下，数据分片最多丢失设置的EC值的数量。任然能够恢复。超过这个值，数据将无法恢复。

- 当我直接删除一个桶中的对象的时候，我重新向minio发送一个get请求，桶中被删除的对象就直接会被恢复。

- 当我在pv中，直接删除一个桶。然后直接发送一个get请求，被删除的桶不会被恢复。我直接使用 `mkdir test` 命令创建一个桶。 
然后重新发送一个GET请求， 被删除的数据仍然没有被恢复。 但是我使用 `mc admin heal test/test` 重建这个桶。然后重新
发送GET请求。被删除的对象将会被恢复。这是一个有趣的现象。

但是我能正常的将文件下载下来，显而易见的事情就是。数据已经被恢复了。但是因为在这个drivers上面的桶不存在了。minio认为
这个pvc已经存坏了。然后就将恢复的数据重新存储到另外一个pvc上面。

但是有趣的事情就是，就算数据已经恢复了。我在当前这个pvc上面。使用 `mc admin heal test/test` 命令重建这个桶。重新
下载数据的时候，这个数据又重新回到了这个pvc上面。但是不确定回到这个pvc上面的数据分片是不是原来的数据分片。

[minio的阈值和简单的规则限制](https://min.io/docs/minio/linux/operations/concepts/thresholds.html)
