---
layout: post
title: GUI frustration
date: '2011-02-21T19:14:00-08:00'
comments: true
categories:
    - Objective-C
    - Cocoa
    - Ruby
    - GUI
tumblr_url: http://jorgenpt.tumblr.com/post/3437297505/gui-frustration
---

Yesterday I needed to whip together a simple little app to "vet" photos - a tool to quickly let me go through each photo in a directory and choose between "thumbs up" or "boo", then give me a list of the ones I said "boo" to.


Ideally, I'd write this app in Ruby, but I have had only bad experiences trying to get a useful, good GUI written in either Ruby or Python. Years ago I looked into GTK, QT and WxWidgets, but none of them convinced me they were capable. So I googled around for "simple GUI ruby", and "Shoes" caught my eye. I tried it out, and it was indeed simple, but sadly **too** simple. It was hellbent on not letting me do anything complex, even when it's "intuitive behavior" was wrong.


I've been writing Objective-C since last summer ([GrabBox](http://grabbox.devsoft.no) is written in it), so I gave up on my Ruby track and tried putting something together using Interface Builder and Cocoa. To my amazement, writing the whole app took me much less time than researching "simple" GUI frameworks for Ruby: ~1 hour from scratch to working app:


{% img center /images/image_vetter.jpg %}


Is this really the state of GUI frameworks for Ruby and the likes (say, Python and Perl) - or have things improved significantly since I last looked at GTK, WxWidgets and QT? Is there nothing that can even remotely compete with Interface Builder and Cocoa? I'm by no means a fanboy, and I'd love to be able to do this as efficiently in Ruby or Python! Please post a comment if you have any tips or thoughts on this. :)


*If you wonder what was wrong with Shoes: Things like the image class **only** letting you load data by providing a path or an URL, and having some caching behavior that rendered the app very slow after changing the ~4MB displayed image tens of times.*
