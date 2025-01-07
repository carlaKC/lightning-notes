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
