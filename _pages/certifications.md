---
title: Certifications
layout: default
permalink: /certifications/
published: true
---
<a href="{{ '/' | relative_url }}" class="back-link">← Home</a>

<h1 class="projects-heading">Certifications</h1>
<p class="projects-intro">Certificates of participation, language certifications and digital badges.</p>

<div class="cert-list">
  {% for cert in site.data.certifications %}
  <article class="cert-card" id="{{ cert.id }}">
    <div class="cert-media">
      {% if cert.image %}
      <img src="{{ cert.image | relative_url }}" alt="{{ cert.title }}" loading="lazy">
      {% else %}
      <svg class="cert-icon" viewBox="0 0 64 64" aria-hidden="true">
        <path d="M22 36 14 58l9-4 5 8 6-18M42 36l8 22-9-4-5 8-6-18" fill="currentColor" opacity=".55"/>
        <circle cx="32" cy="24" r="18" fill="none" stroke="currentColor" stroke-width="4"/>
        <path d="m32 13 3.4 7 7.6 1.1-5.5 5.4 1.3 7.6-6.8-3.6-6.8 3.6 1.3-7.6-5.5-5.4 7.6-1.1z" fill="currentColor"/>
      </svg>
      {% endif %}
    </div>
    <div class="cert-body">
      <p class="project-card-meta">{{ cert.date }}{% if cert.kind %} · {{ cert.kind }}{% endif %}</p>
      <h2 class="cert-title">{{ cert.title }}</h2>
      <p class="cert-issuer">{{ cert.issuer }}</p>
      {% if cert.description %}<p class="cert-desc">{{ cert.description }}</p>{% endif %}
      {% if cert.links %}
      <div class="cert-links">
        {% for link in cert.links %}
        {% assign first = link.url | slice: 0 %}
        <a class="button-pill{% if forloop.first %} is-primary{% endif %}" href="{% if first == '/' %}{{ link.url | relative_url }}{% else %}{{ link.url }}{% endif %}"{% unless link.url contains '/projects/' %} target="_blank" rel="noopener noreferrer"{% endunless %}>
          {% if link.icon %}<i class="{{ link.icon }}"></i>{% endif %} {{ link.label }}
        </a>
        {% endfor %}
      </div>
      {% endif %}
    </div>
  </article>
  {% endfor %}
</div>
