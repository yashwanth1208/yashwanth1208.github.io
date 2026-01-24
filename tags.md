---
layout: default
title: Tags
permalink: /tags
---

# Tags

{% for tag in site.my_tags %}
{% assign tag_writings = site.writings | where_exp: 'item', 'item.tags contains tag.slug' | size %}
\# [{{ tag.title }}]({{ tag.url }}) ({{ tag_writings }})
{% endfor %}
