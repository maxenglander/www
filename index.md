---
layout: default
title: squarkle
---

{% for post in site.posts %}
  [{{post.title}}]({{post.url}})
{% endfor %}
