+++
title = 'Firmwhat?'
date = 2024-01-24
+++

Firmware development has been much of the same for decades.

The stagnation of industry has resulted in overly complicated, fragile, and unmaintainable code-bases for even the simplest of applications.

Nomad deems this level of inadequacy **unacceptable** for safety critical systems.

---

{{< video src="rust_logo.mp4" controls="false" muted="true" autoplay="true">}}

## The Solution

[The Rust Programming Language][5] boasts an enormous amount of state of the art tooling, ideologies, and technology to achieve maximum safety, performance, and robustness across many domains.

Of course, the domain of interest to us is *Embedded Systems*.

### Safety
#### Static Analysis

Rust makes it easy to conduct *static analysis* on our code.

Rust spreads it's tendrils down the branches of your program, tracking how memory is moved about.

This enables powerful compile-time inferences relating to types, flow, etc.

In terms of safety critical applications, the more compile-time checks the better, as it minimizes the opportunity for error at runtime.

#### RAII

Rust strictly abides by [RAII][1]. This directly applies to embedded applications as peripheral or memory access is bound to variables and tracked by rust as a natural consequence of the language.

This makes peripheral related code extremely easy to use and.

### Performance

Rust uses the [LLVM][2] compiler backend, this enables massive size and speed optimizations.

In addition to this, many of Rust's principles like [ownership][4], [borrowing][3], explicitness, and decoupling result in highly performant yet elegant code.

### Robustness

Developing firmware for multiple platforms is not only possible, but *easy* with Rust's tools.

Flashing, remote debugging, logging, and conditional compilation features are built in and make the development process a breeze.

Rust's procedural macro system makes complex code generation simple and flexible. Huge platform-specific structures can be generated in seconds, and hyperspecialization can be achieved with a general solution.

## Conclusion

For a massively complex and safety critical system such as *The Monster*, Embedded Rust proves to be the *only* right choice.

## References

1. [The Rust Programming Language][5]
2. [Resource Acquisition is Initialization - Rust By Example][1]
3. [LLVM - Wikipedia][2]
4. [Ownership - Rust Book][4]
5. [Borrowing - Rust By Example][3]

[1]: https://doc.rust-lang.org/rust-by-example/scope/raii.html
[2]: https://en.wikipedia.org/wiki/LLVM
[3]: https://doc.rust-lang.org/rust-by-example/scope/borrow.html
[4]: https://doc.rust-lang.org/book/ch04-01-what-is-ownership.html
[5]: https://www.rust-lang.org