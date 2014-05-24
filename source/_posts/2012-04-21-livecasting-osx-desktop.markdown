---
layout: post
title: Livecasting your desktop in OS X
date: '2012-04-21T00:04:00-07:00'
comments: true
categories:
    - ffmpeg
    - VLC
    - sox
    - OpenAL
    - justin.tv
    - Mac OS X
    - Livecasting
tumblr_url: http://jorgenpt.tumblr.com/post/21484803576/livecasting-osx-desktop
alias: /post/21484803576/livecasting-osx-desktop
---

**EDIT**: Found a way to get audio using Soundflower & sox.  
**EDIT 2**: RTP+SDP was somehow garbling the output, new and improved script now uses FIFOs!

Turns out that livecasting your desktop in OS X is surprisingly hard. I found two ways online, none of which satisfied me, so I hunted for my own.

### Adobe Flash Media Live Encoder, CamTwist & Soundflower

This method, [described by Mike Chambers](http://www.mikechambers.com/blog/2011/05/29/setting-up-desktop-streaming-on-mac-os-x/), relied on three separate GUI tools (one of which was required the Adobe Flash Media Live Encoder), consumed all my CPU, and refused to support my 16:10 aspect ratio. Yikes.

### Ustream Producer

[One simple app](http://www.ustream.tv/producer) to do everything, but sadly it only supports streaming to Ucast, downscales to somewhere near the resolution of a feature-phone from 2000, and still constantly complains that I don't have enough bandwidth. You can get the "Pro" version for $200 which gives you "HD Broadcasting" - but no thanks.

### My own

So, based on [Tyler's Linux approach](http://unethicalblogger.com/2012/04/04/live-coding-with-ffmpeg.html), I tried getting something similar working on OS X. It only used ffmpeg, and seemed to work pretty well.

Surprisingly, my researched ended up with the following results:
 1. `ffmpeg` doesn't support screen grabbing on OS X
 1. `ffmpeg` doesn't have a single audio input that can read an audio device on OS X (see OpenAL section below)
 1. `ffmpeg` can't record a web cam on OS X (this would allow me to use CamTwist as an input source)
 1. VLC doesn't support streaming to an RTMP server
 1. VLC can't record from an audio device
 1. The only other app that'd stream audio live from an audio device to a fifo or pipe was `sox`.

Any of these statements might be wrong, so please tell me if you know a way :)

So, the best I could do was have [VLC](http://www.videolan.org/) stream the screen over a FIFO to `ffmpeg`, redirect audio through [Soundflower](http://code.google.com/p/soundflower/), then use `sox` to pipe audio to `ffmpeg`.
Finally, `ffmpeg` recodes all that data and sends it to the justin.tv RTMP server. Not very simple, but at least it works.

Turns out that this works best if you do *no* work in VLC other than spew to a FIFO: no encoding or encapsulating. Even using MPEG-TS had issues in this context. Raw video in a dummy mux is passed over the FIFO to `ffmpeg`, and `ffmpeg` is allowed to do all the encoding.

In any case, I figured I should share this with the rest of you.

Here's a modified version of [Tyler's script](http://unethicalblogger.com/2012/04/04/live-coding-with-ffmpeg.html):

{% gist 2435006 stream.sh %}

#### OpenAL issues

`ffmpeg` can theoretically record input devices on OS X using OpenAL - which ships with OS X by default. I spent some time trying to get homebrew to build it ([patch here](https://github.com/jorgenpt/homebrew/commit/d69e9d22ef2b0d04fc4f429e91918c034e19a068)), but when I finally got it building I realized [it was completely broken](http://ffmpeg.org/trac/ffmpeg/ticket/314). Yay!
