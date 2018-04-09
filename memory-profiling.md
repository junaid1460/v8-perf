TODO: merge cranshaft/memory-profiling.md

## Memory Inspection and HeapSnapshots

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
