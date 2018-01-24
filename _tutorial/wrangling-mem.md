---
layout: page
title:  Part 04 - Wrangling Memory
---
Now that we can boot and have a sane project structure, we can move on with developing the kernel itself.  Since we are the kernel, theoretically, we can use any memory
we want at any time.  To impose some order and prevent shooting ourselves in the foot, we need to wrangle all of that memory into a sane system.
The best way to do this is to split memory into 4 KB blocks called *pages*.  Pages allow us to allocate blocks of memory that are not insignificantly small, but also not
so large that they take up a significant fraction of memory.

If you want to download the code and play with it yourself, [see my github](https://github.com/jsandler18/raspi-kernel/tree/62f68bf494eefd372b28ef1ce54841e310f82295)

## Getting the Size of Memory
If we want to organize all of memory, we need to know how much memory we actually have availible.  We can do this by accessing the [atags](/extra/atags.html).

We can match the memory layout of the atags by defining some C types:
``` c
typedef enum {
    NONE = 0x00000000,
    CORE = 0x54410001,
    MEM = 0x54410002,
    ... 
} atag_tag_t;

typedef struct {
    uint32_t size;
    uint32_t start;
} mem_t;

...

typedef struct atag {
    uint32_t tag_size;
    atag_tag_t tag;
    union {
        mem_t mem;
        ...
    };
} atag_t;
```

Then we can iterate over the Atag list until we find the `MEM` tag:
``` c
uint32_t get_mem_size(atag_t * tag) {
   while (tag->tag != NONE) {
       if (tag->tag == MEM) {
           return tag->mem.size;
       }
       tag = ((uint32_t *)tag) + tag->tag_size;
   }
   return 0;

}
```

If you are developing for the VM, this will not work.  The VM does not emulate the bootloader which sets up the atags.  Since you determine the exact size of memory
through QEMU options, you should just have this function return that amount of memory.  My Makefile sets the memory to be 128 MB, so this function should return `1024 *
1024 * 128`.

## Wrangling Memory
Now that we can get the total size of memory, we can divide that memory up into pages.  The number of pages is simply the memory size divided by the page size.  

These pages are going to need some metadata around them.  They will need to keep track of if they have been allocated or not, and for what purpose.  They will need more metadata later when we enable virtual memory. Here is our metadata type:
``` c
typedef struct {
    uint8_t allocated: 1;           // This page is allocated to something
    uint8_t kernel_page: 1;         // This page is a part of the kernel
    uint32_t reserved: 30;
} page_flags_t;

typedef struct page {
    uint32_t vaddr_mapped;  // The virtual address that maps to this page   
    page_flags_t flags;
    DEFINE_LINK(page);
} page_t;
```

In order to hold all of the metadata, we reserve a large swath of memory just after the kernel image for an array of page metadata.  We can get this address by using the symbol `__end` that we declared in [the linker script](/explanations/linker_ld.html).  Addtionally, we will create a linked list (implementation explained [here](/explanations/list_h.html)) to keep track of which pages are free.

Once this is done, all we need to do is iterate over the pages and initialize their metadata and add them to the free list.

Here is the code:
``` c
extern uint8_t __end;

static uint32_t num_pages;

DEFINE_LIST(page);
IMPLEMENT_LIST(page);

static page_t * all_pages_array;
page_list_t free_pages;

void mem_init(atag_t * atags) {
    uint32_t mem_size,  page_array_len, kernel_pages, i;

    // Get the total number of pages
    mem_size = get_mem_size(atags);
    num_pages = mem_size / PAGE_SIZE;

    // Allocate space for all those pages' metadata.  Start this block just after the kernel image is finished
    page_array_len = sizeof(page_t) * num_pages;
    all_pages_array = (page_t *)&__end;
    bzero(all_pages_array, page_array_len);
    INITIALIZE_LIST(free_pages);

    // Iterate over all pages and mark them with the appropriate flags
    // Start with kernel pages
    kernel_pages = ((uint32_t)&__end) / PAGE_SIZE;
    for (i = 0; i < kernel_pages; i++) {
        all_pages_array[i].vaddr_mapped = i * PAGE_SIZE;    // Identity map the kernel pages
        all_pages_array[i].flags.allocated = 1;
        all_pages_array[i].flags.kernel_page = 1;
    }
    // Map the rest of the pages as unallocated, and add them to the free list
    for(; i < num_pages; i++){
        all_pages_array[i].flags.allocated = 0;
        append_page_list(&free_pages, &all_pages_array[i]);
    }

}
```

## Allocating Pages
Now that we have all of the pages under control, it would be nice if we could allocate and free pages dynamically.  This can be done very simply.  Since pages are always 4 KB, all we need to do is find a page that has not been allocated and return a pointer to the memory.  To free, all we need to do is add the page metadata back to the free list.

Since pages are a constant 4 KB, we can get the page memory from the page metadata by just multiplying the page metadata's index in the page array by 4096.  Similarly, we can get the page's metadata from the page memory by dividing by 4096 and using that as an index into the page array.

Here is the code:
``` c
void * alloc_page(void) {
    page_t * page;
    void * page_mem;

    if (size_page_list(&free_pages) == 0)
        return 0;

    // Get a free page
    page = pop_page_list(&free_pages);
    page->flags.kernel_page = 1;
    page->flags.allocated = 1;

    // Get the address the physical page metadata refers to
    page_mem = (void *)((page - all_pages_array) * PAGE_SIZE);

    // Zero out the page, big security flaw to not do this :)
    bzero(page_mem, PAGE_SIZE);

    return page_mem;
}

void free_page(void * ptr) {
    page_t * page;

    // Get page metadata from the physical address
    page = all_pages_array + ((uint32_t)ptr / PAGE_SIZE);

    // Mark the page as free
    page->flags.allocated = 0;
    append_page_list(&free_pages, page);
}

```

The ability to allocate pages is nice, but the strict 4 KB size is a bit restrictive for most dynamic memory use cases.  Next, we are going to add to this code to implement a dynamic memory allocator that can give you any sized allocation you want!

**Previous**:
[Part 3 - Organizing our Project](/tutorial/organize.html)

**Next**:
[Part 5 - Dynamic Memory Allocator](/tutorial/dyn-mem.html)
