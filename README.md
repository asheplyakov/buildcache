# BuildCache [![Build Status](https://travis-ci.org/mbitsnbites/buildcache.svg?branch=master)](https://travis-ci.org/mbitsnbites/buildcache)

This is a simple compiler accelerator that caches and reuses build results to
avoid unnecessary re-compilations, and thereby speeding up the build process.

It is similar in spirit to [ccache](https://ccache.samba.org/), [sccache](https://github.com/mozilla/sccache) and [clcache](https://github.com/frerich/clcache).

## Features

* Works on different operating systems:
  * Linux
  * macOS
  * Windows
  * Though untested, it probably works on most BSD:s
* A modular compiler support system:
  * Built-in support for [popular compilers](#supported-compilers-and-languages).
  * Extensible via custom [Lua](https://www.lua.org/) scripts.
  * In addition to caching compilation results, BuildCache can be used for caching almost any reproducible program artifacts (e.g. test results, [rendered images](https://en.wikipedia.org/wiki/Rendering_(computer_graphics)), etc).
* A fast local file system cache.
* Can optionally use a remote, shared server as a second level cache.
* Optional compression with [LZ4](https://github.com/lz4/lz4) (almost negligable overhead).

## Status

**NOTE:** BuildCache is still in early development and should not be considered
ready for production projects yet!

## Building

Use [CMake](https://cmake.org/) and your favorite C++ compiler to build the
BuildCache program:

```bash
$ mkdir build
$ cd build
$ cmake -DCMAKE_BUILD_TYPE=Release ../src
$ cmake --build .
```

Note: For S3 support on non-macOS/Windows systems you need OpenSSL (e.g. install `libssl-dev` on Ubuntu before running CMake).

## Usage

To use BuildCache for your builds, simply prefix the build command with
`buildcache`. For instance:

```bash
$ buildcache g++ -c -O2 hello.cpp -o hello.o
```

A convenient solution for bigger CMake-based projects is to use the
`RULE_LAUNCH_COMPILE` property to use BuildCache for all compilation commands,
like so:

```cmake
find_program(buildcache_program buildcache)
if(buildcache_program)
  set_property(GLOBAL PROPERTY RULE_LAUNCH_COMPILE "${buildcache_program}")
endif()
```

Another alternative is to create symbolic links that redirect invokations of your favourite compiler to go via BuildCache instead. For instance, if `$HOME/bin` is early in your PATH, you can do the following:

```bash
$ ln -s /path/to/buildcache $HOME/bin/cc
$ ln -s /path/to/buildcache $HOME/bin/c++
$ ln -s /path/to/buildcache $HOME/bin/gcc
$ ln -s /path/to/buildcache $HOME/bin/g++
…
```

You can check that it works by invoking the compiler with BuildCache debugging enabled:

```bash
$ BUILDCACHE_DEBUG=1 gcc
BuildCache[52286] (DEBUG) Invoked as symlink: gcc
…
```

## Using with icecream

[icecream](https://github.com/icecc/icecream) (or ICECC) is a tool for
distributed compilation. To use icecream you can set the environment variable
`BUILDCACHE_PREFIX` to the icecc executable, e.g:

```bash
$ BUILDCACHE_PREFIX=/usr/bin/icecc buildcache g++ -c -O2 hello.cpp -o hello.o
```

## Using a shared remote cache

To improve the cache hit ratio in a cluster of machines that often perform
the same or similar build tasks, you can use a shared remote cache (in
addition to the local cache).

To do so, set `BUILDCACHE_REMOTE` to a valid remote server address (see below).

### Redis

[Redis](https://redis.io/) is a fast, in-memory data store with built in
[LRU](https://en.wikipedia.org/wiki/Cache_replacement_policies#Least_recently_used_(LRU))
eviction policies. It is suitable for build systems that produce many small
object files, such as is typical for C/C++ compilation.

Example:
```bash
$ BUILDCACHE_REMOTE=redis://my-redis-server:6379 buildcache g++ -c -O2 hello.cpp -o hello.o
```

### S3

[S3](https://en.wikipedia.org/wiki/Amazon_S3) is an open HTTP based protocol
that is often provided by [object storage](https://en.wikipedia.org/wiki/Object_storage)
solutions. [Amazon AWS](https://aws.amazon.com/) is one such service. An open
source alternative is [MinIO](https://min.io/).

Compared to a Redis cache, an S3 object store usually has a higher capacity and a slightly higher performance overhead. Thus it is better suited for larger build artifacts.

When using an S3 remote, you also need to define `BUILDCACHE_S3_ACCESS` and
`BUILDCACHE_S3_SECRET`. You will also need to create a bucket for BuildCache
in your S3 storage, and configure some retention policy (e.g. periodic LRU
eviction).

Example:
```bash
$ BUILDCACHE_REMOTE=s3://my-minio-server:9000/my-buildcache-bucket BUILDCACHE_S3_ACCESS="ABCDEFGHIJKL01234567" BUILDCACHE_S3_SECRET="sOMloNgSecretKeyThatsh0uldnotBeshownatAll" buildcache g++ -c -O2 hello.cpp -o hello.o
```

## Supported compilers and languages

Currently the following compilers and languages are supported:

| Compiler | Languages | Support |
| -------- | --------- | --------- |
| GCC      | C, C++    | Built-in  |
| Clang    | C, C++    | Built-in  |
| Microsoft Visual C++ | C, C++    | Built-in  |
| Green Hills Software    | C, C++    | Built-in  |
| Texas Instruments TMS320C6000™ | C, C++  | Lua example |

New backends are relatively easy to add, both as built-in wrappers in C++ and as Lua wrappers (see below).

## Using custom Lua plugins

It is possible to extend the capabilities of BuildCache with
[Lua](https://www.lua.org/). See [lua-examples](lua-examples/) for some examples
of Lua wrappers.

BuildCache first searches for Lua scripts in the paths given in the environment
variable `BUILDCACHE_LUA_PATH` (colon separated on POSIX systems, and semicolon
separated on Windows), and then continues searching in `$BUILDCACHE_DIR/lua`.
If no matching script file was found, BuildCache falls back to the built in
compiler wrappers (as listed above).

**Note:** To use Lua standard libraries (`coroutine`, `debug`, `io`, `math`,
`os`, `package`, `string`, `table` or `utf8`), you must first load them by
calling `require_std(name)`. For convenience it is possible to load all standard
libraries with `require_std("*")`, but beware that it is slower than to load
only the libraries that are actually used.

All program arguments are available in the global `ARGS` array (an array of
strings).

The following methods can be implemented (see
[program_wrapper.hpp](src/wrappers/program_wrapper.hpp) for a more detailed
documentation):

| Function | Returns | Default |
| --- | --- | --- |
| can_handle_command () | Can the wrapper handle this program? | true |
| resolve_args () | (nothing) | - |
| get_capabilities () | A list of supported capabilities | An empty table |
| preprocess_source () | The preprocessed source code (e.g. for C/C++) | An empty string |
| get_relevant_arguments () | Arguments that can affect the build output | All arguments |
| get_relevant_env_vars () | Environment variables that can affect the build output | An empty table |
| get_program_id () | A unique program identification | The MD4 hash of the program binary |
| get_build_files () | A table of build result files | An empty table |

## Configuration options

BuildCache can be configured via environment variables and a per-cache JSON
configuration file. The optional configuration file is located in the cache
root directory, and is called `config.json` (e.g.
`$HOME/.buildcache/config.json`).

The following options control the behavior of BuildCache:

| Env | JSON | Description | Default |
| --- | --- | --- | --- |
| `BUILDCACHE_DIR` | - | The cache root directory | `$HOME/.buildcache` |
| `BUILDCACHE_PREFIX` | `prefix` | Prefix command for cache misses | None |
| `BUILDCACHE_REMOTE` | `remote` | Address of remote cache server (`protocol://host:port/path`, where `protocol` can be `redis` or `s3`, and `port` and `path` are optional) | None |
| `BUILDCACHE_S3_ACCESS` | `s3_access` | S3 access key | None |
| `BUILDCACHE_S3_SECRET` | `s3_secret` | S3 secret key | None |
| `BUILDCACHE_LUA_PATH` | `lua_paths` | Extra path(s) to Lua wrappers | None |
| `BUILDCACHE_DEBUG` | `debug` | Debug level | None |
| `BUILDCACHE_LOG_FILE` | `log_file` | Log file path (empty for stdout) | None |
| `BUILDCACHE_MAX_CACHE_SIZE` | `max_cache_size` | Cache size limit in bytes | 5368709120 |
| `BUILDCACHE_MAX_LOCAL_ENTRY_SIZE` | `max_local_entry_size` | Local cache entry size limit in bytes (uncompressed) | 134217728 |
| `BUILDCACHE_MAX_REMOTE_ENTRY_SIZE` | `max_remote_entry_size` | Remote cache entry size limit in bytes (uncompressed) | 134217728 |
| `BUILDCACHE_HARD_LINKS` | `hard_links` | Allow the use of hard links when caching | false |
| `BUILDCACHE_COMPRESS` | `compress` | Allow the use of compression when caching (overrides hard links) | false |
| `BUILDCACHE_PERF` | `perf` | Enable performance logging | false |
| `BUILDCACHE_DISABLE` | `disable` | Disable caching (bypass BuildCache) | false |

An example configuration file:

```json
{
  "max_cache_size": 10000000000,
  "prefix": "icecc",
  "remote": "redis://my-server:6379",
  "debug": 3,
  "lua_paths": [
    "/home/myname/buildcache-lua",
    "/opt/buildcache-lua"
  ],
  "compress": true
}
```

To see the configuration options that are in effect, run:

```bash
$ buildcache -s
```

## Debugging

To get debug output from a BuildCache run, set the environment variable
`BUILDCACHE_DEBUG` to the desired debug level (debug output is disabled by default):

| BUILDCACHE_DEBUG | Level | Comment              |
| ---------------- | ----- | -------------------- |
| 1                | DEBUG | Maximum printouts    |
| 2                | INFO  |                      |
| 3                | ERROR |                      |
| 4                | FATAL |                      |
| -1               | -     | Disable debug output |

For instance:

```bash
$ BUILDCACHE_DEBUG=2 buildcache g++ -c -O2 hello.cpp -o hello.o
```

It is also possible to redirect the log output to a file using the
`BUILDCACHE_LOG_FILE` setting.
