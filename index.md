---
layout: page
title: Slipperless in Suburbia
tagline: Supporting tagline
---
{% include JB/setup %}

Hi, I'm Kelly Rauchenberger, and this is my personal blog. I'm a second year computer science student at Carnegie Mellon University. Here, I write about programming adventures, music I enjoy, and other musings.

* [Blog feed](/atom.xml) ![](/assets/themes/tom/images/atom.png)
* [My twitter](http://twitter.com/fefferburbia) ![](/assets/themes/tom/images/twitter_alt.png)
* [My last.fm](http://last.fm/user/fefferburbia) ![](/assets/themes/tom/images/lastFM.png)

<br />
## Recent posts

<ul class="posts">
  {% for post in site.posts %}
    <li><span>{{ post.date | date_to_string }}</span> &raquo; <a href="{{ BASE_PATH }}{{ post.url }}">{{ post.title }}</a></li>
  {% endfor %}
</ul>