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
  margin-bottom: 0.5rem;
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
