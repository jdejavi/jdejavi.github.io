---
layout: default
title: Inicio
lang: es
permalink: /es/
---

<section class="hero">
  <h1>0xm4t1</h1>
  <p class="hero-sub">
    Seguridad Ofensiva · Writeups CTF · Investigación Técnica
  </p>
</section>

<section class="recent">
  <h2>Últimos Writeups</h2>

  {% assign posts_es = site.posts | where: "lang", "es" %}
  {% for post in posts_es limit:5 %}
    <div class="post-card">
      <a href="{{ post.url }}">
        <h3>{{ post.title }}</h3>
      </a>
      <p class="post-meta">
        {{ post.date | date: "%d %b %Y" }}
      </p>
    </div>
  {% endfor %}
</section>
