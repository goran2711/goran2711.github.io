---
layout: default
title: Mind Archipelago
---
# My Journaling Method

For now, the content on this page is very sporadic. At some point I would like
to go over and polish it all to give a nice overview of my approach to journaling.

<hr>

## Recent Posts

<ul class="posts">
  {% for post in site.posts %}
    <li>
      <h3><a href="{{ post.url | relative_url }}">{{ post.title }}</a></h3>
      <p class="post-date">{{ post.date | date: "%B %-d, %Y" }}</p>
      <p>{{ post.excerpt }}</p>
    </li>
  {% endfor %}
</ul>