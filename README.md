Name
====

openresty/luajit2 - OpenResty's LuaJIT branch

Table of Contents
=================

* [Name](#name)
* [Description](#description)
    * [New API](#new-api)
        * [table.isempty](#tableisempty)
        * [table.isarray](#tableisarray)
        * [table.nkeys](#tablenkeys)
        * [thread.exdata](#threadexdata)
        * [table.clone](#tableclone)
        * [jit.prngstate](#jitprngstate)
    * [New JIT parameter defaults](#new-jit-parameter-defaults)
    * [String table hashing optimization](#string-table-hashing-optimization)
    * [New command-line option `-bL`](#new-command-line-option--bl)
    * [More detailed LuaJIT bytecode textual listing](#more-detailed-luajit-bytecode-textual-listing)
    * [New macros](#new-macros)
    * [Miscellaneous](#miscellaneous)
* [Copyright & License](#copyright--license)

# Description

This is the official OpenResty branch for LuaJIT. This is not really a fork
since we still synchronize any upstream changes all the time.

We introduce our own changes which will never merge or haven't yet merged into
the upstream LuaJIT (https://github.com/LuaJIT/LuaJIT), which are as follows

## New API

### table.isempty

We added the `table.isempty()` builtin Lua API.

This Lua API can be JIT compiled.

Returns true when the table contains non-nil array elements or non-nil
key-value pairs.

Usage:

```lua
local isempty = require "table.isempty"
print(isempty({}))  -- true
print(isempty({nil, dog = nil}))  -- true
print(isempty({"a", "b"}))  -- false
print(isempty({nil, 3}))  -- false
print(isempty({cat = 3}))  -- false
```

[Back to TOC](#table-of-contents)

### table.isarray

We implemented the new table.isarray() builtin function which returns true
for purely array-like Lua tables and false otherwise.

Empty Lua tables are treated as arrays.

Use it like this:

```lua
isarr = require "table.isarray"
print(isarr{"a", true, 3.14})  -- true
print(isarr{"dog" = 3})  -- false
print(isarr{})  -- true
```

[Back to TOC](#table-of-contents)

### table.nkeys

We added the new table.nkeys Lua API.

This Lua builtin returns the number of elements in a Lua table, i.e.,
the number of non-nil array elements plus the number of non-nil key-value
pairs in a single Lua table.

It can be used like this:

```lua
local nkeys = require "table.nkeys"

print(nkeys({}))  -- 0
print(nkeys({ "a", nil, "b" }))  -- 2
print(nkeys({ dog = 3, cat = 4, bird = nil }))  -- 2
print(nkeys({ "a", dog = 3, cat = 4 }))  -- 3
```

This Lua API can be JIT compiled.

[Back to TOC](#table-of-contents)

### thread.exdata

We implemented the new Lua and C API functions for thread exdata to allow users
to embed user data into a thread. This feature needs LuaJIT FFI and hence is
not available when built with `-DLUAJIT_DISABLE_FFI`.

The Lua API can be used like below:

```lua
local exdata = require "thread.exdata"
exdata(0xdeadbeefLL)  -- set the exdata of the current Lua thread
local ptr = exdata()  -- fetch the exdata of the current Lua thread
```

The exdata value on the Lua land is represented as a cdata object of the
ctype `void*`.

Right now the reading API, i.e., `exdata()` calls without any arguments,
can be JIT compiled.

Also exposed the following public C API functions for manipulating
exdata on the C land:

```C
void lua_setexdata(lua_State *L, void *exdata);
void *lua_getexdata(lua_State *L);
```

The exdata pointer is initialized to NULL when the main thread is
created. Any child Lua threads created will inherit the parent's exdata
but still have their own exdata storage. So child Lua threads can always
override the inherited parent exdata pointer values.

This API is used internally by the OpenResty core so never ever mess
with it yourself in the context of OpenResty.

[Back to TOC](#table-of-contents)

### table.clone

We implemented the table.clone() builtin Lua API.

This change only support shallow clone. e.g

```lua
local tab_clone = require "table.clone"
local x = {x=12, y={5, 6, 7}}
local y = tab_clone(x)
... use y ...
```

Deep clone will be supported by adding 'true' as the 2nd argument to table.clone

We observe 7% over-all speedup in the edgelang-fan compiler's compiling
speed whose Lua is generated by the fanlang compiler.

[Back to TOC](#table-of-contents)

### jit.prngstate

We implemented new API function jit.prngstate() for reading or setting
the current PRNG state number used in the JIT compiler.

[Back to TOC](#table-of-contents)

## New JIT parameter defaults

We use more appressive JIT compiler parameters as the default to help
large OpenResty Lua apps.

We now use the following `jit.opt` defaults:

```lua
maxtrace=8000
maxrecord=16000
minstitch=3
maxmcode=40960  -- in KB
```

[Back to TOC](#table-of-contents)

## String table hashing optimization

The `lj_str_new()` primitiven now uses randomized hash functions based on crc32 when
`-msse4.2` is specified for Intel CPUs supporting SSE 4.2 instruction set.

[Back to TOC](#table-of-contents)

## New command-line option `-bL`

We added the bytecode option `L` to display lua source line numbers.
For example, `luajit -bL -e 'print(1)'` now produces bytecode dump like
below:

```
-- BYTECODE -- "print(1)":0-1
0001     [1]    GGET     0   0      ; "print"
0002     [1]    KSHORT   1   1
0003     [1]    CALL     0   1   2
0004     [1]    RET0     0   1
```

The column `[N]` is the Lua source line number. For example, `[1]` means on
the first source line.

[Back to TOC](#table-of-contents)

## More detailed LuaJIT bytecode textual listing

The `-bl` option also prints out the constant tables of each Lua prototype.
For example,

```
-- BYTECODE -- a.lua:0-48
KGC    0    "print"
KGC    1    "hi"
KGC    2    table
KGC    3    a.lua:17
KN    1    1000000
KN    2    1.390671161567e-309
...
```

[Back to TOC](#table-of-contents)

## New macros

In the header file `luajit.h`, we defined the macro `OPENRESTY_LUAJIT` for our branch of
LuaJIT.

[Back to TOC](#table-of-contents)

## Miscellaneous

* various important fixes for bugs in the JIT compiler and the VM which have not
been merged in upstream LuaJIT.
* removed the GCC 4 requirement for x86 for older systems like
Solaris i386.
* In the `Makefile` file, make sure we always install the symlink for "luajit" even for
alpha or beta versions.
* feature: jit.dump: output Lua source location after every BC.
* feature: added internal memory-buffer-based trace entry/exit/start-recording
event logging, mainly for debugging bugs in the JIT compiler. it requires
`-DLUA_USE_TRACE_LOGS` when building LuaJIT.
* feature: save `g->jit_base` to `g->saved_jit_base` before `lj_err_throw` clears
`g->jit_base` which makes it impossible to get Lua backtrace in such states.
* applied a patch to fix DragonFlyBSD compatibility. Note: this is not an
officially supported target.
* turned off string comparison optimizations for 64-bit architectures when
the build option `LUAJIT_USE_VALGRIND` is specified. now LuaJIT is (almost)
valgrind clean.

[Back to TOC](#table-of-contents)

# Copyright & License

LuaJIT is a Just-In-Time (JIT) compiler for the Lua programming language.

Project Homepage: http://luajit.org/

LuaJIT is Copyright (C) 2005-2019 Mike Pall.

Additional patches for OpenResty are copyrighted by Yichun Zhang and OpenResty
Inc.:

Copyright (C) 2017-2019 Yichun Zhang. All rights reserved.

Copyright (C) 2017-2019 OpenResty Inc. All rights reserved.

LuaJIT is free software, released under the MIT license.
See full Copyright Notice in the COPYRIGHT file or in luajit.h.

Documentation for the official LuaJIT is available in HTML format.
Please point your favorite browser to:

doc/luajit.html

[Back to TOC](#table-of-contents)

