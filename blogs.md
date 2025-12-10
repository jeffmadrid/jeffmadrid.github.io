---
title: "/blogs"
layout: default
permalink: /blogs/
---

# Blogs

A collection of blog posts that I learn and experience in my journey in Software Engineering 💻

{% for post in site.posts %}
  <article>
    <h2><a href="{{ post.url | relative_url }}">{{ post.title }}</a></h2>
    <time datetime="{{ post.date | date: '%Y-%m-%d' }}">
      {{ post.date | date: "%B %d, %Y" }}
    </time>
    {% if post.excerpt %}
      <p>{{ post.excerpt | strip_html | truncatewords: 50 }}</p>
    {% endif %}
  </article>
{% endfor %}
