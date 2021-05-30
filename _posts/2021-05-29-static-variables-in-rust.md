---
layout: post
title: Static Variables in Rust
excerpt: "In this post I talk about static variables in Rust, compare them with static variables in C++. This post also tries to reason about the rules imposed by Rust on static variables and talks about why such rules are important."
comments: false
tags:
  - code
  - rust
---

Hello there, I hope you are doing ok. Today I would like to talk about static variables in Rust, compare them with static variables in C++ and try to reason about the rules imposed by Rust on static variables.

## Introduction
Static variable are variables declared with a `static` keyword and represent a specific global memory location (They are also known as global variables). Static variables have static life-time, a static life-time never goes out of scope and is guaranteed to out live any other variable. Meaning even if they are declared inside a scope their life-time does not begin or end with the scope.

```rust
fn func() -> &'static i23 {
   // global variable, same global memory location for every call.
   static SOMETHING: i32 = 0;

   return &SOMETHING;
}

// every call to func() returns reference to same memory location.
func()
func()
```

*In Rust static variable must be initialized at compile-time* (Meaning they cannot be initialized with state which can only be known at runtime). Also, *mutating a static variable is only possible in an unsafe context*. In this post we'll try to reason about these rules that are imposed by Rust on the static variables and also talk about why such rules are important.

```rust
// OK - 0 can be known at comple time.
static SOME_THING: i32 = 0;

// Error heap allocation is only possible in runtime.
static MEM: Box<i32> = Box::new(0)
```

To understand rules behind static variables let us take a short dive into the land of assembly.


## Land of Assembly
Rust is a native language that compiles down to assembly. An assembly program is generally divided into three sections:

- data
- bss
- text

The `data` section contains all the initialized static variables with their initial value, `bss` section contains all uninitialized/zero-initialized static variables and finally the `text` section contains all our code in assembly. You can read more about assembly layout [here](https://en.wikipedia.org/wiki/Data_segment). (This stuff is platform dependent so take it with a grain of salt.)


## Back to Rust

Rust does not allow uninitialized variables (static or non-static). So, the data, bss section may contain either initialized or zero-initialized static variables. Also, **since assembly is generated after compiling the Rust code and the assembly must contain static variables in special sections the static variable must be initialized at compile time**.


This does not mean you cannot have static variable which stores a state which can only be known at runtime. This just means that you need to initialize static with compile-time known state or value. We can easily store a value which can only be known at runtime using `enum` (variant) or something like `Option<T>`, setting them to a compile-time known value and then updating again later at runtime.


```rust
// Ok - initialized with compile-time known state/value.
static mut MEM: Option<Box<i32>> = None;

// ....... somewhere ........ //

// Ok
unsafe { MEM = Some(Box::new(5)) };
```
<p class="message">
It is not recommended to use mutable static since it is quite easy to run into an undefined behavior with it.<br />
I recommend using <a herf="https://crates.io/crates/lazy_static">lazy_static</a> or checking end part of this article for slightly better implementation.
</p>

In Rust, reading or writing a mutable static is unsafe because static variables are shared between threads and a mutable static might run into race conditions in a concurrent program, so it is particularly important to guard a mutable static with lock. *One of Rust's goals is to make concurrency bugs hard to run into.*

Let us now move our focus to C++,

## Static Initialization in C++

C++ allows initialization of a static variable even with a state which can only be known at runtime. This is possible mainly because of two reasons: First, C++ allows uninitialized variables and Second C++ can do static initialization in runtime before main executes if necessary.


C++ if necessary runs static initialization code before executing main which may lead to an extremely hard to detect problem known as [the static initialization order fiasco](https://www.cs.technion.ac.il/users/yechiel/c++-faq/static-init-order.html). Also, In C++ it is not clear if a variable is being initialized at compile time or at runtime (C++20 solves this problem with [constinit](https://en.cppreference.com/w/cpp/language/constinit) which makes sure that a static variable can be initialized at compile-time, but there is currently no solution for the static initialization fiasco in C++).

```c++

struct Test {
   // unique_pointer is a smart pointer similar to Box in rust.
   static unique_pointer<ComplexType> st_ptr;
};

// make_unique is similar to Box::new().
// This runs before main to initalized static st_ptr.
Test::st_tpr = make_unique(ComplexType());
```

In C++ local static variables (static variables declared inside a function, whose value is persistent across function calls) are initialized by the first function call, so they need to be implicitly provided with a lock guard by the compiler in order to avoid race conditions while initializing them. (i.e, when two thread try to initialize same local static.)



```c++
auto some_function() -> ComplexType {
   // First call to some_function initailizes ct.
   // Other calls will share the same ct initialized by the first call.
   // Compiler adds lock guard to avoid any race conditions.
   static ComplexType ct = ComplexType();

   return ct;
}
```

Rust solves all these issues that C++ suffers from by allowing static variables to only be initialized with a state which can be known at compile-time and making mutable static variables unsafe.

```rust

fn some_function() -> SomeStruct {
   // st is initialized at compile-time (data section set)
   // all call share same st.
   static SomeStruct st = SomeStruct{ a: 0 };

   return st;
}


```

Hence, when it comes to static variables Rust has fairly good reasons to impose its restrictions on how a static variable can be initialized. We can easily overcome this restriction to store pretty much anything in a static variable safely with the help of locks and proper abstraction.

## Better Example

As promised a better example for static variable that stores a value which can be only known at runtime. Try it live on [godbolt](https://godbolt.org/z/81oe7Mc58).

```rust
use std::sync::Once;
use std::cell::Cell;
use std::hint::unreachable_unchecked;

struct Test {
  pub a : Box<i32>,
}

fn get_static() -> &'static Test {

   // struct that stores our data + a lock guard
   struct Stt {
      data: Cell<Option<Test>>,
      once: Once // lock guard to make sure static is set only once
   }

   // we know that it is safe to use get_static() in a concurrent program
   unsafe impl Sync for Stt {}

   // static variable
   static A: Stt = Stt{data: Cell::new(None), once: Once::new() };

   // lock, call_once makes sure that the block is execture only once
   A.once.call_once(|| {

      // init static with a state at runtime - Heap allocation
      A.data.set(Some(Test{a: Box::new(5)}));
   });

   // get reference, dereferencing a raw pointer is unsafe
   let v = unsafe { match *A.data.as_ptr() {
      Some(ref a) => a,
      None => {
         // unreachable code, we are sure that data is never None
         unreachable_unchecked();
      }
   }};

   return v;
}

pub fn main() {
   let a = get_static(); // reference to static
   let b = get_static(); // another reference to static
}
```

Here we are using `Cell<T>` instead of `mut static` to update the state of the static variable once at runtime (on the first function call). This is much safer than mutable static approach, and we are also using a lock guard to avoid race any conditions.

[lazy_static](https://crates.io/crates/lazy_static) uses this approach under the hood behind all its macro magics.

## Conclusion
Static variables in Rust are quite different from programming language like C++ but in a safe way. At first, it may seem like the Rust's static variables are somewhat limited but with the help of library like [lazy_static](https://crates.io/crates/lazy_static) we can utilize static safely and effectively.

<p class="message">
This is my first blog post, so I would love to receive some feedback. You can reach me at <a href = "mailto:{{site.hydeout.email}}">{{site.hydeout.email}}</a>
</p>