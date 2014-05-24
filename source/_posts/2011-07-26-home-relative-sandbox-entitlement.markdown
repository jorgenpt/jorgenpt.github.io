---
layout: post
title: The secret to the home-relative Sandbox entitlement
date: '2011-07-26T16:45:55-07:00'
comments: true
categories:
    - Cocoa
    - Objective-C
    - Sandbox
    - Mac OS X
tumblr_url: http://jorgenpt.tumblr.com/post/8105269529/home-relative-sandbox-entitlement
alias: [/post/8105269529, /post/8105269529/home-relative-sandbox-entitlement]
---

With the launch of Lion, Apple added a new security feature to the operating system: The Application Sandbox. It encourages application authors to specify what subset of system functionality their app needs to function correctly, in order to reduce the impact of a malicious or compromised app. See the [Mac OS X Developer Library][dev-library-sandbox] or the [ars technica Lion review][ars-lion-review] for more info on this.

As a part of this, Apple added a set of entitlements labeled "temporary exceptions" ([here's a complete list](http://developer.apple.com/library/mac/#documentation/Security/Conceptual/CodeSigningGuide/ApplicationSandboxingEntitlementKeys/ApplicationSandboxingEntitlementKeys.html)), most likely to simplify and speed up adoption of this new technology. Your app can claim to need one of these "temporary" entitlements to do certain things that otherwise wouldn't be allowed by the Sandbox. For [GrabBox](http://grabbox.devsoft.no) I need to have read-only access the users desktop -- which falls under this category.

I've been spending some time today trying to figure out how to get the `com.apple.security.temporary-exception.files.home-relative-path.read-only` entitlement working. The documentation is sparse, and there're no samples as far as I can tell. After many attempts, I finally figured out the key piece of information keeping me from getting this working: The path you specify in the entitlement needs to **start with a slash**. For example, instead of specifying `Desktop`, you specify `/Desktop`.

Here's an example of a valid entitlement plist:

```xml Info.plist
<pre><?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
        <key>com.apple.security.app-sandbox</key>
        <true/>
        <key>com.apple.security.temporary-exception.files.home-relative-path.read-only</key>
        <array>
                <string>/Desktop</string>
                <string>/Dropbox</string>
        </array>
</dict>
</plist>
```

I assume this applies to the `com.apple.security.temporary-exception.files.home-relative-path.read-write` entitlement as well.

I hope this saves other people trying to get this working a little bit of time. :-)

[dev-library-sandbox]: http://developer.apple.com/library/mac/documentation/General/Conceptual/MOSXAppProgrammingGuide/AppRuntime/AppRuntime.html#//apple_ref/doc/uid/TP40010543-CH2-SW7
[ars-lion-review]: http://arstechnica.com/apple/2011/07/mac-os-x-10-7.ars/9#sandboxing
