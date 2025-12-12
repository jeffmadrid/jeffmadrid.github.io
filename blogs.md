---
title: "/blogs"
layout: default
permalink: /blogs/
---

# Blogs

A collection of blog posts documenting my learning and experiences throughout my Software Engineering journey. 💻

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
