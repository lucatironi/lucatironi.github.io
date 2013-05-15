---
layout: page
title: Homepage
tagline:
description:
---
{% include JB/setup %}

I'm Luca, I'm 29 and I live in Milano, Italy. I like to code.

Here's some of the stuff I'd like to share.

<ul class="posts">
{% for post in site.posts %}
  <li>
    <strong class="muted">{{ post.date | date_to_long_string }}</strong><br>
    <h3><a href="{{ post.url }}">{{ post.title }}</a></h3>
    {% if post.description %}<p>
      {{ post.description }}
      {% if post.updated %}<br><strong>{{ post.updated }}</strong>{% endif %}
    </p>{% endif %}
  </li>
{% endfor %}
</ul>