## minio Bucket 备份方案

### addon

通过调研发现，Bucket级别的数据复制，当源桶开启加密的情况下，目标桶也要开启加密。所以，目标桶和源桶的部署要一致，就是直接部署一套与源集群完全相同的minio集群。



### 准备工作

#### 1、源桶

在创建Bucket时，需要开启 版本控制。同时明确，当前Bucket是否开启了加密服务。需要注意的是，加密服务是Bucket级别的，需要在创建Bucket之后觉得，手动开启



#### 2、目标桶

创建一个桶，开启版本控制。是否开启加密需要与源桶保持一致

创建一个AK/SK，如下图所示：

![image-20250102100135849](/Users/chenhaiqing/Library/Application Support/typora-user-images/image-20250102100135849.png)

然后需要给这个“用户”赋予权限：
![image-20250102100239653](/Users/chenhaiqing/Library/Application Support/typora-user-images/image-20250102100239653.png)

这里主要是，需要目标集群的这个用户，对当前操作桶有所有权限，不然的话，源桶无法将数据同步过来。



### 操作

在源集群的Bucket中，找到需要被同步复制的集群，添加一个replicate：

![image-20250102100839949](/Users/chenhaiqing/Library/Application Support/typora-user-images/image-20250102100839949.png)

![image-20250102100935134](/Users/chenhaiqing/Library/Application Support/typora-user-images/image-20250102100935134.png)

最后就可以在源集群中上传和删除文件，验证是否成功了。