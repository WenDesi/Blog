---
layout: page
title: Archive
permalink: /archive/
---

<div class="archive">
  {% assign postsByYear = site.posts | group_by_exp: "post", "post.date | date: '%Y'" %}

  {% for year in postsByYear %}
    <div class="archive-year">
      <h2>{{ year.name }}</h2>
      <ul class="archive-list">
        {% for post in year.items %}
          <li>
            <span class="archive-date">{{ post.date | date: "%b %-d" }}</span>
            <a href="{{ post.url | relative_url }}">{{ post.title }}</a>
          </li>
        {% endfor %}
      </ul>
    </div>
  {% endfor %}
</div>
