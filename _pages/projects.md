---
title: Projects
layout: default
permalink: /projects/
published: true
---
<a href="{{ '/' | relative_url }}" class="back-link">← Home</a>

<h1 class="projects-heading">Projects</h1>
<p class="projects-intro">Research, competitions and engineering work. Open any project for the full write-up.</p>

{% assign projects = site.projects | sort: "date" | reverse %}
<div class="project-grid">
  {% for project in projects %}
    {% if project.redirect %}
      {% assign href = project.redirect %}
    {% else %}
      {% assign href = project.url | relative_url %}
    {% endif %}
  <a class="project-card" href="{{ href }}"{% if project.redirect %} target="_blank" rel="noopener noreferrer"{% endif %}>
    {% if project.cover %}
    <div class="project-card-media">
      <img src="{{ project.cover | relative_url }}" alt="{{ project.title }}" loading="lazy">
      {% if project.badge %}<span class="project-card-badge">{{ project.badge }}</span>{% endif %}
    </div>
    {% endif %}
    <div class="project-card-body">
      {% if project.period %}<p class="project-card-meta">{{ project.period }}</p>{% elsif project.date %}<p class="project-card-meta">{{ project.date | date: "%B %Y" }}</p>{% endif %}
      <h2 class="project-card-title">{{ project.title }}</h2>
      <p class="project-card-desc">{{ project.description }}</p>
      {% if project.tags %}
      <ul class="tag-list">
        {% for tag in project.tags limit: 4 %}<li>{{ tag }}</li>{% endfor %}
      </ul>
      {% endif %}
    </div>
  </a>
  {% endfor %}
</div>
