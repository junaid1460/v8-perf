## Function Bind

### Why Was Bind Slow?

- performance of `Function.prototype.bind` and `bound` functions suffered from performance
  issues in crankshaft days
- language boundaries C++/JS were crossed both ways which is expensive (esp.  calling back from
  C++ into JS)
- two temporary arrays were created on every invocation of a bound function
- dut to crankshaft limitations this couldn't be fixed easily there

### What Changed?

- entirely new approach to how _bound function exotic objects_ are implemented
- crossing C++/JS boundaries no longer needed
- pushing bound receiver and bound arguments directly and then calling target function allows
  futher compile time optimizations and enables inlining the target function into the
  caller
- resulted in **~400x** speed improvement

### Facit

- developers should use bound functions freely wherever they apply without having to worry
  about performance penalties

### Resources

- [A new approach to Function.prototype.bind](http://benediktmeurer.de/2015/12/25/a-new-approach-to-function-prototype-bind/)
- [Optimizing bound functions
  further](http://benediktmeurer.de/2016/01/14/optimizing-bound-functions-further/)
- [bound function exotic
  objects](https://tc39.github.io/ecma262/#sec-bound-function-exotic-objects)

## instanceof and @@hasInstance

- latest JS allows overriding behavior of `instanceOf` via the `@@hasInstance` _well known
  symbol_
- naívely this requires a check if `@@hasInstance` is defined for the given object every time
  `instanceof` is invoked for it (in 99% of the cases it won't be defined)
- initially that check was skipped as long as no overrides were added EVER (global protector
  cell)
- Node.js `Writable` class used `@@hasInstance` and thus incurred huge performance bottleneck
  for `instanceof` ~100x, since now checks were no longer skipped
- optimizations weren't possible in these cases initially
- by avoiding to depend on global protector cell for TurboFan and allowing inlining `instancof`
  code this performance bottleneck has been fixed
- similar improvements were made in similar fashion to other _well-known symbols_ like
  `@@iterator` and `@@toStringTag`

### Facit

- developers can use `instanceof` freely without worrying about non-deterministic performance
  characteristics
- developers should think hard before overriding its behavior via `@@hasInstance` since this
  _magical behavior_ may confuse others, but using it will incurr no performance penalties

### Resources

- [V8: Behind the Scenes (November Edition)](http://benediktmeurer.de/2016/11/25/v8-behind-the-scenes-november-edition/)
- [Investigating Performance of Object#toString in ES2015](http://benediktmeurer.de/2017/08/14/investigating-performance-object-prototype-to-string-es2015/)

## Array Builtins

- `Array` builtins like `map` and `forEach` can be inlined into TurboFan optimized code which
  results in considerable performance improvement

- [V8: Behind the Scenes (February Edition)](http://benediktmeurer.de/2017/03/01/v8-behind-the-scenes-february-edition/)

## const

- `const` has more overhead when it comes to temporal deadzone related checks since it isn't
  hoisted
- however the `const` keyword also guarantees that once a value is assigned to its slot it
  won't change in the future
- as a result TurboFan skips loading and checking `const` slot values slots each time they are
  accessed (_Function Context Specialization_)
- thus `const` improves performance, but only once the code was optimized

### Facit

- `const`, like `let` adds cost due to TDZ (temporal deadzone) and thus performs slightly worse
  in unoptimized code
- `const` performs a lot better in optimized code than `var` or `let`

### Resources

- [JavaScript Optimization Patterns (Part 2)](http://benediktmeurer.de/2017/06/29/javascript-optimization-patterns-part2/)

## Iterating Maps and Sets via `for of`

- `for of` can be used to walk any collection that is _iterable_
- this includes `Array`s, `Map`s, `Set`s, `WeakMap`s and `WeakSet`s

### Why was it Slow?

- set iterators where implemented via a mix of self-hosted JavaScript and C++
- allocated two objects per iteration step (memory overhead -> increased GC work)
- transitioned between C++ and JS on every iteration step (expensive)

### What Changed?

- improved optimization of calls to `iterator.next()`
- avoid allocation of `iterResult` via _store-load propagation_, _escape analysis_ and _scalar
  replacement of aggregates_
- avoid alloation of the _iterator_
- fully implemented in JavaScript via [CodeStubAssembler](https://github.com/v8/v8/wiki/CodeStubAssembler-Builtins)
- only calls to C++ during GC

### Facit

- use `for of` wherever needed without having to worry about performance cost

### Resources

- [Faster Collection Iterators](http://benediktmeurer.de/2017/07/14/faster-collection-iterators/)

## Iterating Maps and Sets via Callbacks

- both `Map`s and `Set`s provide a `forEach` method which allows iterating over it's items by
  providing a callback

### Why was it Slow?

- were mainly implemented in C++
- thus needed to transition to C++ first and to handle the callback needed to transition back
  to JavaScript (expensive)

### What Changed?

- `forEach` builtins were ported to the
  [CodeStubAssembler](https://github.com/v8/v8/wiki/CodeStubAssembler-Builtins) which lead to
  a significant performance improvement
- since now no C++ is in play these function can further be optimized and inlined by TurboFan
  (TODO: has this happened yet?)

### Facit

- performance cost of using builtin `forEach` on `Map`s and `Set`s has been reduced drastically
- however an additional closure is created which causes memory overhead
- the callback function is created new each time `forEach` is called (not for each item but
  each time we run that line of code) which could lead to it running in unoptimized mode
- therefore when possible prefer `for of` construct as that doesn't need a callback function

### Resources

- [Faster Collection Iterators - Callback Based Iteration](http://benediktmeurer.de/2017/07/14/faster-collection-iterators/#callback-based-iteration)
