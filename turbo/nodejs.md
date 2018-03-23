## Node.js Perf And Tooling

[watch](https://youtu.be/EdFDJANJJLs?t=23m51s)

- specific language features like `instanceof`, `buffer.length` were improved with Node.js in mind
- ensured that `streams` in Node.js are fast
- measured via [AcmeAir benchmark](https://github.com/nodejs/benchmarking/tree/master/experimental/benchmarks/acmeair) (10% improvement)
- Node.js is _inspectable_ via the `--inspect` and [similar flags](https://nodejs.org/en/docs/inspector/#command-line-options)
- multiple tools, like DevTools (chrome://inspect) and VS Code integrate with it to allow debugging and profiling Node.js applications
	- DevTools includes dedicated Node.js window that auto connects to any Node.js process that is launched with the debugger enabled
	- _in line_ breakpoints allow breaking on specific statement on a line with multiple statements
	- async code flow debugging is supported (async stack traces)
