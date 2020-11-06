---
layout: page
permalink: /blog/
title: Blog archive
---

<ul class="posts">
{% assign postsByYear =
    site.posts | group_by_exp:"post", "post.date | date: '%Y'" %}
{% for year in postsByYear %}
  <h1>{{ year.name }}</h1>
    <ul>
      {% for post in year.items %}
        <li><span>{{ post.date | date_to_string }}</span> » <a href="{{ post.url }}" title="{{ post.title }}">{{ post.title }}</a></li>
      {% endfor %}
    </ul>
{% endfor %}
</ul>
