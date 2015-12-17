---
layout: page
title: 我活在世上/无非想要明白些道理/遇见些有趣的事
tagline:  by 王小波
---
{% include JB/setup %}

<div>
   <img src="/images/index/cover_min.jpg" class="img-responsive">
</div>

## 我的文章

<ul class="posts">
  {% for post in site.posts %}
    <li><span>{{ post.date | date_to_long_string }}</span> &raquo; <a href="{{ BASE_PATH }}{{ post.url }}">{{ post.title }}</a></li>
  {% endfor %}
</ul>
