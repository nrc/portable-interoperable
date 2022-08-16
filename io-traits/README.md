# Design of async IO traits

`Read`, `Write`, `Seek`, `BufRead`, ...

See also [issue 5](https://github.com/nrc/portable-interoperable/issues/5).

## Blog posts

* [Async Read and Write traits](https://ncameron.org/blog/async-read-and-write-traits/)
* [Async IO with completion-model IO systems](https://ncameron.org/blog/async-io-with-completion-model-io-systems/)

## Requirements

Not in priority order.

* The traits should be ergonomic to implement and to use
  - The async traits should be similar to the sync traits, ideally, the two sets of traits should be symmetric. Every concept from the sync versions should be expressible in the async versions.
* The traits support the same variations as the sync traits
  - The traits must support vectored reads and writes
  - The traits must support reading into uninitialized memory
* Generic usage
  - The traits must support concurrent reading and writing of a single resource
  - The traits should work well as trait objects
* The traits must work performantly with both readiness- (e.g., epoll) and completion-based systems (e.g., io_uring, IOCP)
  - When working with completion-based systems, the traits should support zero-copy reads and writes (i.e., the OS can read data directly into the user's buffer)
  - When working with readiness-based systems, the traits should not require access to buffers until IO is ready
* The traits should permit maximum flexibility of buffers
  - buffers should not be constrained to a single concrete type.
  - We should support buffers that are allocated on the stack or owned by other data structures
* The traits should work in no_std scenarios

## Context: completion and readiness

For efficient performance and memory usage, we must understand how async IO is performed at the OS level. There are multiple mechanisms across different OSes, but they can be broadly categorised into following either a readiness or completion model.

The obvious starting place for the async IO traits is to simply add `async` to each method which performs IO. E.g., `fn read(&mut self, buf: &mut [u8]) -> Result<usize>` becomes `async fn read(&mut self, buf: &mut [u8]) -> Result<usize>`. This is very easy for the user because it abstracts all the details of when the task waits for the OS and how the OS communicates status of the operation to the IO library. However, this abstraction has some costs...

### Readiness

Readiness-model IO is currently the most well-supported in async Rust. It is the model used by epoll (Linux) among others.

From the perspective of the IO library, readiness IO has the following steps (I use read as an example, other operations are similar):

* The library initiates the read.
* The OS immediately returns and the library can schedule other work.
* Later, the OS notifies the library that the read is ready.
* The library passes a buffer to the OS to read into. The OS reads into the buffer and returns the bytes read, or returns an error. This step will never block.
* The library may need to retry if the read failed, in particular if the OS gave an `EWOULDBLOCK` error indicating that no data was ready to read (i.e., the ready notification was a false positive).

A strong advantage of this model is that the OS does not keep a buffer while waiting for IO to complete. That means a buffer can be allocated just in time to be used, or can be shared by multiple tasks (or the user can use many other memory handling strategies). This is important in the implementation of network servers where there may be many concurrent connections, the wait for IO can be long (since the wait is for a remote client), and the buffer must be fairly large since the size of the read is not known in advance.

To implement readiness IO using the naive async read method described above, `read` takes the buffer reference. The IO library initiates the read with the OS and then waits to be scheduled; it must hold the buffer reference during this time. When the OS is ready, the library passes the buffer reference to the OS to read into and will retry if necessary. Finally it returns to the caller of `read`. This makes for an attractively simple API - the user does not need to be concerned with readiness notifications or retries, etc., however, there is no opportunity for the user to pass in the buffer just in time. I.e., it must be pre-allocated, which loses the primary advantage of the readiness model.

### Completion

Completion-model IO is less well supported in the async Rust ecosystem, though the rise of io_uring is changing that (e.g., [Glommio](https://github.com/DataDog/glommio) and [Tokio-uring](https://github.com/tokio-rs/tokio-uring)). It is the model used by IOCP (Windows) and io_uring (Linux).

From the perspective of the IO library, completion IO has the following steps (again, I use read as an example):

* The library initiates the read and passes a buffer to read into.
* The OS returns and the library can schedule other work.
* Later, the OS reads directly into the buffer. When reading is complete (or if there is an error), it notifies the user process.

In terms of sequencing, this is much closer to the naive read method given above. The advantage of this model is that the OS can read directly into the user's buffer without using an intermediate, internal buffer. Furthermore, the user can pass a reference into a target data structure. So, the IO can be zero copy: data is read directly from a device into its final destination.

Unfortunately, there is a problem mapping completion to the naive Rust method too: cancellation. If the user wants to cancel the read, then it can send a cancellation message to the OS. This message is also async, it returns immediately but completes some time later when the OS will notify the user process that the IO was cancelled (or that there was an error). It is possible that an IO completes before the cancellation is processed.

Now consider the lifetime of the buffer. The user process passes a reference to a buffer to the OS. The user process must keep the buffer alive (and must ensure nothing is written to the buffer) until the OS is done with it, i.e., either the IO completes or the IO is cancelled and the cancellation completes. Note that even if the IO is cancelled the buffer must be kept alive until either the IO or cancellation completes, it cannot be destroyed immediately.

This is problematic in the Rust async model. In Rust, a future can be cancelled at any time (cancellation is not blockable on progress of any underlying operation). When a future is cancelled, the buffer passed to the IO library in `read` may be destroyed. If the buffer has been passed to the OS, that violates the required guarantee that the buffer is preserved until the IO completes. Even if the IO is cancelled, the cancellation completes from Rust's perspective before it completes from the OS's perspective. I.e., cancellation is unsound.

Solutions will be explored below, but if we must fit the naive `read` signature, then any solution must require the library to own buffers used for IO and then to copy the contents of its buffer into the one provided by the user. Obviously, this loses the zero-copy advantage of completion IO.



## Read and Write proposal

```rust
pub use std::io::Result;

pub trait Ready {
    async fn ready(&mut self, interest: Interest) -> Result<Readiness>;
}

pub trait Read: Ready {
    fn non_blocking_read_buf(&mut self, buf: &mut ReadBuf<'_>) -> Result<NonBlocking<()>>;
    fn non_blocking_read_buf_vectored(&mut self, bufs: &mut ReadBufVec<'_>) -> Result<NonBlocking<usize>> { ... }

    async fn read(&mut self, buf: &mut [u8]) -> Result<usize> { ... }
    async fn read_buf(&mut self, buf: &mut ReadBuf<'_>) -> Result<())> { ... }
    async fn read_exact(&mut self, buf: &mut [u8]) -> Result<()> { ... }
    async fn read_buf_exact(&mut self, buf: &mut ReadBuf<'_>) -> Result<()> { ... }
    async fn read_buf_vectored(&mut self, bufs: &mut ReadBufVec<'_>) -> Result<usize> { ... }
    async fn read_to_end(&mut self, buf: &mut Vec<u8>) -> Result<usize> { ... }
    async fn read_to_string(&mut self, buf: &mut String) -> Result<usize> { ... }

    fn is_read_vectored(&self) -> bool { ... }

    fn by_ref(&mut self) -> &mut Self
    where
        Self: Sized,
    { ... }
}

pub trait Write: Ready {
    fn non_blocking_write(&mut self, buf: &[u8]) -> Result<NonBlocking<usize>>;
    fn non_blocking_write_vectored(&mut self, bufs: &[IoSlice<'_>]) -> Result<NonBlocking<usize>> { ... }
    fn non_blocking_flush(&mut self) -> Result<NonBlocking<()>>;

    async fn write(&mut self, buf: &[u8]) -> Result<usize> { ... }
    async fn write_vectored(&mut self, bufs: &mut [IoSlice<'_>]) -> Result<usize> { ... }
    async fn write_all(&mut self, buf: &[u8]) -> Result<()> { ... }
    async fn write_all_vectored(&mut self, bufs: &mut [IoSlice<'_>]) -> Result<()> { ... }
    async fn write_fmt(&mut self, fmt: Arguments<'_>) -> Result<()> { ... }
    async fn flush(&mut self) -> Result<()> { ... }

    fn is_write_vectored(&self) -> bool { ... }

    fn by_ref(&mut self) -> &mut Self
    where
        Self: Sized,
    { ... }
}

/// Express which notifications the user is interested in receiving.
#[derive(Copy, Clone)]
pub struct Interest(u32);

/// Describes which operations are ready for an IO resource.
#[derive(Copy, Clone)]
pub struct Readiness(u32);

impl Interest {
    pub const READ = ...;
    pub const WRITE = ...;
    pub const READ_WRITE = Interest(Interest::Read.0 | Interest::Write.0);
}

impl Debug for Interest { ... }

impl Readiness {
    /// The resource is ready to read from.
    pub fn read(self) -> bool { ... }

    ///  The resource is ready to write to.
    pub fn write(self) -> bool { ... }

    /// The resource has hung up.
    ///
    /// Note there may still be data to read.
    /// Note that the user does not *need* to check this method, even if the resource has hung up,
    /// the behaviour of `non_blocking_read` and `non_blocking_write` is defined and they should not
    /// panic.
    /// Note that the user does not need to request an interest in hup notifications, they may always
    /// be returned
    pub fn hup(self) -> bool { ... }
}

impl Debug for Readiness { ... }

/// Whether an IO operation is ready for reading/writing or would block.
#[derive(Copy, Clone, Debug, ...)]
enum NonBlocking<T> {
    Ready(T),
    WouldBlock,
}

/// A helper macro for handling the 'would block' case returned by `Ready::ready`.
macro_rules! continue_on_block {
    ($e: expr) => {
        match $e {
            NonBlocking::Ready(t) => t,
            NonBlocking::WouldBlock => continue,
        }
    }
}
```

I haven't included methods for iterating readers (`bytes`, `chain`, `take`), since work on async iterators is ongoing. I leave these for future work and do not believe they should block implementation or stabilisation.

I've included methods for vectored reads into uninitialized memory (`read_buf_vectored`, etc.). These are not yet in sync `Read` trait but have been accepted as an RFC and are expected to be added soon.

Some rationale:

* The non-blocking read functions are only required for high performance usage and implementation. Therefore, there is no need for variations which take slices of bytes rather than a `ReadBuf` (used for reading into uninitialised memory, see [RFC 2930](https://www.ncameron.org/rfcs/2930)). Converting from a byte slice to a `ReadBuf` is trivial.
* We elide async functions for vectored reads using byte slices - vectored IO is a high-performance option and the overhead of converting to a `ReadBuf` is trivial, so I don't think it is necessary.
* The `read_exact` functions are convenience functions and by their nature it does not make sense to have non-blocking variations (c.f., the async functions) since they require multiple reads (and potentially waits).

Note that we expect implementers to implement the provided non-blocking methods in some cases for optimal performance (e.g., for vectored IO). But expect the provided async methods to almost never be overridden. This will be documented.

### Usage - ergonomic path

For the most ergonomic usage, where having the most optimal memory usage is not important, the async IO traits are used just like their sync counterparts. E.g.,

```rust
use std::async::io::{Read, Result};

async fn read_stream(stream: &TcpStream) -> Result<()> {
    let mut buf = [0; 1024];
    let bytes_read = stream.read(&mut buf).await?;

    // we've read bytes_read bytes
    // do something with buf

    Ok(())
}
```

The important thing here is that asynchronous reading is exactly like synchronous reading, it just requires an await (and similarly for writing).

### Usage - memory-sensitive path

Although closely matching the behaviour of synchronous IO is an important ergonomic goal, async IO does have different semantics around timing. Thus for absolutely optimal performance, it requires different usage patterns. In the above usage example, we allocate a buffer to read into, then call read, then await the result. In the sync world we would block until read returns. However, by using async and await we have a non-blocking wait which allows another task to allocate another buffer and await a different read. If we do this hundreds, thousands, or millions of times (which is what we want to be able to do with async IO), then that's a lot of memory allocated in buffers and not doing anything. Better behaviour would be to await the read and only allocate the buffer once the data is ready to be read. In this way we don't have a bunch of memory just sat around. In fact we could then just use a single buffer rather than being forced to allocate new ones!

This usage pattern is facilitated by the `Ready` trait and the `non_blocking_` methods. For this usage, reading might look like:

```rust
use std::async::io::{continue_on_block, Interest, Read, Ready, Result};
use std::io::ErrorKind;

async fn read_stream(stream: &TcpStream) -> Result<()> {
    loop {
        stream.ready(Interest::READABLE).await?;

        let mut buf = vec![0; 1024];
        let bytes_read = continue_on_block!(stream.non_blocking_read(&mut buf)?);

        // we've read bytes_read bytes
        // do something with buf

        return Ok(());
    }

    Ok(())
}
```

This is less ergonomic than the previous example, but it is more memory-efficient.

### Implementations

Implementers of the IO traits must implement `async::Ready` and either or both of `async::{Read, Write}`. They only need to implement the synchronous `non_blocking` method, the async method is provided (the default implementation follows the memory-sensitive usage example above).

Where possible these traits should be implemented on reference types as well as value types (following the synchronous traits). E.g., all three traits should be implemented for both `TcpStream` and `&TcpStream`. This permits simultaneous reads and writes even when using the ergonomic path (simultaneous read/write is possible in the memory-optimal path without such implementations).

Such implementations are not possible today in some runtimes due details of their waiting implementation. They would be ok with the traits as proposed, since there is no explicit polling.

There is a bit of a foot-gun here because reference implementations make it possible to have multiple simultaneous readers or writers. However, that is somewhat difficult to do unintentionally, is already possible with the sync traits, and is possible even without the reference impls by cloning the underlying handles.

The readiness mechanism using the `Ready` trait is extensible to other readiness notifications in the future. However, this is somewhat complicated since these notifications are nearly all platform-specific. (HUP is not, but requires special handling).


## `BufRead` proposal

An async `BufRead` trait has two purposes: to support reads with transparent buffering functionality, as in the sync version, and to support completion-model IO where the library manages the buffers. In the former case, an async reader might be wrapped in a `BufReader` (see note below). In the latter case, the IO resource would likely implement `async::BufRead` directly.

```rust
pub trait BufRead: Read {
    async fn fill_buf(&mut self) -> Result<&[u8]>;
    fn consume(&mut self, amt: usize);

    async fn read_until(&mut self, byte: u8, buf: &mut Vec<u8>) -> Result<usize> { ... }
    async fn read_line(&mut self, buf: &mut String) -> Result<usize> { ... }
    #[unstable]
    async fn has_data_left(&mut self) -> Result<bool> { ... }
}
```

* `fill_buf` and `consume` are required methods. `consume` does not need to be async since it is just about buffer management, no IO will take place.
* `has_data_left` must be async since it might fill the buffer to answer the question (it is currently unstable in `BufRead` which is why I've added that annotation).
* I've elided `split` and `lines` methods since these are async iterators and there are still open questions there. I assume we will add these later. Besides the questions about async iterators, I don't think there is anything too interesting about these methods.

TODO BufWrite

### ` BufReader`

`BufReader` is a concrete type, it's a utility type for converting objects which implement `Read` into objects which implement `BufRead`. I.e., it encapsulates a reader with a buffer to make a reader with an internal buffer. `BufReader` provides its own buffer and does not let the user customise it.

I think that we don't need a separate `async::BufReader` type, but rather we need to duplicate the `impl<R: Read> BufReader<R>` impl for `R: async::Read` and to implement `async::BufRead` where `R: async::Read` (this might be an area where async overloading is useful).

TODO There's also the question of `seek_relative`.

## Owned read

An extension we should consider is permitting reads into owned (rather than borrowed) buffers. We could add either an `OwnedRead` trait or an `owned_read` function to `async::Read`. We would want to do this as well as supporting `async::BufRead`, since the former supports explicitly internal buffers. Owned read is useful for completion-based systems where the user manages the buffers rather than the library or resource. (We can't use borrowed slices due to cancellation).

Design with new trait:

```rust
trait OwnedRead {
    async fn read<T: OwnedBuf>(&mut self, buffer: T) -> Result<T>;
}

trait OwnedBuf {
    // ...
}

// An implementation using the initialised part of a Vector.
impl OwnedBuf for Vec<u8> {
    // ...
}
```

For a possible (WIP) design of `OwnedBuf`, see [nrc/read-buf/../owned.rs](https://github.com/nrc/read-buf/blob/main/src/owned.rs).

TODO OwnedWrite

The alternative is to add `read` as `owned_read` or `moving_read` or similar to `async::Read`.

In either case, we may want to add non-async versions, both for symmetry with async, and because the concept might be generally useful (question: are there non-async use cases?).

### Alternative: an abstract owning pointer type

If the language had an abstract owning pointer type, we could use that rather than the `OwnedReadBuf` trait (using a strawman `~` syntax):

```rust
trait OwnedRead {
    async fn read(&mut self, buffer: ~[u8]) -> Result<~[u8]>;
}
```

In the same way that `&` lets us abstract any borrowing pointer type, `~` would let us abstract any owning pointer type. `~T` is a pointer to a value of `T` with move semantics and which on destruction calls the destructor for the underlying value and the pointer. Any owning pointer (e.g., `Box<T>`, `Arc<T>`, etc.) can be converted into a `~T`. A `~T` can be downcast into any concrete owning pointer with a runtime check. Similarly, an owning collection can be converted into an owned slice, e.g., `Vec<T>` to `~[T]`. An owning slice can be shrunk like a borrowed slice. We would probably need some mechanism to restore an owned slice. E.g., if a `Vec<T>` is converted to a `~[T]` and then shrunk, we would need some way to restore the original `Vec`.

This idea is somewhat similar to some ideas around `dyn*` pointers. It is different to previous `&move` proposals in that `&move` does not call the destructor of the underlying storage on its destruction and thus must be constrained by the lifetime of the storage.

The advantage of this approach is that it works better with trait objects since there is no generic parameter in the `read` method. The disadvantage is that it requires a significant language change.

## Seek

```rust
pub trait Seek {
    async fn seek(&mut self, pos: SeekFrom) -> Result<u64>;

    async fn rewind(&mut self) -> Result<()> { ... }
    async fn stream_len(&mut self) -> Result<u64> { ... }
    async fn stream_position(&mut self) -> Result<u64> { ... }
}
```

The async `Seek` trait is a simple `async`-ification of the sync trait.

The `Ready` trait could be extended to support seeking, but I don't think that is necessary. Seek is only useful with buffered readers, files, and similar. In these cases, the memory advantages of using `Ready` are diminished.

There was some discussion about `Seek` in Tokio. One of the key sticking points which led to their `start_seek`/`seek_complete` API was that a future should not have any observable side effects until it is ready, and `poll_seek` method does not satisfy that invariant (since the state of a file might be changed by a seek that did not complete before the seek was cancelled). I believe this is not an issue for async methods, since there can be no assumption of side-effect freedom because polling is encapsulated.

### Extension: `read_at`/`write_at`

`read_at`/`write_at` is arguably a better API than using `seek` and read/write, especially in async programming, because the operation is atomic and therefore not susceptible to race condition errors. However, we should still have an `async::Seek` trait for symmetry with the sync trait, so `read_at`/`write_at` is an extension rather than an alternative.


## Alternatives

There are several alternatives or tweaks possible to the design proposed above. The first few are presented in separate files and I consider them feasible (though inferior to the above proposal), the later few are clearly sub-optimal and I haven't described them in depth.

### Async traits

[async-traits.md](alternatives/async-traits.md)

### A Split trait

[split-trait.md](alternatives/split-trait.md)

### Tweak: make vectored IO a separate trait

[tweaks.md](alternatives/tweaks.md)

### Tweak: only provide ReadBuf methods

[tweaks.md](alternatives/tweaks.md)

### Tweak: no impls on reference types

[tweaks.md](alternatives/tweaks.md)

### Polling read/write methods

We could continue to use `poll_read` and `poll_write` instead of the `non_blocking_` methods. This would allow using trait objects without allocating and can support simultaneous read/write. However, this is much less ergonomic than this proposal and doesn't permit impls on reference types.

For the sake of having some code to look at, here is the current `Read` trait from futures.rs:

```rust
pub trait Read {
    fn poll_read(
        self: Pin<&mut Self>, 
        cx: &mut Context<'_>, 
        buf: &mut [u8],
    ) -> Poll<Result<usize, Error>>;

    fn poll_read_vectored(
        self: Pin<&mut Self>, 
        cx: &mut Context<'_>, 
        bufs: &mut [IoSliceMut<'_>],
    ) -> Poll<Result<usize, Error>> { ... }
}
```

Given how un-ergonomic this approach is, I don't think it is worth pursuing unless other approaches turn out to be dead ends. (We could add the async methods as provided methods to make this approach more ergonomic for users, however, it is still less ergonomic for implementers and there is no real benefit other than backwards compatibility).

### Elide the `non_blocking_` methods

In this alternative, we'd keep the `Ready` trait, but the `Read` and `Write` traits would only have the async methods, not the `non_blocking_` ones. In the common case, the async read would return immediately and the user would not need to handle 'would block' errors. However, since in some cases the user would need to wait for the method to return, one could not share a single buffer between all reads on a thread. Furthermore, these functions couldn't be called from polling functions, or other non-async contexts.

An alternative alternative would be to use the synchronous version of the `Read::read`, rather than `async::Read::non_blocking_read`. That has the right async-ness, but would need to handle 'would block' errors differently, since we would lose asynchronous-ness if we blocked on the `read` call. I don't think that can be done at the moment. It's possible we could do that with some form of async overloading, if we could have sync, async, and non-blocking versions of the same method (though note that that is an extension to the usual proposal for async overloading).

### `read_ready` and `write_ready` methods rather than a `Ready` trait

This would lead to fewer traits and therefore a simpler API. However, there would be more methods overall (which would lead to code duplication for implementers). The mechanism would not be extensible to other readiness notifications, and it means that a single thread can't concurrently wait for both read and write readiness.

### Make `Ready` optional, rather than a super-trait

This lets implementers choose if they want to support the memory-optimal path or just the ergonomic path. However, it also means that in generic code there is no way to use the memory-optimal path unless it is explicitly declared (i.e., changing the implementation becomes a breaking change, or code is reliant on intermediate generic code to require the `Ready` bound as well as `Read` or `Write`).

### Offer only `BufRead`, and not `Read` (likewise for `Write`)

It might be possible to reduce the set of IO traits to only the buffered varieties by using a sophisticated buffer or buffer manager type to abstract over the various ways in which buffers can be managed and passed to read methods. By having the buffer manager supply the buffer, there is no need to wait on readiness and instead the buffer manager creates or provides the buffer when the reader is ready.

The approach would work well with completion based IO as well as readiness. However, this approach adds some complexity for the simple cases of read, would be a radical departure from the existing sync traits, and it's unclear if the lifetimes can be made to work in all cases.

### Add non-async version of readiness support

Although using the readiness API is more strongly motivated in async code, there is no reason it can't be used in non-async code. We might consider adding support for explicit readiness support to std. This would increase the symmetry between sync and async libraries at the expense of increasing the surface are of std.

## Requirements discussion

TODO evaluate Seek
TODO evalute BufRead/OwnedRead, evaluate against requirements (in particular zero-copy support)

### The traits should be ergonomic to implement and to use

The ergonomic usage of the primary proposal and the simple async traits alternative are the most straightforward to use and equally ergonomic. The split trait and polling alternatives are more complex and less ergonomic. Implementing the IO traits in the primary proposal is more complex than in the simple async traits alternative, but that affects fewer users (and users who tend to be more sophisticated) and is not too bad - there are not too many methods to implement, and those methods have an easily understood purpose which should map well to the OS API.

In terms of symmetry, the simple async traits alternative is optimal. The primary proposal is good for users who use the ergonomic path, and less good for implementers. The traits are a strict superset of their sync equivalents.

I believe that putting the vectored methods and methods for reading into uninit memory makes things simpler and thus more ergonomic, however, it makes the traits less symmetric with the sync versions.

All the other tweaks or minor alternatives make the primary proposal more complex or less symmetric.

### The traits support the same variations as the sync traits

All alternatives support vectored reads and writes and reading into uninitialized memory, in a similar way to the sync traits. There is nothing specific to any alternative which makes this better or worse.

### Generic usage

The primary proposal supports concurrent reading and writing by implementing `Read` and `Write` for reference types in the same manner as for the sync traits, or by using the explicit ready loop. There is a tweak which removes the first method, concurrent reads and writes would still be possible via the second method, but less ergonomically.

Other alternatives support concurrent reads and writes similarly, or by polling or using a 'split' trait.

Sometimes it is necessary to permit concurrent reads and writes in generic code. In this case, the reference impls solution does not work: it requires `T: Read + Write` and we can't easily require reference impls for concurrent reads and writes. The primary proposal would work in this case via the explicit ready loop. The `Split` trait also works here, using `T: Split` rather than `T: Read + Write`.

The traits are usable as trait objects in all variations, assuming that we can support async trait objects in the language.

### The traits must work performantly with both readiness- (e.g., epoll) and completion-based systems (e.g., io_uring, IOCP)

For readiness-based systems, only the primary proposal (and its variations) do not require access to buffers until IO is ready.

For completion-based systems, none of the proposals or alternatives work optimally. They all require copying the buffer into the buffer provided by the user.

I believe that to optimally use compeltion-based IO, we must use alternative traits, such as `BufRead`, these are discussed below.

The high-level view here is that using `Read`/`Write` will work for any platform, but that for optimal performance the user must choose the buffered or unbuffered traits depending on the platform, not just the constraints of the application. That seems acceptable since for optimal performance, one must always take account of the characteristics of the underlying platform. However, it means the libraries which want to offer excellent performance on all platforms cannot treat buffering as orthogonal and must provide versions of their API using both `Read` and `BufRead` (respectively for `Write`) traits.

### The traits should permit maximum flexibility of buffers

All proposals, except for only having a `BufRead` trait, take a borrowed slice as a buffer. This is optimally flexible in most cases. All proposals support the vectored and uninintialised buffers also supported by sync `Read`, so here too we are optimally flexible (we may of course need to add support in the future for other buffers, but there are no known cases at the moment).

### The traits should work in no_std scenarios

The major blocker here is that the `io::Error` type relies on the `Error` trait. There is work underway to move the `Error` trait out of std, at which point the IO traits can follow.

None of the alternatives present any further difficulty in working with no_std crates.
