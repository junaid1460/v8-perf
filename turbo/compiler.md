# Ignition and TurboFan Compiler Pipeline

Fully activated with v8 version 5.9. Earliest LTS Node.js release with a TurboFan activated
pipleline is Node.js v8.

## Goals

[watch](https://youtu.be/HDuSEbLWyOY?t=7m22s)

> Speed up real world performance for modern JavaScript, and enable developers to build a
> faster future web.

- fast startup vs. peak performance
- low memory vs. max optimization
- Ignition Interpreter allows to run code with some amount of optimization very quickly and has
  very low memory footprint
- TurboFan makes functions that run a lot fast, sacrificing some memory in the process
- designed to support entire JavaScript language and make it possible to quickly add new
  features and to optimize them fast and incrementally

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

### Detailed Phases of Frontend, Optimization and Backend Stages

<img alt="phases" src="http://benediktmeurer.de/images/2017/turbofan-20171213.png" width="70%">

## Advantages Over Old Pipeline

[watch old architecture](https://youtu.be/HDuSEbLWyOY?t=8m51s) | [watch new architecture](https://youtu.be/HDuSEbLWyOY?t=9m21s)

- reduces memory and startup overhead significantly
- AST no longer source of truth that compilers need to agree on
- AST much simpler and smaller in size
- TurboFan uses Ignition bytecode directly to optimize (no re-parse needed)
- bytecode is 25-50% the size of equivalent baseline machine code
- combines cutting-edge IR (intermediate representation) with multi-layered translation +
  optimization pipeline
- generates better quality machine code than Crankshaft JIT
- to achieve that fluid code motion, control flow optimizations and precise numerical range
  analysis are used
- clearer separation between JavaScript, v8 and the target architectures allows cleaner, more
  robust generated code and adds flexibility
- relaxed [sea of nodes]() (IR) allows more effective reordering and optimization
- crossing from JS to C++ land has been minimized using techniques like CodeStubAssembler
- as a result optimizations can be applied in more cases and are attempted more aggressively
- for the same reason (and due to other improvements) TurboFan inlines code more aggressively,
  leading to even more performance improvements

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

### Startup Time Improved

[watch](https://youtu.be/M1FBosB5tjM?t=43m25s)

- bytecode smaller and faster to generate than machine code (crankshaft)
- bytecode better suited for smaller icache (low end mobile)
- code parsed + AST converted to bytecode only once and optimized from bytecode
- data driven ICs reduced slow path cost (collected in feedback form, previously collected in code form)

### Memory Usage Reduced

[watch](https://youtu.be/M1FBosB5tjM?t=47m20s)

- most important on mobile
- Ignition code up to 8x smaller than Full-Codegen code (crankshaft)

### Baseline Performance Improved

[watch](https://youtu.be/M1FBosB5tjM?t=37m)

- no longer relying on optimizing compiler for _sufficiently_ fast code
- thus improved baseline performance allows delaying optimization until more feedback is collected
- leads to less time and resources spent optimizing

### New Language Features

[watch](https://youtu.be/M1FBosB5tjM?t=29m3s) | [watch](https://youtu.be/EdFDJANJJLs?t=20m) | [watch](https://youtu.be/HDuSEbLWyOY?t=11m22s)

- can address optimization killers that Crankshaft couldn't b/c it never supported fundamental techniques needed to do so
- as a result no specific syntax (like `try/catch`) inside a function will cause it not being optimized
- other subtle optimization killers that made performance unpredictable are no longer an issue and if they are they can be easily fixed in TF
	- passing `undefined` as first parameter to `Math.max.apply`
	- mixing strict and sloppy modes
- easier to support future JavaScript features as the JavaScript frontend is clearly separated
  from the architecture dependent backends
- new language features are not useful by just being implemented
- need to be fast (at least matching transpiled code), related optimizations are easier with
  new pipeline
- need to support debugging and be inspectable, this is archieved via better integration with
  Chrome DevTools
- new language features are easier optimized which makes them useable after much shorter time
  after they are introduced to v8 (previously performance issues for new features prevented
  their use in code that needed to run fast)
- performance of ES6 features relative to the ES5 baseline operations per second tracked at [sixspeed](http://incaseofstairs.com/six-speed/)
- at this point ES6 features are almost on par with ES5 versions of same code for most cases

#### New Language Features Support And Transpilers

[watch how to leverage babel optimally](https://youtu.be/HDuSEbLWyOY?t=15m5s)| [read deploying es2015 code](https://philipwalton.com/articles/deploying-es2015-code-in-production-today/)

- using features directly, instead of transpiling, results in smaller code size [watch](https://youtu.be/HDuSEbLWyOY?t=13m)
- additionally less parse time for untranspiled code and easier optimized
- use [babel-preset-env](https://github.com/babel/babel/tree/master/packages/babel-preset-env) to specify browsers to target
- therefore transpile es2015+ selectively

### Resources

- [Digging into the TurboFan JIT](https://v8project.blogspot.com/2015/07/digging-into-turbofan-jit.html)

## Ignition Interpreter

[watch](https://youtu.be/EdFDJANJJLs?t=13m16s) | [read](https://v8project.blogspot.com/2016/08/firing-up-ignition-interpreter.html)

- uses TurboFan's low-level architecture-independent macro-assembly instructions to generate
  bytecode handlers for each _opcode_
- TurboFan compiles these instructions to target architecture including low-level instruction
  selection and machine register allocation
- bytecode passes through inline-optimization stages as it is generated
  - common patterns replaced with faster sequences
  - redundant operations removed
  - minimize number of register transfers
- this results in highly optimized and small interpreter code which can execute the bytecode instructions
  and interact with rest of v8 VM in low overhead manner
- Ignition Interpreter uses a [register machine](https://en.wikipedia.org/wiki/Register_machine)
  with each bytecode specifying inputs and outputs as explicit register operands
- holds its local state in _interpreter registers_
  - some map to _real_ CPU registers
  - others map to specific slots in native machine _stack memory_
- last computed value of each bytecode is kept in special _accumulator_ register minimizing
  load/store operations (from/to explicit registers)
- current stack frame is identified by stack pointer
- program counter points to currently executed instruction in the bytecode

## Collecting Feedback via ICs

[watch hidden classes/maps](https://youtu.be/u7zRSm8jzvA?t=6m12s) | [watch](https://youtu.be/u7zRSm8jzvA?t=8m20s) | [watch feedback workflow]((https://youtu.be/u7zRSm8jzvA?t=14m58s))

- main concept is the same as with old compiler chain, therefore [find more information here](https://github.com/thlorenz/v8-perf/blob/master/compiler.md#inline-caches)
- changes to how/what data is collected
  - data-driven approach
  - uses _FeedbackVector_ attached to every function, responsible to record and manage all
    execution feedback to later speed up its execution
  - _FeedbackVector_ linked from function closure and contains slots to store different kinds
    of feedback
- we can inspect what's inside the _FeedbackVector_ of a function in a debug build of d8 by
  passing the `--allow-natives-syntax` flag and calling `%DebugPrint(fn)`
- polymorphic: 2-4 different types seen
- if monomorphic compare maps and if they match just load prop at offset in memory, i.e. `mov eax, [eax+0xb]`
- IC feedback slots reserved when AST is created, see them via `--print-ast`, i.e. `Slot(0) at 29`
- collect typeinfo for ~24% of the ICs before attempting optimization
- see optimization + IC info via `--trace-opt`
	-  _generic ICs_ are _bad_ as if lots of them are present, code will not be optimized
	- _ICs with typeinfo_ are _good_
- feedback vectors aren't embedded in optimized code but map ids or specific type checks, like for SMIs
- evaluate ICs via  `--trace-ic test.js && ./v8/tools/ic-processor`

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

## TurboFan

[watch TurboFan history](https://youtu.be/EdFDJANJJLs?t=10m22s) | [watch TurboFan goals](https://youtu.be/EdFDJANJJLs?t=11m44s)

TurboFan is a simple compiler + backend responsible for the following:

- instruction selection + scheduling
  - innovative scheduling algorithm makes use of reordering freedom ([sea of nodes]()) to move
    code out of loops into less frequently executed paths
- register allocation
- code generation
- generates fast code via _speculative optimization_ from the feedback collected while running
  unoptimized bytecode
- architecture specific optimizatinos exploit features of each target platform for best quality
  code

TurboFan is not just an optimizing compiler:

- interpreter bytecode handlers run on top of TurboFan
- builtins benefit from TurboFan
- code stubs / IC subsystem runs on top of TurboFan
- web assembly code generation (also runs on top of TurboFan)



## Speculative Optimization

- main concept is the same as with old compiler chain, therefore [read this for more info](https://github.com/thlorenz/v8-perf/blob/master/compiler.md#optimizing-compiler-1)
- compiler speculates that kinds of values seen in the past will be see in the future as well
- generates optimized code just for those cases which is not only smaller but also executes at
  peak speed

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

[watch bailout example](https://youtu.be/u7zRSm8jzvA?t=26m43s) | [watch walk through TurboFan optimized code with bailouts](https://youtu.be/u7zRSm8jzvA?t=19m36s)

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

### Inlining Functions

[watch](https://youtu.be/u7zRSm8jzvA?t=26m12s)

- smart heuristics, i.e. how many times was the function called so far

## Sea Of Nodes

- doesn't include total order of program, but dependencies between operations
- total ordering is built from that so machine can execute it
- entrypoints are TF optimizing compiler and WASM Compiler

## CodeStubAssembler

[watch](https://youtu.be/M1FBosB5tjM?t=23m38s)

- provides C++ based DSL to generate highly portable machine code
- can generate highly efficient code for parts of slow-paths in JS without crossing to C++
  runtime
- builtins are coded in that DSL
- all interpreter bytecodes written using TurboFan
- entry-point stubs into C++ can easily be called from CSA
- very fast property accesses
- basis for fast builtins, i.e. [faster Regular Expressions](./js-feature-improvements.md#regular-expressions)

### Improvements via CodeStubAssembler

- `Object.create` has predictable performance by using CodeStubAssembler
- `Function.prototype.bind` archieved final boost when ported to CodeStubAssembler for a total
  60,000% improvement
- `Promise`s where ported to CodeStubAssembler which resulted in 500% speedup for `async/await`

## Facit

[watch](https://youtu.be/M1FBosB5tjM?t=52m54s) | [watch](https://youtu.be/HDuSEbLWyOY?t=10m36s)

- performance of your code is improved
- less _anti patterns_ aka _you are holding it wrong_
- write idiomatic, declarative JavaScript as in _easy to read_ JavaScript with good data structures and algorithms, including all language features (even functional ones) will execute with predictable, good performance
- instead focus on your application design
- now can handle exceptions where it makes sense as `try/catch/finally` no longer ruins the performance of a function
- use appropriate collections as their performance is on par with the raw use of Objects for same task
	- Maps, Sets, WeakMaps, WeakSets used where it makes sense results in easier maintainable JavaScript as they offer specific functionality to iterate over and inspect their values
- avoid engine specific workarounds aka _CrankshaftScript_, instead file a bug report if you discover a bottleneck

## Resources

- [V8: Behind the Scenes (November Edition)](http://benediktmeurer.de/2016/11/25/v8-behind-the-scenes-november-edition/)
- [V8: Behind the Scenes (February Edition)](http://benediktmeurer.de/2017/03/01/v8-behind-the-scenes-february-edition/)
- [An Introduction to Speculative Optimization in V8](http://benediktmeurer.de/2017/12/13/an-introduction-to-speculative-optimization-in-v8/)
- [High-performance ES2015 and beyond](https://v8project.blogspot.com/2017/02/high-performance-es2015-and-beyond.html)
- [Launching Ignition and TurboFan](https://v8project.blogspot.com/2017/05/launching-ignition-and-turbofan.html)

### Videos

- [performance improvements in latest v8](https://youtu.be/HDuSEbLWyOY?t=4m58s)
- [v8 and how it listens to you - ICs and FeedbackVectors](https://www.youtube.com/watch?v=u7zRSm8jzvA)
