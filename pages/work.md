---
layout: page
title: Work Experience
permalink: /work/
---

# Just something I want to remind myself about.

{% assign sorted_work = site.work | sort: 'date' | reverse %}
{% for work in sorted_work %}

  <h2><a href="{{ site.baseurl }}{{ work.url }}">{{ work.title }}</a></h2>
  <p>{{ work.date | date: "%B %d, %Y" }}</p>
  <p>{{ work.content | strip_html | truncatewords: 30 }}</p>
{% endfor %}
