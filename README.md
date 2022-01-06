# Compiler Brainworms

I write compilers semi-professionally and, boy, oh, boy, do they give me thoughts.

## Java and JVM
My big development on the JVM was [Shesmu](https://oicr-gsi.shesmu.github.io/), which I developed at the [Ontario Institute for Cancer Research](https://oicr.on.ca/).

- [`MethodHandles` for life-cycle management in Shesmu](shesmu-methodhandles.html) -- using the JVM's `MethodHandles` in the Shesmu compiler
- [Using a Custom Compiler to Reduce Toil](shesmu-toil.html) -- using a compiler to do things a human would always get wrong to reduce operational overhead

## Rust

- [Getting it done fast and fast with Rust](rust-barcodex.html) -- In a performance crunch, I was able to port a Python program to Rust in under two days and get a 10× performance improvement.

## C and C++

- [Don't argue with me, I know `ptrace`](ptrace-bcl2fastq.html) -- We had an uncooperative piece of closed-source software from Illumina, so we used Linux's ptrace to change its behaviour.
- [Using LLVM for fun and performance](llvm-bamql.html) -- To filter BAM files, a common biological data format, I built a domain-specific query language that compiles using LLVM. The resulting code is high-performance and hides a lot of complexity from the user.
