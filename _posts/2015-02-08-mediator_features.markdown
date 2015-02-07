---
layout: post
title:  "Mediator Features"
date:   2015-02-08 14:34:25
categories: mediator feature
tags: test
image: /assets/article_images/2014-11-30-mediator_features/night-track.JPG
---
#Mediator Formats and CSS features

Examples for different formats and css features

#Header Formats
#Header1
##Header2

#Blockquotes
>Lorem ipsum dolor sit amet, consetetur sadipscing elitr, sed diam nonumy eirmod tempor invidunt ut labore et dolore magna aliquyam erat, sed diam voluptua. At vero eos et accusam et justo duo dolores et ea rebum. Stet clita kasd gubergren, no sea takimata sanctus

#Lists
##orderd lists
1. one
2. two
3. three

##unorderd lists
- Apple
- Banana
- Plum

#Links
This is an [example link](http://example.com/ "With a Title").

#Combinations
>Lorem ipsum dolor sit amet, consetetur sadipscing elitr, sed diam nonumy eirmod tempor invidunt ut labore et dolore magna aliquyam erat, sed diam voluptua. At vero eos et accusam et justo duo dolores et ea rebum. Stet clita kasd gubergren, no sea takimata sanctus
>
> - Apple
> - Banana
> - Plum

#Highlight JS
{% highlight js %}

<footer class="site-footer">
 <a class="subscribe" href="{{ "/feed.xml" | prepend: site.baseurl }}"> <span class="tooltip"> <i class="fa fa-rss"></i> Subscribe!</span></a>
  <div class="inner">a
   <section class="copyright">All content copyright <a href="mailto:{{ site.email}}">{{ site.name }}</a> &copy; {{ site.time | date: '%Y' }} &bull; All rights reserved.</section>
   <section class="poweredby">Made with <a href="http://jekyllrb.com"> Jekyll</a></section>
  </div>
</footer>
{% endhighlight %}

