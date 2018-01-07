---
layout: page
title:  Part 4 - Dynamic Memory
---
Now that we can boot and have a sane project structure, we can move on with developing the kernel itself.  Since we are the kernel, theoretically, we can use any memory
we want at any time.  To impose some order and prevent shooting ourselves in the foot, it makes sense to impliment a simple memory allocator.  It is important to note
that this allocator is not managing all of the memory.  It just manages a small piece that the kernel will use for its data structures.

If you want to download the code and play with it yourself, [see my git repo](https://github.com/jsandler18/raspi-kernel/tree/9eeb138e06d287cb8f42bc71587a2f6aa61c6583).

## Allocating Memory
In order to allocate memory, we need memory to allocate in the first place!  Since we are the
kernel, we are the boss.  We can take whatever memory we want and use it for whatever we want (with the excption of the [peripheral address
space](/extra/peripheral.html)).  I have decided to use the memory range 0x100000 - 0x200000.  This range starts at 1 MB and runs for 1 MB.  I chose this because it is
large enough that it is well outside the range of the kernel code, and small enough that it does not use up a significant chunk of memory that user code might want.

Now that we have a large chunk of memory, all we need is a function to divide it up!  We want to expose the familiar interface `void * malloc(uint32_t bytes)` in the files `src/kernel/mem.c` and `include/kernel/mem.h` to do this.
We are going to manage this by associating each allocation with a header.  The headers will form a linked list, so we can easily traverse from the beginning of one
allocation to the next.  It will also include a size, and whether the allocation is currently in use.  Here is the definition of the header structure:

``` c
typedef struct heap_segment{
    struct heap_segment * next;
    struct heap_segment * prev;
    uint8_t is_allocated: 1;
    uint32_t segment_size: 31;  // Includes this header
} heap_segment_t;
```

To allocate, all we need to do is find the allocation that best fits the number of requested bytes and is not in use. If that allocation is very large relative to the
size of the request, we can split up that allocation into two smaller ones, and only use one of them.  The criterion that I used to determine if an allocation needs to be
split is if the allocation is at least twice the size of a header, since it doesn't do much good to have many allocations that are half header and half data.  Once we have an allocation, we jus return a pointer to the memory directly after the header.  Here are those ideas implimented in code:

``` c
void * kmalloc(uint32_t bytes) {
    heap_segment_t * curr, *best = NULL;
    int diff, best_diff = 0x7fffffff; // Max signed int

    // Add the header to the number of bytes we need and make the size 4 byte aligned
    bytes += sizeof(heap_segment_t);
    bytes += bytes % 4 ? 4 - (bytes % 4) : 0;

    // Find the allocation that is closest in size to this request
    for (curr = heap_segment_list_head; curr != NULL; curr = curr->next) {
        diff = curr->segment_size - bytes;
        if (!curr->is_allocated && diff < best_diff && diff >= 0) {
            best = curr;
            best_diff = diff;
        }
    }

    // There must be no free memory right now :(
    if (best == NULL)
        return NULL;

    // If the best difference we could come up with was large, split up this segment into two.
    // Since our segment headers are rather large, the criterion for splitting the segment is that
    // when split, the segment not being requested should be twice a header size
    if (best_diff > (int)(2 * sizeof(heap_segment_t))) {
        bzero(((void*)(best)) + bytes, sizeof(heap_segment_t));
        curr = best->next;
        best->next = ((void*)(best)) + bytes;
        best->next->next = curr;
        best->next->prev = best;
        best->next->segment_size = best->segment_size - bytes;
        best->segment_size = bytes;
    }

    best->is_allocated = 1;

    return best + 1;
}
```

## Freeing Memory
Now that we have `malloc`, naturally we need `free` so that we don't fill up our 1 MB with data that will never be used again.  Obviously, we need to mark allocations as unused, so that another call to `malloc` will come and claim it back.  In addition, we need to coalesce adjacent free allocations back into one large one.  Here is the code:

``` c
void kfree(void *ptr) {
    heap_segment_t * seg = ptr - sizeof(heap_segment_t);
    seg->is_allocated = 0;

    // try to coalesce segements to the left
    while(seg->prev != NULL && !seg->prev->is_allocated) {
        seg->prev->next = seg->next;
        seg->prev->segment_size += seg->segment_size;
        seg = seg->prev;
    }
    // try to coalesce segments to the right
    while(seg->next != NULL && !seg->next->is_allocated) {
        seg->segment_size += seg->next->segment_size;
        seg->next = seg->next->next;
    }
}
```

## Initializing the Heap
Now that we have the algorithms, we need to initialize it.  To do this, we must put a header before the beginning of our 1 MB allocation that says there is a 1 MB unused allocation there, and assign `heap_segment_list_head` to this header. Finally, we must call this initialization function in `kernel_main`. Here is the code:

``` c
static heap_segment_t * heap_segment_list_head;

void mem_init(void) {
    // Set up malloc for use within the kernel
    heap_init();
    // TODO set up paging
}

static void heap_init(void) {
   heap_segment_list_head = (heap_segment_t *) KERNEL_HEAP_START;
   bzero(heap_segment_list_head, sizeof(heap_segment_t));
   heap_segment_list_head->segment_size = KERNEL_HEAP_SIZE;
}
```

The reason `heap_init` is inside this `mem_init` function is that later, `mem_init` will set up the rest of the memory.

Next up, we are going to get our kernel to print to a real screen.

**Previous**:
[Part 3 - Organizing our Project](/tutorial/organize.html)

**Next**:
[Part 5 - Printing to a Real Screen](/tutorial/hdmi.html)
