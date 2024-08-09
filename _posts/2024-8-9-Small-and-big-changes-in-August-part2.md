---
layout: post
title: Small & big changes in August - part 2
---

Almost 2 weeks ago, I wrote about plans I have for August, and since then there
are updates and exciting things to write on. There's not a lot to see (yet, as many efforts are still done locally)
but hopefully they will be published soon :)

## More general improvements for filesystems and mounts

The [pull request](https://github.com/SerenityOS/serenity/pull/24734) to change
the storage of filesystem-specific options to be in a `AK::HashMap` instead of a raw `KBuffer` was merged this week :)

Now that work is merged, there's an opportunity to do many other cool things on the filesystem front. One of them is [immutable mounts](https://github.com/SerenityOS/serenity/pull/24915), which should enforce non-removable mounts as another layer of security and stability in the system.

I also started to work on is a mechanism similar to `dirent cache` (which is inspired from the Linux kernel).
Currently, it's quite immature so there's only a local git branch on my machine. Hopefully this work will get done soon so I can publish this :)

## Making `BuggieBox` almost completely static

I managed to compile an almost-completely static version of `BuggieBox` locally.
I did so in 2 separate git branches - one branch for a binary that has no dependencies
and a second branch that keeps `libc.so`, `libgcc_s.so` and `libsystem.so` as dependencies.

The first branch has some issues, with the PLT/GOT getting corrupted or unpopulated accordingly, which leads
to trying to run code from a function which its pointer is null.

The second branch works completely OK, but still has 3 libraries that should be included so the program can work.

## EROFS support

I started this week to work on `EROFS` support, which is a filesystem designed for containers' usage on Linux systems.

The design is extremely simple, because `EROFS` is always immutable when mounted, and is designed to drastically improve direct IO over other archiving solutions such as squashfs images or tar archives, by keeping the internal filesystem structures as simple as possible.

I plan on making use of `EROFS` as a fundamental block in the upcoming containers' ecosystem. I also plan on using it for creating initramfs images, for a rescue environment or just boot-from-RAM situations.

There's still not much to see for now, as I work on it locally until I have a basic functioning code to mount an `EROFS` archive (probably using a loop device).

## Auto-jailing programs for free & improving the durability of `execve` syscall

As I already spoke about this, I sent [a pull request](https://github.com/SerenityOS/serenity/pull/24764) to introduce an auto-jailing version of the dynamic loader (the normal version's path is `/usr/lib/Loader.so`), which if used on a program, forces running of the program in jailed mode.

It now turned into something similar in the idea of having a separate auto-jailing dynamic loader, but
instead of compiling a whole new version of `/usr/lib/Loader.so`, we instead use a symlink, which is more lean
on storage usage.

As a result, it makes the `execve` syscall much more understandable and approachable now, by re-arranging the huge `Process::do_exec` as more atomic, smaller methods.

It should be noted that the pull request now directly relies on [another pull request](https://github.com/SerenityOS/serenity/pull/24791), to get rid of bad assumptions earlier in the `execve` syscall on how large the read buffer should be.

## Small fixes for userspace utilities

I really love the way our utilities grow organically. Many utilities have way less features you would expect
from, compared to their versions on a typical Linux system (either GNU-based or not). I strongly believe we should continue improving these utilities, one step after another. Of course, there are alternatives like `coreutils`, but I still find it satisfying that we don't rely on these alternatives for making a complete system.

Recently I sent 2 pull request improving 3 utilities:
 - [Utilities: Add an option to get total size in du, Add option to only print paths in elfdeps](https://github.com/SerenityOS/serenity/pull/24900) - the purpose is to run the `du -Th $(elfdeps -s /bin/BuggieBox)` command, which helps me identifying what should be removed from the list of `BuggieBox`'s dependencies and what is the size of each dependency.
 - [Utilities/du: Set minimum width when printing sizes](https://github.com/SerenityOS/serenity/pull/24901) - I just noticed a small alignment issue when printing human readable sizes, so this should resolve it now.
