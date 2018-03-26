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
- [Optimizing bound functions further](http://benediktmeurer.de/2016/01/14/optimizing-bound-functions-further/)
- [bound function exotic objects](https://tc39.github.io/ecma262/#sec-bound-function-exotic-objects)

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

## Reflection API

- `Reflect.apply` and `Reflect.construct` received 17x performance boost in v8 v6.1 and
  therefore should be considered performant at this point

### Resources

- [V8 Release 6.1](https://v8project.blogspot.com/2017/08/v8-release-61.html)

## Array Builtins

- `Array` builtins like `map` and `forEach` can be inlined into TurboFan optimized code which
  results in considerable performance improvement
- optimizations are applied to all _major non-holey_ elements kinds

- [V8: Behind the Scenes (February Edition)](http://benediktmeurer.de/2017/03/01/v8-behind-the-scenes-february-edition/)
- [V8 Release 6.1](https://v8project.blogspot.com/2017/08/v8-release-61.html)

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
- additionally each `for of` is implicitly wrapped in a `try/catch` block as per the language
  specification, which prevented its optimization due to crankshaft not ever optimizing
  functions which contained a `try/catch` statement

### What Changed?

- improved optimization of calls to `iterator.next()`
- avoid allocation of `iterResult` via _store-load propagation_, _escape analysis_ and _scalar
  replacement of aggregates_
- avoid alloation of the _iterator_
- fully implemented in JavaScript via [CodeStubAssembler](https://github.com/v8/v8/wiki/CodeStubAssembler-Builtins)
- only calls to C++ during GC
- full optimization now possible due to TurboFan's ability to optimize functions that include a
  `try/catch` statement

### Facit

- use `for of` wherever needed without having to worry about performance cost

### Resources

- [Faster Collection Iterators](http://benediktmeurer.de/2017/07/14/faster-collection-iterators/)
- [V8 Release 6.1](https://v8project.blogspot.com/2017/08/v8-release-61.html)

## Iterating Maps and Sets via `forEach` and Callbacks

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
- [V8 Release 6.1](https://v8project.blogspot.com/2017/08/v8-release-61.html)

## Iterating Object properties via for in

### Incorrect Use of For In To Iterate Object Properties

```js
var ownProps = 0
for (const prop in obj) {
  if (obj.hasOwnProperty(prop)) ownProps++
}
```

- problematic due to `obj.hasOwnProperty` call
  - may raise an error if `obj` was created via `Object.create(null)`
  - `obj.hasOwnProperty` becomes megamorphic if `obj`s with different shapes are passed
- better to replace that call with `Object.prototype.hasOwnProperty.call(obj, prop)` as it is
  safer and avoids potential performance hit

### Correct Use of For In To Iterate Object Properties

```js
var ownProps = 0
for (const prop in obj) {
  if (Object.prototype.hasOwnProperty.call(obj, prop)) ownProps++
}
```

### Why was it Fast?

- crankshaft applied two optimizations for cases were only enumerable fast properties on
  receiver were considered and prototype chain didn't contain enumerable properties or other
  special cases like proxies
- _constant-folded_ `Object.hasOwnProperty` calls inside `for in` to `true` whenever
  possible, the below three conditions need to be met
  - object passed to call is identical to object we are enumerating
  - object shape didn't change during loop iteration
  - the passed key is the current enumerated property name
- enum cache indices were used to speed up property access

### What Changed?

- _enum cache_ needed to be adapted so TurboFan knew when it could safely use _enum cache
  indices_ in order to avoid deoptimization loop (that also affected crankshaft)
- _constant folding_ was ported to TurboFan
- separate _KeyAccumulator_ was introduced to deal with complexities of collecting keys for
  `for-in`
- _KeyAccumulator_ consists of fast part which support limited set of `for-in` actions and slow part which
  supports all complex cases like ES6 Proxies
- coupled with other TurboFan+Ignition advantages this led to ~60% speedup of the above case

### Facit

- `for in` coupled with the correct use of `Object.prototype.hasOwnProperty.call(obj, prop)` is
  a very fast way to iterate over the properties of an object and thus should be used for these
  cases

### Resources

- [Restoring for..in peak performance](http://benediktmeurer.de/2017/09/07/restoring-for-in-peak-performance/)
- [Require Guarding for-in](https://eslint.org/docs/rules/guard-for-in)
- [Fast For-In in V8](https://v8project.blogspot.com/2017/03/fast-for-in-in-v8.html)

## Object Constructor Subclassing and Class Factories

- pure object subclassing `class A extends Object {}` by itself is not useful as `class B
  {}` will yield the same result even though [`class A`'s constructor will have different
  prototype chain than `class B`'s](https://github.com/thlorenz/d8box/blob/8ec3c71cb6bdd7fe8e32b82c5f19d5ff24c65776/examples/object-subclassing.js#L22-L23)
- however subclassing to `Object` is heavily used when implementing mixins via class factories
- in the case that no base class is desired we pass `Object` as in the example below

```js
function createClassBasedOn(BaseClass) {
  return class Foo extends BaseClass { }
}
class Bar {}

const JustFoo = createClassBasedOn(Object)
const FooBar = createClassBasedOn(Bar)
```

- TurboFan detects the cases for which the `Object` constructor is used as the base class and
  fully inlines object instantiation

### Facit

- class factories won't incur any extra overhead if no specific base class needs to be _mixed
  in_ and `Object` is passed to be extended from
- therefore use freely wherever if mixins make sense

### Resources

- [Optimize Object constructor subclassing](http://benediktmeurer.de/2017/10/05/connecting-the-dots/#optimize-object-constructor-subclassing)

## Tagged Templates

- [tagged templates](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Template_literals#Tagged_templates)
  are optimized by TurboFan and can be used where they apply

### Resources

- [optimize tagged templates](http://benediktmeurer.de/2017/10/05/connecting-the-dots/#optimize-tagged-templates)

## Typed Arrays and ArrayBuffer

- typed arrays are highly optimized by TurboFan
- calls to [`Function.prototype.apply` with TypedArrays as a parameter](http://benediktmeurer.de/2017/10/05/connecting-the-dots/#fast-path-for-typedarrays-in-functionprototypeapply)
  were sped up which positively affected calls to `String.fromCharCode`
- [`ArrayBuffer` view checks](http://benediktmeurer.de/2017/10/05/connecting-the-dots/#optimize-arraybuffer-view-checks)
  were improved by optimizing `ArrayBuffer.isView` and `TypedArray.prototype[@@toStringTag]
- storing booleans inside TypedArrays was improved to where it now is identical to storing
  integers

### Facit

- TypedArrays should be used wherever possible as it allows v8 to apply optimizations faster
  and more aggressively than for instance with plain Arrays
- any remaining bottlenecks will be fixed ASAP as TypedArrays being fast is a prerequisite of
  Webgl performing smoothly

### Resources

- [Connecting the dots](http://benediktmeurer.de/2017/10/05/connecting-the-dots)

## Object.is

- one usecase of `Object.is` is to check if a value is `-0` via `Object.is(v, -0)`
- previously implemented as C++ and thus couldn't be optimized
- now implemented via fast CodeStubAssembler which improved performance by ~14x

### Resources

- [Improve performance of Object.is](http://benediktmeurer.de/2017/10/05/connecting-the-dots/#improve-performance-of-objectis)

## Regular Expressions

- migrated away from JavaScript to minimize overhead that hurt performance in previous
  implementation
- new design based on [CodeStubAssembler](compiler.md#codestubassembler)
- entry-point stub into RegExp engine can easily be called from CodeStubAssembler
- make sure to neither modify the `RegExp` instance or its prototype as that will interfere
  with optimizations applied to regex operations

### Resources

- [Speeding up V8 Regular Expressions](https://v8project.blogspot.com/2017/01/speeding-up-v8-regular-expressions.html)

## Destructuring

- _array destructuring_ performance on par with _naive_ ES5 equivalent

### Facit

- employ destructuring syntax freely in your applications

### Resources

- [High-performance ES2015 and beyond](https://v8project.blogspot.com/2017/02/high-performance-es2015-and-beyond.html)

## Promises Async/Await

- native Promises in v8 have seen huge performance improvements as well as their use via
  `async/await`
- v8 exposes C++ API allowing to trace through Promise lifecycle which is used by Node.js API
  to provide insight into Promise execution
- DevTools async stacktraces make Promise debugging a lot easier
- DevTools _pause on exception_ breaks immediately when a Promise `reject` is invoked

### Resources

- [V8 Release 5.7](https://v8project.blogspot.com/2017/02/v8-release-57.html)k

## Generators

- weren't optimizable in the past due to control flow limitations in Crankshaft
- new compiler chain generates bytecodes which de-sugar complex generator control flow into
  simpler local-control flow bytecodes
- these resulting bytecodes are easily optimized by TurboFan without knowing anything specific
  about generator control flow

### Resources

- [High-performance ES2015 and beyond](https://v8project.blogspot.com/2017/02/high-performance-es2015-and-beyond.html)
