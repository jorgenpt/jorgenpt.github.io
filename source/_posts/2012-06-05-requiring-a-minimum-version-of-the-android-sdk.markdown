---
layout: post
title: Requiring a minimum version of the Android SDK
date: '2012-06-05T16:40:00-07:00'
comments: true
categories:
    - Android SDK
    - Android
tumblr_url: http://jorgenpt.tumblr.com/post/24502661219/requiring-a-minimum-version-of-the-android-sdk
alias: /post/24502661219/requiring-a-minimum-version-of-the-android-sdk
---

(This is similar to [Requiring a minimum version of the Android NDK](/post/2012/03/02/requiring-a-minimum-version-of-the-android-ndk/), but for SDK versions)

Again, I was tinkering with our build system at work, which is a set of small Makefiles that are responsible for invoking ndk-build (to build our C component) and ant (for the Java component). These files also maintain the dependency graph for the cross-domain dependencies, so things like header files being generated from class-files using `javah` and APKs depending on the produced shared libraries.

I recently made some changes to the `ant` build step by creating [our own `custom_rules.xml`](https://gist.github.com/2878806), exposing the "hidden" -compile target. What I noticed was that `build.xml` only did an `<import file="custom_rules.xml" optional="true" />` if you were on a fairly recent Android SDK version. This isn't a problem for our Jenkins builds, since we've got an in-house system that ensures a strict version dependency between a specific source checkout and SDK/NDK versions, so they were always using the newer SDK. It was a problem for our developers - we've yet to roll this system to our development machines, so developers are responsible for checking out and updating their own SDKs.

To prevent this from getting in the way, I wrote a little snippet of bash that's run from the Makefile, that ensures that the SDK version is at least the given version.

You can find [the shellscript as a gist on GitHub](https://gist.github.com/2878774)

Put the script into `assert_sdk_version.sh`, and put the following at the top of your `Makefile`, and voilà! Builds should now fail with a more understandable message if someone's using the wrong NDK version. :-)

```make Makefile
ifneq ($(shell $(LOCAL_PATH)/assert_sdk_version.sh "r19"),true)
  $(error SDK version r19 or greater required)
endif
```

If you're curious how this works: It checks the `tools/source.properties` file in your Android SDK, looking for a line like `Pkg.Revision=XX`, and extracts the version (`XX`) from that.

It's pretty straight forward, but I couldn't find anything online on how to check the SDK version from the command line, so I figured I'd share it.
