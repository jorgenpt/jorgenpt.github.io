---
layout: post
title: "DT_RPATH (ld) & @rpath (dyld)"
date: 2014-05-20 22:06:29 -0700
comments: true
categories:
    - Linux
    - Mac OS X
---

Mac and Linux have two similarly named concepts that both deal with
dynamic loading, that behave quite differently: `@rpath` (under Mac OS
X's dyld) and `DT_RPATH` (or just rpath, under Linux' ld.)

Having done development (and more importantly, deployment) on both of
these platforms, I've experienced first-hand how those concepts can get
a little jumbled in your mind, so here's a brief overview.

<!-- more -->

## DT\_RPATH

DT\_RPATH, or more commonly just rpath, is a property set on an ELF
file[^1]. It points to a list of directories that the dynamic linker
will consider when loading a shared library. DT\_RPATH is set at
link-time with the `-rpath` option to `ld`.  If you invoke `ld` through
`gcc` (or another compiler, like `g++`), then you can use the `-Wl`
option to pass arguments through to `ld`. You use commas to separate
arguments passed to `-Wl`.

```
$ gcc program.c -lm -o program '-Wl,-rpath,$ORIGIN/lib'
$ ldd program | grep libm
        libm.so.6 => /lib/x86_64-linux-gnu/libm.so.6 (0x00007f3b9ad94000)
$ mkdir lib && cp /lib/x86_64-linux-gnu/libm.so.6
$ ldd program | grep libm
        libm.so.6 => /home/jorgenpt/lib/libm.so.6 (0x00007f1440b0c000)
```

The snippet above also shows one of the three special variables[^2] you
can include in an rpath, $ORIGIN. $ORIGIN gets replaced at runtime with
the directory in which our executable lives. DT\_RPATH is transitive,
meaning it applies to any dependencies of our dependencies (unlike
DT\_RUNPATH, but I won't talk about that here.) If our executable links
with libfoo, and libfoo depends on libbar, libfoo will include our rpath
in its search for libbar.

$ORIGIN is also commonly expanded by bash or zsh, so we use single
quotes around our `-Wl,-rpath,$ORIGIN/lib` option to prevent that from
happening.

To specify multiple paths, separate them by a colon, like
`-Wl,-rpath,$ORIGIN/lib:$ORIGIN/lib/amd64`.

As you might be able to tell, rpath is great for creating self-contained
applications. You still have to be careful, as any libraries that are
missing from your rpath will still be (silently) searched for in the
system directories. I highly recommend asking users for `ldd` output if
you're trying to debug something with your dependencies.

Many people use LD\_LIBRARY\_PATH to achieve a similar effect.
LD\_LIBRARY\_PATH is not set at link-time, but rather as an environment
variable when your application is run. This is for example what [Valve's
steam-runtime][steam-runtime] does to guarantee that your dynamically
linked libraries will be picked from the Steam runtime libraries rather
than the system libraries.

The benefit of using LD\_LIBRARY\_PATH is that it can be set for
applications you cannot edit, but the downside is that it also applies
to any applications launched by the application in question. Say that
you have an application that launches `dbus-send` or `aplay` -- since
they're system applications, you'd want them to pick their dependencies
from the system, not your LD\_LIBRARY\_PATH.

Interaction between LD\_LIBRARY\_PATH and your application's rpath is
well-defined: Your rpath is searched first, and anything it can't find
there it'll look for in LD\_LIBRARY\_PATH. Finally, if searches the
system directories[^3].

## @rpath

While @rpath is named similarly to its Linux cousin, it behaves a bit
differently. When you dynamically link to a library on Mac OS X, the
linker stores the "install name" of the library inside your executable.
The install name is something that comes from the dylib you're linking
against, and by default it is the absolute path of the linked file. You
can change the install name by modifying the dylib after linking[^4].

After your application has been linked, you can change what the
application thinks the install name is for one of its dependent
libraries[^5].

Your application can set its own rpath at link-time using the same
`-Wl,-rpath,@executable_path` magic, but note that instead of $ORIGIN,
you use @executable\_path or @loader\_path. @executable\_path behaves
like $ORIGIN, @loader\_path is the directory of whatever object is doing the
loading, which could be a dylib that your application has loaded. For
details, [read this excellent article by Wincent Colaiuta][at-paths] and
[this blog post by Mike Ash][at-paths-ash].

This rpath does *not* do anything by default. To make it take effect,
the install name for the shared library has to start with `@rpath/` --
and the dynamic linker will then substitute each of the possible values
for `@rpath` in order. This means that you'll typically change the
install name of the dylib (if it's a dylib you built yourself) or change
the install name inside the application.

Under Mac OS X, you have the DYLD\_LIBRARY\_PATH environment variable --
and this behaves just like it does on Linux. When DYLD\_LIBRARY\_PATH is
set, it is checked before the install name (and therefore, @rpath) is
consulted.


## Summary

Hopefully this helps you understand some nuances of dynamic linking on
Mac OS X versus Linux. In my next blog post, I hope to show how you can
use DT\_RPATH on Linux to link with the Steam runtime when distributing
your game outside of Steam.

[^1]: This also applies to .so's - when one of your dynamically loaded libraries load another dynamic library, their rpath is searched first (if any), then your main application's rpath is searched.
[^2]: The other two variables are `$LIB` and `$PLATFORM`, and they deal with finding architecture-specific binaries.
[^3]: The truth is a little more complicated, see the ld.so manpage for more info. (http://man7.org/linux/man-pages/man8/ld.so.8.html)
[^4]: See the man page for install\_name\_tool (`install_name_tool -id @rpath/my.dylib my.dylib`)
[^5]: See the man page for install\_name\_tool (`install_name_tool -change old.dylib @rpath/new.dylib my_application`)

[steam-runtime]: https://github.com/ValveSoftware/steam-runtime
[at-paths]: https://wincent.com/wiki/@executable_path,_@load_path_and_@rpath
[at-paths-ash]: https://www.mikeash.com/pyblog/friday-qa-2009-11-06-linking-and-install-names.html
