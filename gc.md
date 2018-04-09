# v8 Memory Management

## Orinoco Garbage Collector

[watch orinoco overview]([watch](https://youtu.be/EdFDJANJJLs?t=15m10s)) | [jank and concurrent GC](https://youtu.be/HDuSEbLWyOY?t=5m14s) |
[read](https://v8project.blogspot.com/2016/04/jank-busters-part-two-orinoco.html)

- mostly parallel and concurrent garbage collector without _strict_ generational boundaries
- most parts of GC taken off the main thread (56% less GC on main thread)
- optimized weak global handles
- unified heap for full garbage collection
- optimized v8's black allocation additions
- reduced peak memory consumption of on-heap peak memory by up to 40% and off-heap peak memory
  by 20% for low-memory devices by tuning several GC heuristics

### Generational Garbage Collector

- two garbage collectors are implemented, each focusing on _young_ and _old_ generation
  respectively
- young generation evacuation ([more details](#tospace-fromspace-memory-exhaustion))
  - objects initially allocated in _nursery_  of the _young generation_
  - objects surviving one GC are copied into _intermediate_ space of the _young generation_
  - objects surviving two GCs are moved into _old generation_

```
        young generation         |   old generation
                                 |
  nursery     |  intermediate    |
              |                  |
 +--------+   |     +--------+   |     +--------+
 | object |---GC--->| object |---GC--->| object |
 +--------+   |     +--------+   |     +--------+
              |                  |
```

### Parallel Scavenger

[read](https://v8project.blogspot.com/2017/11/orinoco-parallel-scavenger.html)

- introduced with v8 v6.2 which is part of Node.js v8
- older v8 versions used Cheney semispace copying garbage collector that divides young
<<<<<<< HEAD
  generation in two equal halves, read more about it [here](../gc.md#tospace-fromspace-memory-exhaustion) TODO fix link once repo rearranged
- single threaded scavenger made sense on single-core environments, but at this point Chrome,
=======
  generation in two equal halves and [performed moving/copying of objects that survived GC
  synchronously](crankshaft/gc.md#tospace-fromspace-memory-exhaustion)
  - single threaded scavenger made sense on single-core environments, but at this point Chrome,
>>>>>>> ba4dddc... fix: gc
  Node.js and thus v8 runs in many multicore scenarios
- new algorithm similar to the [Halstead semispace copying collector](https://www.cs.cmu.edu/~guyb/papers/gc2001.pdf)
  except that v8 uses dynamic instead of static _work stealing_ across multiple threads

#### Scavenger Phases

As with the previous algorithm scavenge happens in four phases.
All phases are performed in parallel and interleaved on each task, thus maximizing utilization
of worker tasks.

1. scan for roots
    - majority of root set are the references from the old generation to the young generation
    - [remembered sets](#tracking-pointers) are maintained per page and thus naturally distributes
      the root sets among garbage collection threads
2. copy objects within the young generation
3. promote objects to the old generation
    - objects are processed in parallel
    - newly found objects are added to a global work list from which garbage collection threads can
      _steal_
4. update pointers

##### Distribution of scavenger work across one main thread and two worker threads

![parallel scavenger](https://1.bp.blogspot.com/-fqUIuq6zXEg/Wh2T1lAM5nI/AAAAAAAAA8M/g183HuHqOis6kENwJGt9ctloHEaXEQlagCLcBGAs/s1600/image4.png)
![threads](https://3.bp.blogspot.com/-IQcY0MHevKs/Wh2T08XW7wI/AAAAAAAAA8I/EluBNmwT2XIPZNkznSRUml6AmOJWZiJwQCLcBGAs/s1600/image3.png)


#### Results

- just a little slower than the optimized Cheney algorithm on very small heaps
- provides high throughput when heap gets larger with lots of life objects
- time spent on main thread by the scavenger was reduced by 20%-50%

### Techniques to Improve GC Performance

#### Memory Partition and Parallelization

- heap memory is partitioned into fixed-size chunks, called _pages_
- _young generation evacuation_ is archieved in parallel by copying memory based on pages
- _memory compaction_ parallelized on page-level
- young generation and old generation compaction phases don't depend on each other and thus are
  parallelized
- resulted in 75% reduction of compaction time

#### Tracking Pointers

[read](https://v8project.blogspot.com/2016/04/jank-busters-part-two-orinoco.html)

- GC tracks pointers to objects which have to be updated whenever an object is moved
- all pointers to old location need to be updated to object's new location
- v8 uses a _rembered set_ of _interesting pointers_ on the heap
- an object is _interesting_ if it may move during garbage collection or if it lives in heavily
  fragmented pages and thus will be moved during compaction
- _remembered sets_ are organized to simplify parallelization and ensure that threads get
  disjoint sets of pointers to update
- each page stores offsets to _interesting_ pointers originating from that page

#### Black Allocation

[read](https://v8project.blogspot.com/2016/04/jank-busters-part-two-orinoco.html)

- assumption: objects recently allocated in the old generation should at least survive the next
  old generation garbage collection and thus are _colored_ black
- _black objects_ are allocated on black pages which aren't swept
- speeds up incremental marking process and results in less garbage collection

### Resources

- [Getting Garbage Collection for Free](https://v8project.blogspot.com/2015/08/getting-garbage-collection-for-free.html)
  _maybe outdated except the scheduling part at the beginning_?
- [Jank Busters Part One](https://v8project.blogspot.com/2015/10/jank-busters-part-one.html)
  _outdated_?
- [Jank Busters Part Two: Orinoco](https://v8project.blogspot.com/2016/04/jank-busters-part-two-orinoco.html)
  _outdated_ except for paging, pointer tracking and black allocation?
- [V8 Release 5.3](https://v8project.blogspot.com/2016/07/v8-release-53.html)
- [V8 Release 5.4](https://v8project.blogspot.com/2016/09/v8-release-54.html)
- [Optimizing V8 memory consumption](https://v8project.blogspot.com/2016/10/fall-cleaning-optimizing-v8-memory.html)
- [Orinoco: young generation garbage collection](https://v8project.blogspot.com/2017/11/orinoco-parallel-scavenger.html)

<<<<<<< HEAD
## Memory Inspection and HeapSnapshots
=======
## Old Generation Garbage Collector Deep Dive

- fast alloc
- slow collection performed infrequently and thus in most cases doesn't affect application
  performance as much as the more frequently performed _scavenge_
- `~20%` of objects survive into **Old Generation**

### Collection Steps

[watch](http://youtu.be/VhpdsjBUS3g?t=12m30s)

- parts of collection run concurrent with mutator, i.e. runs on same thread our JavaScript is executed on
- [incremental marking/collection](http://www.memorymanagement.org/glossary/i.html#term-incremental-garbage-collection)
- [mark-sweep](http://www.memorymanagement.org/glossary/m.html#term-mark-sweep): return memory to system
- [mark-compact](http://www.memorymanagement.org/glossary/m.html#term-mark-compact): move values
>>>>>>> ba4dddc... fix: gc

TODO: merge in older material

- v8's ability to dynamically increase its heap limit allows taking heap snapshot when close to
  running out of memory
- `set_max_old_space_size` is exposed to v8 embedders as part of the _ResourceConstraints_ API
  to allow them to increase the heap limit
- DevTools added feature to pause application when close to running out of memory
  1. pauses application and increases heap limit which allows taking a snapshot, inspect the
  heap, evaluate expressions, etc.
  2. developer can then clean up items that are taking up memory
  3. application can be resumed

### Resources

- [One small step for Chrome, one giant heap for V8](https://v8project.blogspot.com/2017/02/one-small-step-for-chrome-one-giant.html)
