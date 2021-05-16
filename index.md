---
layout: default
title: oberien's Blog
---
<h1>Latest Posts</h1>

{% for post in site.posts %}
  <h3><a href="{{ post.url }}">{{ post.title }}</a></h3>
  {{ post.excerpt }}
{% endfor %}
