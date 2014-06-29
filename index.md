---
layout: page
title: Welcome!
tagline: null
published: true
---

{% include JB/setup %}

<!--  
<div class="jumbotron" style="opacity: 0.8; color:white; background-image: url(grand-canyon.jpg); background-size: 100%; background-repeat: no-repeat">
   <div class="container for-about">
   <h1>Welcome!</h1>
	<p>Software Development, Big Data, & Technology</p>
   </div>
</div>
-->

<div>
{% assign page = site.posts.first %}
{% assign content = page.content %}
{% include themes/bootstrap/post.html %}
</div>
