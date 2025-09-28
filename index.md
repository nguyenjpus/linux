---
layout: default
---

Welcome! I’m documenting my progress toward earning the **CompTIA Linux+**, **CompTIA Server+**, and **Cisco CCNA** certifications through work experience and self-study. This site, built with GitHub Pages and Jekyll, captures my notes, reflections, and projects as I develop practical tech skills. My goals are to achieve Linux+ by early 2026, Server+ by July 2026, and CCNA by 2028 to advance my career in data center, system administration, and networking.

## About This Site

- **Purpose**: Track my learning in Linux+ topics, such as system administration, troubleshooting, and automation.
- **Content**: Blog posts, code snippets, study summaries, and professional experiences.
- **Focus**: Hands-on Linux skills, like mastering the initramfs stage in the boot process.

Explore my posts and work experiences below to follow my journey!

---

## Latest Posts

{% assign sorted_posts = site.posts | sort: "date" | reverse %}
{% for post in sorted_posts limit:10 %}

  <h2><a href="{{ post.url | relative_url }}">{{ post.title }}</a></h2>
  <p><small>{{ post.date | date: "%B %d, %Y" }}</small></p>
  <p>{{ post.excerpt }}</p>
{% endfor %}

<p><a href="{{ '/archives/' | relative_url }}">See All Posts →</a></p>

---

## Work Experience

Discover my professional journey in tech [here]({{ '/work/' | relative_url }}).
