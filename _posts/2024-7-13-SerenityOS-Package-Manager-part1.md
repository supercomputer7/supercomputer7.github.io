---
layout: post
title: SerenityOS Package Manager (part 1/N)
---

The SerenityOS package manager (called `pkg` in short, on the actual system) is a program
that is intended to be used to install port packages in the system in a somewhat standardized
fashion.

## No rule for kernel-userspace ABI compatibility

It should be noted that the SerenityOS doesn't have a restriction on changes to kernel-userspace ABI at all right now.
The project is still considered to be immature on that aspect, and things still change (although not rapidly as in
the first days of the project).

The obvious implication of this, is that we can't ship prebuilt (also called "binary") packages.
Almost any other Linux based operating system (Ubuntu, Debian, archlinux, etc) does that because Linux has a very
known rule of not breaking userspace.

In SerenityOS, we have a rule of not breaking userspace, but this rule has another meaning. If you're interested
to understand this rule, you may want to look it [here](https://github.com/SerenityOS/serenity/blob/master/Documentation/Kernel/DevelopmentGuidelines.md#we-dont-break-userspace---the-serenityos-version).

## But why there's no rule of not breaking kernel-userspace ABI compatibility?

Simply put, in SerenityOS there's a strong policy of writing everything ourselves.
That includes all the userspace utilities, libraries, etc.

With that mindset effectively creating a monorepo as a codebase for all base system (kernel, userspace utilities, programs and services), we're free to change what we want in the kernel syscall ABI and then just re-compile everything. This is how it has been done for a very long time by now, and everyone seem to be happy with this.

## Why is that approach actually fine for the time being

When you start developing a big project like SerenityOS, there's nothing that hold you back -
you don't have users, you do mistakes and you need to fix them immediately, you don't have release schedules, etc.

What you obviously don't want to do is to tie yourself so early to keep old stuff as a compatibility layer.

We might have to re-think this policy and be more conservative on breaking kernel-userspace ABI sometime
in the future, but we're still not there as far as I consider this, so maybe another review should be done in a couple of years :)

## The implication of doing port packages

Ports are always considered "optional" in the SerenityOS project. They're prone
to breakage over time, and it takes some manual work to ensure they're usable.

As such, system upgrades could potentially make a port unusable, because of changes in the base system. 
With that mindset being taken into consideration, port packages are no different from that implication.

For that problem, we could technically try to make LibC a non-moving part as much as possible.
We also try to be compatible with POSIX as much as possible, but programs that don't adhere to the POSIX specification are probably never going to be stable in terms of workability.

We are probably never going to officially support statically compiled ported software, because ABI breakage somewhere is guaranteed to happen at some point and we probably don't even want to allow massive static compilation on our base system.

As for shipping packages from remote repositories - I hardly believe anyone in the project would want to mess around with shipping prebuilt port packages. Dealing with such aspect of ports will increase the amount of problems our community will need to solve (for example, where do we store prebuilt packages, how do we sign them, etc).

## Why a port package manager is needed

Currently our ports' ecosystem has grown quite organically and has over 300+ ported software!

The defacto maintainer-in-charge for this subsystem in our project is [Tim](https://github.com/timschumi) and he does
a lovely job ensuring everything stays intact.
I work closely with Tim on bringing new ports, updating some others from time to time and also working on
the package manager.

What our ports' ecosystem fails to do right now is to ensure users can manage ports on the actual system.
When installing a port, you currently need to run a shell script from the host (for example, your Linux machine) and install everything to the guest (SerenityOS VM). Uninstalling a port is almost impossible to do, and if you want to uninstall a port which is a dependency to another, you can't easily observe which ports still need it.

Therefore, the goal of the package manager (which is still under development) is to create a standardized
way to do all of these:
- Install prebuilt port packages (more on that later)
- Compile a port package
- Remove an installed port
- Query for a port that can be compiled & installed
- View a dependency graph for a port

Basically, this incomplete list sets a line where we might feel comfortable enough to use ports
in the actual system without resorting to the development (host) environment for actual management.

## Building a port package & installing pre-built port packages

This might confuse some people - I just said we don't want to ship pre-built port packages,
so why on earth should I allow to install one?

The answer is quite simple - we might not want to deal with shipping with pre-built packages
in a way that is used to exist on Linux-based operating system, but building a port package locally
still has a value instead of just compiling the port and blindly installing it - it gives us the control
of what can be installed and removed when needed.

Therefore, to properly manage the ports' ecosystem on SerenityOS, you would need to compile
and build a port package and move it to an accessible location on the filesystem to SerenityOS.
Presumably, this package would be a TAR archive, with all needed files to track and install the port.

Then, you simply use the package manager to install that port package, which is tracked by some metadata
files inside the TAR archive.

## The structure of a port package (as a TAR archive)

You might want to view the current implementation [here](https://github.com/supercomputer7/serenity/commit/4e04b6f7ec9d070138a26f1bf6f79e2ba538f2a1).

Currently, each port package has 3 components inside:
- A `Root` directory, which has all the needed files that should be copied over to the root filesystem.
- A file called `files` to indicate which files are installed and should be tracked by this port.
- A file called `details`, which has a list of all necessary dependencies for this port.

## Where to follow from now on

As stated before, you can view my progress on my forked repository WIP branch [here](https://github.com/supercomputer7/serenity/commits/build-essential-port/).

I already [sent this feature as pull request](https://github.com/SerenityOS/serenity/pull/24681), so stay tuned for more progress :)
