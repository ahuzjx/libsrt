master branch: [![Build Status (gcc/clang/tcc)](https://travis-ci.org/faragon/libsrt.svg?branch=master)](https://travis-ci.org/faragon/libsrt) [![Build Status (vs win32)](https://ci.appveyor.com/api/projects/status/bw52jnfgv7qff6x4/branch/master?svg=true)](https://ci.appveyor.com/project/faragon/libsrt)

master branch analysis: [![Build Status](https://scan.coverity.com/projects/5366/badge.svg)](https://scan.coverity.com/projects/5366)

documentation (generated by doing ./make\_test.sh 8): [HTML](https://faragon.github.io/libsrt.html)

libsrt has been included into Paul Hsieh's [String Library Comparisons](http://bstring.sourceforge.net/features.html) table. What a great honor! :-)

libsrt: Safe Real-Time library for the C programming language
===

libsrt is a C library that provides string, vector, bit set, set, and map handling. It's been designed for avoiding explicit memory management when using dynamic-size data structures, allowing safe and expressive code, while keeping high performance. It covers basic needs for writing high level applications in C without worrying about managing dynamic size data structures. It is also suitable for low level and hard real time applications, as functions are predictable in both space and time (asuming OS and underlying C library is also real-time suitable).

Key points:

* Easy: write high-level-like C code. Write code faster and safer.
* Fast: using O(n)/O(log n)/O(1) state of the art algorithms.
* Useful: strings supporting raw data handling (per-byte), vector, tree, and map structures.
* Efficient: space optimized, minimum allocation calls (heap and stack support).
* Compatible: OS-independent (e.g. built-in space-optimized UTF-8 support).
* Predictable: suitable for microcontrollers and hard real-time compliant code.
* Unicode: although string internal representation is raw data (bytes), functions for handling Unicode interpretation/generation/transformation are provided, so when Unicode-specific functions are used, the result of these functions is stored internally as UTF-8 (also, caching some operations, like Unicode string length -e.g. ss_len()/ss_size() give length in bytes, and ss_len_u() the length in Unicode characters-).

Generic advantages
===

* Ease of use
  * Use strings, vectors, maps, sets, and bit sets in a similar way to higher level languages.

* Space-optimized
  * Dynamic one-block linear addressing space.
  * Internal structures use indexes instead of pointers (i.e. similar memory usage in both 32 and 64 bit mode).
  * Details: [doc/benchmarks.md](https://github.com/faragon/libsrt/blob/master/doc/benchmarks.md)

* Time-optimized
  * Buffer direct access
  * Preallocation hints (reducing memory allocation calls)
  * Heap and stack memory allocation support
  * Details: [doc/benchmarks.md](https://github.com/faragon/libsrt/blob/master/doc/benchmarks.md)

* Predictable (suitable for hard and soft real-time)
  * Predictable execution speed: all API calls have documented time complexity. Also space complexity, when extra space involving dynamic memory is required.
  * Hard real-time: allocating maximum size (strings, vector, trees, map) will imply calling 0 or 1 times to the malloc() function (C library). Zero times if using the stack as memory or externally allocated memory pool, and one if using the heap. This is important because malloc() implementation has both memory and time overhead, as internally manages a pool of free memory (which could have O(1), O(log(n)), or even O(n), depending on the compiler provider C library implementation).
  * Soft real-time: logarithmic time memory allocation (when not doing preallocation): for 10^n new elements just log(n) calls to the malloc() function. This is not a guarantee for hard real time, but for soft real time (being careful could provide almost same results, except in the case of very poor malloc() implementation in the C library being used -not a problem with modern compilers-).

* RAM, ROM, and disk operation
  * Data structures can be stored in ROM memory.
  * Data structures are suitable for memory mapped operation, and disk store/restore. This is currently true for strings, vectors, and bitsets. For maps, only when using integer data (as currently strings inside a tree are external references -this will be fixed in the future, adding the possibility of in-map arbitrary fixed data, allowing e.g. fixes size strings inside a map-)

* Known edge case behavior
  * Allowing both "carefree code" and per-operation error check. I.e. memory errors and UTF8 format error can be checked after every operation.

* Implementation
  * Simple internal structures, with linear addressing. It allows to reduce memory allocation calls to the minimum (using the stack it is possible to avoid heap usage).
  * Implementation kept as simple as possible/known.
  * Focus in avoiding code-bloat, avoiding expensive code duplication/inlining (inlining is used when being "free" or because of big speed impact)
  * Small code, suitable for static linking. In fact, currently no dynamic linking is provided (will be added, but it is not a priority).
  * Coverage, profiling, and memory leak/overflow/uninitialized checks (Valgrind, CLang static analysis, Coverity, MS VS).

* Compatibility
  * C99 (and later) compatible (requiring alloca() function)
  * C++11 (and later) compatible (requiring alloca() function)
  * CPU-aware: little and big endian CPU support (middle/mixed/PDP endian is not supported), aligned memory accesses
  * POSIX builds (Linux, BSD's) require GNU Make (e.g. on FreeBSD use 'gmake' instead of 'make')

Generic disadvantages/limitations
===

* Double pointer usage: because of using just one allocation, wrte operations require to address a double pointer, so in the case of reallocation the source pointer could be changed.

* Concurrent read-only operations are safe, but concurrent read/write must be protected by the user (e.g. using mutexes or spinlocks). That can be seen as a disadvantage or as a "feature" (it is faster).

* It may work in C89 mode only if following language extensions are available:
  * \_\_VA\_ARGS\_\_ macro
  * alloca()
  * type of bit-field in 'struct'
  * %S printf extension (only for unit testing)
  * Format functions (\*printf) rely on system C library, so be aware if you write multi-platform software before using compiler-specific extensions or targetting different C standards).
  * Allow mixed code and variable declaration
* It may work in C++98 mode only if following language extensions are available:
  * Anonymous variadic macros
  * Long long integer constant (LL): required only for the tests, not by the library itself

String-specific advantages (ss\_t)
===

* Binary data support
  * I.e. strings can have zeros in the middle
  * Search and replace into binary data is supported
* Unicode support
  * Although strings internal storage is binary, Unicode-aware functions store data in UTF-8.
  * Search and replace into UTF-8 data is supported
  * Full and fast Unicode lowercase/uppercase support without requiring "setlocale" nor hash tables.
* Efficient raw and Unicode (UTF-8) handling. Unicode size is tracked, so resulting operations with cached Unicode size, will keep that, keeping the O(1) for getting that information afterwards.
  * Find/search: O(n), one pass.
  * Replace: O(n), one pass. Worst case overhead is limited to a realloc and a copy of the part already processed.
  * Concatenation: O(n), one pass for multiple concatenation. I.e. Optimal concatenation of multiple elements require just one allocation, which is computed before the concatination. When concatenating ss\_t strings the allocation size compute time is O(1).
  * Resize: O(n) for worst case (when requiring reallocation for extra space. O(n) for resize giving as cut indicator the number of Unicode characters. O(1) for cutting raw data (bytes).
  * Case conversion: O(n), one pass, using the same input if case conversion requires no allocation over current string capacity. If resize is required, in order to keep O(n) complexity, the string is scanned for computing required size. After that, the conversion outputs to the secondary string. Before returning, the output string replaces the input, and the input becomes freed.
  * Avoid double copies for I/O (read access, write reserve)
  * Avoid re-scan (e.g. commands with offset for random access)
  * Transformation operations are supported in all dup/cpy/cat functions, in order to both increase expressiveness and avoid unnecessary copies (e.g. tolower, erase, replace, etc.). E.g. you can both convert to lower a string in the same container, or copy/concatenate to another container.
* Space-optimized
  * Using just 4 byte overhead for strings with size <= 255 bytes
  * Using sizeof(size\_t) * 5 byte overhead for strings with size >= 256 bytes (e.g. 20 bytes for a 32-bit CPU, 40 for 64-bit)
  * Data structure has no pointers, i.e. just one allocation is required for building a string. Or zero, if using the stack.
  * No additional memory allocation for search.
  * Extra memory allocation may be required for: UTF-8 uppercase/lowercase and replace.
  * Strings can grow from 0 bytes to ((size\_t)~0 - metainfo\_size)
* String operations
  * Copy, cat, tolower/toupper, find, split, printf, cmp, base64, data compression, crc32 on buffers, etc.
  * All string operations allow C strings and raw buffers as input, without extra copies (ss\_[c]ref[a]() functions)
  * Allocation, buffer pre-reserve,
  * Raw binary content is allowed, including 0's.
  * "Wide char" and "C style" strings R/W interoperability support.
  * I/O helpers: buffer read, reserve space for async write
  * Aliasing suport, e.g. ss\_cat(&a, a) is valid
* Focus on reducing verbosity:
  * ss\_cat(&t, s1, ..., sN);
  * ss\_cat(&t, s1, s2, ss\_printf(&s3, "%i", cnt), ..., sN);
  * ss\_free(&s1, &s2, ..., &sN);
  * Expressive code without explicit memory handling
* Focus on reducing errors, e.g.
  * If a string operation fails, the string is kept in the last successful state (e.g. ss\_cat(&a, b, huge\_string, other))
  * String operations always return valid strings, e.g.
		This is OK:
			ss\_t *s = NULL;
			ss_cpy_c(&s, "a");
		Same behavior as:
			ss\_t *s = ss_dup_c("a");
  * ss\_free(&s1, ..., &sN);  (no manual set to NULL is required)

String-specific disadvantages/limitations
===

* No reference counting support. Rationale: simplicity.

Vector-specific advantages (sv\_t)
===

* Variable-length concatenation and push functions.
* Allow explicit size for allocation (8, 16, 32, 64 bits) with size-agnostic generic signed/unsigned functions (easier coding).
* Allow variable-size generic element.
* Sorting
  * O(n) for 8-bit elements (counting sort algorithm), much faster than GNU/Clang qsort() (C), and up to 5x faster than GNU/Clang std::vector sort (C++)
  * O(n log n) -pseudo O(n)- for 16/32/64-bit elements (in-place MSD binary radix sort algorithm), 2x-3x faster than GNU/Clang qsort() (C), performing similar to GNU/Clang std::vector sort (C++)
  * O(n log n) for generic elements using the C library (qsort())

Vector-specific disadvantages/limitations
===

* No insert function. Rationale: insert is slow (O(n)). Could be added, if someone asks for it.

Map-specific advantages (sm\_t)
===

* Abstraction over Red-Black tree implementation using linear memory pool with just 8 byte per node overhead, allowing up to (2^32)-1 nodes (for both 32 an 64 bit compilers). E.g. one million 32 bit key, 32 bit value map will take just 16MB of memory (16 bytes per element \-8 byte metadata, 4 + 4 byte data\-).
* Keys: integer (8, 16, 32, 64 bits) and string (ss\_t)
* Values: integer (8, 16, 32, 64 bits), string (ss\_t), and pointer
* O(1) for allocation
* O(1) for deleting maps without strings (one or zero calls to 'free' C function)
* O(n) for deleting maps with strings (n + one or zero calls to 'free' C function)
* O(n) for map copy (in case of maps without strings, would be as fast as a memcpy())
* O(log n) insert, search, delete
* O(n) sorted enumeration (amortized O(n log n))
* O(n) unsorted enumeration (faster than the sorted case)
* O(n) copy: tree structure is copied as fast as a memcpy(). For map types involving strings, additional allocation is used for duplicating strings.

Map-specific disadvantages/limitations
===

* Because of being implemented as a tree, it is slower than a hash-map, on average. However, in total execution time is not that bad, as because of allocation heuristics a lot of calls to the allocator are avoided.
* There is room for node deletion speed up (currently deletion is a bit slower than insertion, because of an additional tree search used for avoiding having memory fragmentation, as implementation guarantees linear/compacted memory usage, it could be optimized with a small LRU for cases of multiple delete/insert operation mix).

Test-covered platforms
===

| ISA | Word size | Endianess | Unaligned memory access HW support | OS | Compilers | Code analysis | Test coverage |
| --- | --------- | --------- | ---------------------------------- | --- | --------- | ------------- | ------------- |
| x86, x86-64 (Core i5) | 32, 64 | little | yes | Linux Ubuntu 12.04/14.04/17.10 | gcc, g++, tcc, clang, clang++ | Valgrind, clang, Coverity | Travis CI (automatic, every public commit) |
| x86, x86-64 (Core i5) | 32, 64 | little | yes | Windows | Visual Studio Express 2013, AppVeyor's VS | VS | AppVeyor (automatic, every public commit) |
| x86, x86-64 (Core i5) | 32, 64 | little | yes | FreeBSD 10.2 | gcc, g++, clang, clang++ | Valgrind clang | manual |
| x86, x86-64 (Core2Duo) | 32, 64 | little | yes | Darwin 11.4.2 | gcc, g++, clang, clang++ | none | manual |
| ARMv5 (ARM926EJ-S) | 32 | little | no | Arch Linux | gcc, g++, clang, clang++ | none | manual |
| ARMv5 (Feroceon) | 32 | little | no | Linux Debian 7.0 "Wheezy" | gcc, g++ | none | manual |
| ARMv6 (ARM1176JZF-S) | 32 | little | yes | Linux Raspbian | gcc, g++, clang, clang++ | Valgrind, clang | manual |
| ARMv7-A (Krait 400) | 32 | little | yes | Linux Android 5.1.1 + BusyBox | gcc, g++ | none | manual |
| ARMv8-A (Cortex A53) | 64 | little | yes | Debian 8.5 "Jessie" | gcc, g++, clang, clang++ | Valgrind, clang | manual |
| MIPS, MIPS64 (Octeon) | 32, 64 | big | yes | EdgeOS v1.6.0 (Linux Vyatta-based using Debian 7 "Wheezy" packages) | gcc, g++, clang, clang++ | Valgrind, clang | manual |
| MIPS (EE R5900) | 32 | little | no | Playstation 2 Linux (Red Hat based) | gcc, g++ 2.95.2 | none | manual |
| PowerPC (G4) | 32 | big | yes | Linux Ubuntu 12.04 | gcc, g++ | none | manual |
| PowerPC, PPC64 (Cell) | 32, 64 | big | yes | Yellow Dog Linux 6.2 (Red Hat based) | gcc, g++ 4.1.2 | none | manual |

License
===

Copyright (c) 2015-2018, F. Aragon. All rights reserved.
Released under the BSD 3-Clause License (see the LICENSE file included).

Contact
===

email: faragon.github (GMail account, add @gmail.com)

Other
===

Status
---

Beta. API still can change.

"to do" list
---

Check [doc/todo.md](https://github.com/faragon/libsrt/blob/master/doc/todo.md)


Acknowledgements and references
---

Check [doc/references.md](https://github.com/faragon/libsrt/blob/master/doc/references.md)

