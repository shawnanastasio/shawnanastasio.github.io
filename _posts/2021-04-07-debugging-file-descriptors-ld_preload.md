---
layout: post
title:  "Debugging incorrectly closed file descriptors with LD_PRELOAD"
date:   2021-04-07 12:00:00 -0500
categories: linux
---
I recently tracked down a bug in a relatively complex piece of software using the LD_PRELOAD mechanism and I
figured it was worth documenting it here in case anybody finds it useful or interesting.

For those unfamiliar, [LD_PRELOAD](https://en.wikipedia.org/wiki/Dynamic_linker#Systems_using_ELF) is a hack
present in the dynamic linker of most unix-like systems that allows hooking calls to any functions located in a
dynamic library.  It goes without saying that this is a pretty powerful tool that can be used in many
different ways, from [faking the system time](https://github.com/wolfcw/libfaketime), to [forcing all network
traffic through a proxy](https://github.com/haad/proxychains), to debugging, which is what I'll cover here.

# Discovering the Bug

As always, the first step in debugging is discovering an issue that you need to solve. In my case, I found an
issue while working on my [libkvmchan](https://github.com/shawnanastasio/libkvmchan) project, specifically its
daemon which is a multi-process multi-thread codebase with a homebrew IPC mechanism and many moving parts.

After adding a slew of seemingly minor changes to the codebase, I noticed that whenever one of the newly added
codepaths was executed, statements printing to stdout would no longer make it to my console window. The
logging facility that uses stderr was unaffected, which narrowed it down. After checking the usual suspects
(missing newline to flush the buffer, print statements not being hit, etc.), I began to suspect that the
process' stdout file descriptor had been closed somehow.

To confirm my suspicion, I checked the process' `/proc` entry, which is a kernel interface for querying
information on a running process including any file descriptors it has open.

```
$ ls -l /proc/$(pgrep kvmchand)/fd
total 0
lrwx------. 1 root root 64 Apr  7 13:28 0 -> /dev/pts/4
lrwx------. 1 root root 64 Apr  7 13:28 2 -> /dev/pts/4
lrwx------. 1 root root 64 Apr  7 13:28 3 -> 'socket:[358583]'
lrwx------. 1 root root 64 Apr  7 14:06 4 -> 'anon_inode:[eventpoll]'
lrwx------. 1 root root 64 Apr  7 14:06 5 -> 'socket:[358585]'
lrwx------. 1 root root 64 Apr  7 14:06 7 -> 'socket:[358587]'
```

Sure enough, file descriptors 0 and 2 (stdin and stderr respectively) were present and pointed to my console's
allocated pseudo-tty, but file descriptor 1 corresponding to stdout is absent. I double checked and confirmed
that fd 1 was present before my newly-added codepath gets hit and disappears afterwards, so the issue
definitely had to do with my new code causing fd 1 to be closed.

Strangely though, the new code I added had absolutely nothing to do with closing file descriptors! Grepping
the modified files for `close()` calls and adding a check for file descriptor 1 didn't catch anything either,
so it was clear that the new code was triggering a bug in an entirely different part of the codebase.

So now the question is, what is the most effective way to track down the erroneous close that is occurring
in a completely unknown part of this large, multi-threaded codebase? LD_PRELOAD to the rescue!

# LD_PRELOAD to the Rescue

Now that we know the bug is likely caused due to an erroneous call to `close()` with a file descriptor of 1, we
can discover the location of the bug rather trivially by intercepting all `close()` calls with LD_PRELOAD and
checking for a parameter of fd 1.

First, we create a new C file that defines a function in the global namespace with the same name and signature
as the function we want to hook. In our case, that signature is `int close(int fd)`. Then we simply need to
inspect the argument and perform some action if it's equal to 1, and forward it to the actual `close()` function
in libc otherwise.

What should be done when the invalid file descriptor argument is detected though? The easiest thing I could
think of was to execute an illegal instruction which would allow us to catch the exception in a debugger.
I'm on a POWER9 system, so I used the assembly pseudo-op `.long 0` to do this. On x86_64 you might want to
use the `int3` instruction instead to generate a software breakpoint.

Here's the implementation of the hook:
{% highlight C %}
#define _GNU_SOURCE

#include <stdio.h>
#include <unistd.h>
#include <dlfcn.h>

int close(int fd) {
    static int (*close_fn)(int) = NULL;
    if (!close_fn)
        close_fn = dlsym(RTLD_NEXT, "close");

    if (fd == 1)
        // Tried to close fd 1 - execute an illegal instruction
        asm volatile(".long 0\n");

    return close_fn(fd);
}
{% endhighlight %}

Breaking this down, the first thing our hook does is declare a static function pointer which is used to store
the address of the actual `close()` function provided by our system libc. It gets populated by a call to
`dlsym` which dynamically resolves the address of the real `close()`. For more information, check out
[dlsym's man page](https://linux.die.net/man/3/dlsym), specifically the section about `RTLD_NEXT`.

Next comes the check for file descriptor 1, which should only be hit by the bug. As discussed earlier, if the
check passes the code executes the opcode 0x00000000 which is guaranteed by the Power ISA to be an invalid
instruction and should thus raise a SIGILL. On x86_64 this should probably be replaced with the `int3` instruction.

Finally we simply forward the argument to the real `close()` function and return the result.

Now all that's left is building the hook and using it to track down the bug.

# Using the LD_PRELOAD hook

With the hook written and saved, compiling it is straightforward - we just need to provide a few flags to gcc
telling it to output a shared library and to link against `libdl`, which provides the `dlsym()` function we
used to grab the function pointer to the real `close()`.

```
$ gcc -shared -fPIC -ldl hook_close.c -o hook_close.so
```

We can now use LD_PRELOAD to run our application with the hook installed and then gdb to it:
```
$ LD_PRELOAD=$PWD/hook_close.so ./my_application &
$ gdb -p $(pgrep my_application)
...
(gdb) continue
```

It's also possible to launch the application with the hook from within gdb:
```
$ gdb ./my_application
...
(gdb) set environment LD_PRELOAD ./hook_close.so
(gdb) run
```

In my case, I went with the former method since the application I was debugging spawns multiple child
processes and it was easier to launch it normally and then attach gdb to the relevant PID.

With gdb attached, it's just a matter of triggering the bug and using the `backtrace` command to find
the erroneous callsite:
```
...
Thread 3 "kvmchand" received signal SIGILL, Illegal instruction.
Switching to Thread 0x7fff9ce4ed80 (LWP 54610)]
0x00007fffa02c0724 in close () from /home/shawnanastasio/opt/libkvmchan/hook_close.so
(gdb) backtrace
#0  0x00007fffa02c0724 in close () from /home/shawnanastasio/opt/libkvmchan/hook_close.so
#1  0x000000012702d990 in server_receiver_thread (data_=0x127050608 <g_ipc_data+144>) at daemon/ipc.c:529
#2  0x00007fff9fb59618 in start_thread () from /lib64/libpthread.so.0
#3  0x00007fff9fa68cb4 in clone () from /lib64/libc.so.6
(gdb) quit
```

The backtrace points to daemon/ipc.c:529 as the culprit. After observing the surrounding code the bug became
clear:

{% highlight C %}
int *fds = (msg.flags & IPC_FLAG_FD) ? msg.fds : NULL;
if (ipcmsg_send(socfd, &msg, sizeof(struct ipc_message), fds, msg.fd_count) < 0)
    goto fail_errno;

// Close fds now that we're done forwarding them
if (fds) {
    for (uint8_t i=0; i<IPC_FD_MAX; i++) {
        if (fds[i] != -1)
            close(fds[i]);
    }
}
{% endhighlight %}

This code is responsible for closing file descriptors passed via IPC messages after they have been forwarded
to their destination. The issue is that instead of iterating through the number of file descriptors present in
the array (`msg.fd_count`), it iterates through all values (`IPC_FD_MAX`). This works fine if
the user always sends `IPC_FD_MAX` (5) file descriptors (as was the case before my recent changes), but if the user sends
less, the code will end up passing uninitialized values to `close()` as long as they're not equal to -1.

In this case, it seems one of the uninitialized values of the array ended up being 1, which resulted in stdout
being closed. The fix was very straightforward to implement and was [committed as 8855a5680e59](https://github.com/shawnanastasio/libkvmchan/commit/8855a5680e59a2eb7b02aee6ea759b6e8e2dda36).


# Conclusion

So what did we learn then? Well firstly, care needs to be taken when dealing with array sizes and
uninitialized values when working in C (yes, I know you already knew that), and secondly, LD_PRELOAD is
a very powerful tool that allows for tracking down some pretty nasty bugs with relatively little effort.

I'm certainly not the first person to write about LD_PRELOAD debugging, but hopefully you found this 
write-up helpful or at least interesting.

Happy hacking!

