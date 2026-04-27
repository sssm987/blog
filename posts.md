---
layout: page
title: Posts
permalink: /posts/
---

{% for post in site.posts %}
  <article class="post-card">
    <div class="post-date">{{ post.date | date: "%Y.%m.%d" }}</div>
    <a class="post-title" href="{{ post.url | relative_url }}">{{ post.title }}</a>
    {% if post.description %}
      <div class="post-desc">{{ post.description }}</div>
    {% endif %}
    {% if post.categories and post.categories.size > 0 %}
      <div class="tags">{{ post.categories | join: " / " }}</div>
    {% endif %}
  </article>
{% endfor %}
