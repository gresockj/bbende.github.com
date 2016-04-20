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
  <div class="col-md-8">

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

  <div class="col-md-4">
    {% include JB/personal_info %}
    <div class="well">
      {% include JB/tag_cloud %}
      <div class='clear'></div>
    </div>

    <!--
    <a class="twitter-timeline" href="https://twitter.com/BBende" data-widget-id="627527690225061888">Tweets by @BBende</a>
    <script>!function(d,s,id){var js,fjs=d.getElementsByTagName(s)[0],p=/^http:/.test(d.location)?'http':'https';if(!d.getElementById(id)){js=d.createElement(s);js.id=id;js.src=p+"://platform.twitter.com/widgets.js";fjs.parentNode.insertBefore(js,fjs);}}(document,"script","twitter-wjs");</script>
    -->
  </div>
</div>
