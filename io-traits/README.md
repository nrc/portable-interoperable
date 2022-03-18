# Design of async IO traits

`Read`, `Write`, `Seek`, `BufRead`, ...

See also [issue 5](https://github.com/nrc/portable-interoperable/issues/5).

## Blog posts

* [Async Read and Write traits](https://ncameron.org/blog/async-read-and-write-traits/)
* [Async IO with completion-model IO systems](https://ncameron.org/blog/async-io-with-completion-model-io-systems/)

## Requirements

* The traits should be ergonomic to implement and to use
  - The async traits should be similar to the sync traits, ideally, the two sets of traits should be symmetric. Every concept from the sync versions should be expressible in the async versions.
* The traits must support vectored reads and writes
* The traits must support reading into uninitialized memory
* The traits must support concurrent reading and writing of a single resource
* The traits should work well as trait objects
* The traits should work in no_std scenarios
* The traits must work well both readiness (e.g., epoll) and completion-based systems (e.g., io_uring, IOCP)
* When working with completion-based systems, the traits should support zero-copy reads and writes
* When working with readiness-based systems, the traits should not require access to buffers until IO is ready
* The traits should permit maximum flexibility of buffers (i.e., buffers should not be constrained to a single concrete type. We should support buffers that are allocated on the stack or owned by other data structures)

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

        let mut buf = [0; 1024];
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

### Requirements discussion

#### Ergonomics

For many users, this proposal will be optimally ergonomic to use. Implementing the IO traits is not ideally ergonomic, but that affects fewer users (and users who tend to be more sophisticated) and is not too bad - there are not too many methods to implement, and those methods have an easily understood purpose which should map well to the OS API. Likewise, for those users who need optimal control or performance, the ergonomics are not great, but such users must expect to be exposed to some details. We can also offer some good helpers to make implementation easier.

I think there is an intrinsic trade-off between wanting to be close to the sync versions for ergonomics and wanting to be close to the underlying IO model for performance and control. I think this proposal is a good compromise point on that trade-off.

#### Vectored reads and writes, and reading into uninitialized memory

These goals are satisfied by the presence of the respective methods and in the same way as for the sync equivalent traits in std.

#### Concurrent reading and writing of a single resource

This works in both API paths (in most cases). In the memory-optimal path because `Ready::ready` can wait on readiness for both reading and writing at the same time. In the ergonomic path, for most resources at least, due to implementations of `Read` and `Write` on reference types.

#### Work well as trait objects

Assuming we get async functions in trait objects working well, the proposed traits work optimally as trait objects. With current plans, that will require some boxing, but that should only be one allocation per read, and only necessary if using trait objects.

#### Work well with completion based systems

Async IO systems which are completion-based, such as IOCP and io_uring will work with this proposal, but will not perform optimally. In particular, the implementation must copy a buffer on every read or write. The optimal way to use completion-based IO is to use the async version of the `BufRead` trait (see below).

The high-level view here is that using `Read`/`Write` will work for any platform, but that for optimal performance the user must choose the buffered or unbuffered traits depending on the platform, not just the constraints of the application. That seems acceptable since for optimal performance, one must always take account of the characteristics of the underlying platform. However, it means the libraries which want to offer excellent performance on all platforms cannot treat buffering as orthogonal and must provide versions of their API using both `Read` and `BufRead` (respectively for `Write`) traits.


TODO other requirements

### Alternatives

There are several alternatives or tweaks possible to the design proposed above. The first few are presented in separate files and I consider them feasible (though inferior to the above proposal), the later few are clearly sub-optimal and I haven't described them in depth.

#### Async traits

[async-traits.md](alternatives/async-traits.md)

#### A Split trait

[split-trait.md](alternatives/split-trait.md)

#### Tweak: make vectored IO a separate trait

[tweaks.md](alternatives/tweaks.md)

#### Tweak: only provide ReadBuf methods

[tweaks.md](alternatives/tweaks.md)

#### Tweak: no impls on reference types

[tweaks.md](alternatives/tweaks.md)

#### Polling read/write methods

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

#### Elide the `non_blocking_` methods

In this alternative, we'd keep the `Ready` trait, but the `Read` and `Write` traits would only have the async methods, not the `non_blocking_` ones. In the common case, the async read would return immediately and the user would not need to handle 'would block' errors. However, since in some cases the user would need to wait for the method to return, one could not share a single buffer between all reads on a thread. Furthermore, these functions couldn't be called from polling functions, or other non-async contexts.

#### `read_ready` and `write_ready` methods rather than a `Ready` trait

This would lead to fewer traits and therefore a simpler API. However, there would be more methods overall (which would lead to code duplication for implementers). The mechanism would not be extensible to other readiness notifications, and it means that a single thread can't concurrently wait for both read and write readiness.

#### Make `Ready` optional, rather than a super-trait

This lets implementers choose if they want to support the memory-optimal path or just the ergonomic path. However, it also means that in generic code there is no way to use the memory-optimal path unless it is explicitly declared (i.e., changing the implementation becomes a breaking change, or code is reliant on intermediate generic code to require the `Ready` bound as well as `Read` or `Write`).

#### Offer only `BufRead`, and not `Read` (likewise for `Write`)

It might be possible to reduce the set of IO traits to only the buffered varieties by using a sophisticated buffer or buffer manager type to abstract over the various ways in which buffers can be managed and passed to read methods. By having the buffer manager supply the buffer, there is no need to wait on readiness and instead the buffer manager creates or provides the buffer when the reader is ready.

The approach would work well with completion based IO as well as readiness. However, this approach adds some complexity for the simple cases of read, would be a radical departure from the existing sync traits, and it's unclear if the lifetimes can be made to work in all cases.


## `BufRead` proposal

```rust
trait OwnedRead {
    async fn read(&mut self, buffer: impl OwnedReadBuf) -> Result<OwnedReadBuf>;
}

trait OwnedReadBuf {
    fn as_mut_slice(&mut self) -> *mut [u8];
    unsafe fn assume_init(&mut self, size: usize);
}

// An implementation using the initialised part of a Vector.
impl OwnedReadBuf for Vec<u8> {
    fn as_mut_slice(&mut self) -> *mut [u8] {
        &mut **self
    }

    unsafe fn assume_init(&mut self, size: usize) {
        self.truncate(size);
    }
}
```

## Owned read

See [blog post](https://ncameron.org/blog/async-io-with-completion-model-io-systems/).

We could add either an `OwnedRead` trait or an `owned_read` function to `async::Read`.

Strawman design:

```rust
trait OwnedRead {
    async fn read(&mut self, buffer: impl OwnedReadBuf) -> Result<OwnedReadBuf>;
}

trait OwnedReadBuf {
    fn as_mut_slice(&mut self) -> *mut [u8];
    unsafe fn assume_init(&mut self, size: usize);
}

// An implementation using the initialised part of a Vector.
impl OwnedReadBuf for Vec<u8> {
    fn as_mut_slice(&mut self) -> *mut [u8] {
        &mut **self
    }

    unsafe fn assume_init(&mut self, size: usize) {
        self.truncate(size);
    }
}
```

Or add `read` as `owned_read` or `moving_read` or something to `async::Read`.