---
layout: page
title: Archives
permalink: /archives/
---

# Archives

Below are my blog posts and work experiences, organized by month.

## Blog Posts

{% assign posts_by_month = site.posts | group_by_exp: 'post', 'post.date | date: "%Y-%m"' | sort: 'name' | reverse %}
{% for month in posts_by_month %}

  <h2>{{ month.name | date: "%B %Y" }}</h2>
  <ul>
    {% for post in month.items %}
      <li><a href="{{ site.baseurl }}{{ post.url }}">{{ post.title }}</a> ({{ post.date | date: "%d %B %Y" }})</li>
    {% endfor %}
  </ul>
{% endfor %}

## Work Experiences

{% assign work_by_month = site.work | group_by_exp: 'work', 'work.date | date: "%Y-%m"' | sort: 'name' | reverse %}
{% for month in work_by_month %}

  <h2>{{ month.name | date: "%B %Y" }}</h2>
  <ul>
    {% for work in month.items %}
      <li><a href="{{ site.baseurl }}{{ work.url }}">{{ work.title }}</a> ({{ work.date | date: "%d %B %Y" }})</li>
    {% endfor %}
  </ul>
{% endfor %}
