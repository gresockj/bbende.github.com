---
layout: page
title: bryanbende.com
tagline: null
published: true
theme :
  name : bootstrap
---

{% include JB/setup %}

<div class="row">
  <div class="col-md-11">
    <div class="well">

      <div class="row">
        <div class="col-xs-3 col-md-2">
          <img src="{{ BASE_PATH }}/self_photo_bw.jpg" class="img-responsive img-rounded">
        </div>
        <div class="col-xs-9 col-md-10">
          <h2>{{ site.author.name }}</h2>
          <p class="post-meta">Software developer interested in Java, big-data, and open-source development / Apache NiFi PMC & Committer</p>
        </div>
      </div>

    </div>
  </div>
</div>

<div class="row">
  <div class="col-md-11">
  <hr/>
  {% for post in site.posts offset: 0 limit: 5  %}
      <div class="post-preview">
        <a href="{{ post.url }}">
          <h2 class="post-title">{{ post.title }}</h2>
          <h3 class="post-subtitle">
              {{ post.content | strip_html | truncatewords:10}}
          </h3>
        </a>
        <p class="post-meta">Posted by {% if post.author %}{{ post.author }}{% else %}{{ site.title }}{% endif %} on {{ post.date | date: "%B %-d, %Y" }}</p>
      </div>
      <hr/>
   {% endfor %}
  </div>
</div>
