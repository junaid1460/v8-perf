# Ignition and TurboFan Compiler Pipeline

## Simplified Pipeline

- once crankshaft was taken out of the mix the below pipeline was possible

![new v8 pipeline](http://benediktmeurer.de/images/2016/v8-new-pipeline-20161125.png)

- enabled lots more optimizations
- reduces memory and startup overhead significantly
- AST no longer source of truth that compilers need to agree on (TODO: which is, IR generated
  by Ignition)?
- AST much simpler and smaller in size

## Goals

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
