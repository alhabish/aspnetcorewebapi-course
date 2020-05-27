---
layout: default
title: "الصفحة الرئيسية"
---

# قائمة الدروس

<div style="float: left;">

<ul>
  {% assign posts = site.posts | sort: 'date' %}
  {% for post in posts %}
    <li>
      <a href="{{site.url}}{{post.url}}">{{ post.title }}</a>
    </li>
  {% endfor %}
</ul>

</div>
<!--[back](./)-->

---
