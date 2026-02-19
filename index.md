---
layout: default
title: Homepage
---

# 👋 Welcome to QbProg's C++ and Geometry blog

Welcome to my blog! Here I will publish some articles and news about computational geometry (CAD and CAM) and c++, mostly about integrations and advanced techniques.


<ul class="post-list">
  {% for post in site.posts %}
    <li>
      <h2><a href="{{ post.url | relative_url }}">{{ post.title }}</a></h2>
      <span class="post-meta">{{ post.date | date: "%b %d, %Y" }}</span>
      
      <div class="post-excerpt">
        {{ post.excerpt | strip_html | truncatewords: 60 }}
      </div>
      
      <a href="{{ post.url | relative_url }}">Leggi di più...</a>
      <hr>
    </li>
  {% endfor %}
</ul>