---
layout: default
title: Yashwanth's Vault
permalink: /
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

## Writings

{% if site.writings and site.writings.size > 0 %}
<ul class="writings-list">
  {% assign sorted_writings = site.writings | sort: 'date' | reverse %}
  {% for writing in sorted_writings %}
    <li>
      <span>{{ writing.date | date: "%Y-%m-%d" }}</span>
      <a href="{{ writing.url }}">{{ writing.title }}</a>
    </li>
  {% endfor %}
</ul>
{% else %}
<p><em>No writings yet. Coming soon.</em></p>
{% endif %}
