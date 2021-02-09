---
layout: post
title: Where everything went wrong...
categories: [Rust, Error, Minidump]
mastodon:
  id: 105700724543137442
---

**Today you are frustrated.**

This is so annoying. You've written a Rust crate and now that you want to test it for the very first time, _it doesn't work!_

Come on, Rust! How dare you? You promised that once one gets past the compiler, it.  
_Just.  
**Works!**_  
And now this!

Ok, ok. You calm yourself down. Lets start from the beginning. You want to create so called [minidumps](https://docs.sentry.io/platforms/native/guides/minidumps/). This is a file that contains information about a crashed program (like stacks of all threads, CPU registers, system info, etc.).
The minidump consists of various sections, such as the minidump header (including time of day, versions and basically a table of contents), a thread section (including all threads of the process and their stacks), memory mappings and libraries, etc. [Just to give some context, as all of this is actually not really important.]

For this, you created a [crate](https://github.com/msirringhaus/minidump_writer_linux). One section gets written after the other, while information about the targeted process is retrieved from the system. You even created a nice, simple API. You hand in a process ID and an open file, where the minidump should be written to. like this:

```rust
    MinidumpWriter::new(pid, blamed_thread)
        .dump(&mut dump_file)
        .expect("Dumping failed!")
```

You can also hand in user specified memory regions that should be included in the dump, like so:

```rust
    let app_memory = AppMemory {
        ptr: some_address,
        length: memory_size,
    };

    MinidumpWriter::new(pid, pid)
        .set_app_memory(vec![app_memory])
        .dump(&mut tmpfile)
        .expect("Dumping failed");
```

# A wild error appears

But when you run your nice library code in an application, you get `'Dumping failed: "Failed in ptrace::read: Sys(EIO)"'`.

_How useless is that?!_

Okay, maybe you could enhance your library error handling, a little. And by enhance, you mean "implement one in the first place".

## State of the dart

Your current approach is to define

```rust
type Error = Box<dyn error::Error + std::marker::Send + std::marker::Sync>;
pub type Result<T> = result::Result<T, Error>;
```

and using `Result<T>` in all of your functions as the return value and handing all of them to the parent function using `?`. Thus the original error pierces through your callstack like a dart through....jelly (Yes, you are good with words and you know it.).

```rust
    pub fn init(&mut self) -> Result<()> {
        self.read_auxv()?;
        self.enumerate_threads()?;
        self.enumerate_mappings()?;
        Ok(())
    }
```

In Rust parlance, this is also called bubbling up errors.

Usually, you just bubble up errors from libraries you use, but for the rare errors you have to define yourself, you currently just do
```rust
Err("Found no auxv entry".into())
```

Well, now you know there is an error, at least. And that it has _something_ to do with your usage of `ptrace`. But you have no idea where that happens. You use that functionality in various places. Is it during the init-phase? During one of the sections? And if so, which one? What are you trying to read? And from where? Or in short: **What is going on?!**

## Shoes off, get some tea: Research time!

Well, Rust has been around for quite some time now and they always boast about how error handling is a first class citizen and all that. So error handling should be a done deal, right? With a canonical way of dealing with errors, officially documented and all that should be right there, correct?

Oh boy, were you wrong.

Turns out, this is a very active field of...mh...experimentation, lets say. There has been [a survey](https://blog.yoshuawuyts.com/error-handling-survey/) recently, listing and quickly describing most the different libraries and ways for error handling that emerged, fallen out of favor, got forked, died anyways, got superseded, fallen out of favor again, etc.
And the opinions seem to change frequently, if you should use `error-chain` or `failure` or `fehler` or `snafu` or `thiserror` or `anyhow` or `eyre` or...

You opened a can of hornets there, or whatever that saying is.

Then you find [this gem](https://blog.rust-lang.org/inside-rust/2020/11/23/What-the-error-handling-project-group-is-working-on.html) and don't know if you should laugh or cry. Almost six years after Rust hit 1.0 an error handling project group is formed. Six. Years. _(heavy breathing)_

Well, okay. At least they are sorting it out now. Problem is, you need...._SIX YEARS? Are you serious?_...ahem, sorry...Problem is, you need helpful error messages now.

After reading a few decent blogs on the topic (like [this](http://www.sheshbabu.com/posts/rust-error-handling/) or [that](https://nick.groenen.me/posts/rust-error-handling/)), there seems to emerge a consensus, at least for libraries: Return something that derives from `std::error::Error`. Either implement them by hand, or use a crate that does it for you, using macro magic. like `thiserror`. Which method you use depends on your level of laziness plus your patience regarding compile times.

## Examples vs. Reality

Another post highlighted [error wrapping](https://doc.rust-lang.org/rust-by-example/error/multiple_error_types/wrap_error.html), a particularly intriguing idea to you.

Unfortunately, all the articles have the understandable, but rather annoying tendency to use very simple example code for illustration purposes. Unrealistically simple, you might even say. They have callstacks of depth 1, return only three kinds of error in total in their API, and their errors are obvious and easily describable (e.g. "Input file XY not found in your 'counting words' program").

You have a more complicated callstack, with tons of different errors and code reuse in different places. For example, the function you think is to blame for the above error is `copy_from_process()`, which calls `ptrace::read()`, which probably returns something like `Failed in ptrace::read: Sys(EIO)`.
This function is used in multiple places in your code, e.g.:

```
├─ init()
│   ├─ read_auxv()
│   │  ├─ open(format!("/proc/{}/auxv", self.pid))
│   │  └─ some_parsing()
│   ├─ ...
│   ├─ enumerate_mappings()
│   │  ├─ open(format!("/proc/{}/maps", self.pid))
│   │  └─ some_parsing()
│   │
│   └─ some_more_checks()
│      └─ copy_from_process()
│
└─ dump()
   │
   ├─ sections::header::write()
   │
   ├─ sections::thread_list_stream()
   │  └─ copy_from_process()
   │
   ├─ sections::mappings::write()
   │  └─ elf_identifier_for_mapping()
   │     └─ copy_from_process()
   │
   ├─ sections::app_memory::write()
   │  └─ copy_from_process()
   │
   └─ ...
```

Same goes for opening files that happens in multiple places (two examples of which are shown in `init()`), so getting `FileNotFound` without context is going to be equally fun, and so on.

# Wrapping up (your errors)

Wrapping errors still sounds like a nice idea, but one layer alone is not going to ~~wrap it~~ cut it.
Going with `copy_from_process()` as an example, you see a few possibilities:
1. Wrapping the `ptrace` error into an `CopyFromProcessError`, but that gives you nothing (except maybe some context, if you add some)
2. With `InitError`s and `DumpingError`s that wrap the `ptrace` errors, you will still not know which section failed and why, but know if it was during `init()` or not.

You might add context to option 2 as well (see below on how), but each section has a variety of reasons why it could fail. Some unique to the section, some shared among a few, some among all of them.

Complex problems sometimes require complex solutions, maybe?

## Inc _Err()_ ption

Using `thiserror` and the fabulous `#[from]` macro, you quickly define a plethora of errors and wrappers, starting from the deepest, darkest places in your callstack, wrapping your way up:

```rust
#[derive(Debug, Error)]
pub enum PtraceDumperError {
    #[error("nix::ptrace() error")]
    PtraceError(#[from] nix::Error),
    ...
}

#[derive(Debug, Error)]
pub enum SectionAppMemoryError {
    #[error("Failed to copy memory from process")]
    CopyFromProcessError(#[from] DumperError),
    ...
}

#[derive(Debug, Error)]
pub enum DumpError {
    #[error("Error during init phase")]
    InitError(#[from] InitError),
    #[error(transparent)]
    PtraceDumperError(#[from] PtraceDumperError),
    #[error("Failed when writing section AppMemory")]
    SectionAppMemoryError(#[from] SectionAppMemoryError),
    ...
```

The fun part is: You have to touch very little of your existing code, thanks to the automatic conversion from one error to the other, conveniently provided by `#[from]`:
```diff
- pub fn init(&mut self) -> Result<()> {
+ pub fn init(&mut self) -> Result<(), InitError> {
     self.read_auxv()?;
     self.enumerate_threads()?;
     self.enumerate_mappings()?;
     Ok(())
 }
```

or

```diff
- pub fn get_stack_info(&self, int_stack_pointer: usize) -> Result<(usize, usize)> {
+ pub fn get_stack_info(&self, int_stack_pointer: usize) -> Result<(usize, usize), DumperError> {
 // snip

    let mapping = self
        .find_mapping(stack_pointer)
-        .ok_or("No mapping for stack pointer found")?;
+        .ok_or(DumperError::NoStackPointerMapping)?;
    let offset = stack_pointer - mapping.start_address;
    let distance_to_end = mapping.size - offset;
  // snip
```

If you run your test binary again, you now get
```
Failed when writing section AppMemory
```
which is...._(Throws a stack of papers from the desk)_...short. Too short, and not that much more helpful, actually. Well, you know which section is failing. Thats good. But where are all the nice error messages you specified in your errors?

Hm, you do only use `println!("{}", error);`. Maybe `{:?}` is better?
```
SectionAppMemoryError(CopyFromProcessError(PtraceError(Sys(EIO))))
```

Aha! Now you are getting somewhere! Tiny, tiny, painfully **tiny** steps, but you are getting somewhere! No error texts, but at least a chain!

Normal printing doesn't seem to recursively go through all the wrapped errors, but stop at the top most. For this, you need to either go through all the errors yourself by hand, or use a crate that does this for you. There are a number of them that provide this, but `anyhow` will do (its by the same author as `thiserror`, so interoperability shouldn't be an issue).

```rust
    println!("{:#}", anyhow::Error::new(error));
```

aaaaand:

```
Failed when writing section AppMemory: Failed to copy memory from process: nix::ptrace() error: EIO: I/O error
```

_Collects papers from the floor. Throws them into the air in jubilation_

## Out of context?

With that sweet, sweet error chain and the resulting error message, you now know where everything goes wrong. Not really _why_, though. You lack context. Luckily, you are not out of context (Note to yourself: You need a drum set for acoustically emphasizing your puns).

Adding more context is rather easy. Instead of using `#[from]`, you use `#[source]`, which will not implement an automatic conversion function anymore, but keep the error chain intact:

```rust
#[derive(Debug, Error)]
pub enum PtraceDumperError {
    #[error("Copy from process {0} failed (source: 0x{1:x}, offset: {2}, length: {3})")]
    PtraceError(Pid, usize, usize, usize, #[source] nix::Error),
```

This can be done on all layers of your error chain. For this example, we only do it at the last link.
No matter where you do it, you have to map your error now (`eyre` could do this a bit more ergonomically, but you want to keep your dependency list small):
```
let word = ptrace::read(pid, (src + idx) as *mut c_void)
    .map_err(|e| PtraceError(child, src, idx, num_of_bytes, e))?;
```

leading to

```
Failed when writing section AppMemory: Failed to copy memory from process: Copy from process 12111 failed (source: 0x0, offset: 0, length: 4096): EIO: I/O error
```

You handed in a `nullptr`. Why would you do that? Why?

To get an easy example error case for this article, thats why!

## Human or machine?

Well, your error messages, including lots of context, is now a well of information for humans. Its perfect for logging (for example to put into a JSON-file accompanying your minidump). Is it any good to use programmatically by application developers, though?

You think it could be. You envision scenarios like "Can't find file XY, because of not yet mounted filesystem" or the like. These can easily be matched for in the application code by using the last link in your error-chain, which `anyhow` provides with the `root_cause()` function.
Or you could match only the top most error and redoing the minidump-section on your own, if the library code fails in it.

Only time will tell.
