# v8 Garbage Collector

## Orinoco Garbage Collector

[watch orinoco overview]([watch](https://youtu.be/EdFDJANJJLs?t=15m10s)) | [jank and concurrent GC](https://youtu.be/HDuSEbLWyOY?t=5m14s)

- most parts of GC taken off the main thread (56% less GC on main thread)
- optimized weak global handles
- unified heap for full garbage collection
- optimized v8's black allocation additions
- reduced peak memory consumption of on-heap peak memory by up to 40% and off-heap peak memory
  by 20% for low-memory devices by tuning several GC heuristics

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
