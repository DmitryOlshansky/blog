---
layout: post
title:  "Inside D's GC"
date:   2017-06-14 00:38:00
categories: gc runtime dlang
---

At the DConf 2017 hackathon I adventurously led a druntime group (of 2 persons) hacking on D's GC. After a few hours I couldn't shake off the nagging feeling "boy, this could use a rewrite". So I decided to start on the quest of better GC for D, the first iteration being faster classic mark-sweep collector.

To explain my motivation I'm going to describe the internals of current GC, enumerating issues with the design. All in all, one should analyze what we have to understand where to go from here.

## Pools, pools everywhere
If we were to ignore some global paraphernalia the GC is basically an array of pool objects. Each pool is a chunk of mmap-ed memory + a bunch of malloc-ed metadata such as tables for mark bits, free bits and so on. All allocation happens inside of a pool, if not a single pool is capable to service an allocation a new pool is allocated.  The size of a pool is determined by arithmetic progression on the number of pools or 150% of the size of allocation, whatever is bigger. 

Importantly pools come in two flavors: small object pool and large object pool. Small pools allocate objects up to 2kb in size, the rest is serviced by large pools. Small pools are actually more interesting so let's look at them first. 

![Small pool structure](/assets/images/SmallPool.jpg "Small pool structure")
Any small allocation is first rounded up to one of power of 2 size classes - 16, 32, 64, 128, 256, 512, 1024, 2048.  Then a global freelist for this size class is checked, failing that we go on to locate a small pool. That small pool would allocate a new page and link it up as a free list of objects of this size class. Here comes the first big mistake of the current design - size class is assigned on a page basis, therefore we need a table that maps each page of a pool to a size class (confusingly called pagetable). Now to find the start of an object by internal pointer we first locate the page it belongs to, then lookup the size class, and finally do a bitmask to get to the start of object. More over metadata is a bunch of simple bit-tables that now has to cope with heterogeneous pages, it does so by having ~7 bits per 16 bytes regardless of the object size.

What motivated that particular design? I have 2 hypotheses. First is being afraid to reserve memory for underutilized pools, which is a non-issue due to virtual memory with lazy commit. Second is being afraid of having too many pools, slowing down allocation and interestingly marking. The last one is more likely the reason, as indeed GC does a linear scan over pools quite often and a binary search for every potential pointer(!) during the marking phase.

That brings us to the second mistake - pool search in logP where P is a number of pools, which makes mark a NlogP business. A hash table could have saved quite a few cycles.

Concluding our overview of small pool, we should also look at the selection of size classes. This is a third issue (not a mistake, but controversial choice) having power of 2 sizes guarantees us up to 50% of [internal fragmentation](https://en.m.wikipedia.org/wiki/Fragmentation_(computing)#Internal_fragmentation).  Modern allocators such as [jemalloc](https://m.facebook.com/notes/facebook-engineering/scalable-memory-allocation-using-jemalloc/480222803919/)  typically provide for one more size class in between powers of 2. Modulo by a constant that is not a power of 2 is a bit slower than a single bit AND but still quite affordable. 

![Large pool structure](/assets/images/LargePool.jpg "Large pool structure")

Let's have a look at large object pools. First thing to note is that its granularity is a memory page (4kb) for both metadata and allocations. Free runs of pages are linked in one free list which is linearly scanned for every allocation request. This is the 4th mistake, that is not bothering with performance of large object allocation at all. To locate a start of an object a separate table is maintained where for every page an index of the start of the object it belongs to is stored. The scheme is sensible until one considers big allocations of 100+ Mb, as it will likely fail to reallocate in place causing a new pool to be allocated and would waste huge amounts of memory on metadata for essentially one object.

## Collection

So far we observed the allocation pipeline, deallocation follows the same paths. What's more interesting is automatic reclamation which is the whole point of GC.  Let me first dully note that D's GC is conservative meaning that it doesn't know if something is a pointer or not, and secondly it supports finalizers, actions that are run on object before reclaiming its memory.  These two decisions heavily constrain the design of a collector.

From a high level view a collection is surprisingly a whole 4 phase process: [prepare](https://github.com/dlang/druntime/blob/master/src/gc/impl/conservative/gc.d#L2106), [markAll](https://github.com/dlang/druntime/blob/master/src/gc/impl/conservative/gc.d#L2144), [sweep](https://github.com/dlang/druntime/blob/master/src/gc/impl/conservative/gc.d#L2172) and [recover](https://github.com/dlang/druntime/blob/master/src/gc/impl/conservative/gc.d#L2291).  

Prepare stage is the most dubious, essentially it should have been "copy of free bits to mark bits" (to prevent scanning of free memory). The waters are muddied by actually calculating all of free space by walking free lists. This is  5th(?) mistake - leap frogging an additional untold amount of pointers is the last thing to do during stop the world pause.  A better design would be  flipping free bits during allocation/deallocation, especially since free list maintains pointers to pool for each object so pool search is not required.

The actual marking phase is markAll call, that just delegates apropriate ranges of memory to mark function. That [mark](https://github.com/dlang/druntime/blob/master/src/gc/impl/conservative/gc.d#L1955) deserves a closer look. 
1. For each pointer in memory range it will first check if it hits the address range of the GC heap (smallest address in a pool to highest address in a pool). 
2. Following that a dreaded binary search to find the right pool for the pointer.
3. Regardless of the pool type a lookup into a pagetable for the current pool to see its size class or if it's a large object or even free memory that is not scanned. There is a tiny optimization in that this lookup also tells us if it's a start of large object or continuation. We have 3 cases  - small object, large object start or large object continuation, the last two are identical save for one extra table lookup.
4. Determining the start of the object - bit masking with the right mask plus in case of large object continuation an offset table lookup. 
5. In case of large object there is a "no interior pointer" bit that allows ignoring internal pointers to the object.
6. Finally check and set bit in mark bits, if wasn't marked before or noscan bit is not set add the object to the stack of memory to scan.

Ignoring the curious stack limiting manipulations (to avoid stack overflow yet try to use the stack allocation) this is all there is to mark function. Apart from already mentioned pool search, deficiencies are still numerous. Mixing no-pointers memory (noscan) with normal memory in the same pool gives us an extra bit-table lookup on hot path. Likewise a pagetable lookup could be easily avoided had we segregated the pools by size class. Not to mention the dubious optimization of no interior pointers bit that not only promotes unsafe code (an object can be collected while still being pointed at) but also introduces a few extra checks on the critical path for all large objects, including a potential bit-table lookup.

That was quite involved but keep in mind that mark phase is the heart of any collector. Now to the 3rd phase - sweep. Ironically in current D's GC sweep doesn't sweep to free lists as one would expect. Instead all it is concerned about is calling finalizers (if present) and setting free bits and other tables. Bear in mind that this is doing a linear pass through the memory of each pool looking at mark bits.

Final stage - recover, this will actually rebuild free lists. It is again a linear pass across pools but only the small ones.  Again a pagetable lookup for every page of a pool to learn a size class, it just makes me want to cry. But the main unanswered question is why? Why an extra pass? I tried hard to come up with a reasonable cause but couldn't, "for simplicity" is a weak but probable explanation. This is the last big mistake by my count. 

## What's not there
So far I've been criticizing things that are in plain sight, now it's time to go on to things that are simply non-existent. 

Thread cache is one big omission, keeping in mind that D's GC is dated by early 2000s it's not that surprising.  Every modern allocator has some form of thread cache, some try to maintain a per processor cache. A cache works by each thread basically doing allocations in bulk, keeping a stash of allocations for the future. This amortizes the cost of locking the shared data-structures of heap. Speaking of which there is a bit of fine grained locking present, but not say on per pool level.

Parallel marking is another example of modern feature that is now totally expected of any garbage collector. Also quite popular are concurrent and mostly concurrent GCs whereby the marking and less often sweep/compaction is done while application threads are running.

## Closing thoughts

The post got pretty lengthy and more involved then I would hope. Still I belive it carries the message across - D's GC is slow not because of some fundamental limitation but because of a half a dozen or so of bad implementation decisions. In the same vein one could easily build a precise generational GC that is slow, purely missing out on good implementation techniques. 

Now to sum up what my first iteration attempts to change compared to this baseline.
- Segregate small pools on size class.
- Make O(1) pool search.
- Try to use more size classes including non-power of 2,  a-la jemalloc, to defeat internal fragmentation.
- Segregate all pools based on no-scan attribute, this streamlines marking.
- Provide 3rd class of "pools" for huge allocations (16+ Mb) intended for single objects only.
- Large object pool allocation needs more thought, a kind of tree keyed on block length might be in order. jemalloc uses red-black trees.
- Drop no interior pointer attribute, it tries to combat false pointers due to conservative GC. However it is fundamentally unsafe and the price is too high, lastly it's completely pointless on 64-bit systems where your servers are.
- No whacky multi-phase collection in mark-sweep cycle, it's mark and sweep, that's it.
- Fine grained locking from the start, I see no problem with per pool locking.

The second iteration will focus on more juicy stuff such as thread cache, parallel marking and concurrent marking utilizing the fork trick.

The third iteration (if we get to it) would be a completely new design - a mark-region collector with design inspired by [immix](http://www.cs.utexas.edu/users/speedway/DaCapo/papers/immix-pldi-2008.pdf). 

This concludes my plans and on this optimistic note I will warn that it's going to start as Linux specific, slowly becoming POSIX specific with Windows support being a distant posibility.

