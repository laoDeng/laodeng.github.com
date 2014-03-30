---
layout: page
title: Galatica
tagline: The battlestar always in my heart.
---
{% include JB/setup %}

## New update

<ul class="posts">
  {% for post in site.posts limit:10 offset:0 %}
    <li class="listing-item">
      <time datetime="{{ post.date | date:"%Y-%m-%d" }}">{{ post.date | date:"%Y-%m-%d" }}</time>
      <a href="{{ post.url }}" title="{{ post.title }}">{{ post.title }}</a>
    </li>
  {% endfor %}
</ul>

