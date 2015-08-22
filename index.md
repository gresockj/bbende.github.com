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

    <div class="panel panel-default">
        <div class="panel-heading">
            <h2 class="panel-title"><b>Recent Posts</b></h2>
        </div>
        <ul class="list-group blog-list">
        {% for post in site.posts offset: 0 limit: 5  %}
            <li class="list-group-item">
              <h3>{{ post.title }}</h3>
              <span><b>{{ post.date | date_to_string }}</b></span> -
              {{ post.content | strip_html | truncatewords:30}}<br/>
              <a href="{{ post.url }}">Read More »</a>
          </li>
         {% endfor %}
          </ul>
    </div>
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
