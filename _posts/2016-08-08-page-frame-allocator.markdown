---
layout: post
title:  "OSDEV: Implementing a basic x86 page frame allocator in C"
date:   2016-08-08 16:42:10 -0500
categories: osdev memory
---
When writing an operating system for any architecture, one of the most important
functions you'll have to write is the page frame allocator. The page frame allocator allows the OS
to divide the physical memory available on a system into chunks called page frames,
which can then be used later to allocate memory to applications with a separate
paging function.

There are many methods of page frame allocation, each with varying levels of
complexity and efficiency. For an overview of many of these methods, see the
[OSDev Wiki](http://wiki.osdev.org/Page_Frame_Allocation). In this post we'll be
going over a much less efficient method than those outlined in the wiki for simplicity's
sake, but you can improve upon it later.

Unlike those outlined in the wiki, our page frame allocator isn't actually going
to be storing any data of its own. Instead, we're going to piggyback off of
the Multiboot information structure. If you're following this tutorial, you
hopefully already have a basic Multiboot kernel set up, as well as a basic
understanding of the Multiboot specification (the PDF for which can be found
[here][mboot-pdf]).

Without reading the entire specification, the rundown is that your Multiboot-compliant
bootloader (probably GRUB) will provide you with an information structure that contains
a lot data about the system you're running on. Among this data is the Multiboot
memory map, which provides a listing of all memory regions in the system, and
which ones you're allowed to write to (many regions are reserved for things like
VGA video output).

To access this memory map, we must first pass a pointer to the information structure
to our kernel's entry point. Luckily, our bootloader will put this pointer in the
`ebx` register when we first boot up, so all we have to do is pass it as a parameter
to our entrypoint in assembly. Along with `ebx`, we'll also pass `eax` which contains
a "magic value", `0x2BADB002`, which you can use to verify that the bootloader is
in fact Multiboot-compliant.

## Getting the Multiboot information structure
Before we start, we have to make sure that we've declared to our bootloader
that we want the memory map to be included in the information structure, as it is
optional by default. If you're using the boot assembly code from the
[OSDev Bare Bones][osdev-barebones] tutorial, this is already done for you.
The code to specify we want the memory map looks like this (at the top of your file):
{% highlight nasm %}
MEMINFO     equ 1<<1
; You can add more flags here separated by |
FLAGS       equ MEMINFO
MAGIC       equ 0x1BADB002
CHECKSUM    equ -(MAGIC + FLAGS)

.section multiboot
align 4
    dd MAGIC
    dd FLAGS
    dd CHECKSUM
{% endhighlight %}

We'll also have to make sure that the initial stack is at least 4096 bytes
(the size of a single page). As with the previous step, if you're using the
OSDev Bare Bones code this is done already. I'll be setting it to 16384 bytes,
but anything greater than 4096 bytes should suffice.
This change needs to be done in your `boot.asm` or equivalent file.
{% highlight nasm %}
section .bootstrap_stack, nobits
align 4
stack_bottom:
    resb 16384
stack_top:

...

section .text
global _start
_start:
    ; Use the stack we allocated above
    mov esp, stack_top

    ; Ensure stack is 16-bit aligned
    and esp, -16
{% endhighlight %}

Now that our stack is big enough, we can pass the Multiboot information structure
as well as the magic value to our kernel entrypoint. Remember that we must push the
values in the reverse order that we want to use them:

{% highlight nasm %}
section .text
global _start
_start:
    ...
    ; Push Multiboot information structure
    push ebx

    ; Push Multiboot magic value
    push eax

    ; Call our kernel entrypoint
    call kernel_main
{% endhighlight %}

Now we can access these values in our kernel's C entrypoint, we just have to
change its parameters:

{% highlight C %}
void kernel_main(uint32_t mboot_magic, void *mboot_header) {
    ...
}
{% endhighlight %}

## Defining the Multiboot information structure
To read the Multiboot information structure, we first need to declare a few
`struct`s in C that define its layout. Luckily the aforementioned
[Multiboot specification PDF][mboot-pdf] has them written out for you on page 16.
While it does declare more than we'll need for this tutorial, it's not a bad idea
to include the whole file anyways as it may be useful in the future.

Once you have the structures and constants from the PDF included into your kernel,
there's one more thing we need to do before we can start parsing the information
structure.

## Validating the Multiboot magic value
As stated before, the Multiboot magic value is simply a constant that all
Multiboot-compliant bootloaders will provide to prove that they are compliant.
Before we even attempt to read the Multiboot information structure, we should
verify this value so that we don't try to read garbage in the event that the kernel
is not booted properly.
{% highlight C %}

void kernel_main(uint32_t mboot_magic, void *mboot_header) {
    ...
    // MULTIBOOT_BOOTLOADER_MAGIC is defined on page 16 of the PDF too
    if (mboot_magic != MULTIBOOT_BOOTLOADER_MAGIC) {
        // If the magic is wrong, we should probably halt the system.
        printf("Error: We weren't booted by a compliant bootloader!\n");
        panic();
    }
}
{% endhighlight %}

## Preparing to read the Multiboot information structure
Now that we've verified the magic value, we can read the Multiboot information structure
and look for the memory map. As it's possible the memory map wasn't included
(in the case of an incomplete bootloader or other error), it's always good to
check for its existence.
{% highlight C %}
void kernel_main(uint32_t mboot_magic, void *mboot_header) {
    ...
    // First, cast the pointer to a multiboot_info_t struct pointer
    multiboot_info_t * mboot_hdr = (multiboot_info__t *)mboot_header;

    // The specification states that bit 6 signifies the presence of the memory map
    // We can check the header flags to see if it's there
    if ((mboot_hdr->flags & (1<<6)) == 0) {
        // The memory map is not present, we should probably halt the system
        printf("Error: No Multiboot memory map was provided!\n");
        panic();
    }
}
{% endhighlight %}

We're almost there, but first, let's set some global variables that will help us
keep track of things and make our lives easier.

{% highlight C %}
/**
 * At the top of your file, we'll make some global variables that will help us
 * keep track of frames. I'll explain what these do in a bit.
 */
multiboot_info_t *verified_mboot_hdr;
uint32_t mboot_reserved_start;
uint32_t mboot_reserved_end;
uint32_t next_free_frame;

void kernel_main(uint32_t mboot_magic, void *mboot_header) {
    ...
    /**
     * After we've verified the information structure, let's update our
     * global variables.
     */
    verified_mboot_hdr = mboot_hdr;
    mboot_reserved_start = (uint32_t)mboot_hdr;
    mboot_reserved_end = (uint32_t)(mboot_hdr + sizeof(multiboot_info_t));
    next_free_frame = 1;
}
{% endhighlight %}

As you can see, we've made quite a few variables. First, we've got a pointer,
`verified_mboot_hdr` where we can store our pointer to the information structure
so other functions can access it easily. Next, we've got `mboot_reserved_start`
and `mboot_reserved_end` which, surprisingly enough, store the start and end
locations of the Multiboot information structure in memory so we don't accidentally
overwrite it later. Finally, we've got `next_free_frame`, which stores the number
of the next free frame. This will be explained in greater detail in a later section.


## A page frame allocator
We've gone through all the steps to verify and prepare to read the Multiboot
information structure, but up until now you didn't know why. As stated before,
our page frame allocator is going to piggyback off of the Multiboot memory map
to keep track of allocated frames. We're going to do this by having a single
counter that increases by one every time we allocate a 4096 chunk (frame) of free memory
in the Multiboot memory map from start to end. This way, frame #1 is always the
first 4096 bytes marked as free in the memory map, #2 is the second 4096 bytes,
and so on. This is what the `next_free_frame` variable in the previous section is
for.

Before we can write the actual allocator, we need a method to read the memory map
and retrieve memory addresses that correspond to frame numbers, and vice versa.
For this we're going to create a separate function that accesses those varibles
we created earlier.
{% highlight C %}
#define MMAP_GET_NUM 0
#define MMAP_GET_ADDR 1
#define PAGE_SIZE 4096

/**
 * A function to iterate through the multiboot memory map.
 * If `mode` is set to MMAP_GET_NUM, it will return the frame number for the
 * frame at address `request`.
 * If `mode` is set to MMAP_GET_ADDR, it will return the starting address for
 * the frame number `request`.
 */
uint32_t mmap_read(uint32_t request, uint8_t mode) {
    // We're reserving frame number 0 for errors, so skip all requests for 0
    if (request == 0) return 0;

    // If the user specifies an invalid mode, also skip the request
    if (mode != MMAP_GET_NUM && mode != MMAP_GET_ADDR) return 0;

    // Increment through each entry in the multiboot memory map
    uintptr_t cur_mmap_addr = (uintptr_t)verified_mboot_hdr->mmap_addr;
    uintptr_t mmap_end_addr = cur_mmap_addr + verified_mboot_hdr->mmap_length;
    uint32_t cur_num = 0;
    while (cur_mmap_addr < mmap_end_addr) {
        // Get a pointer to the current entry
        multiboot_memory_map_t *current_entry =
                                        (multiboot_memory_map_t *)cur_mmap_addr;

        // Now let's split this entry into page sized chunks and increment our
        // internal frame number counter
        uint64_t i;
        uint64_t current_entry_end = current_entry->addr + current_entry->len;
        for (i=current_entry->addr; i + PAGE_SIZE < current_entry_end; i += PAGE_SIZE) {
            if (mode == MMAP_GET_NUM && request >= i && request <= i + PAGE_SIZE) {
                // If we're looking for a frame number from an address and we found it
                // return the frame number
                return cur_num+1;
            }

            // If the requested chunk is in reserved space, skip it
            if (current_entry->type == MULTIBOOT_MEMORY_RESERVED) {
                if (mode == MMAP_GET_ADDR && cur_num == request) {
                    // The address of a chunk in reserved space was requested
                    // Increment the request until it is no longer reserved
                    ++request;
                }
                // Skip to the next chunk until it's no longer reserved
                ++cur_num;
                continue;
            } else if (mode == MMAP_GET_ADDR && cur_num == request) {
                // If we're looking for a frame starting address and we found it
                // return the starting address
                return i;
            }
            ++cur_num;
        }

        // Increment by the size of the current entry to get to the next one
        cur_mmap_addr += current_entry->size + sizeof(uintptr_t);
    }
    // If no results are found, return 0
    return 0;
}
{% endhighlight %}

While this code looks daunting, the basic concept behind it is quite simple.
It essentially does the following:


* Read each entry in the memory map
* Split each entry into frame-sized chunks (4096 bytes)
* Iterate through each chunk until a valid address is found for the given frame number
or a frame number is found for the given address.

Now that we can read the memory map, creating an allocator can be done quite
simply:
{% highlight C %}
/**
 * Allocate the next free frame and return it's frame number
 */
uint32_t allocate_frame() {
    // Get the address for the next free frame
    uint32_t cur_addr = mmap_read(next_free_frame, MMAP_GET_ADDR);

    // Verify that the frame is not in the multiboot reserved area
    // If it is, increment the next free frame number and recursively call back.
    if (addr >= multiboot_reserved_start && addr <= multiboot_reserved_end) {
        ++next_free_frame;
        return allocate_frame();
    }

    // Call mmap_read again to get the frame number for our address
    uint32_t cur_num = mmap_read(cur_addr, MMAP_GET_NUM);

    // Update next_free_frame to the next unallocated frame number
    next_free_frame = cur_num + 1;

    // Finally, return the newly allocated frame num
    return cur_num;
}
{% endhighlight %}

Now, to allocate a page frame you simply have to call
{% highlight C %}
uint32_t new_frame = allocate_frame();
uint32_t new_frame_addr = mmap_read(new_frame, MMAP_GET_ADDR);
printf("New frame allocated at: 0x%x\n", new_frame_addr);
{% endhighlight %}

## Conclusion
Congratulations! You now have a working (albeit primitive) method of page frame
allocation for your operating system. It cannot yet free frames, but it provides
a good starting point for developing a paging system. In the future, I may write
another post improving upon this allocator with freeing implemented, as well as
some performance enhancements.

## Resources
If you get stuck anywhere, I recommend taking a peek at my example OS
[ShawnOS](https://github.com/shawnanastasio/ShawnOS), as well as the plethora
of online resources related to OS development, such as the
[OSDev Wiki](http://wiki.osdev.org/).

[mboot-pdf]: https://www.gnu.org/software/grub/manual/multiboot/multiboot.pdf
[osdev-barebones]: http://wiki.osdev.org/Bare_Bones
