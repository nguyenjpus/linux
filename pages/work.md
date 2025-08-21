---
layout: page
title: Work Experience
permalink: /work/
---

# Work Experience

Below are my professional experiences in tech.
{% assign sorted_work = site.work | sort: 'date' | reverse %}
{% for work in sorted_work %}

  <h2><a href="{{ work.url }}">{{ work.title }}</a></h2>
  <p>{{ work.date | date: "%B %d, %Y" }}</p>
  <p>{{ work.content | strip_html | truncatewords: 30 }}</p>
{% endfor %}
