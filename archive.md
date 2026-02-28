---
layout: page
title: Archive
permalink: /archive/
---

{% assign postsByYear = site.posts | group_by_exp:"post", "post.date | date: '%Y'" %}

{% for year in postsByYear %}
## {{ year.name }}

<ul>
  {% for post in year.items %}
    <li>
      <span>{{ post.date | date: "%b %d" }}</span> —
      <a href="{{ post.url }}">{{ post.title }}</a>
    </li>
  {% endfor %}
</ul>
{% endfor %}