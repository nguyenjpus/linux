---
layout: home
---

# My Learning Journey

Hi! I'm documenting my progress in studying for the **CompTIA Linux+** certification and other tech-related skills from work and self-study. This site, built with GitHub Pages and Jekyll, will include notes, reflections, and projects.

## About This Site

- **Purpose**: Track my learning, especially Linux+ topics like system administration and troubleshooting.
- **Content**: Blog posts, code snippets, study summaries, and work experiences.
- **Focus**: Practical Linux skills, like understanding the initramfs stage in the boot process.
  Check out my posts and work experiences below to follow along!

## Latest Posts

{% assign sorted_posts = site.posts | sort: 'date' | reverse %}
{% for post in sorted_posts %}

  <h2><a href="{{ site.baseurl }}{{ post.url }}">{{ post.title }}</a></h2>
  <p>{{ post.date | date: "%B %d, %Y" }}</p>
  <p>{{ post.content | strip_html | truncatewords: 30 }}</p>
{% endfor %}
## Work Experience
Explore my professional journey in tech [here]({{ site.baseurl }}/work/).
