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

## Custom Startup Snapshots

- v8 uses snapshots and lazy deserialization to _retrieve_ previously optimized code for builtin
  functions
- snapshot API exposed to embedders 

TODO: notes from more up to date post

### Resources

- [custom startup snapshots](https://v8project.blogspot.com/2015/09/custom-startup-snapshots.html)
  slightly out of date as embedder API changed and lazy deserializatoin was introduced
- [lazy deserialization](https://v8project.blogspot.com/2018/02/lazy-deserialization.html)
