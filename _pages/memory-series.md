---
layout: single
title: "内存原理专栏"
permalink: /series/memory/
author_profile: true
---

## 专栏定位

这是一套面向后端 / 系统工程师的内存专栏，目标是把"会调 `malloc`、知道有 GC"升级为"看懂从应用调用到物理页框落地的完整链路、能在生产 OOM / 内存泄漏现场做定位"。

覆盖范围：

- 用户态：glibc / ptmalloc2 / tcmalloc / jemalloc / Go runtime 的分配器
- 内核态：虚拟地址空间、VMA、页表、缺页、伙伴系统、slab、OOM Killer
- 硬件：MMU / TLB / 页大小与大页机制

## 阅读顺序

{% assign series_posts = site.posts | where: "series_id", "memory" | sort: "series_order" %}
{% if series_posts.size > 0 %}
1. 建议按顺序阅读，前一篇是后一篇的前提。
2. 每篇文末都提供专栏导航，可直接跳转上一篇 / 下一篇。

### 目录（共 {{ series_posts.size }} 篇）

{% for post in series_posts %}
- 第 {{ post.series_order }} 篇：[{{ post.title }}]({{ post.url | relative_url }})
{% endfor %}
{% else %}
当前专栏暂无文章。
{% endif %}

## 适合人群

- 想把"内存到底从哪里来、什么时候真正占用"这件事彻底搞清楚的工程师
- 经常需要排查 OOM / 内存泄漏 / 内存增长异常的后端、SRE
- 想理解 Go / Java runtime 内存模型与 Linux 内存管理之间关系的开发者
