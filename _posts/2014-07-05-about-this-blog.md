---
layout: post
title: "About This Blog"
description: ""
category: "General"
tags: [Jekyll, Bootstrap, Jekyll-Bootstrap, IntelliJ]
---
{% include JB/setup %}

I spend a significant amount of time reading technical information on the 
internet, and I've often felt that I should contribute something back to 
the community. I created this blog with the hope that it will motivate me 
to share some of my knowledge, and maybe something I write will eventually 
help someone else. For this first post I thought I would  share how I 
created this site while it was still fresh in my mind.

#### GitHub Pages

When I set out to create this site I researched what the most popular approaches 
for creating a blog were. As a developer and regular user of Git, I was naturally 
intrigued by the idea of hosting a blog directly on GitHub. For those who are 
unaware, [GitHub Pages](https://pages.github.com/) allows you to create one user 
or organization site, and unlimited project sites.

Creating a user or organization site is as simple as creating a repository named
username.github.io. You then clone the repository, create your content, commit, 
& push. At this point you will have a live site at http://username.github.io. One 
of the neat features of GitHub pages is that it supports Jekyll, a static website
generator. You can read more about Jekyll [here](http://jekyllrb.com/).

#### Jekyll-Bootstrap

While doing research I came across [Jekyll-Bootstrap](http://jekyllbootstrap.com/), a 
framework for creating Jekyll based blogs, backed by Twitter Bootstrap. Since I didn't
know anything about Jekyll, and already had some minor experience with Bootstrap, this
seemed like a good fit. 

At the time of writing this, the main Jekyll-Bootstrap repository
was based off Bootstrap 2, and didn't seem to have much current development taking place, 
so I ended up using [this fork](https://github.com/dbtek/jekyll-bootstrap-3) which upgraded
the original Jekyll-Boostrap to Bootstrap 3 (thanks to dbtek for maintaining this repo). I
followed the exact steps outlined in the README of this repository to get up and running.

#### Blog Template

The default theme that comes with Jekyll-Bootstrap-3 is not totally complete, and I was
interested in customizing it a bit. After looking at example Bootstrap blog templates, 
I decided to use the template provided in this article - ["How to create an awesome blog template
using Bootstrap 3"](http://prideparrot.com/blog/archive/2014/4/blog_template_using_twitter_bootstrap3_part1). 
This is a great starting point for creating a layout that will work well across desktop 
and mobile browsers.

Customizing the theme basically amounts to editing the files under _includes/themes/bootstrap. 
The default.html file is the overall layout of the site, post.html is the template for posts, 
and page.html is the template for pages. I supposed I could have created my own theme, but 
since I don't plan to ever change the themes, it seemed easier to use the existing one as a 
starting point and the modify it.

#### Editing

I've recently started using IntelliJ IDEA for Java development so I decided to give it a try
as an HTML and Markdown editor. I created a static website project and pointed it at the 
directory where my repository was cloned. The first time I opened a Markdown file it prompted 
me to automatically install a Markdown plugin which I did. So far IntelliJ has worked well and 
I really don't have any complaints. 
 
I also wanted to mention [prose.io](http://prose.io/) which is a very cool web-based editor
for GitHub pages, with support for Jekyll and Markdown. I played around with prose in
the beginning, but since I was initially doing heavy editing of the theme/layout, I preferred
working locally running *jekyll server --watch* to constantly preview my changes before pushing
them. I may consider using prose again just for authoring posts since I liked the idea of being
able to work from any device with a browser and internet connection.

#### Conclusion

In the end I was able to create a basic blog with minimal front-end development skills, and almost 
no knowledge of Jekyll. Hopefully the existence of this blog will motivate me to write more posts 
in the future.