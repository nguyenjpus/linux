---
layout: page
title: Work Experience
permalink: /work/
---

# Professional Experiences in Tech

Below are my professional experiences, organized from most recent:

{% assign sorted_work = site.work | sort: "date" | reverse %}
{% for work in sorted_work %}

<div class="post-card">
  {% for tag in work.tags %}
    <span class="post-badge">{{ tag }}</span>
  {% endfor %}
  <h2><a href="{{ work.url | relative_url }}">{{ work.title }}</a></h2>
  <p><small>{{ work.date | date: "%B %d, %Y" }}</small></p>
  <p>{{ work.content | strip_html | truncatewords: 50 }}</p>
</div>

{% endfor %}
