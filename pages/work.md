---
layout: page
title: Work Experience
permalink: /work/
---

Below are my professional experiences in tech.

{% assign sorted_work = site.work | sort: 'date' | reverse %}

{% comment %}
Limit to 10 items on this page, like homepage
{% endcomment %}
{% for work in sorted_work limit:10 %}

<div class="post-card">
  <h2><a href="{{ work.url | relative_url }}">{{ work.title }}</a></h2>
  <p><small>{{ work.date | date: "%B %d, %Y" }}</small></p>
  <p>{{ work.content | strip_html | truncatewords: 30 }}</p>
  
  {% if work.tags %}
    {% for tag in work.tags %}
      <span class="post-badge">{{ tag }}</span>
    {% endfor %}
  {% endif %}
</div>
{% endfor %}

<p><a href="{{ '/archives/#work-experiences' | relative_url }}">See All Work Experience →</a></p>
