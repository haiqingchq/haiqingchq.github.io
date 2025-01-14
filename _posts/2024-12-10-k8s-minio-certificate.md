---
category: k8s
layout: post_layout
title: k8s部署minio 证书自动续期
time: 2024年12月15日 星期日
location: 杭州
pulished: true
excerpt_separator: "##"
---

1、在刚遇到这个问题的时候，首先考虑的就是参考官方文档，得到的结果是需要开启自动更新的字段。但是开启之后遇到以下问题：

- 续期的时长是7天

- 只会自动更新 Minio 自己的证书，相关的kes，和minio-operator 的 sts 证书并不能自动更新。任然需要手动运维。

2、当时的方案就是，就是直接修改了minio-operator的源码，给相关的证书添加自动续期的功能。

3、但是后续发现了一个更加简单的方案，这里涉及的所有证书都是可以自签的。可以在部署的时候，手动签一个100年的证书替代原来的证书就可以了。

##

