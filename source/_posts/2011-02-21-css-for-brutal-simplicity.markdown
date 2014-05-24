---
layout: post
title: Brutal Simplicity and Disqus
date: '2011-02-21T21:00:00-08:00'
tags:
- tumblr
- css
- disqus
tumblr_url: http://jorgenpt.tumblr.com/post/3439183544/css-for-brutal-simplicity
---

I currently use the <a title="Brutal Simplicity Theme blog" href="http://brutalsimplicitytheme.tumblr.com/">Brutal Simplicity</a> theme for my tumblr blog with the "Dark Layout", and when I set up Disqus it was really unreadable. To remedy this I threw together some CSS to make it more readable:

<pre>#brutal_disqus_container h3,
 #brutal_disqus_container a,
 #brutal_disqus_container a:active,
 #brutal_disqus_container a:focus,
 #brutal_disqus_container a:link
 #brutal_disqus_container {
      color: #BEBEBE;
}
#brutal_disqus_container a:hover {
      color: #A9A9A9;
}</pre>

This uses the colors from the Brutal Simplicity (dark) theme for the text and links in the comment field. To set it up, go to your disqus admin page, and under Settings &gt; Appearances put the above in the the Custom CSS text field.


A different style (but similar goal) can be found at <a title="HippoSounds blog" href="http://hipposounds.tumblr.com/post/2144909858/css-tweaking">HippoSounds tumblr</a> :-)

