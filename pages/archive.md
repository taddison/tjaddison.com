---
layout: page
title: Archive
permalink: /archive/
---
{% assign postsByYearMonth = site.posts | group_by_exp:"post", "post.date | date: '%Y %b'"  %}
{% for yearMonth in postsByYearMonth %}
  <h3>{{ yearMonth.name }}</h3>
  <div>
    <ul>
      {% for post in yearMonth.items %}
        <li><a href="{{ post.url | prepend: site.baseurl }}">{{ post.title }}</a></li>
      {% endfor %}
    </ul>
  </div>
{% endfor %}