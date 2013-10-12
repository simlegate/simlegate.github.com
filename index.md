---
layout: page
title: Hello World!
tagline: Supporting tagline
---

我是一位程序员，主要从事Ruby开发，目前就职于成都奥震电子科技有限公司  
平时主要研究[Ruby](https://www.ruby-lang.org/)，[Clojure](http://clojure.org/)编程语言，对AI(人工智能)比较感兴趣  
常年活跃在[Github](https://github.com/)，[Ruby China](http://ruby-china.org/)中  

***

<ul class="posts">
  {% for post in site.posts %}
    <li>
    <!-- <span>{{ post.date | date_to_string }}</span> &raquo; <a href="{{ BASE_PATH }}{{ post.url }}">{{ post.title }}</a> -->
     <a href="{{ BASE_PATH }}{{ post.url }}">{{ post.title }}</a>
    </li>
  {% endfor %}
</ul>
