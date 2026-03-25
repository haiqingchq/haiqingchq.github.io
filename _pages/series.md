---
layout: single
title: "专栏"
permalink: /series/
author_profile: true
---

## 使用说明

写新文章时，只要在 front matter 里增加并维护这三个字段，就会自动归入专栏目录并生成文末导航：

- `series`：专栏名称（如 `云原生网络专栏`）
- `series_id`：专栏唯一标识（如 `cloud-native-network`）
- `series_order`：专栏内顺序（数字，1 开始）

---

{% assign series_enabled_posts = site.posts | where_exp: "post", "post.series_id" %}
{% assign grouped_series = series_enabled_posts | group_by: "series_id" %}

{% if grouped_series.size > 0 %}
  {% for group in grouped_series %}
    {% assign series_posts = group.items | sort: "series_order" %}
    {% assign series_name = series_posts[0].series | default: group.name %}
    <section id="series-{{ group.name }}" style="margin-bottom: 2rem;">
      <h2>{{ series_name }}</h2>
      <p>共 {{ series_posts.size }} 篇</p>
      <ol>
        {% for post in series_posts %}
          <li>
            第 {{ post.series_order }} 篇：
            <a href="{{ post.url | relative_url }}">{{ post.title }}</a>
          </li>
        {% endfor %}
      </ol>
    </section>
  {% endfor %}
{% else %}
当前暂无专栏文章。
{% endif %}
