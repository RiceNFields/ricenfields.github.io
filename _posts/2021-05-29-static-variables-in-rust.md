---
layout: post
title: Understanding Static Variables in Rust
excerpt: "In this post I talk about static variables in Rust, compare them with static variables in C++. This post also tries to reason about the rules imposed by Rust on static variables and talks about why such rules are important."
comments: false
tags:
  - code
  - rust
---

Hello there, I hope you are doing ok. Today I would like to talk about static variables in Rust, compare them with static variables in C++ and also   try to reason about the rules imposed by Rust on static variables.

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

In Rust,
* *static variables must be initialized at compile-time* (Meaning they cannot be initialized with state which can only be known at runtime),
* *The type of static variable must have the [Sync trait](https://doc.rust-lang.org/nomicon/send-and-sync.html) bound* (Meaning the type should be safe to share between threads, `Sync` is an automatically derived trait with some exceptions) and
* *mutating a static variable is only possible in an unsafe context*.

In this post we'll try to reason about these rules that are imposed by Rust on the static variables and also talk about why such rules are important.

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

Rust does not allow uninitialized static variables. So, the data, bss section may contain either initialized or zero-initialized static variables. Also, **since assembly is generated after compiling the Rust code and the assembly must contain static variables in special sections, the static variable must be initialized at compile time**.


This does not mean you cannot have static variable that stores a state which can only be known at runtime. This just means that you need to initialize static with compile-time known state or value. There is an easy way to store a value that can only be known at runtime utilizing a `enum` (variant) or something like `Option<T>` by setting them to a compile-time known value and updating them later at runtime.

```rust
// Ok - initialized with compile-time known state/value.
static mut MEM: Option<Box<i32>> = None;

// ....... somewhere ........ //

// Ok
unsafe { MEM = Some(Box::new(5)) };
```
<p class="message">
It is not recommended to use mutable static since it is quite easy to run into an undefined behavior with it.<br />
I recommend using <a href="https://crates.io/crates/lazy_static">lazy_static</a> instead or checking end part of this article for slightly better implementation.
</p>

As *one of Rust’s goals is to make concurrency bugs harder to run into*, reading or writing a mutable static is unsafe because static variables are shared between threads and a mutable static might run into race conditions in a concurrent program. This is why it is particularly important to guard a mutable static with lock. Also, for same reasons the type of non-mutable static variable should only allow thread safe access.

Let us now move our focus to C++.

## Static Initialization in C++

C++ allows initialization of a static variable even with a state which can only be known at runtime. This is possible mainly because of two reasons:
* First, C++ allows uninitialized variables.
* Second, C++ can do static initialization in runtime before main executes if necessary.


Since C++ can carry out static initialization before the main method executes, it might lead to an extremely hard to detect problem known as [the static initialization order fiasco](https://www.cs.technion.ac.il/users/yechiel/c++-faq/static-init-order.html). It is also not clear if a variable is being initialized at compile time or at runtime. [C++20](https://en.cppreference.com/w/cpp/20) solves this problem with [constinit](https://en.cppreference.com/w/cpp/language/constinit), which makes sure that a static variable can be initialized at compile-time. That being said, there is still no solution for the static initialization fiasco in C++.

```c++

struct Test {
   // unique_pointer is a smart pointer similar to Box in rust.
   static unique_pointer<ComplexType> st_ptr;
};

// make_unique is similar to Box::new().
// This runs before main to initalized static st_ptr.
Test::st_tpr = make_unique(ComplexType());
```

In C++ local static variables (static variables declared inside a function, whose value is persistent across function calls) are initialized by the first function call, because of which they need to be implicitly provided with a lock guard by the compiler. This helps to avoid any race conditions that might occur during initialization, when two or more threads try to initialize the same local static variable.


```c++
auto some_function() -> ComplexType {
   // First call to some_function initailizes ct.
   // Other calls will share the same ct initialized by the first call.
   // Compiler adds lock guard to avoid any race conditions.
   static ComplexType ct = ComplexType();

   return ct;
}
```

Rust solves all these issues that C++ suffers from by making mutable static variables unsafe and at the same time, allowing static variables to be initialized only with a state which can be known during compile-time.

```rust

fn some_function() -> SomeStruct {
   // st is initialized at compile-time (data section set)
   // all call share same st.
   static SomeStruct st = SomeStruct{ a: 0 };

   return st;
}


```

Hence, when it comes to static variables, Rust has fairly good reasons to impose the restrictions on how a static variable can be initialized. However, we can easily bypass these restrictions and store pretty much anything in a static variable safely with the help of lock and proper abstraction.

## Better Example

As promised, here is a better example for static variable that stores a value which can be only known at runtime. Try it live on [godbolt](https://godbolt.org/z/6xcGKzKEx).

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

   // static variable type must have the Sync trait bound.
   // and we also make sure that Stt can only be accessed in a thread safe manner.
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

In this example, we are using `Cell<T>` instead of `mut static` in order to update the state of the static variable once at runtime (on the first function call). This is much safer than the mutable static approach, we are also using a lock guard to avoid any race conditions.

Also, since Rust doesn't automatically derive [Sync trait](https://doc.rust-lang.org/nomicon/send-and-sync.html) for our type `Stt` because of `Cell<T>`(`Cell<T>` is not thread safe type). We have to implement the `Sync` trait manually, and make sure that our type `Stt` can only be accessed in a thread safe manner.

As I mentioned earlier, you should use [lazy_static](https://crates.io/crates/lazy_static). Under the hood, behind all its macro magic lazy_static also uses similar approach.

## Conclusion
Static variables in Rust are quite different from programming language such as C++, because they can be used in a much safer way. At first, it may seem like the Rust's static variables are somewhat limited but with the help of library like [lazy_static](https://crates.io/crates/lazy_static), we can utilize static safely and effectively.

<p class="message">
This is my first blog post, so I would love to receive some feedback. You can reach me at <a href = "mailto:{{site.hydeout.email}}">{{site.hydeout.email}}</a>
</p>
