---
layout: post
title: "steam-runtime without Steam"
date: 2014-05-28 08:28:28 -0700
comments: true
categories:
  - Linux
  - Steam Runtime
  - Game Development
---

**Updated 2014-06-03:** Added information about `STEAM_RUNTIME` variable under [the new embedded search path subsection][runtime-deps-of-runtime].

If you've ever had customers report errors like these, then this post
might be for you:

 * ``./foo: /usr/lib/x86_64-linux-gnu/libstdc++.so.6: version
   `GLIBCXX_3.4.16` not found (required by ./foo)``
 * `./foo: error while loading shared libraries: libSDL2-2.0.so.0:
    cannot open shared object file: No such file or directory`


In my [previous post about self-contained distributions][runtime-post],
we started looking at how the [steam-runtime project][steam-runtime]
works. In this post, we'll make the steam-runtime work for us in a
self-contained distribution that you can ship without depending on
Steam.

I will present two possible ways of doing it:

1. Using [a wrapper script][wrapper-solution].
1. Using [an "embedded search path"][embedded-solution].

If you're wondering why you would prefer the second approach, that
section starts with a rundown of the benefits inherent to it!

<!-- more -->

## Assumptions

The remainder of this article makes a few assumptions, no matter which
of the two approaches you choose.

I assume that you've extracted the steam-runtime into a directory named
`steam-runtime/` next to the executable. The easiest way to do this is
to use the two helper scripts I wrote, see the section on [repackaging
the steam-runtime][repackaging]. You should include the steam-runtime
directory when distributing *outside of* Steam, and distribute the exact
same package **except** for the steam-runtime directory when
distributing *through* Steam.

Excluding the steam-runtime can be done trivially inside your Steam
depot build script. Assuming you're building a depot from `build/linux`
(relative to your ContentRoot) with the binary living directly in that
directory, your script would contain something like this:

    "DepotBuildConfig"
    {
        "DepotID" "1001"

        "FileMapping"
        {
            "LocalPath" "build\linux\*"
            "DepotPath" "."
            "recursive" "1"
        }

        "FileExclusion" "build\linux\steam-runtime"
    }

It's worth noting that the FileExclusion is matched against your local
paths, not your depot paths, and it is implicitly recursive (the latter
doesn't seem to be documented [in the SteamPipe docs][steampipe-docs] as
of 2014-05-28.)

I assume you're already building your game with [the steam-runtime
SDK][steam-runtime]. This is how you make sure your game is depending on
the right version of the libraries.

Finally, for simplicity sake I'm also assuming you don't mind ~100MB of
additional data in your package, which is the size of the entire
steam-runtime for one architecture. If this is too much for you, you can
always manually strip out any unneeded libraries from the runtime.

### Preparing the steam-runtime for repackaging

I've created [two helper scripts][runtime-helpers], one to make sure
you've [downloaded the latest runtime][update-rt], and one to [extract
the parts of the runtime you care about][extract-rt] (to reduce runtime
size from 400MB to 100MB, by excluding documentation and whatever
architecture you're **not** using.)

You would invoke them like this to download the latest runtime and
extract the 64bit libraries from it into the `build/linux/steam-runtime`
directory.

    ./update_runtime.sh
    ./extract_runtime.sh steam-runtime-release_latest.tar.xz amd64 build/linux/steam-runtime


## Solution 1: The wrapper script

The least invasive way to accomplish what we want is to basically do
what Steam does: Set up the runtime environment variables via
`LD_LIBRARY_PATH`, and launch the main binary.

To make it even easier, I've put together [a little wrapper
script][wrapper-script] that does exactly that. Name the script `foo.sh`
or `foo`, and put it in the same directory as your executable, which it
will then assume is named `foo.bin`.

The script should gracefully handle being launched from Steam, as it'll
detect that the runtime has already been set up.

## Solution 2: Embedded search path

First off, why would you prefer this approach to using a wrapper script?

 * Shell scripts are fragile -- it's easy to get something wrong, like
   incorrectly handling spaces in filenames, or something equally silly.
 * A shell script gives you another file that you have to be careful to
   maintain the executable bit on.
 * Shell scripts are text files, and your VCS / publishing process might
   mangle the line endings, which makes everyone sad (`bad interpreter:
   /bin/bash^M: no such file or directory`)
 * A customer could accidentally launch the wrong thing (i.e. the
   `.bin`-file rather than the script), which might work on some
   machines, fail in subtle ways on other machines, and not work at all
   on the rest of them.
 * Launching the game in a debugger requires more complexity in your
   script, like the `--gdb` logic in
   [launcher_wrapper.sh][wrapper-script], to make the game, but not the
   debugger, pick up the runtime libraries.
 * If you launch any system binaries from outside of the runtime without
   taking care to unset `LD_LIBRARY_PATH`, they will implicitly be using
   the runtime libraries, which might not cause problems.

The alternative to the wrapper script is using `DT_RPATH`, which I've
talked about in [a previous blog post][rpath-post]. This approach is a
little more invasive to your build process, but overall it should
require less code.

Simply invoke your linker with the `-rpath` option pointing to various
subdirectories of the steam-runtime directory. For GCC and Clang, you
would add `-Wl,-rpath,<path1>:<path2>:...` to the linking step to
accomplish this.

These are the paths to the 64bit libraries in the steam-runtime:

 * amd64/lib/x86_64-linux-gnu
 * amd64/lib
 * amd64/usr/lib/x86_64-linux-gnu
 * amd64/usr/lib

These are the paths to the 32bit libraries:

 * i386/lib/i386-linux-gnu
 * i386/lib
 * i386/usr/lib/i386-linux-gnu
 * i386/usr/lib

Assuming you're using GCC and the steam-runtime lives next to the
executable, you'd use these GCC options for a 64bit binary:

    -Wl,-z,origin -Wl,-rpath,$ORIGIN/steam-runtime/amd64/lib/x86_64-linux-gnu:$ORIGIN/steam-runtime/amd64/lib:$ORIGIN/steam-runtime/amd64/usr/lib/x86_64-linux-gnu:$ORIGIN/steam-runtime/amd64/usr/lib

And you would use these option for a 32bit binary:

    -Wl,-z,origin -Wl,-rpath,$ORIGIN/steam-runtime/i386/lib/i386-linux-gnu:$ORIGIN/steam-runtime/i386/lib:$ORIGIN/steam-runtime/i386/usr/lib/i386-linux-gnu:$ORIGIN/steam-runtime/i386/usr/lib

### Runtime dependencies of the steam-runtime

In addition to redirecting the ELF loader to the steam-runtime, there are some runtime dependencies within those dynamic libraries that need to be redirected as well. Luckily, Valve has done this work for us, and [patched these libraries to look elsewhere][runtime-patches]. In order to know what the "base" of the runtime is, it looks at the `STEAM_RUNTIME` environment variable. 

The first version of this post didn't include this detail, and you might've run into errors like these:

    symbol lookup error: /usr/lib/x86_64-linux-gnu/gio/modules/libdconfsettings.so: undefined symbol: g_mapped_file_get_bytes

This is because glib has a [runtime search for plugins][glib-plugins] that directly calls `dlopen()` on an absolute path.

The solution to this problem is to have the first thing in your `main()` method on Linux be:

``` c
if (!getenv("STEAM_RUNTIME")) {
    setenv("STEAM_RUNTIME", figureOutSteamRuntimePath(), 1);
}
```

A full sample for your `main()` is [available in the helpers GitHub repository][embedded-path-c-sample].

## Conclusion

With just a small modification to your build system and a ~100MB larger
distribution, you can make your executables run across a wide variety of
Linux distributions and user setups. I highly recommend the embedded
search path solution, which is what I used for [Planetary
Annihilation][pa]'s Linux release.

When shipping your own steam-runtime, you are responsible for updating
the runtime. The date of the latest update can be found inside the
[runtime MD5 file][runtime-md5]. In addition, you are responsible for
respecting the licenses of all the packages included in the runtime --
including any clauses regarding redistribution.


[rpath-post]: /post/2014/05/20/dt-rpath-ld-and-at-rpath-dyld/
[runtime-post]: /post/2014/05/26/self-contained-game-distribution-on-linux/
[steam-runtime]: https://github.com/ValveSoftware/steam-runtime
[runtime-md5]: http://media.steampowered.com/client/runtime/steam-runtime-release_latest.tar.xz.md5
[pa]: http://www.uberent.com/pa/
[runtime-helpers]: https://github.com/jorgenpt/steam-runtime-helpers
[update-rt]: https://github.com/jorgenpt/steam-runtime-helpers/blob/master/update_runtime.sh
[extract-rt]: https://github.com/jorgenpt/steam-runtime-helpers/blob/master/extract_runtime.sh
[wrapper-script]: https://github.com/jorgenpt/steam-runtime-helpers/blob/master/launch_wrapper.sh
[steampipe-docs]: https://partner.steamgames.com/documentation/steampipe
[wrapper-solution]: /post/2014/05/28/steam-runtime-without-steam/#Solution.1:.The.wrapper.script
[embedded-solution]: /post/2014/05/28/steam-runtime-without-steam/#Solution.2:.Embedded.search.path
[repackaging]: /post/2014/05/28/steam-runtime-without-steam/#Preparing.the.steam-runtime.for.repackaging
[runtime-deps-of-runtime]: /post/2014/05/28/steam-runtime-without-steam/#Runtime.dependencies.of.the.steam-runtime
[runtime-patches]: https://github.com/ValveSoftware/steam-runtime/tree/master/patches
[glib-plugins]: https://github.com/ValveSoftware/steam-runtime/blob/master/patches/glib2.0/01_steam_runtime_path.patch#L16
[embedded-path-c-sample]: https://github.com/jorgenpt/steam-runtime-helpers/blob/master/sample_embedded_path_main.c
