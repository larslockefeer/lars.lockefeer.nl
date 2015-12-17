---
layout: page
title: Links
permalink: /links/
---

<ul class="post-list">
  {% for post in site.posts %}
    {% if post.link %}
      <li>
        <span class="post-meta">{{ post.date | date: "%b %-d, %Y" }}</span>

        <h2>
          <a class="post-link" href="{{ post.link }}">&nbsp;&#10132; {{ post.title }}</a>
        </h2>

        <div class="post-content" itemprop="articleBody">
          {{ post.content }}
        </div>
      </li>
    {% endif %}
  {% endfor %}
</ul>
