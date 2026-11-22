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

.intro-tagline {
  font-size: 1.6rem;
  font-weight: 700;
  line-height: 1.3;
  margin: 0.5rem 0 1rem 0;
}
</style>

<p class="intro-tagline">Writing code that holds other people's money.</p>

[CV](/CV) - [GitHub](https://github.com/yashwanth1208) - [LinkedIn](https://www.linkedin.com/in/yashwanth-sai-sollu-009a50201/) - [X](https://x.com/Yash_1208)

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
