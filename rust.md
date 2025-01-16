# Error Handling

[Rust Book / Errors](https://doc.rust-lang.org/book/ch09-00-error-handling.html)
- Panic: unrecoverable errors which could land your code in a bad state
  - Doesn't include bad user input, for example - it's unexpected
- Result: recoverable errors where you want to give the caller agency

Rustaceans (pg 57)
Options for defining errors are:
- Enumeration: create an enum with specific error cases, should
  - Implement `std:error::Error`
  - Implement `Display` and `Debug` (lowercase/no punctuation to fit inline)
  - Implement `Send` and `Sync`
  - Should be `'static`
- Erasure: something went wrong, it doesn't matter what (and you don't
  want to complicate API with details)
  - `Box<dyn Error + Send + Sync + 'static>` completely erases an error
    - If there is interesting information associated with the error, 
      you may want to do something else.
  - This makes errors easy to combine, as you can propagate with ? all
    the way up.
  - `'static` is useful because it allows you to downcast the `dyn Error`
    to a specific type using `downcast_ref -> Option(error of that type)`
    - There isn't agreement about whether the downcasted error counts
      as part of a library's stable API (ie, does changing the 
      underlying type constitute a break)

[Rust Conf / Error Handling](https://www.youtube.com/watch?v=WzC7H83aDZc)

[Reddit thread on libraries specifically](https://www.reddit.com/r/rust/comments/9x17hn/comment/e9p5c9t/?utm_source=share&utm_medium=web3x&utm_name=web3xcss&utm_term=1&utm_content=share_button})
- Panic if your library has a bug
- Panic if your library's precondition is not met (they have a bug)

# References when `Copy` is implemented

I have a ~50 byte struct, and was wondering when it's better to pass
it by reference vs just allowing it to be copied.

Re-read rust book ch04 (specifically [borrowing](https://doc.rust-lang.org/book/ch04-02-references-and-borrowing.html)).
- Seems like the [general rule](https://users.rust-lang.org/t/what-is-faster-copy-or-passing-reference/80733/3)
  is that structs that will fit in a register (<64 bytes) are faster
  to copy because CPUs are optimized for register access nowadays,
  but generally you have to benchmark to understand in the context of
  your code.
- Adding a reference can [complicate](https://www.reddit.com/r/rust/comments/zzxz2e/comment/j2f551r/?utm_source=share&utm_medium=web3x&utm_name=web3xcss&utm_term=1&utm_content=share_button)
  underlying in CPU; even if you only end up passing an 8 byte reference
  you end up doing more juggling under the hood to create that reference
  when you could have just accessed a few registers that the struct was
  stored in
- When you pass by value, the compiler may [optimize](https://www.reddit.com/r/rust/comments/r8vxkv/comment/hn89fsn/?utm_source=share&utm_medium=web3x&utm_name=web3xcss&utm_term=1&utm_content=share_button)
  to pass by reference under the hood anyway
- Nice [summary here](https://www.reddit.com/r/rust/comments/r8vxkv/comment/hn8qux0/?utm_source=share&utm_medium=web3x&utm_name=web3xcss&utm_term=1&utm_content=share_button).

tl;dr: 
- pass by value for small structs
- confirm compiler optimization for large structs by value if I run
  into it (if yes, then always pass by value)

[Warp speed rust](http://troubles.md/rust-optimization/) looks like a
nice read if I get the time.

# Generics vs Dyn

Per [the book](https://www.lurklurk.org/effective-rust/generics.html#basic-comparisons):
* `dyn` requires 2x dereferences (fat pointer to vtable + value itself)
* Generics are slower to compile, _slightly_ faster to run
  * Don't optimize on this unless you've actually benchmarked a 
    difference
* Generics are helpful for conditionally implementing features based
  on the traits that something implements:
  * ie, you can implement with bound `A + B` to only surface that
    functionality if both traits are available
* Trait objects (`dyn`) also must adhere to object safety:
  * No generics: generics are technically infinite impls, can't fit in
    a vtable
  * No `self`: the size of self is not known, can't assign stack space
    * Exception if it is sized

tl;dr: trait objects give you type erasure, which allows you to use a
heterogeneous set of implementations of a trait that _only_ need the
trait methods - eg you have trait X and you want to use A, B, and C that
all implement X just as an "erased" implementation of X.

Key question: do I need to call this with different types?
- Yes: trait object
- No: generics
