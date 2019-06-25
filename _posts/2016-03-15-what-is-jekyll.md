---
layout: post
date: 2016-03-15 22:00
title:  "What is Jekyll?"
mood: speechless
category: 
- docs
---

Jekyll is a parsing engine bundled as a ruby gem used to build static websites from dynamic components such as templates, partials, liquid code, markdown, etc. Jekyll is known as "a simple, blog aware, static site generator".

![Alt Text](https://images.unsplash.com/photo-1449247709967-d4461a6a6103?ixlib=rb-0.3.5&q=80&fm=jpg&crop=entropy&s=d1b9ba0f56bbb8a965d26e8c4177f4af)

<!--more-->

### What does Jekyll do?

Jekyll is installed as a ruby gem local computer. Once installed you can call jekyll serve in the terminal in a directory and provided that directory is setup in a way jekyll expects, it will do magic stuff like parse markdown/textile files, compute categories, tags, permalinks, and construct your pages from layout templates and partials.

Once parsed, Jekyll stores the result in a self-contained static _site folder. The intention here is that you can serve all contents in this folder statically from a plain static web-server.

You can think of Jekyll as a normalish dynamic blog but rather than parsing content, templates, and tags on each request, Jekyll does this once beforehand and caches the entire website in a folder for serving statically.

*Source: [http://jekyllbootstrap.com/lessons/jekyll-introduction.html](http://jekyllbootstrap.com/lessons/jekyll-introduction.html){:target="_blank"}*