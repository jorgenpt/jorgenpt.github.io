---
layout: post
title: Requiring a minimum version of the Android NDK
date: '2012-03-02T13:22:00-08:00'
categories:
    - Android
    - Android NDK
tumblr_url: http://jorgenpt.tumblr.com/post/18620908643/requiring-a-minimum-version-of-the-android-ndk
---

I was tinkering with our build system at work, and I realized that builds were failing on r5b and below. After a bit of debugging (and reading the NDK changelog), I realized that `LOCAL_WHOLE_STATIC_LIBRARIES` was broken in r5b and missing in r4 and before. To prevent other engineers from running into this, I wanted to make our project depend on r5c or above.

There doesn't seem to be a facility in place to do this, so I created a shellscript you can invoke from your `Makefile` to assert that the right version of the NDK is present. You can find [the shellscript as a gist on GitHub](https://gist.github.com/1961404)


**EDIT**: This script depends on the environment variable ANDROID_NDK_ROOT to point to the base of your NDK. The script has been updated to test that this is the case (thanks to David R. for pointing this out).

The only caveat is that it does *not* support asserting versions below r5 - this is because only r5 and above have a version identifier in the NDK tree. It will correctly identify r4 and below as not being good enough for your build, but you can't say "I need r3 or above".

Put the script into `jni/assert_ndk_version.sh`, and put the following at the top of your `jni/Android.mk`, and voilà! Builds should now fail with a more understandable message if someone's using the wrong NDK version. :-)

```make jni/Android.mk
ifneq ($(shell $(LOCAL_PATH)/assert_ndk_version.sh "r5c"),true)
  $(error NDK version r5c or greater required)
endif
```
