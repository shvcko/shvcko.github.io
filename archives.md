---
layout: page
title: 归档
permalink: /archives/
---

<div class="post">
  <div class="post-archive">
  {% for post in site.posts %}
    {% capture currentyear %}{{post.date | date: "%Y"}}{% endcapture %}
    {% if currentyear != year %}
        <h2>{{ currentyear }}</h2>
        {% capture year %}{{currentyear}}{% endcapture %}
    {% endif %}
    <ul class="listing">
      <li>
      <span class="date">{{ post.date | date: "%Y/%m/%d" }}</span>
      <a href="{{ post.url | prepend: site.baseurl }}">
      {% if post.title %}
                {{ post.title }}
          {% else %}
                {{ site.page_no_title }}
          {% endif %}
          </a>
        </li>
    </ul>
  {% endfor %}
  </div>
</div>
