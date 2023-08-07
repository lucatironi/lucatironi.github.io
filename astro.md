---
layout: page
sidebar: true
title: Astrophotography
astro_list:
  - name: Soul Nebula
    label: soul_nebula
  - name: Heart Nebula
    label: heart_nebula
  - name: Messier 31
    label: m31
  - name: Messier 27
    label: m27
  - name: Messier 13
    label: m13
  - name: Caldwell 33
    label: c33
  - name: Messier 92
    label: m92
  - name: Cygnus Wall
    label: cygnus_wall
  - name: Caldwell 4
    label: c4
  - name: Elephant Trunk
    label: elephant_trunk
---

<div id="gallery">
{% for item in page.astro_list %}
  <a class="gallery-item" href="/astro/{{ item.label }}">
    <img src="/assets/astro/{{ item.label }}_small.jpg" alt="{{ item.name }}">
    <div class="overlay">
      <div class="text">{{ item.name }}</div>
    </div>
  </a>
{% endfor %}
</div>