---
layout: page
title: Post Index
permalink: /postindex/
---
<ul>
  {% for post in site.posts %}
    <li>
        <a href="{{ post.url | prepend: site.baseurl }}">
            {{ post.date | date: "%B %-d, %Y" }} - {{ post.title }}
        </a>
    </li>
  {% endfor %}
</ul>