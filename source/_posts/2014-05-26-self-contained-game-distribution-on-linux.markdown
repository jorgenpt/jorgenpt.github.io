---
layout: post
title: "Self-contained game distribution on Linux"
date: 2014-05-26 21:10:28 -0700
comments: true
categories:
  - Linux
  - Steam Runtime
  - Game Development
---

Distributing a game on Linux can be a little intimidating, and there are
definitely pitfalls. The main problem is making sure your game runs on
all of your users' machines, and outside of hardware and drivers, the
root of the problem is usually one of two things:

1. You make an assumption about what libraries are present on the system.
1. You make an assumption about what version of a library is present on the system.

This is very easy to accidentally do, as adding `-lSDL2` to the linker's
command line might work perfectly fine on your machine, but you forgot
that you installed SDL2 by hand 4 months ago. Another cause could be
that while **your** Linux distribution came with SDL2 preinstalled,
another distribution (that your users use) might not. Finally, maybe
your distribution came with v2 of some library, but your users only have
v1.

The best way to avoid this is to make your game distribution "hermetic,"
meaning that it contains all of its own dependencies. There are two main
ways to achieve this:

1. Statically linking with all of your dependencies.
1. Dynamically linking with all of your dependencies, and pointing the
   system's runtime loader at a copy of the libraries you bundle with
   your game.

Statically linking comes with its own set of problems, so this post
talks about solving the problem with dynamic linking.

## Introducing the steam-runtime

It turns out that Valve has already solved this problem in Steam with
[something called the steam-runtime][steam-runtime]. Contrary to what
its name indicates, it has **no direct dependency on Steam nor does it
even assume that it is installed**. It is merely a controlled set of
open source libraries (with some patches) and associated tools to use
those libraries - to make your game build hermetic.

<!-- more -->

If your game is running under Steam, you don't need to do much. Build
your game with the steam-runtime SDK, make sure all of your dependencies
exist inside of the runtime, and ship the game binaries to Steam. On the
receiving end, Steam will make sure that your users have the latest
version of the steam-runtime, and execute your game inside of it.

If you, like many others, also distribute your game outside of Steam,
you'll need to find another solution. The obvious solution is to build
on their work - it's an open source project that solves the problem
perfectly!

## Workings of the steam-runtime

When I say that your game is executed "inside" of the runtime when
launched through Steam, I specifically mean that:

* The steam-runtime being present in some location Steam knows about
* Steam sets the `LD_LIBRARY_PATH` environment variable before launching
  your game to [instruct the dynamic loader to search the specified
  directory for libraries][ld_library_path].

This way, your game suddenly prefers the runtime versions of libraries
rather than your own. It's worth noting that if your game depends on a
library that is **not** present in the runtime, but the user has it
installed on their system, your game will run without error. This is
something to be wary of, since you don't know what version of the
library you're getting, and it'll fail to execute on some users'
systems.

The [runtime SDK][steam-runtime] is just a set of tools that have been
told to look for libraries and headers inside the SDK rather than in the
system directories, so that the linker and compiler knows about the
right version of the libraries.

## Contents of the runtime

Since the steam-runtime doesn't require Steam, let's take a look at what
the runtime contains, and see if there's a way to use this in our
non-Steam distributions.

You can find the runtime binaries hosted on the Steam CDN as a tar
archive:
http://media.steampowered.com/client/runtime/steam-runtime-release_latest.tar.xz

I've provided a script on GitHub that you can use to make sure you have
the latest runtime downloaded to the current directory. [The helper
script is update_runtime.sh][update_runtime].

The runtime tar archive contains some helper scripts, and the various
files needed for each library, as well as the libraries themselves. For
each library, there's a 32bit version (in the i386 directory) and a
64bit version (in the amd64 directory.)

Surprisingly enough, the runtime *also* contains (as of 2014-05-26) the
documentation needed for each library, which takes up almost half of the
space required by an extracted version of the runtime. To strip out the
documentation, and extract just the architecture you care about, I've
written [another little helper script called
extract_runtime.sh][extract_runtime].

With this script, you'll be left with about ~100MB of libraries per
architecture. You can probably tailor the set of libraries for your
title to reduce the size even further, but that's left as an exercise
for the reader.

## Conclusion

The Steam runtime is a useful collection of libraries that helps solve
the important problem of operating system fragmentation (different Linux
distributions, different versions). It has a lot of value outside of
Steam as well, and should be trivially re-usable for your non-Steam
distribution.

In my next blog post, I will cover the details of distributing a game
that relies on the steam-runtime to hermeticize its environment, outside
of Steam.

[steam-runtime]: https://github.com/ValveSoftware/steam-runtime
[ld_library_path]: http://tldp.org/HOWTO/Program-Library-HOWTO/shared-libraries.html#AEN80
[extract_runtime]: https://gist.github.com/jorgenpt/07f207aefdd49b61c7b6#file-extract_runtime-sh
[update_runtime]: https://gist.github.com/jorgenpt/07f207aefdd49b61c7b6#file-update_runtime-sh
