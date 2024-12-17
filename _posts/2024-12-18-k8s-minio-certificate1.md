---
category: k8s
layout: post_layout
title: k8s部署minio 证书自动续期
time: 2024年12月15日 星期日
location: 杭州
pulished: true
excerpt_separator: "##"
---
##

## 1、部署K8s集群
首先，要有一个K8s集群，这里就不赘述了。如果没有的话，可以使用kind起一个简单的集群。
## 2、起minio-operator
要想要K8s认识minio，我们需要部署一个minio的operator。K8s集群中部署minio-operator。
## 3、部署minio，开启SSE
接下来，我们正式开启在K8s中部署minio，并开启SSE。

需要注意的是。这样子部署minio，会自动生成一个证书。这个证书的过期时间是一年。这个证书是自动续期的。

