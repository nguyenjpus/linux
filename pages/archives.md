---
layout: page
title: Archives
permalink: /linux/archives/
---

# Archives

{% assign posts_by_month = site.posts | group_by: 'date' | sort: 'name' | reverse %}
{% for month in posts_by_month %}

  <h2>{{ month.name | date: "%B %Y" }}</h2>
  <ul>
    {% for post in month.items %}
      <li><a href="{{ site.baseurl }}{{ post.url }}">{{ post.title }}</a> ({{ post.date | date: "%d %B" }})</li>
    {% endfor %}
  </ul>
{% endfor %}

## Work Archives

{% assign work_by_month = site.work | group_by: 'date' | sort: 'name' | reverse %}
{% for month in work_by_month %}

  <h2>{{ month.name | date: "%B %Y" }}</h2>
  <ul>
    {% for work in month.items %}
      <li><a href="{{ site.baseurl }}{{ work.url }}">{{ work.title }}</a> ({{ work.date | date: "%d %B" }})</li>
    {% endfor %}
  </ul>
{% endfor %}
