## Code Caching

NOTE: most likely superceded by [custom startup snapshots](#custom-startup-snapshots)

- lessens overhead of parsing + compiling script
- uses cached data to recreate previous compilation result
- exposed via v8's API to embedders
  - pass `v8::ScriptCompiler::kProduceCodeCache` as an option when compiling script
  - cached data is attached to source object to be retrieved via
    `v8::ScriptCompiler::Source::GetCachedData`
  - can be persisted for later
  - later cache data can be attached to the source object and passed
    `v8::ScriptCompiler::kConsumeCodeCache` as an option to cause v8 to bypass compileing the
    code and deserialize the provided cache data instead

### Resources

- [Code caching](https://v8project.blogspot.com/2015/07/code-caching.html)

## Startup Snapshots

### Custom Startup Snapshots

- v8 uses snapshots and lazy deserialization to _retrieve_ previously optimized code for builtin
  functions
- powerful snapshot API exposed to embedders via `v8::SnapshotCreator` 
- among other things this API allows embedders to provide an additional script to customize a
  start-up snapshot
- new contexts created from the snapshot are initialized in a state obtained _after_ the script
  executed
- native C++ functions are recognized and encoded by the serializer as long as they have been
  registered with v8
- serializer cannot _directly_ capture state outside of v8, thus outside state needs to be
  attached to a JavaScript object via _embedder fields_

TODO: notes from more up to date post

### Resources

- [custom startup snapshots](https://v8project.blogspot.com/2015/09/custom-startup-snapshots.html)
  slightly out of date as embedder API changed and lazy deserialization was introduced
- [Energizing Atom with V8's custom start-up snapshot](https://v8project.blogspot.com/2017/05/energizing-atom-with-v8s-custom-start.html)
- [lazy deserialization](https://v8project.blogspot.com/2018/02/lazy-deserialization.html)

