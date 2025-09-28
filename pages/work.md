---
layout: page
title: Work Experience
permalink: /work/
---

# Work Experience

Below are highlights from my professional experiences in tech.

{% assign sorted_work = site.work | sort: 'date' | reverse %}
{% for work in sorted_work limit:10 %}

  <h2><a href="{{ work.url | relative_url }}">{{ work.title }}</a></h2>
  <p><small>{{ work.date | date: "%B %d, %Y" }}</small></p>
  <p>{{ work.excerpt }}</p>
{% endfor %}

<p><a href="{{ '/archives/#work-experiences' | relative_url }}">See All Work Experience →</a></p>
