## Short Lived Classes

- when class or prototype definition is collected it's hidden class (associated maps) are
  collected as well
- need to re-learn hidden classes for short living objects including metadata and all feedback
  collected by inline caches
- references to maps and JS objects from optimized code are considered weak to avoid memory
  leaks
- deoptimizations can also occur as a result
- therefore it is recommended to define classes outside functions at module scope so a
  reference to them is maintained 

### Resources

- [The case of temporary objects in Chrome](http://benediktmeurer.de/2016/10/11/the-case-of-temporary-objects-in-chrome/)
