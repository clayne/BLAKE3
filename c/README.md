The official C implementation of BLAKE3.

# Example

An example program that hashes bytes from standard input and prints the
result:

```c
#include "blake3.h"
#include <errno.h>
#include <stdio.h>
#include <string.h>
#include <unistd.h>

int main(void) {
  // Initialize the hasher.
  blake3_hasher hasher;
  blake3_hasher_init(&hasher);

  // Read input bytes from stdin.
  unsigned char buf[65536];
  while (1) {
    ssize_t n = read(STDIN_FILENO, buf, sizeof(buf));
    if (n > 0) {
      blake3_hasher_update(&hasher, buf, n);
    } else if (n == 0) {
      break; // end of file
    } else {
      fprintf(stderr, "read failed: %s\n", strerror(errno));
      return 1;
    }
  }

  // Finalize the hash. BLAKE3_OUT_LEN is the default output length, 32 bytes.
  uint8_t output[BLAKE3_OUT_LEN];
  blake3_hasher_finalize(&hasher, output, BLAKE3_OUT_LEN);

  // Print the hash as hexadecimal.
  for (size_t i = 0; i < BLAKE3_OUT_LEN; i++) {
    printf("%02x", output[i]);
  }
  printf("\n");
  return 0;
}
```

The code above is included in this directory as `example.c`. If you're
on x86\_64 with a Unix-like OS, you can compile a working binary like
this:

```bash
gcc -O3 -o example example.c blake3.c blake3_dispatch.c blake3_portable.c \
    blake3_sse2_x86-64_unix.S blake3_sse41_x86-64_unix.S blake3_avx2_x86-64_unix.S \
    blake3_avx512_x86-64_unix.S
```

# API

## The Struct

```c
typedef struct {
  // private fields
} blake3_hasher;
```

An incremental BLAKE3 hashing state, which can accept any number of
updates. This implementation doesn't allocate any heap memory, but
`sizeof(blake3_hasher)` itself is relatively large, currently 1912 bytes
on x86-64. This size can be reduced by restricting the maximum input
length, as described in Section 5.4 of [the BLAKE3
spec](https://github.com/BLAKE3-team/BLAKE3-specs/blob/master/blake3.pdf),
but this implementation doesn't currently support that strategy.

## Common API Functions

```c
void blake3_hasher_init(
  blake3_hasher *self);
```

Initialize a `blake3_hasher` in the default hashing mode.

---

```c
void blake3_hasher_update(
  blake3_hasher *self,
  const void *input,
  size_t input_len);
```

Add input to the hasher. This can be called any number of times. This function
is always single-threaded; for multithreading see `blake3_hasher_update_tbb`
below.


---

```c
void blake3_hasher_finalize(
  const blake3_hasher *self,
  uint8_t *out,
  size_t out_len);
```

Finalize the hasher and return an output of any length, given in bytes.
This doesn't modify the hasher itself, and it's possible to finalize
again after adding more input. The constant `BLAKE3_OUT_LEN` provides
the default output length, 32 bytes, which is recommended for most
callers. See the [Security Notes](#security-notes) below.

## Less Common API Functions

```c
void blake3_hasher_init_keyed(
  blake3_hasher *self,
  const uint8_t key[BLAKE3_KEY_LEN]);
```

Initialize a `blake3_hasher` in the keyed hashing mode. The key must be
exactly 32 bytes.

---

```c
void blake3_hasher_init_derive_key(
  blake3_hasher *self,
  const char *context);
```

Initialize a `blake3_hasher` in the key derivation mode. The context
string is given as an initialization parameter, and afterwards input key
material should be given with `blake3_hasher_update`. The context string
is a null-terminated C string which should be **hardcoded, globally
unique, and application-specific**. The context string should not
include any dynamic input like salts, nonces, or identifiers read from a
database at runtime. A good default format for the context string is
`"[application] [commit timestamp] [purpose]"`, e.g., `"example.com
2019-12-25 16:18:03 session tokens v1"`.

This function is intended for application code written in C. For
language bindings, see `blake3_hasher_init_derive_key_raw` below.

---

```c
void blake3_hasher_init_derive_key_raw(
  blake3_hasher *self,
  const void *context,
  size_t context_len);
```

As `blake3_hasher_init_derive_key` above, except that the context string
is given as a pointer to an array of arbitrary bytes with a provided
length. This is intended for writing language bindings, where C string
conversion would add unnecessary overhead and new error cases. Unicode
strings should be encoded as UTF-8.

Application code in C should prefer `blake3_hasher_init_derive_key`,
which takes the context as a C string. If you need to use arbitrary
bytes as a context string in application code, consider whether you're
violating the requirement that context strings should be hardcoded.

---

```c
void blake3_hasher_update_tbb(
  blake3_hasher *self,
  const void *input,
  size_t input_len);
```

Add input to the hasher, using [oneTBB] to process large inputs using multiple
threads. This can be called any number of times. This gives the same result as
`blake3_hasher_update` above.

[oneTBB]: https://uxlfoundation.github.io/oneTBB/

NOTE: This function is only enabled when the library is compiled with CMake option `BLAKE3_USE_TBB`
and when the oneTBB library is detected on the host system. See the building instructions for
further details.

To get any performance benefit from multithreading, the input buffer needs to
be large. As a rule of thumb on x86_64, `blake3_hasher_update_tbb` is _slower_
than `blake3_hasher_update` for inputs under 128 KiB. That threshold varies
quite a lot across different processors, and it's important to benchmark your
specific use case.

Hashing large files with this function usually requires
[memory-mapping](https://en.wikipedia.org/wiki/Memory-mapped_file), since
reading a file into memory in a single-threaded loop takes longer than hashing
the resulting buffer. Note that hashing a memory-mapped file with this function
produces a "random" pattern of disk reads, which can be slow on spinning disks.
Again it's important to benchmark your specific use case.

This implementation doesn't require configuration of thread resources and will
use as many cores as possible by default. More fine-grained control of
resources is possible using the [oneTBB] API.

---

```c
void blake3_hasher_finalize_seek(
  const blake3_hasher *self,
  uint64_t seek,
  uint8_t *out,
  size_t out_len);
```

The same as `blake3_hasher_finalize`, but with an additional `seek`
parameter for the starting byte position in the output stream. To
efficiently stream a large output without allocating memory, call this
function in a loop, incrementing `seek` by the output length each time.

---

```c
void blake3_hasher_reset(
  blake3_hasher *self);
```

Reset the hasher to its initial state, prior to any calls to
`blake3_hasher_update`. Currently this is no different from calling
`blake3_hasher_init` or similar again.

# Security Notes

Outputs shorter than the default length of 32 bytes (256 bits) provide less security. An N-bit
BLAKE3 output is intended to provide N bits of first and second preimage resistance and N/2
bits of collision resistance, for any N up to 256. Longer outputs don't provide any additional
security.

Avoid relying on the secrecy of the output offset, that is, the `seek` argument of
`blake3_hasher_finalize_seek`. [_Block-Cipher-Based Tree Hashing_ by Aldo
Gunsing](https://eprint.iacr.org/2022/283) shows that an attacker who knows both the message
and the key (if any) can easily determine the offset of an extended output. For comparison,
AES-CTR has a similar property: if you know the key, you can decrypt a block from an unknown
position in the output stream to recover its block index. Callers with strong secret keys
aren't affected in practice, but secret offsets are a [design
smell](https://en.wikipedia.org/wiki/Design_smell) in any case.

# Building

The easiest and most complete method of compiling this library is with CMake.
This is the method described in the next section. Toward the end of the
building section there are more in depth notes about compiling manually and
things that are useful to understand if you need to integrate this library with
another build system.

## CMake

The minimum version of CMake is 3.9. The following invocations will compile and
install `libblake3`. With recent CMake:

```bash
cmake -S c -B c/build "-DCMAKE_INSTALL_PREFIX=/usr/local"
cmake --build c/build --target install
```

With an older CMake:

```bash
cd c
mkdir build
cd build
cmake .. "-DCMAKE_INSTALL_PREFIX=/usr/local"
cmake --build . --target install
```

The following options are available when compiling with CMake:

- `BLAKE3_USE_TBB`: Enable oneTBB parallelism (Requires a C++20 capable compiler)
- `BLAKE3_FETCH_TBB`: Allow fetching oneTBB from GitHub (only if not found on system)
- `BLAKE3_EXAMPLES`: Compile and install example programs

Options can be enabled like this:

```bash
cmake -S c -B c/build "-DCMAKE_INSTALL_PREFIX=/usr/local" -DBLAKE3_USE_TBB=1 -DBLAKE3_FETCH_TBB=1
```

## Building manually

We try to keep the build simple enough that you can compile this library "by
hand", and it's expected that many callers will integrate it with their
pre-existing build systems. See the `gcc` one-liner in the "Example" section
above.

### x86

Dynamic dispatch is enabled by default on x86. The implementation will
query the CPU at runtime to detect SIMD support, and it will use the
widest instruction set available. By default, `blake3_dispatch.c`
expects to be linked with code for five different instruction sets:
portable C, SSE2, SSE4.1, AVX2, and AVX-512.

For each of the x86 SIMD instruction sets, four versions are available:
three flavors of assembly (Unix, Windows MSVC, and Windows GNU) and one
version using C intrinsics. The assembly versions are generally
preferred. They perform better, they perform more consistently across
different compilers, and they build more quickly. On the other hand, the
assembly versions are x86\_64-only, and you need to select the right
flavor for your target platform.

Here's an example of building a shared library on x86\_64 Linux using
the assembly implementations:

```bash
gcc -shared -O3 -o libblake3.so blake3.c blake3_dispatch.c blake3_portable.c \
    blake3_sse2_x86-64_unix.S blake3_sse41_x86-64_unix.S blake3_avx2_x86-64_unix.S \
    blake3_avx512_x86-64_unix.S
```

When building the intrinsics-based implementations, you need to build
each implementation separately, with the corresponding instruction set
explicitly enabled in the compiler. Here's the same shared library using
the intrinsics-based implementations:

```bash
gcc -c -fPIC -O3 -msse2 blake3_sse2.c -o blake3_sse2.o
gcc -c -fPIC -O3 -msse4.1 blake3_sse41.c -o blake3_sse41.o
gcc -c -fPIC -O3 -mavx2 blake3_avx2.c -o blake3_avx2.o
gcc -c -fPIC -O3 -mavx512f -mavx512vl blake3_avx512.c -o blake3_avx512.o
gcc -shared -O3 -o libblake3.so blake3.c blake3_dispatch.c blake3_portable.c \
    blake3_avx2.o blake3_avx512.o blake3_sse41.o blake3_sse2.o
```

Note above that building `blake3_avx512.c` requires both `-mavx512f` and
`-mavx512vl` under GCC and Clang. Under MSVC, the single `/arch:AVX512`
flag is sufficient. The MSVC equivalent of `-mavx2` is `/arch:AVX2`.
MSVC enables SSE2 and SSE4.1 by default, and it doesn't have a
corresponding flag.

If you want to omit SIMD code entirely, you need to explicitly disable
each instruction set. Here's an example of building a shared library on
x86 with only portable code:

```bash
gcc -shared -O3 -o libblake3.so -DBLAKE3_NO_SSE2 -DBLAKE3_NO_SSE41 -DBLAKE3_NO_AVX2 \
    -DBLAKE3_NO_AVX512 blake3.c blake3_dispatch.c blake3_portable.c
```

### ARM NEON

The NEON implementation is enabled by default on AArch64, but not on
other ARM targets, since not all of them support it. To enable it, set
`BLAKE3_USE_NEON=1`. Here's an example of building a shared library on
ARM Linux with NEON support:

```bash
gcc -shared -O3 -o libblake3.so -DBLAKE3_USE_NEON=1 blake3.c blake3_dispatch.c \
    blake3_portable.c blake3_neon.c
```

To explicitiy disable using NEON instructions on AArch64, set
`BLAKE3_USE_NEON=0`.

```bash
gcc -shared -O3 -o libblake3.so -DBLAKE3_USE_NEON=0 blake3.c blake3_dispatch.c \
    blake3_portable.c 
```

Note that on some targets (ARMv7 in particular), extra flags may be
required to activate NEON support in the compiler. If you see an error
like...

```
/usr/lib/gcc/armv7l-unknown-linux-gnueabihf/9.2.0/include/arm_neon.h:635:1: error: inlining failed
in call to always_inline ‘vaddq_u32’: target specific option mismatch
```

...then you may need to add something like `-mfpu=neon-vfpv4
-mfloat-abi=hard`.

### Other Platforms

The portable implementation should work on most other architectures. For
example:

```bash
gcc -shared -O3 -o libblake3.so blake3.c blake3_dispatch.c blake3_portable.c
```

### Multithreading

Multithreading is available using [oneTBB], by compiling the optional C++
support file [`blake3_tbb.cpp`](./blake3_tbb.cpp). For an example of using
`mmap` (non-Windows) and `blake3_hasher_update_tbb` to get large-file
performance on par with [`b3sum`](../b3sum), see
[`example_tbb.c`](./example_tbb.c). You can build it like this:

```bash
g++ -c -O3 -fno-exceptions -fno-rtti -DBLAKE3_USE_TBB -o blake3_tbb.o blake3_tbb.cpp
gcc -O3 -o example_tbb -lstdc++ -ltbb -DBLAKE3_USE_TBB blake3_tbb.o example_tbb.c blake3.c \
    blake3_dispatch.c blake3_portable.c blake3_sse2_x86-64_unix.S blake3_sse41_x86-64_unix.S \
    blake3_avx2_x86-64_unix.S blake3_avx512_x86-64_unix.S
```

NOTE: `-fno-exceptions` or equivalent is required to compile `blake3_tbb.cpp`,
and public API methods with external C linkage are marked `noexcept`. Compiling
that file with exceptions enabled will fail. Compiling with RTTI disabled isn't
required but is recommended for code size.
