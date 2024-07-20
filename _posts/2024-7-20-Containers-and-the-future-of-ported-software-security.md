---
layout: post
title: Containers and the future of ported software's security
---

Last week I posted my thoughts on the [upcoming changes on the package manager](https://supercomputer7.github.io/SerenityOS-Package-Manager-part1/).
While I did that, I also thought thoroughly on what the future holds for a viable security when using ported software on SerenityOS.

## Porting software is a no-man's land, full of surprises

This title might confuse someone that read my previous (linked above) post. In that post, I said clearly that
we have a maintainer for this subsystem, which does a very good job maintaining it.

This is true, and the title doesn't contradict this statement. Nobody can attest instantly on the level of quality of any ported software they just encounter with.

What actually I mean by "no-man's land" or "full of surprises" is basically that nobody knows what can break at given time, and nobody knows each quirk or bug in each software package we would want to use.

This is actually fine in some way, because there are so many software packages out there (Debian 12, for example, has 64419 software packages in its repositories!), and knowing every bit or shtick of every software package is simply not possible.

Also, I did mention in that previous post that we don't require building ports in the base system.
Therefore, at any given moment we could technically change something that some ported software will behave badly with, and we simply won't have a clue until we test that software again.

Another major problem is the known [dependency hell problem](https://en.wikipedia.org/wiki/Dependency_hell), which luckily we avoided so far. There's also the issue of the actual porting software work, which could be simple as
writing a 8-line port script and call it a day, or spending hours or even days building a port and testing it.

As a conclusion - ports are a vast, never-ending land, for a user to explore, which makes it mostly fun but sometimes extremely tedious and frustrating.

## Review cycles and progress so far

I know I spoke a bit on containers on [my actual first blogpost](https://supercomputer7.github.io/June-2024-Recent-changes-in-SerenityOS/). Tim reviewed my patches, and I committed to review the suggestions he wrote, which led to a bunch of nice fixes on top of what I did already.
I am still waiting for the next review cycle so that patch series takes a final shape, and hopefully [that pull request](https://github.com/SerenityOS/serenity/pull/22968) will be merged soon :)

For example, Tim pointed out that we shouldn't break LibC API for `mount` or `umount` functions.
Another nice example was to put `BuggieBox` in `/bin` instead of `/tmp` when launching a container
as most people would expect it to just be there.

## Containers as security measure for ported software

For the base system we already have [`pledge`](https://man.serenityos.org/man2/pledge.html) and [`unveil`](https://man.serenityos.org/man2/unveil.html) syscalls as a simple sandboxing mechanism. Both of these syscalls are originated from the OpenBSD operating system project.

Therefore, almost all userspace programs already benefit from these syscalls to some extent.
However, there's a known caveat with `pledge` and `unveil` - the restrictions are cleared upon calling the `execve`
syscall (unless you set `unveil`/`pledge` promises for after-`execve`).

Containers play an important role in the overall system security paradigm I strive to create.

I wrote containers with simplicity in mind - no extra features which are not necessary right now, no new obscure syscalls or device nodes - everything in the flow of creating and launching a container should be as simple as possible, to avoid introducing bugs or misleading the user, which might result in a bad configuration that would hurt the overall system security.

By using containers, we can impose PID/filesystem restrictions in runtime, with no need to re-compile anything. This will make it extremely easy to create a tested sandbox for almost any software, with no need of
potentially wasting hours or even days to re-compile and test what works best, because containers are launched with a simple configuration file which can be changed instantly.

Containers are meant to be used on software which is not part of the base system - which is mostly ported software packages. We could technically add more patches to enable `pledge` or `unveil` on some
ports, but that will be a tedious effort. It's also guaranteed to break at some point, especially in a situation of updating the software version, or adding a new `pledge` promise, for example.

It should be noted that once a software is running in a container (which is set as jailed as well), it can never escape from the container and its restrictions, in contrast to `pledge` or `unveil`.

To summarize everything:

|                                | `pledge` & `unveil`  | Containers |
|--------------------------------|:---------------------------:|:----------:|
| Survives `execve`?             | No* [1]              | Yes                      | 
| Time of applying rules         | Compile-time         | Run-time                 | 
| User learning-curve* [2]       | Negligible           | Needs some tinkering to fully understand  |
| Requires a configuration file? | No                   | Yes                             |
| Current well-tested state      | Very good            | Needs more testing              |
| Introduction time              | 2020                 | 2024* [3]                       |
| Userland's sector              | Base system          | Ports (mostly), base system     |

<div><br></div>
<div class="small-text">[1] - As mentioned above, it's possible to use <span class="small-code">execpromises</span> or <span class="small-code">Kernel::UnveilFlags::AfterExec</span> to set settings after <span class="small-code">execve</span> is called, but it's not used often</div>
<div class="small-text">[2] - While containers might require some tinkering, one of the key features is verbosity by design, which simplifies understanding of configuration errors</div>
<div class="small-text">[3] - If the containers feature is merged, which will happen probably this year</div>

## Mitigating the "dependency hell" problem and making islands of conflicting versions of software packages

Like on Linux, the containers feature has an opportunity to not only improve the overall system security by
creating a sandbox for many ports, but it also might help installing the same software packages
with different versions and make them separated with containers, so each software package has its own dependencies
within the container and there's no conflict or contamination of a different version of some shared library, etc.

While this sounds great, there's little interest in this aspect of containers for the time being.

## Optimizations on filesystem isolation

Currently, firing a full-fledged filesystem-isolated [`BuggieBox`](https://github.com/SerenityOS/serenity/tree/master/Userland/BuggieBox) container takes a hot 2 seconds to complete until a shell is acquired.

While this might sound not much, `BuggieBox` is a relatively small program, so this can quickly pile up to multiple seconds, or maybe even worse, on more complicated programs.

I searched recently (for about half a year, to be precise) for a solution and came up with a couple of ideas.

### EROFS

Adding support for the [`EROFS`](https://lwn.net/Articles/934047/) in the kernel and in userspace (in form of `mkfs.erofs` utility) is the "best" option so far I came up with.

Then, we could simply build a prebuilt `EROFS` archive and mount it with a loop device as a root filesystem
for a container.

This approach obviously has a big advantage that once we have a prebuilt archive, we could deploy it
multiple times and also not worry about changing dependencies of a program in another version.

I also like the idea of having a non-mutable filesystem by default, which is optimized for containers' usage by default as well.

Arguably, it has a small disadvantage that we lose the neatness of launching an updated software instantly, but that
could be addressed otherwise by keeping an option to not use a prebuilt archive just like how it is now, when needed.

### TAR archives

Adding support for launching from a packed TAR archive is another option, similar to the previous one.

This approach has the advantage like the option above that we have everything we need in one archive.
It also doesn't require kernel support, because only userspace needs to care about extracting an archive to a new container environment.

The problem is that it might take sometime to extract everything when needed, and in contrast to `EROFS`, where
the kernel instantly exposes the archive to userspace (because the actual data is aligned on page boundaries), if we use a TAR archive there's no other option but to extract each file in the archive to the new environment.

### static `BuggieBox`

We could also make `BuggieBox` a statically-linked program. While this might be desireable anyway for other purposes (like creating a rescue environment), this will only solve the problem for `BuggieBox`.

This option is the least preferable in terms of flexibility or scalability. It could be almost impossible to re-compile more complicated programs statically, and this could be a huge effort for anyone trying to create a new container configuration, which defeats one of the key features of all of this - simplicity by design.
