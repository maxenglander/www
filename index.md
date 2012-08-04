---
layout: default
title: squarkle
---

{% for post in site.posts %}
  {{ post.date | date: "%Y %B %d" }} - [{{post.title}}]({{post.url}})
{% endfor %}
