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

          <div class="row">
            <div class="col-md-6">
              <h2>{{ site.author.name }}</h2>
            </div>
            <div class="col-md-6">
              <ul class="list-inline">
                <li>
                    <a href="{{ BASE_PATH }}/atom.xml" class="social-link">
                        <span class="fa-stack fa-lg">
                            <i class="fa fa-circle fa-stack-2x"></i>
                            <i class="fa fa-rss fa-stack-1x fa-inverse"></i>
                        </span>
                    </a>
                </li>
                <li>
                    <a href="https://twitter.com/{{ site.author.twitter }}" class="social-link">
                        <span class="fa-stack fa-lg">
                            <i class="fa fa-circle fa-stack-2x"></i>
                            <i class="fa fa-twitter fa-stack-1x fa-inverse"></i>
                        </span>
                    </a>
                </li>
                <li>
                    <a href="https://github.com/{{ site.author.github }}" class="social-link">
                        <span class="fa-stack fa-lg">
                            <i class="fa fa-circle fa-stack-2x"></i>
                            <i class="fa fa-github fa-stack-1x fa-inverse"></i>
                        </span>
                    </a>
                </li>
                <li>
                    <a href="http://www.linkedin.com/pub/bryan-bende/7/76/600" class="social-link">
                        <span class="fa-stack fa-lg">
                            <i class="fa fa-circle fa-stack-2x"></i>
                            <i class="fa fa-linkedin fa-stack-1x fa-inverse"></i>
                        </span>
                    </a>
                </li>
                <li>
                    <a href="http://www.slideshare.net/BryanBende/presentations" class="social-link">
                        <span class="fa-stack fa-lg">
                            <i class="fa fa-circle fa-stack-2x"></i>
                            <i class="fa fa-slideshare fa-stack-1x fa-inverse"></i>
                        </span>
                    </a>
                </li>
              </ul>
            </div>
          </div>

          <div class="row">
            <div class="col-md-12">
              <p class="post-meta">Software developer interested in Java, big-data, and open-source development / Apache NiFi PMC & Committer</p>
            </div>
          </div>

        </div>
      </div>

    </div>
  </div>
</div>

<div class="row">
  <div class="col-md-11">
  <hr/>
  {% for post in site.posts offset: 0 limit: 6  %}
      <div class="post-preview">
        <a href="{{ post.url }}">
          <h2 class="post-title">{{ post.title }}</h2>
          <h3 class="post-subtitle">
              {{ post.content | strip_html | truncatewords:20}}
          </h3>
        </a>
        <p class="post-meta">Posted by {% if post.author %}{{ post.author }}{% else %}{{ site.title }}{% endif %} on {{ post.date | date: "%B %-d, %Y" }}</p>
      </div>
      <hr/>
   {% endfor %}
  </div>
</div>
