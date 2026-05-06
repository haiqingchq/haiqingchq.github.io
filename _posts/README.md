# `_posts/` 目录组织约定

本目录用于存放所有要发布到博客的文章。**子目录名必须等于该文章 front matter 里 `categories` 的第一个元素**——这是 URL 不变的硬性约定。

> 本 README 不会出现在站点上：Jekyll 只把 `_posts/` 里满足 `YYYY-MM-DD-*.md` 命名的文件当成 post，其它文件（包括本文件）会被忽略。

---

## 1. 当前目录结构

```
_posts/
├── docker/             # Docker / 镜像构建
├── ebpf/               # eBPF 系列
├── go/                 # Go 语言、底层数据库等
├── k8s/                # Kubernetes 相关
├── mcp/                # MCP 协议
├── memory/             # 内存原理专栏（用户态分配器 + 内核内存管理）
├── network/            # 云原生网络专栏（含 iptables / Envoy / kube-proxy / metallb 等）
│   └── xds/            # network 下的 xDS 子专栏
├── openTelemetry/      # 可观测性 - OpenTelemetry
├── prometheus/         # 可观测性 - Prometheus
├── test/               # 测试稿（不要发到生产分类）
├── tools/              # 小工具、脚本
└── ES.md               # 历史草稿，没有日期前缀，Jekyll 不会发布它
```

---

## 2. URL 约定（关键，别违反）

`_config.yml` 里设置：

```yaml
permalink: /:categories/:title/
```

含义：文章的最终 URL = `/<categories 列表用 / 拼接>/<title 标题 slug>/`。

举例：

| 文章 categories | 最终 URL |
|---|---|
| `[network]` | `/network/<slug>/` |
| `[network, xds]` | `/network/xds/<slug>/` |
| `[k8s]` | `/k8s/<slug>/` |

### Jekyll 行为提醒

Jekyll 会**自动把 `_posts/` 的子目录名注入到 categories 里**（与 front matter 合并去重）。所以：

- 子目录名 = front matter 里 `categories` 的**第一个**元素 ⇒ URL 与平铺时**完全一致**
- 子目录名 ≠ category ⇒ Jekyll 会注入新 category，URL 改变，**会破坏所有外链 / SEO / RSS**

迁移本目录时已经按此规则走过一遍，所有现有 URL 没变。

---

## 3. 怎么新增一篇文章

### 步骤 1：决定分类

先想好这篇文章属于哪个一级分类（也是要落到哪个子目录）。

- 已有分类：直接放进对应子目录
- 全新分类：先决定分类名（小写英文，无空格），再新建对应子目录

### 步骤 2：命名

文件名格式：

```
YYYY-MM-DD-<英文-slug>.md
```

- 日期就是发布日期，未来日期默认不会发布（除非 `_config.yml` 开 `future: true`）
- slug 建议用纯英文小写 + 连字符，URL 友好；中文文件名也行（Jekyll 支持），但部分外链工具会做编码处理，不便维护
- **不要**在文件名里出现空格或方括号

### 步骤 3：front matter

最小模板：

```yaml
---
title: "文章标题"
date: 2026-05-06 10:00:00 +0800
categories: [k8s]           # 第一个元素必须等于子目录名
tags: [kubernetes, runtime] # 可选，多个用数组
---
```

### 步骤 4：放进对应子目录

```
_posts/k8s/2026-05-06-my-new-post.md
```

校验：本地起 `bundle exec jekyll serve`，访问 `http://localhost:4000/k8s/my-new-post/`。

---

## 4. 专栏（series）字段

专栏功能由 front matter 三个字段驱动，**与目录结构无关**。聚合页面在 `_pages/series.html`，文末导航在 `_includes/series-nav.html`。

```yaml
series: "云原生网络专栏"               # 显示名
series_id: "cloud-native-network"      # 唯一标识，全站只能一个值代表一个专栏
series_order: 3                        # 该专栏内顺序，从 1 开始
```

只要这三个字段填对，文章无论放在哪个子目录都能进同一专栏。例如 `network/` 和 `network/xds/` 下的文章都可以同属一个专栏。

---

## 5. 新增一个分类时怎么做

1. 在 `_posts/` 下创建子目录，目录名就是新分类名（如 `_posts/database/`）
2. 把第一篇文章 front matter 设 `categories: [database]`
3. **不需要**改 `_config.yml`、不需要改主题
4. 分类页 `/categories/` 会自动列出该分类（来自 `_pages/category-archive.md`）

---

## 6. 几个反模式（不要做）

- ❌ 把文章放进与 `categories[0]` **不一致**的子目录 → URL 会变、外链会断
- ❌ 在子目录名里用大写或空格 → URL 会带大写或 `%20`
- ❌ 用同一个 `series_id` 但 `series` 名字不一致 → 专栏聚合页标题会乱
- ❌ 修改已发布文章的标题或 categories（会改变 URL）→ 必须做的话，配合 `redirect_from` 加 301

---

## 7. 历史信息

- `ES.md`：早期草稿，无日期前缀、无 front matter，Jekyll 不会发布它，保留作为历史记录
- 2026-05-06 完成首次按分类分目录的整理，全部 31 篇文章 URL 保持不变
