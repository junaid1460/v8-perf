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
