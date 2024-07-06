---
layout: post
title: Recent kernel & userspace changes in SerenityOS
---

Since [the fork with Ladybird has happened a month ago](https://awesomekling.substack.com/p/forking-ladybird-and-stepping-down-serenityos), things changed in the administration
of the project, as the project no longer has a project leader, so everything in the project is decided by a group of maintainers. It will be very interesting to see the new path of the project from now on, and I trust that group
to do well on our lovely project :)

Regardless of that change, I'm mainly focused on continuing the work on my upstreamed patches which includes some work on the kernel, but also on userspace as well.

Since I'm not a maintainer (but I do some review on interesting patches from time to time), [Nico](https://github.com/nico) and [Tim](https://github.com/timschumi) have been extremely helpful on code review and merging my patches recently :)

Here's a list of interesting work (in my opinion) this month and what can be
expected in the foreseeable future:

## Containers instead of Jails

There's a waiting [a pull request ](https://github.com/SerenityOS/serenity/pull/22968) that intends to replace
most of what I started as "Jails".
Simply put, Jails were a kernel feature I seemingly thought as way of creating
containerized, isolated spaces for programs, mainly being used for security reasons.

I realized that putting more features together makes the whole machinery a bit complicated
to handle with one kernel object, so I took the Linux's approach:
- Being jailed means you restrict yourself to a baseline of known rules.
It still needs to determined which base restrictions are valid, so further work on this is expected.
- Every other feature in a container is opt-in (PID/filesystem/hostname isolation)
- Building a highly customizable container is something everyone should be able to do!

Currently, as stated before, the pull request is pending further work on my side but I hope that
it will be merged soon.

## Userspace is a lovely place

When I started working on the SerenityOS project about 5 years ago, I didn't really bother to work on
userspace improvements and instead focused almost entirely on the kernel.

Since then, my perspective changed a lot and I had lots of fun working on userspace stuff, especially bringing
new & neat features and polishing some rough edges on things I'd want to ensure will not hurt my productivity
once I use SerenityOS as a daily-driver.

The main feature and seemingly soon-to-be-merged I worked on is preparing the system
for [handling hot-plug events](https://github.com/SerenityOS/serenity/pull/24210) elegantly.
The main idea is to have daemon (called `DeviceMapper`) that propagates events to other
services through creation or removal of nodes in a special directory in `/tmp`.

An interested program could register a file watcher on a corresponding device type (and its associated subdirectory)
and be notified of changes for that device node family.

In addition to that, I've worked last month on these new utilities:
- [A new watchfs utility](https://github.com/SerenityOS/serenity/pull/24494), to be able view changes of files or directories in live.
- [A new `init` utility](https://github.com/SerenityOS/serenity/pull/24574), that takes the responsibility of initializing the system out of `SystemServer`
into the new utility.
- [A new utility](https://github.com/SerenityOS/serenity/pull/24571) to convert human-readable sizes into a raw number. This is useful on many occasions, which I will not discuss here.

I also added a quite known feature of [testing a TCP-listening service](https://github.com/SerenityOS/serenity/pull/24506) with the `nc` utility.

## More kernel work

As I worked on system-wide hotplug handling, I was setting a goal this month to refactor some parts of hot-plug
and events' handling in the networking space of the project.

Some [initial progress has been done](https://github.com/SerenityOS/serenity/pull/24545) already, but it remains
to be seen what path this pull request takes.

Lastly, I sent patches to make [the prekernel stage a bit more reasonable to debug](https://github.com/SerenityOS/serenity/pull/24430).
It should be improved upon, but currently assertions can actually print messages to
the debug console, which was not possible beforehand, and any crash was just hanging the machine
with no apparent reason to view.
