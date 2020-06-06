---
title: "Using Rust in C/C++ code"
category: en
tags: c c++ rust programming
---

There is a Rust joke, that everything must be (re)written in Rust.
Well, it is a compiled language compatible with C, nice to write 
No wonder people started to write __everything__ in Rust

Discord switched from Go to Rust
Fancy terminal task manager? `gotop` has been recently rewritten into `ytop`
Shell? NuShell \
Terminal? AlaCritty \
Browser? Servo \
Games? Veloren \
C library? relibc \
OS? Redox \
BIOS? oreboot

Even Rust compiler is itself written in Rust! ;)

But sometimes it is not possible to just "throw out" C++ and rewrite it in Rust,
because rewrites take lots of time, especially if your code is hundreds of thousands
lines long. 

For example, what if there is a security bug in the old code?
Now both old code and new code have to be corrected.

Rust has this fantastic ability to compile into `.a` files, which can be linked later
with any language that supports them.

So, lets say I want to use Rust to convert UTF-8 `std::string` into UTF-32 
`std::vector<uint32_t>`.

Well, first I have to establish an interface which I'm gonna use.
Rust doesn't support C++ ABI (binary interface), but C++ and Rust support C ABI,
so it is a common choice for 'glueing' languages together.

Rust's allocator is not compatible with C++'s, so returning an `Vec<u32>` wouldn't
be a good choice here, nor passing `std::vector<uint32_t>` as `Vec<u32>`, so what
is an option here is to pass a pointer to `std::vector`, export `push_back` functionality
and then write a loop like this
```rust
for character in input_str.chars() {
  cpp_push_back_u32(vecptr, character as u32);
}
```

With input parameters there shouldn't be any problem - just pass pointer to buffer
and its length.

Putting this together, function looks like this
```rust
#[no_mangle]
pub extern "C" 
fn utf8_to_utf32(ptr: *const u8, len: usize, vec: &mut CppVecU32) -> bool {
  let input_slice = unsafe { slice::from_raw_parts(ptr, len) };
  let input_str = match str::from_utf8(input_slice) {
    Some(s) => s,
    None => return false,
  };

  for character in input_str.chars() {
    unsafe { cpp_push_back_u32(vec, character as u32); }
  }

  return true;
}
```
Calling functions from C is `unsafe`, because compiler can't check the code.

What is still missing is `CppVecU32` and `cpp_push_back_u32` definitions.
On Linux distros C++ stdlib is located in `/usr/include/c++` folder (at least
on my machine)
`std::vector` basically has three pointers:
  * begin (beginning of buffer it holds)
  * end (begin + len)
  * buffer\_end (begin + capacity)

```rust
#[repr(C)]
struct CppVecU32 {
  begin:      *mut u32,
  end:        *mut u32,
  buffer_end: *const u32,
}
```

(although it won't be necessary later, I just left it as a fun fact) \
and `cpp_push_back_u32` is just

```rust
extern "C" {
  fn cpp_push_back_u32(vec: &mut CppVecU32, x: u32);
}
```

To compile it as a library, these three lines need to be written inside Cargo.toml
```
[lib]
name = "rusty"
crate-type = ["staticlib"]
```

Rust should compile this code into `librusty.a` file inside 
target/[debug|release]/ directory.

The C++ file looks like this
```cpp
#include <vector>
#include <stdint.h>
#include <iostream>
#include <string_view>

/* Imports */
extern "C" bool utf8_to_utf32(const char*, size_t n, std::vector<uint32_t>* buf);

/* Exports */
extern "C" void cpp_vector_push_u32(std::vector<uint32_t>* v, uint32_t x) {
  v->push_back(x);
}

int main() {
  std::string s = "Witaj świecie!";
  std::vector<uint32_t> v;
  v.reserve(32);

  bool ok = utf8_to_utf32(s.data(), s.size(), &v);
  if(!ok) {
    std::cerr << "ERROR" << std::endl;
    return 1;
  }

  for(const auto& x : v) {
    std::cout << x << ' ';
  }
  std::cout << std::endl;
}
```

`extern "C"` in C++ prevents compiler from mangling the function name. \
Lets try to compile with `clang++ main.cpp rust/target/debug/librusty.a -std=c++20` and... oh my!
A HUGE wall of linker complaining about `undefined reference`s. \
Well, most of these functions/variables are not used in this program, so passing `-flto`
flag fixes the problem, but I consider it more as a hack than a solution.

Solution here would be to get rid of Rust's standard library.

[Here is a great blog post](https://fasterthanli.me/blog/2020/a-no-std-rust-binary/)
that explains the process of making a `no_std` rust binary. \
I'll do everything as described in it, with exception to panic handler.
What I want to do is pass the error message to C++, print it and then abort.

The C++ part looks like this
```cpp
extern "C" void cpp_rethrow_panic(const char* s, size_t n, uint32_t line) {
  auto filename = std::string_view(s, n);

  if(line == std::numeric_limits<uint32_t>::max()) {
    std::cerr << "Rust panic occured" << std::endl;
  } else {
    std::cerr << "Rust panic occured in file '" << filename 
              << "' at line " << line << std::endl;
  }

  abort();
}
```

And in Rust code one more function is exported, `use std::{str, slice}` has changed to \
`use core::{str, slice}`. \
Notice the `!` return type - this way Rust expects this function to never return.
```rust
#![no_std]
#![feature(lang_items)]

#[lang = "eh_personality"]
fn eh_personality() {}

#[panic_handler]
fn panic(info: &core::panic::PanicInfo) -> ! {
  let (file, line) = match info.location() {
    Some(loc) => (loc.file(), loc.line()),
    None      => ("UNKNOWN FILE", u32::MAX),
  };

  unsafe {
    cpp_rethrow_panic(file.as_ptr(), file.len(), line);
  }
}

/* Imports */
use core::{slice, str};

extern "C" {
  fn cpp_rethrow_panic(s: *const u8, n: usize, line: u32) -> !;
  fn cpp_vector_push_u32(vec: &mut CppVecU32, character: u32);
}

#[repr(C)]
pub struct CppVecU32 {
  start: *mut u32,
  finish: *mut u32,
  buffer_end: *const u32,
}

/* Exports */
#[no_mangle]
pub extern "C" fn utf8_to_utf32(ptr: *const u8, len: usize, vec: &mut CppVecU32) -> bool {
  let input_slice: &[u8] = unsafe { slice::from_raw_parts(ptr, len) };
  let s = match str::from_utf8(input_slice) {
    Ok(x) => x,
    Err(_) => return false,
  };

  for character in s.chars() {
    unsafe { cpp_vector_push_u32(vec, character as u32) };
  }

  return true;
}
```

And this is how the whole C++ code looks like.

```cpp
#include <vector>
#include <stdint.h>
#include <iostream>
#include <string_view>

/* Imports */
extern "C" bool utf8_to_utf32(const char*, size_t n, std::vector<uint32_t>* buf);

/* Exports */
extern "C" void cpp_vector_push_u32(std::vector<uint32_t>* v, uint32_t x) {
  v->push_back(x);
}
extern "C" void cpp_rethrow_panic(const char* s, size_t n, uint32_t line) {
  auto filename = std::string_view(s, n);

  if(line == std::numeric_limits<uint32_t>::max()) {
    std::cerr << "Rust panic occured" << std::endl;
  } else {
    std::cerr << "Rust panic occured in file '" << filename 
              << "' at line " << line << std::endl;
  }

  abort();
}

int main() {
  std::string s = "Witaj świecie!";
  std::vector<uint32_t> v;
  v.reserve(32);

  bool ok = utf8_to_utf32(s.data(), s.size(), &v);
  if(!ok) {
    std::cerr << "ERROR" << std::endl;
    return 1;
  }

  for(const auto& x : v) {
    std::cout << x << ' ';
  }
  std::cout << std::endl;
}
```

Compile the Rust code with `cargo build --release` and C++ with 
`clang++ main.cpp rust/target/release/librusty.a -std=c++20 -O3`
and now everything is done!

Code from this post can be found [here](https://github.com/Soveu/c-and-rust)

