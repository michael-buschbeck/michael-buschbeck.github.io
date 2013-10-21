---
layout: page
title: Program all the things!
---

## Hi!

My name is **Michael Buschbeck**.
I'm a software developer currently living and working in [Frankfurt](http://en.wikipedia.org/wiki/Frankfurt).

## Posts

<ul class="posts">
  {% for post in site.posts %}
    <li><span>{{ post.date | date_to_string }}</span> &raquo; <a href="{{ BASE_PATH }}{{ post.url }}">{{ post.title }}</a></li>
  {% endfor %}
</ul>
