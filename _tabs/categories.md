---
layout: page
icon: fas fa-stream
order: 1
permalink: /categories/
---

{% assign sorted_categories = site.categories | sort %}

{% for category in sorted_categories %}
  {% assign category_name = category | first %}
  {% assign posts = category | last %}

  <h2 id="{{ category_name | slugify }}">{{ category_name }}</h2>
  <ul>
  {% for post in posts %}
    <li><a href="{{ post.url | relative_url }}">{{ post.title }}</a> - {{ post.date | date: "%Y-%m-%d" }}</li>
  {% endfor %}
  </ul>
{% endfor %}
