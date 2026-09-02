---
layout: post
title: "Blog"
eyebrow: "Robot Air Hockey"
blog: true
permalink: /blog/
description: "Posts about the robot air hockey system"
---

Posts on the air hockey system — what we built, how it works, and what we found along the
way. For the system itself, start with the [overview]({{ '/' | relative_url }}).

<ul class="post-list">
  {% for post in site.posts %}
  <li>
    <a class="title" href="{{ post.url | relative_url }}">{{ post.title }}</a>
    <span class="post-meta">
      <time datetime="{{ post.date | date_to_xmlschema }}">{{ post.date | date: "%B %-d, %Y" }}</time>{% if post.author %}
      <span class="sep">&middot;</span> {{ post.author }}{% endif %}
    </span>{% if post.description %}
    <span class="summary">{{ post.description }}</span>{% endif %}
  </li>
  {% endfor %}
</ul>
