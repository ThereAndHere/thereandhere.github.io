---
layout: page
title: Posts
---
  {% for post in site.posts %}
  <div>
    <h3><a href="{{ post.url }}">{{ post.title }}</a><span class="posts-date">{{ post.date | date_to_string }}</span></h3>
  </div>
  {% endfor %}
