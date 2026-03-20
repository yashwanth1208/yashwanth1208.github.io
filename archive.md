---
layout: default
title: Archive
permalink: /archive
---

<style>
.writings-list {
  list-style: none;
  padding: 0;
}

.writings-list li {
  display: flex;
  align-items: center;
  margin-bottom: 0.75rem;
  gap: 1.5rem;
}

.writings-list span {
  min-width: 100px;
  flex-shrink: 0;
}

.writings-list a {
  text-decoration: none;
}
</style>

# Archive

All writings organized by date.

<ul class="writings-list">
{% for writing in site.writings %}
{% if writing.in_archive %}
<li><span>{{ writing.date | date: "%Y-%m-%d" }}</span> <a href="{{ writing.url }}">{{ writing.title }}</a></li>
{% endif %}
{% endfor %}
</ul>
