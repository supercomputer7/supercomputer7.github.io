---
layout: post
title: Small & big changes in August
---

Last week I wrote about containers, and what should be done further on that front. Since then, that PR was merged by Tim, and we can move forward benefiting from that now.
I realized I have many plans for August that I could share them in a blog post. The plans are mostly related to changing things on userspace that should make the system more pleasant to use, and obviously more fun! :)

## Better internal handling of filesystem-specific options in Kernel

This is something I didn't plan beforehand, but after taking a look again
at the code I wrote to allow passing filesystem-specific options during the
mount procedure, I admittedly found it terribly confusing and error-prone.

Therefore, I sent a [pull request](https://github.com/SerenityOS/serenity/pull/24734) to fix this, by changing
the storage of filesystem-specific options to be in a `AK::HashMap` instead of a raw `KBuffer`.
In addition to that, filesystem implementations are longer responsible to store the options
on some obscure structure in that buffer, and instead are only given the key and a value
of a filesystem-specific option, to verify it.

This change also ensures that overriding a filesystem-specific option is not allowed anymore,
as the user needs to remove the option by its key first and then resend it with a new value.

## Interactive mount REPL

This idea has been cooking for a while, which is to allow a user
to test filesystem-specific flags and see the actual result - either from
verification of the filesystem implementation, or rather after mounting the
prepared filesystem on a path.

It's still under heavy development, so I don't have anything to show yet, but
hopefully somewhere in mid-August (but maybe before, who knows) there will be at least a draft pull request for this.

## Making `BuggieBox` a statically compiled program

I actually talked about [this in post last week](https://supercomputer7.github.io/Containers-and-the-future-of-ported-software-security/), that making `BuggieBox` a statically compiled program
is a good idea regardless of the work on containers.

The effort is mostly going locally, but [I sent a pull request](https://github.com/SerenityOS/serenity/pull/24815) to minimize the dependencies' list of `BuggieBox` which includes a built-in shell.

It remains to be seen if that effort takes another turn, and I still discuss
the matter with Tim on the Discord server.

## Loop devices are again "in the loop"

Loop devices are a neat feature to create a virtual block device that is attached to a filesystem inode,
which lets the user to treat it as a backing storage for all sorts of goodies (so you could mount it, for example).

I already spoke about this - the main goal for loop devices is to eventually be a mechanism to expose prebuilt images as root filesystems for containers.

Anyway, loop devices are still an ongoing effort, to ensure they're actually useful.
I sent a [pull request a while back](https://github.com/SerenityOS/serenity/pull/23814), to ensure that loop devices act reasonably. The main features and fixes (as stated in the pull request blurb) are:
- You can't remove a loop device when it's in use. It doesn't make sense to allow this, and we should allow re-use of a loop device (for example, to re-mount it) if so desired.
- When doing a loop mount, the user is responsible for cleaning up the loop device if it's no longer in use
- A user can create a new loop device manually and mount it whenever it's needed.
- When creating a loop device, the open file description should be readable and writable, as the loop device does not check if the file is not writable later on and assume that it was writable when we created it.

That pull request has a merge-conflict, which I should resolve soon and hopefully we can get
more review on this so this will be merged in the near future :)

## Auto-jailing programs for free

I recently sent [a pull request](https://github.com/SerenityOS/serenity/pull/24764) to introduce an auto-jailing
version of the dynamic loader (the normal version's path is `/usr/lib/Loader.so`), which if used on a program, forces running
of the program in jailed mode.

Currently, a user can either run it directly like this:
```
/usr/lib/ldjail.so YOUR_PROGRAM_PATH
```
or, set the program to be "interpreted" with `/usr/lib/ldjail.so`, so the kernel loads it as the
ELF program interpreter.

The pull request adds a new utility called `set-elf-jailed`, which changes the `.INTERP` section's data
to `/usr/lib/ldjail.so`. It does it in quite a quick & dirty way, but it works OK for now.

I also sent [a pull request](https://github.com/SerenityOS/serenity/pull/24761) that was merged by Tim to port the [`patchelf`](https://github.com/NixOS/patchelf/) utility, which can do that job in a more elegant way.

Going forward, I'd want to use this feature on all (and if not possible, on most) ported software that can be
used on SerenityOS, as a another layer of security almost for free.

## Removing singletons in the Kernel

The singleton pattern is currently widespread in the kernel code.
Running a quick `grep` on the kernel directory show these ARM64-specific
instances:

```
Kernel/Arch/aarch64/RPi/GPIO.h:    static GPIO& the();
Kernel/Arch/aarch64/RPi/Framebuffer.h:    static Framebuffer& the();
Kernel/Arch/aarch64/RPi/Mailbox.h:    static Mailbox& the();
Kernel/Arch/aarch64/RPi/UART.h:    static UART& the();
Kernel/Arch/aarch64/RPi/Watchdog.h:    static Watchdog& the();
Kernel/Arch/aarch64/RPi/SDHostController.h:    static SDHostController& the();
Kernel/Arch/aarch64/RPi/MMIO.h:    static MMIO& the();
```

There are also more "generic" instances all over the place:
- `NetworkingManagement`
- `MemoryManager`
- `AudioManagement`
- `SerialDevice`
- `StorageManagement`
- `HIDManagement`
- `PTYMultiplexer`
- `DeviceManagement`
- `GraphicsManagement`
- `InterruptManagement`
- `Timer`
- `InterruptManagement`
- `HPET`
- `APIC`
- `InterruptManagement`
- `TimerQueue`
- `TimeManagement`
- `USBManagement`
- `PCI::Access`
- `KernelRng`
- `SysFSComponentRegistry`
- `SysFSDisplayConnectorsDirectory`
- `SysFSStorageDirectory`
- `SysFSDeviceIdentifiersDirectory`
- `SysFSBlockDevicesDirectory`
- `SysFSCharacterDevicesDirectory`
- `SysFSUSBBusDirectory`

Many instances here are my fault, and I think we should get rid of them eventually.

The main drawback of this pattern is that it usually leads to ugly code
being written with that pattern. Using a C++ namespace instead in most cases makes a lot
more sense as well.

There are many ongoing efforts (also locally for now) to remove most of these singletons and
replace them with better code. One of them was [the containers' pull request](https://github.com/SerenityOS/serenity/pull/22968)
that was already merged, and [the other one](https://github.com/SerenityOS/serenity/pull/24698) is set to remove the `DeviceManagement` singleton.

## Improving the durability of `execve` syscall

Last week, I sent [a pull request](https://github.com/SerenityOS/serenity/pull/24763) to simplify loading of the ELF interpreter path in the kernel.

We had some problems that were resolved by this pull request in the `execve` syscall flow:
- We used to load some sections of the main program and its ELF interpreter and then checking
  the actual ELF type of the main program after all of this work, essentially throwing it all if it fails on that condition.
- We could panic if the specified ELF interpreter leads to a non-regular file.
- We could fail loading a program if the `.INTERP` section's data is after the first 4KiB of the main program.

The result of all of the mentioned problems being resolved, is that the `LibELF` `validate_program_headers` method
looks a lot better now, and the kernel `Process::find_elf_interpreter_for_executable` method acts more reasonably now as well.

After this pull request was merged, I noticed that we still have bad assumptions earlier in the syscall
on how large the read buffer should be.

Therefore, I sent [a pull request](https://github.com/SerenityOS/serenity/pull/24791) to change this, so we only read first the size of `Elf_Ehdr`, and if we detect that it has the shebang sign (known as `#!`), we currently read the first 4 KiB of the program to check what is the interpreter. 

That part still needs to be improved, because assuming the path length will not cross 4096 bytes is reasonable, but technically incorrect.

If we try to load an ELF file, we calculate where are the program headers, how long is that section, and then create a `FixedArray<u8>` with that size to be filled with the actual program headers.

This patch ensures that we no longer assume the program headers of an ELF file are on the first 4 KiB or just don't cross the boundary of the first 4 KiB.
