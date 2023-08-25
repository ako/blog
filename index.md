---
layout: post
title: Ako's notes
---

# Ako's notes

<ul>
  {% for post in site.posts %}
    <li>
      <a href="blog/{{ post.url }}">{{ post.title }}</a>
    </li>
  {% endfor %}
</ul>