---
layout: page
title: Posts
---


<!-- <ul>
  {% assign recent_posts = site.posts | limit:5 %}
  {% for post in recent_posts %}
    <li>
      <a href="{{ post.url | absolute_url }}">{{ post.title }}</a>
      <small>{{ post.date | date: "%B %d, %Y" }}</small>
    </li>
  {% endfor %}
</ul> -->

<ul>
  {% for post in site.posts %}
    <li>
      <a href="{{ post.url | absolute_url }}">{{ post.title }}</a>
      <small>{{ post.date | date: "%B %d, %Y" }}</small>
    </li>
  {% endfor %}
</ul>
