---
layout: default
title: Mind Archipelago
---
# Recent Posts

<ul class="posts">
  {% for post in site.posts %}
    <li>
      <article>
        • <a href="{{ post.url | relative_url }}">{{ post.title }}</a>
      </article>
    </li>
  {% endfor %}
</ul>
