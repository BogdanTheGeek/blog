+++
title = 'MicroAlloc'
date = '2025-09-21T16:13:00+01:00'
author = 'Bogdan Ionescu'
description = 'A very small allocator'
tags = ['programming']
draft = false
cover = 'images/heap.png'
+++

# TLDR;
You can find the project repository [here](https://github.com/BogdanTheGeek/microalloc).

# What? How? Why?
A few Christmases ago, I was browsing the source code for the esp-idf [heap allocator](https://github.com/espressif/esp-idf/blob/release/v4.1/components/heap/multi_heap.c)[^1] and thought:
> This is quite interesting, I should write my own allocator

[^1]: The latest version of the idf ditched the old allocator for [tlsf](https://github.com/mattconte/tlsf)

After a bit of looking around, I discovered that general purpose heap allocators are one of those problems that has no perfect solution (which is the kind of problem I really enjoy). It's all about trade-offs. You can optimise for speed (which seems very much like the wrong thing to optimise for), allocation overhead, memory fragmentation, or all of them at the same time.

It took me a few hours to get a working first-fit free list allocator. Just an allocator, no `free()` which is actually where the complexity really lives, but we'll get to that. With my itched scratched I committed all my work on a local git repo and returned to my roasted turkey and pickled cabbages (we have a very multi-cultural Christmas).

Few months ago, I got interested once again in memory allocators after finding a great paper talking about [Embedded Dynamic Memory Allocator Optimisation](https://trepo.tuni.fi/bitstream/handle/10024/140229/AuvinenEetu.pdf?sequence=2).

After reading the paper end to end, I realised that as far as I could tell, very few people have tried pushing the limit when it came to minimising allocation overhead. With my renewed interest, I sat down and finished implementing `free()`, so that I could start playing with optimisations.

# Public Service Announcement
Before going any further, please allow me to share some industry secrets. Every self-respecting embedded engineer avoids dynamic allocations like the plague, myself included. There are much simpler, faster, and safer ways to deal with data of flexible sizes.

Statically allocating all your memory up front is by far the safest (as long as you leave enough room for your stack to grow), so you should always start with that.

If you need to allocate some large buffers depending on some configuration at startup, it's perfectly safe to allocate it dynamically and never free it.

For anything else, use a scratch buffer, block allocator (a.k.a. object pool), or arena allocator.

I don't think you should *never* use a dynamic allocator, but it should be a last resort, especially for embedded applications.

Now, with that out of the way, lets get back to our horses.

# The Meat and Potatoes
A first-fit free list allocator is quite a simple thing. Each block of memory has some metadata associated with it:

```c
typedef struct
{
    ua_base_t metadata; // LSB is used to indicate if the block is free
    union
    {
        unsigned char data[1]; // The actual data
        ua_base_t next_free;   // Pointer to the next free block
    };
} ua_block_t;
```

We keep track of these blocks as a linked list:

```c
typedef struct
{
    ua_block_t *last_block;
    ua_block_t *first_free_block;
} ua_heap_t;
```

We can actually do a little trick and allocate the `ua_heap_t` structure at the beginning of our `heap`. We then create a single large free block with the remaining memory.

When we want to allocate some memory, we traverse the list of free blocks until we find one that fits. If its bigger than we need, we can split it.
Crucially, we then return to the user the address of `block.data` instead of the address of the block. When we need to free, we can just look "above" the pointer provided by the user to find the metadata. We can use the metadata to store a pointer to the next block in the linked list.

After a block is freed, we can use the space inside the block to store a second linked list that points us to the next free block. This means that our effective overhead is just the size of the metadata.

Here is where we try to do something novel. Pretty much every other similar allocator uses a full `word` to store the pointers in the metadata field. What if you don't need more than 64kiB of heap? If we actually store an offset into the heap instead of an absolute pointer, we can get away with only 2 bytes of overhead per allocation.


# Garbage collection
While implementing `free()`, I realised that I had to have a way of joining adjacent free blocks, otherwise I could run out of large blocks. 
To keep overhead low, I didn't want to use a two way linked list, so it's quite slow to find a previous free block, but what if we could defer this merge? 

I added a compile time flag to disable "merge on free" and allow the user to call `micro_alloc_defrag()` whenever they need to tidy up things. Depending on the application, manual merging may produce a more fragmented heap, but it does improve performance significantly.

# Closing thoughts
Dynamic allocation is not magic, and writing your own allocator is a surprisingly rewarding exercise.
It took me a while to iron out most of the bugs, though some probably still remain.
For a bit of extra fun, here's how the heap looks after some "stress testing":

![heap](images/heap.png)

Green blocks are free, blue ones are allocated and red is all the metadata.
All the code used to generate it is in the repo linked above.



