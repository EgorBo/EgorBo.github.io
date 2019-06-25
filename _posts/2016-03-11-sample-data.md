---
layout: post
date: 2016-03-11 23:56
title:  "Sample Data"
mood: happy
category: 
- docs
---

Markdown (or Textile), Liquid, HTML & CSS go in. Static sites come out ready for deployment.

#### Headings

# Heading 1

## Heading 2

### Heading 3

#### Heading 4

##### Heading 5

###### Heading 6

#### Blockquote

> No more databases, comment moderation, or pesky updates to install—just your content.

#### Unordered List

* Jekyll
    * Nested Jekyll
    * Nested Ruby
* Ruby
* Markdown
* Liquid

#### Ordered List

1. Jekyll
    1. Nested Jekyll
    2. Nested Ruby
2. Ruby
3. Markdown
4. Liquid

#### Link

This is <a href="http://example.com/" title="Title">an example</a> inline link.

#### Paragraph w/ strong, em, etc.

These are just a few of the *available* **configuration** options. Many configuration options can <strike>either</strike> be specified as flags on the <abbr title="Command Line Tool">command line</abbr>, or alternatively (and more commonly) they can be specified in a _config.yml file at the root of the source directory. Jekyll will <a href="http://joro.me/" target="_blank">automatically use</a> the options from this file when run. For example, if you place the following lines in your _config.yml file.

#### Image
<figure class="aligncenter">
	<img src="https://images.unsplash.com/photo-1449452198679-05c7fd30f416?ixlib=rb-0.3.5&q=80&fm=jpg&crop=entropy&s=73181f1c6d56b933b30de2bfe21fdf3b" />
	<figcaption>Photo by <a href="https://unsplash.com/rmaedavis" target="_blank">Rachel Davis</a>.</figcaption>
</figure>

#### Video

<iframe width="560" height="315" src="https://www.youtube.com/embed/iWowJBRMtpc" frameborder="0" allowfullscreen></iframe>

#### Default Code Block

    This is code blog.

#### Styled Code Block
	
{% highlight ruby linenos %}
#!/usr/bin/ruby
$LOAD_PATH << '.'
require "support"

class Decade
include Week
   no_of_yrs=10
   def no_of_months
      puts Week::FIRST_DAY
      number=10*12
      puts number
   end
end
d1=Decade.new
puts Week::FIRST_DAY
Week.weeks_in_month
Week.weeks_in_year
d1.no_of_months
{% endhighlight %}
	
#### Definition Lists
	
<dl>
    <dt>Definition Title</dt>
    <dd>Definition Description</dd>
</dl>

#### Paragraphs w/ Aligned Images

The Jekyll gem makes a jekyll executable available to you in your Terminal window. You can use this command in a number of ways.

<figure class="alignleft">
	<img width="250" src="https://images.unsplash.com/photo-1432821596592-e2c18b78144f?ixlib=rb-0.3.5&q=80&fm=jpg&crop=entropy&s=3f9c78df0edb464244bbabb04d1797d8" />
	<figcaption>Photo by <a href="https://unsplash.com/dustinlee" target="_blank">Dustin Lee</a>.</figcaption>
</figure>

This site aims to be a comprehensive guide to Jekyll. We’ll cover topics such as getting your site up and running, creating and managing your content, customizing the way your site works and looks, deploying to various environments, and give you some advice on participating in the future development of Jekyll itself.

Jekyll is a simple, blog-aware, static site generator. It takes a template directory containing raw text files in various formats, runs it through a converter (like Markdown) and our Liquid renderer, and spits out a complete, ready-to-publish static website suitable for serving with your favorite web server. Jekyll also happens to be the engine behind GitHub Pages, which means you can use Jekyll to host your project’s page, blog, or website from GitHub's servers for free.

<figure class="alignright">
	<img width="250" src="https://images.unsplash.com/photo-1442037025225-e1cffaa2dc23?ixlib=rb-0.3.5&q=80&fm=jpg&crop=entropy&s=7fe04b68b0cb123bf568c6951c14b177" />
	<figcaption>Photo by <a href="https://unsplash.com/lobostudiohamburg" target="_blank">LoboStudio Hamburg</a>.</figcaption>
</figure>

Throughout this guide there are a number of small-but-handy pieces of information that can make using Jekyll easier, more interesting, and less hazardous. Here's what to look out for.

If you come across anything along the way that we haven’t covered, or if you know of a tip you think others would find handy, please file an issue and we’ll see about including it in this guide.

The front matter is where Jekyll starts to get really cool. Any file that contains a YAML front matter block will be processed by Jekyll as a special file. The front matter must be the first thing in the file and must take the form of valid YAML set between triple-dashed lines.