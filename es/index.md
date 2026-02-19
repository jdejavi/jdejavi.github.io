---
layout: default
title: Inicio
---

<section class="hero">
  <h1>0xm4t1</h1>
  <p class="hero-sub">
    Seguridad Ofensiva · Resoluciones de CTF
  </p>
</section>

<section class="stats">
  <div class="stat">
    <span class="stat-number">x</span>
    <span class="stat-label">Máquinas de HTB</span>
  </div>
  <div class="stat">
    <span class="stat-number">1</span>
    <span class="stat-label">Certificaciones</span>
  </div>
  <div class="stat">
    <span class="stat-number">x</span>
    <span class="stat-label">Escaladas de privilegios</span>
  </div>
</section>

<section class="recent">
  <h2>Últimas resoluciones</h2>

  {% for post in site.posts limit:5 %}
    <div class="post-card">
      <a href="{{ post.url }}">
        <h3>{{ post.title }}</h3>
      </a>
      <p class="post-meta">
        {{ post.date | date: "%d %b %Y" }} · {{ post.tags | join: ", " }}
      </p>
    </div>
  {% endfor %}
</section>
