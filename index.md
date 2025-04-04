---
layout: default
title: Mind Archipelago
---
# Recent Posts

<ul class="posts">
  {% for post in site.posts %}
    <li>
      <h3><a href="{{ post.url | relative_url }}">{{ post.title }}</a></h3>
      <p class="post-date">{{ post.date | date: "%B %-d, %Y" }}</p>
      <p>{{ post.excerpt }}</p>
    </li>
  {% endfor %}
</ul>