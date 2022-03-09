---
layout: page
title: Archive
---
<!--Thanks Nikhita! https://www.nikhita.dev/build-blog-using-github-jekyll#archive-->

{% for post in site.posts %}
  * {{ post.date | date_to_string }} &raquo; [ {{ post.title }} ]({{ post.url }})
{% endfor %}
