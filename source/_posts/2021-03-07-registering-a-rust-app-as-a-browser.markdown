---
layout: post
title: "Registering a Rust app as a browser on macOS and Windows"
date: 2021-03-07 17:47:48 -0800
comments: true
categories: 
---

# Registering a Rust app as a browser

## Motivation

During this period of extended work for home I am often using the desktop Slack app to keep up with both my professional and personal communications. In order to keep my credentials and identities somewhat separated when I'm browsing, I use two different Google Chrome profiles -- a built-in feature where I'm signed in to my employer's Google Suite in one window, and my personal gmail.com account in a different window. This seperates cookies, extensions, and settings.

I recently got tired of clicking "obvious" work links (like JIRA) on Slack and having them open in my personal Google Chrome profile. This happens because Chrome's default behavior is to open a link in the most recently activated window, and it doesn't have any way to be more clever than that.

In order to address this, I figured I could write an app that pretends to be my browser, and then uses command line switches to tell Chrome which profile to open it in, based on the URL. Thus, [bichrome] was born. (Which now also supports Firefox, Safari, and Edge.)

## 


[bichrome]: https://github.com/jorgenpt/bichrome