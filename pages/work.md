---
layout: page
title: Work Experience
permalink: /work/
---

# Work Experience

Below are my professional experiences in tech.

{% for work in site.work %}

  <h2><a href="{{ work.url }}">{{ work.title }}</a></h2>
  <p>{{ work.date | date: "%B %d, %Y" }}</p>
  <p>{{ work.content | strip_html | truncatewords: 30 }}</p>
{% endfor %}
