# v8 Garbage Collector

## Orinoco Garbage Collector

[watch orinoco overview]([watch](https://youtu.be/EdFDJANJJLs?t=15m10s)) | [jank and concurrent GC](https://youtu.be/HDuSEbLWyOY?t=5m14s)

- most parts of GC taken off the main thread (56% less GC on main thread)
- optimized weak global handles
- unified heap for full garbage collection
- optimized v8's black allocation additions

## Resources

- [Getting Garbage Collection for Free](https://v8project.blogspot.com/2015/08/getting-garbage-collection-for-free.html)
  _maybe outdated except the scheduling part at the beginning_?
- [Jank Busters Part One](https://v8project.blogspot.com/2015/10/jank-busters-part-one.html)
  _outdated_?
- [Jank Busters Part Two: Orinoco](https://v8project.blogspot.com/2016/04/jank-busters-part-two-orinoco.html)
  _outdated_ except for paging, pointer tracking and black allocation?
- [V8 Release 5.3](https://v8project.blogspot.com/2016/07/v8-release-53.html)