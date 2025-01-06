## Minio扩容与缩容方案

> 参考文档：
>
> - https://blog.min.io/decommissioning/
> - https://min.io/docs/minio/kubernetes/upstream/operations/install-deploy-manage/expand-minio-tenant.html
>
> 开始执行退化的pool，会被锁定为只读状态，（非新写入请求未完成的，任然会执行完成），如果此时任然有新的写入请求，Minio会自动重定向到新的pool。但是为了减少意外情况，最好在使用量少的时候进行扩容/缩容。

#### 1、创建新的pool

直接修改tenant.yaml文件，在spec.pools添加一个新的pool的定义。然后应用这个修改。等待执行完成并生效。

#### 2、退化旧的pool

1. 首先查看集群中已经存在的pool：
   ~~~bash
   [root@iZbp108b5djx9wspy4wfezZ ~]# mc admin decommission status v1/
   ┌─────┬──────────────────────────────────────────────────────────────────────────────┬──────────────────────┬──────────┐
   │ ID  │ Pools                                                                        │ Drives Usage         │ Status   │
   │ 1st │ https://myminio-pool-0-{0...3}.myminio-hl.v1.svc.cluster.local/export{0...3} │ 0.0% (total: 14 PiB) │ Active   │
   │ 2nd │ https://myminio-pool-1-{0...3}.myminio-hl.v1.svc.cluster.local/export{0...3} │ 0.0% (total: 14 PiB) │ Active   │
   └─────┴──────────────────────────────────────────────────────────────────────────────┴──────────────────────┴──────────┘
   ~~~

2. 可以看到两个pool，选择退化掉第一个pool
   ~~~bash
   mc admin decommission start v1/ https://myminio-pool-0-{0...3}.myminio-hl.v1.svc.cluster.local/export{0...3}
   ~~~

3. 最后重新发看pool的状态
   ~~~bash
   [root@iZbp108b5djx9wspy4wfezZ ~]# mc admin decommission status v1/
   ┌─────┬──────────────────────────────────────────────────────────────────────────────┬──────────────────────┬──────────┐
   │ ID  │ Pools                                                                        │ Drives Usage         │ Status   │
   │ 1st │ https://myminio-pool-0-{0...3}.myminio-hl.v1.svc.cluster.local/export{0...3} │ 0.0% (total: 14 PiB) │ Complete   │
   │ 2nd │ https://myminio-pool-1-{0...3}.myminio-hl.v1.svc.cluster.local/export{0...3} │ 0.0% (total: 14 PiB) │ Active   │
   └─────┴──────────────────────────────────────────────────────────────────────────────┴──────────────────────┴──────────┘
   ~~~

   可以看到第一个pool已经处于Complete状态，就是退化完毕了。

#### 3、删除旧的pool

修改tenant.yaml文件，将spec.pools中的旧的pool的定义，然后应用这个这个更改。等待执行完毕即可。