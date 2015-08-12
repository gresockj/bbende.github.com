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
        <h4 class="panel-title"><b>{{ site.author.name }}</b></h4>
      </div>
      <div class="panel-body ">
        <div class="row">
          <div class="col-xs-2 col-md-2">
            <img src="{{ BASE_PATH }}/self_photo_2.jpg" class="img-responsive img-rounded">
          </div>
          <div class="col-xs-2 col-md-2">
              <a href="https://twitter.com/{{ site.author.twitter }}">
                  <img src="{{ ASSET_PATH }}/resources/social_icons/twitter.png" class="img-responsive" />
              </a>
          </div>
          <div class="col-xs-2 col-md-2">
              <a href="https://github.com/{{ site.author.github }}">
                  <img src="{{ ASSET_PATH }}/resources/social_icons/github.png" class="img-responsive" />
              </a>
          </div>
          <div class="col-xs-2 col-md-2">
              <a href="http://www.linkedin.com/pub/bryan-bende/7/76/600">
                  <img src="{{ ASSET_PATH }}/resources/social_icons/linkedin.png" class="img-responsive" />
              </a>
          </div>
          <div class="col-xs-2 col-md-2">
              <a href="http://www.slideshare.net/BryanBende/presentations">
                  <img src="{{ ASSET_PATH }}/resources/social_icons/slideshare.png" class="img-responsive" />
              </a>
          </div>
          <div class="col-xs-2 col-md-2">
              <a href="{{ BASE_PATH }}/rss.xml">
                  <img src="{{ ASSET_PATH }}/resources/social_icons/rss.png" class="img-responsive" />
              </a>
          </div>
        </div>
      </div>
</div>

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
    <a class="twitter-timeline" href="https://twitter.com/BBende" data-widget-id="627527690225061888">Tweets by @BBende</a>
    <script>!function(d,s,id){var js,fjs=d.getElementsByTagName(s)[0],p=/^http:/.test(d.location)?'http':'https';if(!d.getElementById(id)){js=d.createElement(s);js.id=id;js.src=p+"://platform.twitter.com/widgets.js";fjs.parentNode.insertBefore(js,fjs);}}(document,"script","twitter-wjs");</script>
  </div>
</div>
