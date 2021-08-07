---
layout: page
title: 书签
description: 我的收藏
keywords: 书签
comments: true
menu: 链接
permalink: /links/
---

> 我的收藏

<ul>
{% for link in site.data.links %}
  <li><a href="{{ link.url }}" target="_blank">{{ link.name}}</a></li>
{% endfor %}
</ul>

<!-- > 友情链接

<ul>
{% for link in site.data.links %}
  {% if link.src == 'www' %}
  <li><a href="{{ link.url }}" target="_blank">{{ link.name}}</a></li>
  {% endif %}
{% endfor %}
</ul> -->
