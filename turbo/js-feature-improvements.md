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
