---
layout: page
title: 欢迎光临
tagline: 博客装修中...
---
{% include JB/setup %}
 
## 我的文章

<ul class="posts">
  {% for post in site.posts %}
    <li><span>{{ post.date | date_to_string }}</span> &raquo; <a href="{{ BASE_PATH }}{{ post.url }}">{{ post.title }}</a></li>
  {% endfor %}
</ul>
