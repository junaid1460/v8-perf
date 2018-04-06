## HashTables and Hash Codes

- data structures such as Map, Set, WeakSet and WeakMap use hash tables under the hood
- a _hash function_ returns a _hash code_ for given keys which is used to map them to a
  location in the hash table 
- hash code is a random number (independent of object value) and thus needs to be stored
- storing the hash code as private symbol on the object, like was done previously, resulted in
  a variety of performance problems
  - led to slow megamorphic IC lookups of the hash code and
  - triggered hidden class transition in the key on storing the hash code
- performance issues were fixed (~500% improvement for Maps and Sets) by _hiding_ the hashcode
  and storing it in unused memory space that is _connected_ to the JSObject
  - if properties backing store is empty: directly stored in the offset of JSObject
  - if properties backing store is array: stored in extranous 21 bits of 31 bits to store array length
  - if properties backing store is dictionary: increase dictionary size by 1 word to store hashcode
    in dedicated slot at the beginning of the dictionary

### Resources

- [Optimizing hash tables: hiding the hash code - 2018](https://v8project.blogspot.com/2018/01/hash-code.html)


TODO: merge the below with crankshaft/data-types.md and update statements to current what holds
for TurboFan

## Arrays

[watch](http://youtu.be/UJPdhx5zTaw?t=17m25s) | [slide](http://v8-io12.appspot.com/index.html#38)

v8 has two methods for storing arrays.

### Fast Elements

- compact keysets
- linear storage buffer

#### Characteristics

- contiguous (non-sparse) 
- `0` based
- smaller than 64K

### Dictionary Elements

- hash table storage
- slow access

#### Characteristics

- sparse
- large

### Double Array Unboxing

[watch](http://youtu.be/UJPdhx5zTaw?t=20m20s) | [slide](http://v8-io12.appspot.com/index.html#45)

- Array's hidden class tracks element types
- if all doubles, array is unboxed
  - wrapped objects layed out in linear buffer of doubles
  - each element slot is 64-bit to hold a double
  - SMIs that are currently in Array are converted to doubles
  - very efficient access
  - storing requires no allocation as is the case for boxed doubles
  - causes hidden class change
- careless array manipulation may cause overhead due to boxing/unboxing [watch](http://youtu.be/UJPdhx5zTaw?t=21m50s) |
  [slide](http://v8-io12.appspot.com/index.html#47)

### Typed Arrays

[blog](http://mrale.ph/blog/2011/05/12/dangers-of-cross-language-benchmark-games.htm) |
[spec](https://www.khronos.org/registry/typedarray/specs/latest/)

- difference is in semantics of indexed properties
- v8 uses unboxed backing stores for such typed arrays

#### Float64Array

- gets 64-bit allocated for each element

### Considerations

- don't pre-allocate large arrays (`>64K`), instead grow as needed, to avoid them being considered sparse
- do pre-allocate small arrays to correct size to avoid allocations due to resizing
- don't delete elements
- don't load uninitialized or deleted elements [watch](http://youtu.be/UJPdhx5zTaw?t=19m30s) |
  [slide](http://v8-io12.appspot.com/index.html#43)
- use literal initializer for Arrays with mixed values
- don't store non-numeric valuse in numeric arrays
  - causes boxing and efficient code that was generated for manipulating values can no longer be used
- use typed arrays whenever possible

