---
layout: post
title: First playtest of Arcane Deathmatch
date: '2012-10-06T15:24:00-07:00'
comments: true
categories:
    - Arcane Deathmatch
    - Playtesting
    - Unity3D
    - Game Development
tumblr_url: http://jorgenpt.tumblr.com/post/33037031622/first-playtest-of-arcane-deathmatch
alias: /post/33037031622/first-playtest-of-arcane-deathmatch
---

Yesterday I asked two coworkers to play Arcane Deathmatch with me, to get an initial feel for what direction I'm going in. It was a great experience, and a great motivator: I finished up a large amount of basic implementations of features in the last week, so that it'd be ready for the playtest.

I **highly** recommend doing this early for people working on hobby projects - it helps you focus on shipping and creating a real product, and motivate you. During the time leading up to the play-testing, I started thinking of every issue I've filed in the bugtracker in terms of "how important is this to testing gameplay".

Some very useful things I learned include:

* Controls need to be thought about very carefully, and tested on real people. What seems fine when you're just testing around the environment is experienced completely differently when you're in an action-packed fight.
* While art is not important to the "fun" in this concept, UX is very important - even early on. People need to find the information and actions they care about!
* Pacing is very important, and hard to get right - there are a *lot* of factors that influence it, many of which I haven't gotten around to implementing yet.

There were a lot of other subtle bugs and improvements I found through this little hour of gameplay. As an example, camera movement is currently not animated, which makes Teleport **very** jarring.

For now, I'm back to implementing some new content (especially a more interesting level and more spells) - and then I can get a better feel for how pacing and balancing needs to be done.
