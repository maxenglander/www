---
layout: default
title: maxenglander.com
---

{% for post in site.posts %}
  {% if post.external_url %}
  {{ post.date | date: "%Y %B %d" }} - <a href="{{ post.external_url }}" target="_blank">{{post.title}}</a> (external)
  {% else %}
  {{ post.date | date: "%Y %B %d" }} - [{{post.title}}]({{post.url}})
  {% endif %}
{% endfor %}
