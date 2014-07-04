---
layout: page
title: Welcome!
tagline: null
published: true
---

{% include JB/setup %}

<div>
{% assign page = site.posts.first %}
{% assign content = page.content %}
{% include themes/bootstrap/post.html %}
</div>
