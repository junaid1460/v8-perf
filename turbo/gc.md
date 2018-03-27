# v8 Garbage Collector

## Orinoco Garbage Collector

[watch orinoco overview]([watch](https://youtu.be/EdFDJANJJLs?t=15m10s)) | [jank and concurrent GC](https://youtu.be/HDuSEbLWyOY?t=5m14s)

- most parts of GC taken off the main thread (56% less GC on main thread)
- optimized weak global handles
- unified heap for full garbage collection
- optimized v8's black allocation additions
- reduced peak memory consumption of on-heap peak memory by up to 40% and off-heap peak memory
  by 20% for low-memory devices by tuning several GC heuristics

### Generational Garbage Collector

- objects initially allocated in _nursery_ of the _young generation_
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

- two garbage collectors are implemented, each focusing on _young_ and _old_ generation
  respectively
- older v8 versions used Cheney semispace copying garbage collector that divides young
  generation in two equal halves, read more about it [here](../gc.md#tospace-fromspace-memory-exhaustion) TODO fix link once repo rearranged
- starting with v8 6.2 a parallel scavenger is used to collect the young generation

### Parallel Scavenger

[read](https://v8project.blogspot.com/2017/11/orinoco-parallel-scavenger.html)

- introduced end of 2017 with v8 v6.2 which is part of Node.js v8
- single threaded scavenger made sense on single-core environments, but at this point Chrome,
  Node.js and thus v8 runs in many multicore scenarios
- new algorithm similar to the [Halstead semispace copying collector](https://www.cs.cmu.edu/~guyb/papers/gc2001.pdf)
  except that v8 uses dynamic instead of static _work stealing_ across multiple threads

#### Scavenger Phases

As with the previous algorithm scavenge happens in four phases.
All phases are performed in parallel and interleaved on each task, thus maximizing utilization
of worker tasks.

1. scanning for roots

- majority of root set are the references from the old generation to the young generation
- remembered sets are maintained per page (TODO more details in other articles) and thus
  naturally distributes the root sets among garbage collection threads

2. copying objects within the young generation
3. promoting objects to the old generation

- objects are processed in parallel
- newly found objects are added to a global work list from which garbage collection threads can
  _steal_

4. updating pointers

![parallel scavenger](https://1.bp.blogspot.com/-fqUIuq6zXEg/Wh2T1lAM5nI/AAAAAAAAA8M/g183HuHqOis6kENwJGt9ctloHEaXEQlagCLcBGAs/s1600/image4.png)
![threads](https://3.bp.blogspot.com/-IQcY0MHevKs/Wh2T08XW7wI/AAAAAAAAA8I/EluBNmwT2XIPZNkznSRUml6AmOJWZiJwQCLcBGAs/s1600/image3.png)

_Distribution of scavenger work across one main thread and two worker threads_

#### Results

- just a little slower than the optimized Cheney algorithm on very small heaps
- provides high throughput when heap gets larger with lots of life objects
- time spent on main thread by the scavenger was reduced by 20%-50%

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

### Old Generation

TODO

## Memory Inspection and HeapSnapshots

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
