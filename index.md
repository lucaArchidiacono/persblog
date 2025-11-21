---
layout: default
title: Home
---

archiluc's notes and thoughts

## Pages

{% assign subpages = site.pages | where_exp: "p", "p.permalink != '/' and p.layout == 'page'" | sort: "title" %}
<ul>
{% for p in subpages %}
  <li><a href="{{ p.url | relative_url }}">{{ p.title }}</a>{% if p.description %} – {{ p.description }}{% endif %}</li>
{% endfor %}
</ul>
