# Ignition and TurboFan Compiler Pipeline

Fully activated with v8 version 5.9. Earliest LTS Node.js release with a TurboFan activated
pipleline is Node.js v8.

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

## Simplified Pipeline

- once crankshaft was taken out of the mix the below pipeline was possible

<img alt="simplified pipeline" src="http://benediktmeurer.de/images/2016/v8-new-pipeline-20161125.png" width="70%">

### Basic Steps

1. Parse JavaScript into an [AST (abstract syntax tree)](https://en.wikipedia.org/wiki/Abstract_syntax_tree)
2. Generate bytecode from that AST
3. Turn bytecode into sequence of bytecodes by the BytecodeGenerator, which is part of the [Ignition Interpreter](https://v8project.blogspot.com/2016/08/firing-up-ignition-interpreter.html)
  - sequences are divided on a per function basis
4. Execute bytecode sequences via Ignition and collect feedback via inline chaches
  - feedback used by Ignition itself to speed up subsequent interpretation of the bytecode
  - feedback used for speculative optimization by TurboFan when code is optimized
5. _Speculatively_ optimize and compile bytecode using collected feedback to generate optimized machine code
  for the current architecture

### Pipeline as Part of New V8 Architecture

<img alt="new v8 pipeline detailed" src="http://benediktmeurer.de/images/2017/architecture-20170301.png" width="70%">

### Advantages Over Old Pipeline

- enabled lots more optimizations
- reduces memory and startup overhead significantly
- AST no longer source of truth that compilers need to agree on (TODO: which is, IR generated
  by Ignition)?
- AST much simpler and smaller in size

## Ignition Interpreter

- Ignition Interpreter uses a [register machine](https://en.wikipedia.org/wiki/Register_machine)
  instead of stack machine used previously
- holds its local state in _interpreter registers_
  - some map to _real_ CPU registers
  - others map to specific slots in native machine _stack memory_
- last computed value of each bytecode is kept in special _accumulator_ register
- current stack frame is identified by stack pointer
- program counter points to currently executed instruction in the bytecode

## Collecting Feedback via ICs

- main concept is the same as with old compiler chain, therefore [find more information here](https://github.com/thlorenz/v8-perf/blob/master/compiler.md#inline-caches)
- changes to how/what data is collected
  - data-driven approach
  - uses _FeedbackVector_ attached to every function, responsible to record and manage all
    execution feedback to later speed up its execution
  - _FeedbackVector_ linked from function closure and contains slots to store different kinds
    of feedback
- we can inspect what's inside the _FeedbackVector_ of a function in a debug build of d8 by
  passing the `--allow-natives-syntax` flag and calling `%DebugPrint(fn)`

### Feedback Lattice

- the feedback [lattice](https://en.wikipedia.org/wiki/Lattice#Science,_technology,_and_mathematics)
  describes the possible states of feedback that can be collected about the type of a function
  argument
- all states but _Any_ are considered _monomorphic_ and _Any_ is considered _polymorphic_
- states can only change in one direction, thus going back from _Number_ to _SignedSmall_ is
  not possible for instance

<img alt="feedback lattice" src="http://benediktmeurer.de/images/2017/lattice-20171213.png" width="60%">

### Information Stored in Function Closures

```
+-------------+
|   Closure   |-------+-------------------+--------------------+
+-------------+       |                   |                    |
                      ↓                   ↓                    ↓
               +-------------+  +--------------------+  +-----------------+
               |   Context   |  | SharedFunctionInfo |  | Feedback Vector |
               +-------------+  +--------------------+  +-----------------+
                                          |             | Invocation Count|
                                          |             +-----------------+
                                          |             | Optimized Code  |
                                          |             +-----------------+
                                          |             |    Binary Op    |
                                          |             +-----------------+
                                          |
                                          |             +-----------------+
                                          +-----------> |    Byte Code    |
                                                        +-----------------+
```

- function _Closure_ links to _Context_, _SharedFunctionInfo_ and _FeedbackVector_
- Context: contains values for the _free variables_  of the function
  and provides access to global object
  - [free variables](https://en.wikipedia.org/wiki/Free_variables_and_bound_variables)
    are variables that are neither local nor paramaters to the function, i.e. they are in scope
    of the function but declared outside of it
- SharedFunctionInfo: general info about the function like source position and bytecode
- FeedbackVector: collects feedback via ICs as explained above

## Speculative Optimization

- main concept is the same as with old compiler chain, therefore [read this for more info](https://github.com/thlorenz/v8-perf/blob/master/compiler.md#optimizing-compiler-1)
- compiler speculates that kinds of values seen in the past will be see in the future as well
- generates optimized code just for those cases which is not only smaller but also executes at
  peak speed

### Detailed Phases of Frontend, Optimization and Backend Stages

<img alt="phases" src="http://benediktmeurer.de/images/2017/turbofan-20171213.png" width="70%">

### Advantages Over Old Pipeline

- more code can be optimized and inlined as crossing from JS to C++ land has been minimized
  using techniques like CodeStubAssembler
- for the same reason (and due to other improvements) TurboFan inlines code more aggressively,
  leading to even more performance improvements
- new language features are easier optimized which makes them useable after much shorter time
  after they are introduced to v8 (previously performance issues for new features prevented
  their use in code that needed to run fast)

### Add Example of Ignition and Feedback Vector

```
   Bytecode                Interpreter State             Machine Stack

+--------------+          +-------------------+         +--------------+
| StackCheck   | <----+   |   stack pointer   |---+     |   receiver   |
+--------------+      |   +-------------------+   |     +--------------+
|   Ldar a1    |      +-- | program counter   |   |     |      a0      |
+--------------+          +-------------------+   |     +--------------+
| Add a0, [0]  |          |   accumulator     |   |     |      a1      |
+--------------+          +-------------------+   |     +--------------+
|   Return     |                                  |     | return addr. |
+--------------+                                  |     +--------------+
                                                  |     |   context    |
                                                  |     +--------------+
                                                  |     |   closure    |
                                                  |     +--------------+
                                                  +---> | frame pointer|
                                                        +--------------+
                                                        |      ...     |
                                                        +--------------+
```

#### Bytecode annotated

```asm
StackCheck    ; check for stack overflow
Ldar a1       ; load a1 into accumulator register
Add a0, [0]   ; load value from a0 register and add it to value in accumulator register
Return        ; end execution, return value in accum. reg. and tranfer control to caller
```

#### Feedback Used To Optimize Code

- the `[0]` of `Add a0, [0]` refers to _feedback vector slot_ where Ignition stores profiling
  info which later is used by TurboFan to optimize the function
- `+` operator needs to perform a huge amount of checks to cover all cases, but if we assume
  that we always add numbers we don't have to handle those other cases
- additionally numbers don't call side effects and thus the compiler knows that it can
  eliminate the expression as part of the optimization

## Deoptimization

### Bailout

- when assumptions made by optimizing compiler don't hold it bails out to deoptimized code
- _code objects_ are verified via a `test` in the _prologue_ of the generated machine code for a
  particular function
- argument types are verified before entering the function body
- on bail out the code object is _thrown_ away as it doesn't handle the current case

```asm
; x64 machine code generated by TurboFan for the Add Example above
; expecting that both parameters and the result are Smis

leaq rcx, [rip+0x0]             ; load memory address of instruction pointer into rcx
movq rcx, [rcx-0x37]            ; copy code object stored right in front into rcx
testb [rcx+0xf], 0x1            ; check if code object is valid
jnz CompileLazyDeoptimizedCode  ; if not bail out via a jump

[ .. ]                          ; push registers onto stack

cmpq rsp, [r13+0xdb0]           ; enough space on stack to execute code?
jna StackCheck                  ; if not we're sad and raise stack overflow

movq rax, [rbp+0x18]            ; load x into rax
test al, 0x1                    ; check tag bit to ensure x is small integer
jnz Deoptimize                  ; if not bail

movq rbx, [rbp+0x10]            ; load y into rbx
testb rbx, 0x1                  ; check tag bit to ensure y is small integer
jnz Deoptimize                  ; if not bail

[ .. ]                          ; do some nifty conversions via shifts
                                ; and store results in rdx and rcx

addl rdx, rcx                   ; perform add and including overflow check
jo Deoptimize                   ; if overflowed bail

[ .. ]                          ; cleanup and return to caller
```

### Deoptimization Reasons

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

## CodeStubAssembler

- provides C++ based DSL to generate highly portable machne code
- can generate highly efficient code for parts of slow-paths in JS without crossing to C++
  runtime

### Improvements via CodeStubAssembler

- `Object.create` has predictable performance by using CodeStubAssembler
- `Function.prototype.bind` archieved final boost when ported to CodeStubAssembler for a total
  60,000% improvement
- `Promise`s where ported to CodeStubAssembler which resulted in 500% speedup for `async/await`

## Facit

- performance of your code is improved
- write clean code with using idiomatic JavaScript and sensible data structures and it will run
  fast
- no need to work around architectural limitations of v8
- instead focus on your application design

## Resources

- [V8: Behind the Scenes (November Edition)](http://benediktmeurer.de/2016/11/25/v8-behind-the-scenes-november-edition/)
- [V8: Behind the Scenes (February Edition)](http://benediktmeurer.de/2017/03/01/v8-behind-the-scenes-february-edition/)
- [An Introduction to Speculative Optimization in V8](http://benediktmeurer.de/2017/12/13/an-introduction-to-speculative-optimization-in-v8/)
