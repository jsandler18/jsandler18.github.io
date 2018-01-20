---
layout: page
title:  Part 05 - Dynamic Memory Allocator
---
In the last part we organized all memory into pages.  Now we are going to reserve a small chunk of that memory so we can allocate with more precision than  4 KB.

If you want to download the code and play with it yourself, [see my git repo](https://github.com/jsandler18/raspi-kernel/tree/cf8329869218a8f7a0278e495734010d2cb0a9b1).

## Allocating Memory
In order to allocate memory, we need memory to allocate in the first place!  Since we are the
kernel, we are the boss.  We can take the memory directly after the page metadata and reserve that for our heap.  The amount to reserve is somewhat arbitrary, so I chose 1 MB because it is
large enough that it should be sufficient for the kernel's dynamic memory needs, and small enough that it does not use up a significant chunk of memory that user code might want.

Now that we have a large chunk of memory, all we need is a function to divide it up!  We want to expose the familiar interface `void * malloc(uint32_t bytes)` in the files `src/kernel/mem.c` and `include/kernel/mem.h` to do this.
We are going to manage this by associating each allocation with a header.  The headers will form a linked list, so we can easily traverse from the beginning of one
allocation to the next.  It will also include a size, and whether the allocation is currently in use.  Here is the definition of the header structure:

``` c
typedef struct heap_segment{
    struct heap_segment * next;
    struct heap_segment * prev;
    uint32_t is_allocated;
    uint32_t segment_size;  // Includes this header
} heap_segment_t;
```

To allocate, all we need to do is find the allocation that best fits the number of requested bytes and is not in use. If that allocation is very large relative to the
size of the request, we can split up that allocation into two smaller ones, and only use one of them.  The criterion that I used to determine if an allocation needs to be
split is if the allocation is at least twice the size of a header, since it doesn't do much good to have many allocations that are half header and half data.  Once we have an allocation, we jus return a pointer to the memory directly after the header.  Here are those ideas implemented in code:

``` c
void * kmalloc(uint32_t bytes) {
    heap_segment_t * curr, *best = NULL;
    int diff, best_diff = 0x7fffffff; // Max signed int

    // Add the header to the number of bytes we need and make the size 16 byte aligned
    bytes += sizeof(heap_segment_t);
    bytes += bytes % 16 ? 16 - (bytes % 16) : 0;

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
Now that we have the algorithms, we need to initialize it.  To do this, we must reserve the pages we are going to use, put a header at the beginning of our 1 MB allocation that says there is a 1 MB unused allocation there, and assign `heap_segment_list_head` to this header. Finally, we must call this initialization function in `kernel_main`. Here is the code:

``` c
static heap_segment_t * heap_segment_list_head;

void mem_init(void) {
    ...
    kernel_pages = ((uint32_t)&__end) / PAGE_SIZE;
    for (i = 0; i < kernel_pages; i++) {
        all_pages_array[i].vaddr_mapped = i * PAGE_SIZE;    // Identity map the kernel pages
        all_pages_array[i].flags.allocated = 1;
        all_pages_array[i].flags.kernel_page = 1;
    }
    // Reserve 1 MB for the kernel heap
    for(; i < kernel_pages + (KERNEL_HEAP_SIZE / PAGE_SIZE); i++){
        all_pages_array[i].vaddr_mapped = i * PAGE_SIZE;    // Identity map the kernel pages
        all_pages_array[i].flags.allocated = 1;
        all_pages_array[i].flags.kernel_heap_page = 1;
    }
    // Map the rest of the pages as unallocated, and add them to the free list
    for(; i < num_pages; i++){
        all_pages_array[i].flags.allocated = 0;
        append_page_list(&free_pages, &all_pages_array[i]);
    }


    // Initialize the heap
    page_array_end = (uint32_t)&__end + page_array_len;
    heap_init(page_array_end)    
}

static void heap_init(uint32_t heap_start) {
    heap_segment_list_head = (heap_segment_t *) heap_start;
    bzero(heap_segment_list_head, sizeof(heap_segment_t));
    heap_segment_list_head->segment_size = KERNEL_HEAP_SIZE;
}
```

Next up, we are going to get our kernel to print to a real screen.

**Previous**:
[Part 4 - Wrangling Memory](/tutorial/wrangling-mem.html)

**Next**:
[Part 6 - Printing to a Real Screen](/tutorial/hdmi.html)
