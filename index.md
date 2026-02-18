---
layout: default
title: Home
---

archiluc's notes and thoughts

Contact: [mail@archiluc.ch](mailto:mail@archiluc.ch)

## Pages

<ul>
{% for p in site.pages %}
  {% if p.permalink != '/' and p.layout == 'page' %}
    <li><a href="{{ p.url | relative_url }}">{{ p.title }}</a>{% if p.description %} – {{ p.description }}{% endif %}</li>
  {% endif %}
{% endfor %}
</ul>
