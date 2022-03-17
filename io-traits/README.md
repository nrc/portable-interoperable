# Design of async IO traits

Read, Write, Seek, BufRead, ...

See also [issue 5](https://github.com/nrc/portable-interoperable/issues/5).

## Blog posts

* [Async Read and Write traits](https://ncameron.org/blog/async-read-and-write-traits/)
* [Async IO with completion-model IO systems](https://ncameron.org/blog/async-io-with-completion-model-io-systems/)

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
