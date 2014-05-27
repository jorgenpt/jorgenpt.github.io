---
layout: post
title: "Linux games at full Steam ahead"
date: 2014-05-28 19:28:28 -0700
comments: true
categories:
  - Linux
  - Steam Runtime
  - Game Development
---

In my [previous post about self-contained distributions][runtime-post],
we started looking at how [the steam-runtime project][steam-runtime]
works. In this post, we'll make the steam-runtime work for us in
self-contained distributions that you can ship without Steam.

I will present two possible ways of doing it:

1. Using a wrapper script.
1. Using an "embedded search path".

If you're wondering why you would prefer the second approach, read the
first paragraph under the embedded search path heading!

<!-- more -->

## Assumptions

Both of these approaches assume that you've extracted the steam-runtime
into a directory named `steam-runtime/` next to the exectuable. You can
find the latest runtime at
http://media.steampowered.com/client/runtime/steam-runtime-release_latest.tar.xz,
[with accompanying checksum file][runtime-md5]. Include the
steam-runtime directory when distributing outside of Steam, and
distribute the exact same package **except** for the steam-runtime
directory when distributing inside of Steam.

**TODO: Write about FileExclusion depot script in previous paragraph.**

In addition, these approaches assume you're already building with [the
steam-runtime SDK][steam-runtime]. This is how you make sure your game
is depending on the right version of the libraries.

Finally, for simplicity sake I'm assuming you don't mind ~100MB of
additional data in your package (that's the size of the steam-runtime
for one architecture.) If this is too much for you, you can always
manually strip out any unneeded libraries from the runtime.

## Solution 1: The wrapper script

The least invasive way to accomplish what we want is to do basically
what Steam does: Set up the runtime environment variables via
`LD_LIBRARY_PATH`, and launch the main binary.

To make it even easier, I've put together [a little wrapper
script][wrapper-script] that does exactly that. Name the script `foo.sh`
or `foo`, and put it in the same directory as your executable (which
should be named `foo.bin`.)

The script should gracefully handle being launched from Steam, as it'll
detect that the runtime has already been set up.

## Solution 2: Embedded search path

First off, why **wouldn't** you want to use a wrapper script?

 * Shell scripts are fragile -- it's easy to get something wrong or not
   handle spaces right, or something equally silly.
 * You have another file that you have to be careful about the
   permissions of (to keep it executable)
 * Shell scripts are text files, and your VCS / publication process
   might mangle the line endings, which makes everything sad (`bad
   interpreter: /bin/bash^M: no such file or directory`)
 * Your customers could accidentally launch the wrong thing (the .bin
   rather than the script), which might work on some machines, might
   fail in subtle ways on other machines, and not work at all on the
   rest of them.
 * Launching the game in a debugger requires more complexity in your
   script (like the `--gdb` logic I embedded above) to make the game
   pick up the same libraries.
 * If you launch any system binaries from outside of the runtime, they
   will implicitly be using the runtime libraries, which might not work,
   unless you take care to unset `LD_LIBRARY_PATH` before executing
   them.

The alternative to using `LD_LIBRARY_PATH` is using `DT_RPATH`, which
I've talked about in [a previous blog post][rpath-post]. This approach
is a little more invasive to your build process, but overall it should
require less code.

Simply invoke your linker with the `-rpath` option pointing to various
subdirectories of the steam-runtime directory. For GCC and Clang, you
would add `-Wl,-rpath,<paths>` to the linking step to accomplish this.

These are the 64bit paths:

 * amd64/lib/x86_64-linux-gnu
 * amd64/lib
 * amd64/usr/lib/x86_64-linux-gnu
 * amd64/usr/lib

These are the 32bit paths:

 * i386/lib/i386-linux-gnu
 * i386/lib
 * i386/usr/lib/i386-linux-gnu
 * i386/usr/lib

Assuming you're using GCC and the steam-runtime lives next to the
executable, you'd use these GCC options for 32bit:

    -Wl,-z,origin -Wl,-rpath,$ORIGIN/steam-runtime/i386/lib/i386-linux-gnu:$ORIGIN/steam-runtime/i386/lib:$ORIGIN/steam-runtime/i386/usr/lib/i386-linux-gnu:$ORIGIN/steam-runtime/i386/usr/lib

And you would use these option for 64bit:

    -Wl,-z,origin -Wl,-rpath,$ORIGIN/steam-runtime/amd64/lib/x86_64-linux-gnu:$ORIGIN/steam-runtime/amd64/lib:$ORIGIN/steam-runtime/amd64/usr/lib/x86_64-linux-gnu:$ORIGIN/steam-runtime/amd64/usr/lib

## Preparing the steam-runtime for your package

I've created [two helper scripts][runtime-helpers], one to make sure
you've downloaded the latest runtime, and one to extract the parts of
the runtime you care about (to reduce runtime size from 400MB to 100MB,
by excluding documentation and whatever architecture you're **not**
using.)

They're invoked like this to download the runtime and extract the 64bit
libraries from it into the `build/steam-runtime` directory.

    ./update_runtime.sh
    ./extract_runtime.sh steam-runtime-release_latest.tar.xz amd64 build/steam-runtime

## Conclusion

With just a small modification to your build system and a ~100MB larger
package, you can make your executables run across a wide variety of
Linux distributions and user setups. I highly recommend the embedded
search path solution, which is what I've used for [Planetary
Annihilation's Linux release][pa].

When shipping your own steam-runtime, you are responsible for updating
the runtime. The date of the latest update can be found inside the
[runtime MD5 file][runtime-md5]. In addition, you are responsible for
respecting the licenses of all the open source packages included in the
runtime.


[rpath-post]: /post/2014/05/20/dt-rpath-ld-and-at-rpath-dyld/
[runtime-post]: /post/2014/05/26/self-contained-game-distribution-on-linux/
[steam-runtime]: https://github.com/ValveSoftware/steam-runtime
[runtime-md5]: http://media.steampowered.com/client/runtime/steam-runtime-release_latest.tar.xz.md5
[pa]: http://www.uberent.com/pa/
[runtime-helpers]: https://gist.github.com/jorgenpt/07f207aefdd49b61c7b6
[wrapper-script]: https://gist.github.com/jorgenpt/35ded51f96cddad8190d
