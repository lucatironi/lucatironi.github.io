---
layout: page
title: Homepage
tagline:
---
{% include JB/setup %}

I'm Luca, I'm 29 and I live in Milano, Italy. I like to code.

Here's some of the stuff I like to share.

<ul class="posts">
  {% for post in site.posts %}
    <li><span>{{ post.date | date_to_string }}</span> &raquo; <a href="{{ post.url }}">{{ post.title }}</a></li>
  {% endfor %}
</ul>