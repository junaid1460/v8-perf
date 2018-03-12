# Ignition and TurboFan Compiler Pipeline

## Simplified Pipeline

- once crankshaft was taken out of the mix the below pipeline was possible

![new v8 pipeline](http://benediktmeurer.de/images/2016/v8-new-pipeline-20161125.png)

- enabled lots more optimizations
- reduces memory and startup overhead significantly
- AST no longer source of truth that compilers need to agree on (TODO: which is, IR generated
  by Ignition)?
- AST much simpler and smaller in size

### Pipeline as Part of New V8 Architecture

![new v8 pipeline detailed](http://benediktmeurer.de/images/2017/architecture-20170301.png)

## Collecting Feedback via ICs

- data-driven approach
- uses FeedbackVector attached to every function, responsible to record and manage all
  execution feedback to later speed up its execution

## CodeStubAssembler

- provides C++ based DSL to generate highly portable machne code
- can generate highly efficient code for parts of slow-paths in JS without crossing to C++
  runtime

### Improvements via CodeStubAssembler

- `Object.create` has predictable performance by using CodeStubAssembler
- `Function.prototype.bind` archieved final boost when ported to CodeStubAssembler for a total
  60,000% improvement
- `Promise`s where ported to CodeStubAssembler which resulted in 500% speedup for `async/await`

## Deoptimization Reasons

### Class Definitions inside Functions

```js
function createPoint(x, y) {
  class Point {
    constructor(x, y) {
      this.x = x
      this.y = y
    }

    distance(other) {
      const dx = Math.abs(this.x - other.x)
      const dy = Math.abs(this.y - other.y)
      return dx + dy
    }
  }

  return new Point(x, y)
}
function usePoint(point) {
  // do something with the point
}
```

- defining a class inside `createPoint` results in its definition to be executed on each
  `createPoint` invocation
- executing that definition causes a new prototype to be created along with methods and
  constructor
- thus each new point has a different prototype and thus a different object shape
- passing these objects with differing prototypes to `usePoint` makes that function
  become polymorphic
- v8 gives up on polymorphism after it has seen **more than 4** different object shapes, and enters
  megamorphic state
- as a result `usePoint` won't be optimized
- pulling the `Point` class definition out of the `createPoint` function fixes that issue as
  now the class definition is only executed once and all point prototypes match
- the performance improvement resulting from that results from this simple change is
  substantial, the exact speedup factor depends on the `usePoint` function

### Resource

- [optimization patterns part1](http://benediktmeurer.de/2017/06/20/javascript-optimization-patterns-part1/)

## Goals

### Smaller Performance Cliffs

- for most websites the optimizing compiler isn't important and could even hurt performance
  (speculative optimizations arent' cheap)
- pages need to load fast and unoptimized code needs to run fast _enough_, esp. on mobile
  devices
- previous v8 implementations suffered from _performance cliffs_
  - optimized code ran super fast (focus on peak performance case)
  - baseline performance was much lower
  - as a result one feature in your code that caused deoptimization would affect your app's
    performance dramatically, i.e. 100x difference
- TurboFan improves this as
  - widens fast path to ensure that optimized code is more flexible and can accept more types
    of arguments
  - reduces code memory overhead by reusing code generation parts of TurboFan to build Ignition
    interpreter
  - improves slow path

### New Language Features

- new language features are not useful by just being implemented
- need to be fast (at least matching transpiled code), related optimizations are easier with
  new pipeline
- need to support debugging and be inspectable, this is archieved via better integration with
  Chrome DevTools

## Facit

- performance of your code is improved
- write clean code with sensible data structures and it will run fast
- no need to work around architectural limitations of v8

## Resources

- [V8: Behind the Scenes (November Edition)](http://benediktmeurer.de/2016/11/25/v8-behind-the-scenes-november-edition/)
- [V8: Behind the Scenes (February Edition)](http://benediktmeurer.de/2017/03/01/v8-behind-the-scenes-february-edition/)
