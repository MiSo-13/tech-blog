---
layout: default
title: Home
---

# Tech Blog

{% if site.posts.size > 0 %}
<section class="post-list">
  {% for post in site.posts %}
  <article class="post-card">
    <p class="post-date">{{ post.date | date: "%Y.%m.%d" }}</p>
    <h2><a href="{{ post.url | relative_url }}">{{ post.title }}</a></h2>
    {% if post.description %}
    <p>{{ post.description }}</p>
    {% endif %}
  </article>
  {% endfor %}
</section>
{% else %}
아직 발행된 글이 없습니다.
{% endif %}
