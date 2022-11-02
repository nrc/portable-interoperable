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
  - The traits should work well as trait objects (we're assuming the `dyn*` work will progress as expected)
* The traits must work performantly with both readiness- (e.g., epoll) and completion-based systems (e.g., io_uring, IOCP)
  - When working with completion-based systems, the traits should support zero-copy reads and writes (i.e., the OS can read data directly into the user's buffer)
  - When working with readiness-based systems, the traits should not require access to buffers until IO is ready
* The traits should permit maximum flexibility of buffers
  - buffers should not be constrained to a single concrete type (i.e., users can use there own buffer types with the IO traits and are not constrained to using a single concrete type such as `Vec<u8>`).
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

## `Read` proposal

This proposal consists of straightforward async `Read` (and `Write`) traits which closely follow their non-async counterparts. We also have sub-traits which are specialized for readiness and completion IO: `ReadinessRead` and `OwnedRead`. There are 'downcasting' methods on `Read` to facilitate converting to these traits where possible. I expect that we would stabilise the `Read` trait well in advance of the specialized traits.

The `Read` trait (compare to non-async [`Read`](https://doc.rust-lang.org/nightly/std/io/trait.Read.html)):

```rust
pub trait Read {
    async fn read(&mut self, buf: &mut [u8]) -> Result<usize>;
    async fn read_vectored(&mut self, bufs: &mut [IoSliceMut<'_>]) -> Result<usize> { ... }
    async fn read_buf(&mut self, buf: &mut BorrowedCursor<'_>) -> Result<()> { ... }
    async fn read_exact(&mut self, buf: &mut [u8]) -> Result<()> { ... }
    async fn read_buf_exact(&mut self, buf: &mut BorrowedCursor<'_>) -> Result<()> { ... }
    async fn read_buf_vectored(&mut self, bufs: &mut BorrowedSliceCursor<'_>) -> Result<usize> { ... }
    async fn read_to_end(&mut self, buf: &mut Vec<u8>) -> Result<usize> { ... }
    async fn read_to_string(&mut self, buf: &mut String) -> Result<usize> { ... }

    fn is_read_vectored(&self) -> bool { false }

    fn by_ref(&mut self) -> &mut Self
    where
        Self: Sized,
    { ... }

    fn bytes(self) -> Bytes<Self>
    where
        Self: Sized,
    { ... }

    fn chain<R: Read>(self, next: R) -> Chain<Self, R>
    where
        Self: Sized,
    { ... }

    fn take(self, limit: u64) -> Take<Self>
    where
        Self: Sized,
    { ... }

    fn as_ready(&mut self) -> Option<&mut impl ReadyRead> {
        None
    }

    fn as_owned(&mut self) -> Option<&mut impl OwnedRead> {
        None
    }

    fn as_ready_dyn(&mut self) -> Option<&mut dyn ReadyRead> {
        None
    }

    fn as_owned_dyn(&mut self) -> Option<&mut dyn OwnedRead> {
        None
    }
}
```

Notes:

* I have included `read_buf_vectored` based on an [open proposal](https://github.com/rust-lang/libs-team/issues/104).
* `bytes`, `chain`, and `take` are not async methods but return types which implement `AsyncIterator` (c.f., `Iterator` in non-async `Read`).
* The `as_*` methods are for downcasting. I have included both RPITIT and trait object versions, I'm not 100% sure that is necessary. These are the only methods without equivalents in non-async `Read`.

The expectation is that nearly all code will use the `Read` trait only. Libraries which provide types which implement `Read`, should also provide implementations for `OwnedRead` and/or `ReadyRead` where possible. Users of these traits which support such methods can try downcasting before falling back to using plain `Read::read`. Adapter types (those which have an inner type which is a generic type bounded by `Read` and implement `Read` themselves) should implement `OwnedRead` and `ReadyRead` where their inner type does.

## Readiness read

This trait is specialized for readiness IO systems such as epoll.

```rust
pub trait Ready {
    // async
    fn ready(&mut self, interest: Interest) -> Result<Readiness>;
}

pub trait ReadyRead: Ready + Read {
    fn non_blocking_read(&mut self, buf: &mut BorrowedCursor<'_>) -> Result<NonBlocking<()>>;
    fn non_blocking_read_vectored(&mut self, bufs: &mut BorrowedSliceCursor<'_>) -> Result<NonBlocking<usize>> { ... }
}

/// Express which notifications the user is interested in receiving.
#[derive(Copy, Clone)]
pub struct Interest(u32);

/// Describes which operations are ready for an IO resource.
#[derive(Copy, Clone)]
pub struct Readiness(u32);

/// Whether an IO operation is ready for reading/writing or would block.
#[derive(Copy, Clone, Debug)]
pub enum NonBlocking<T> {
    Ready(T),
    WouldBlock,
}

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
```

Notes:

* This approach requires a different idiom for reading, see below. That pattern facilitates avoiding memory allocation before data is ready to read.
* I'm unclear if we should have convenience async methods including `read`, `read_vectored`, `read_exact`, or `read_to_end`. My thinking for not doing so is that if you want the performance benefits of `ReadyRead` then you probably also want the performance benefits of reading into uninitialized memory (or at least, the ergonomic hit of using `BorrowedCursor` rather than `&mut [u8]` is easy enough to ignore when that is not true).

## Owned read

`OwnedRead` is a specialized trait useful for completion IO systems, e.g., io_uring or IOCP. It circumvents the cancellation problem by letting the IO library own the buffer for the duration of the read.

```rust
pub trait OwnedRead: Read {
    async fn read(&mut self, buf: OwnedBuf) -> (OwnedBuf, Result<()>);
    async fn read_exact(&mut self, buf: OwnedBuf) -> (OwnedBuf, Result<()>) { ... }
    async fn read_to_end(&mut self, buf: Vec<u8>) -> (Vec<u8>, Result<usize>) { ... }
}
```

Notes:

* Vectored reads are left as future work.
* OwnedBuf permits reads into uninitialized memory, so there is no need for `read_buf` methods.
* We must return the buffer since it is moved into the read methods.
* `read_to_end` takes a `Vec` since it must extend the buffer and `OwnedBuf` is intentionally not extensible.
* FIXME (open question): the methods take an OwnedBuf with the default allocator. If we permit any allocator then we require a generic method and `OwnedRead` cannot be used as a trait object. I'm not sure what is the right solution here. It's possible `dyn*` helps here, then we can take the allocator as a `dyn*` trait object (which for the common case of a zero or pointer-sized allocator, would not have to allocate).

`OwnedBuf` is a new type similar in API to [`BorrowedBuf`](https://doc.rust-lang.org/nightly/std/io/struct.BorrowedBuf.html), but which owns its data. Since the buffer must be owned by the IO library during the read, it is passed as a buf rather than a cursor; the cursor is still used for writing. We use a concrete type rather than a trait to avoid generic methods or trait objects in `OwnedRead`. The idea is that any contiguous, owned buffer can be represented as an `OwnedBuf` and can be converted to or from the original type. There is provided support for easily converting to and from `Vec<u8>` and `Vec<MaybeUninit<u8>>`.

```rust
pub struct OwnedBuf<A: 'static + Allocator = Global> {
    data: *mut MaybeUninit<u8>,
    dtor: &'static dyn Fn(&mut OwnedBuf<A>),
    capacity: usize,
    /// The length of `self.buf` which is known to be filled.
    filled: usize,
    /// The length of `self.buf` which is known to be initialized.
    init: usize,
    allocator: A,
}

impl<A: 'static + Allocator> OwnedBuf<A> {
    #[inline]
    pub fn new(data: *mut MaybeUninit<u8>, dtor: &'static dyn Fn(&mut OwnedBuf), capacity: usize, filled: usize, init: usize) -> OwnedBuf<Global> {
        OwnedBuf::new_in(data, dtor, capacity, filled, init, Global)
    }

    #[inline]
    pub fn new_in(data: *mut MaybeUninit<u8>, dtor: &'static dyn Fn(&mut OwnedBuf<A>), capacity: usize, filled: usize, init: usize, allocator: A) -> OwnedBuf<A> {
        OwnedBuf {
            data,
            dtor,
            capacity,
            filled,
            init,
            allocator,
        }
    }

    /// SAFETY: only safe if self was created from a Vec<u8, A> or Vec<MaybeUninit<u8>, A>.
    #[inline]
    pub unsafe fn into_vec(self) -> Vec<u8, A> {
        let this = ManuallyDrop::new(self);
        Vec::from_raw_parts_in(this.data as *mut u8, this.filled, this.capacity, unsafe { ptr::read(&this.allocator) })
    }

    /// SAFETY: only safe if self was created from a Vec<u8, A> or Vec<MaybeUninit<u8>, A>.
    #[inline]
    pub unsafe fn into_uninit_vec(self) -> Vec<MaybeUninit<u8>, A> {
        let this = ManuallyDrop::new(self);
        Vec::from_raw_parts_in(this.data, this.filled, this.capacity, unsafe { ptr::read(&this.allocator) })
    }

    /// Returns the length of the initialized part of the buffer.
    #[inline]
    pub fn init_len(&self) -> usize {
        self.init
    }

    /// Returns a shared reference to the filled portion of the buffer.
    #[inline]
    pub fn filled(&self) -> &[u8] {
        // SAFETY: We only slice the filled part of the buffer, which is always valid
        unsafe { MaybeUninit::slice_assume_init_ref(slice::from_raw_parts(self.data, self.filled)) }
    }

    /// Returns a cursor over the unfilled part of the buffer.
    #[inline]
    pub fn unfilled(&mut self) -> OwnedCursor<'_, A> {
        OwnedCursor {
            start: self.filled,
            buf: self,
        }
    }

    /// Clears the buffer, resetting the filled region to empty.
    ///
    /// The number of initialized bytes is not changed, and the contents of the buffer are not modified.
    #[inline]
    pub fn clear(&mut self) -> &mut Self {
        self.filled = 0;
        self
    }

    /// Asserts that the first `n` bytes of the buffer are initialized.
    ///
    /// `OwnedBuf` assumes that bytes are never de-initialized, so this method does nothing when called with fewer
    /// bytes than are already known to be initialized.
    ///
    /// # Safety
    ///
    /// The caller must ensure that the first `n` unfilled bytes of the buffer have already been initialized.
    #[inline]
    pub unsafe fn set_init(&mut self, n: usize) -> &mut Self {
        self.init = max(self.init, n);
        self
    }
}

impl<A: 'static + Allocator> Drop for OwnedBuf<A> {
    fn drop(&mut self) {
        (self.dtor)(self)
    }
}

pub struct OwnedCursor<'buf, A: 'static + Allocator> {
    buf: &'buf mut OwnedBuf<A>,
    start: usize,
}

impl<'a, A: 'static + Allocator> OwnedCursor<'a, A> {
    /// Reborrow this cursor by cloning it with a smaller lifetime.
    ///
    /// Since a cursor maintains unique access to its underlying buffer, the borrowed cursor is
    /// not accessible while the new cursor exists.
    #[inline]
    pub fn reborrow<'this>(&'this mut self) -> OwnedCursor<'this, A> {
        OwnedCursor {
            buf: self.buf,
            start: self.start,
        }
    }

    /// Returns the available space in the cursor.
    #[inline]
    pub fn capacity(&self) -> usize {
        self.buf.capacity - self.buf.filled
    }

    /// Returns the number of bytes written to this cursor since it was created from a `BorrowedBuf`.
    ///
    /// Note that if this cursor is a reborrowed clone of another, then the count returned is the
    /// count written via either cursor, not the count since the cursor was reborrowed.
    #[inline]
    pub fn written(&self) -> usize {
        self.buf.filled - self.start
    }

    /// Returns a shared reference to the initialized portion of the cursor.
    #[inline]
    pub fn init_ref(&self) -> &[u8] {
        let filled = self.buf.filled;
        // SAFETY: We only slice the initialized part of the buffer, which is always valid
        unsafe { MaybeUninit::slice_assume_init_ref(&slice::from_raw_parts(self.buf.data, self.buf.init)[filled..]) }
    }

    /// Returns a mutable reference to the initialized portion of the cursor.
    #[inline]
    pub fn init_mut(&mut self) -> &mut [u8] {
        let filled = self.buf.filled;
        let init = self.buf.init;
        // SAFETY: We only slice the initialized part of the buffer, which is always valid
        unsafe {
            MaybeUninit::slice_assume_init_mut(&mut self.buf_as_slice()[filled..init])
        }
    }

    /// Returns a mutable reference to the uninitialized part of the cursor.
    ///
    /// It is safe to uninitialize any of these bytes.
    #[inline]
    pub fn uninit_mut(&mut self) -> &mut [MaybeUninit<u8>] {
        let init = self.buf.init;
        unsafe { &mut self.buf_as_slice()[init..] }
    }

    /// Returns a mutable reference to the whole cursor.
    ///
    /// # Safety
    ///
    /// The caller must not uninitialize any bytes in the initialized portion of the cursor.
    #[inline]
    pub unsafe fn as_mut(&mut self) -> &mut [MaybeUninit<u8>] {
        let filled = self.buf.filled;
        &mut self.buf_as_slice()[filled..]
    }

    #[inline]
    unsafe fn buf_as_slice(&mut self) -> &mut [MaybeUninit<u8>] {
        slice::from_raw_parts_mut(self.buf.data, self.buf.capacity)
    }

    /// Advance the cursor by asserting that `n` bytes have been filled.
    ///
    /// After advancing, the `n` bytes are no longer accessible via the cursor and can only be
    /// accessed via the underlying buffer. I.e., the buffer's filled portion grows by `n` elements
    /// and its unfilled portion (and the capacity of this cursor) shrinks by `n` elements.
    ///
    /// # Safety
    ///
    /// The caller must ensure that the first `n` bytes of the cursor have been properly
    /// initialised.
    #[inline]
    pub unsafe fn advance(&mut self, n: usize) -> &mut Self {
        self.buf.filled += n;
        self.buf.init = max(self.buf.init, self.buf.filled);
        self
    }

    /// Initializes all bytes in the cursor.
    #[inline]
    pub fn ensure_init(&mut self) -> &mut Self {
        for byte in self.uninit_mut() {
            byte.write(0);
        }
        self.buf.init = self.buf.capacity;

        self
    }

    /// Asserts that the first `n` unfilled bytes of the cursor are initialized.
    ///
    /// `BorrowedBuf` assumes that bytes are never de-initialized, so this method does nothing when
    /// called with fewer bytes than are already known to be initialized.
    ///
    /// # Safety
    ///
    /// The caller must ensure that the first `n` bytes of the buffer have already been initialized.
    #[inline]
    pub unsafe fn set_init(&mut self, n: usize) -> &mut Self {
        self.buf.init = max(self.buf.init, self.buf.filled + n);
        self
    }

    /// Appends data to the cursor, advancing position within its buffer.
    ///
    /// # Panics
    ///
    /// Panics if `self.capacity()` is less than `buf.len()`.
    #[inline]
    pub fn append(&mut self, buf: &[u8]) {
        assert!(self.capacity() >= buf.len());

        // SAFETY: we do not de-initialize any of the elements of the slice
        unsafe {
            MaybeUninit::write_slice(&mut self.as_mut()[..buf.len()], buf);
        }

        // SAFETY: We just added the entire contents of buf to the filled section.
        unsafe {
            self.set_init(buf.len());
        }
        self.buf.filled += buf.len();
    }
}

impl<'a, A: 'static + Allocator> Write for OwnedCursor<'a, A> {
    fn write(&mut self, buf: &[u8]) -> Result<usize> {
        self.append(buf);
        Ok(buf.len())
    }

    fn flush(&mut self) -> Result<()> {
        Ok(())
    }
}

fn drop_vec<A: 'static + Allocator>(buf: &mut OwnedBuf<A>) {
    let buf = ManuallyDrop::new(buf);
    let _vec = unsafe { Vec::from_raw_parts_in(buf.data, buf.filled, buf.capacity, ptr::read(&buf.allocator)) };
}

impl<A: 'static + Allocator> From<Vec<MaybeUninit<u8>, A>> for OwnedBuf<A> {
    fn from(v: Vec<MaybeUninit<u8>, A>) -> OwnedBuf<A> {
        let (data, len, capacity, allocator) = v.into_raw_parts_with_alloc();
        OwnedBuf {
            data,
            dtor: &drop_vec,
            capacity,
            filled: len,
            init: len,
            allocator,
        }
    }
}

impl<A: 'static + Allocator> From<Vec<u8, A>> for OwnedBuf<A> {
    fn from(v: Vec<u8, A>) -> OwnedBuf<A> {
        let (data, len, capacity, allocator) = v.into_raw_parts_with_alloc();
        OwnedBuf {
            data: data as *mut MaybeUninit<u8>,
            dtor: &drop_vec,
            capacity,
            filled: len,
            init: len,
            allocator,
        }
    }
}
```

## Usage

A simple read looks like:

```rust
async fn read_example(reader: &mut impl Read) -> Result<()> {
    let mut buf = [0; 1024];
    reader.read(&mut buf).await?;
    // Use buf
    Ok(())
}
```

The specialized traits can be used by themselves:

```rust
async fn read_example_owned(reader: &mut impl OwnedRead) -> Result<()> {
    let mut buf = vec![0; 1024];
    let result = reader.read(buf).await;
    buf = result.0;
    result.1?;
    // Use buf
    Ok(())
}

async fn read_example_ready(reader: &mut impl ReadyRead) -> Result<()> {
    loop {
        reader.ready(Interest::READABLE).await?;

        let mut buf = [0; 1024];
        if let NonBlocking::Ready(n) = reader.non_blocking_read(buf)? {
            // Use buf
            return Ok(());
        }
    }
}
```

Where a user of read wants optimal performance and cannot know if the reader implements a specialized trait, then it can test by downcasting. I demonstrate testing for both traits; most end user code would only test for the trait that maximizes their performance.

```rust
async fn read_example(reader: &mut impl Read) -> Result<()> {
    if let Some(reader) = reader.as_owned() {
        let mut buf = vec![0; 1024];
        let result = reader.read(buf).await;
        buf = result.0;
        result.1?;
        // Use buf
    } else if let Some(reader) = reader.as_ready() {
        loop {
            reader.ready(Interest::READABLE).await?;

            let mut buf = [0; 1024];
            if let NonBlocking::Ready(n) = reader.non_blocking_read(buf)? {
                // Use buf
                return Ok(());
            }
        }
    } else {
        let mut buf = [0; 1024];
        reader.read(&mut buf).await?;
        // Use buf
    }
    Ok(())
}
```

Wrapper traits should implement all traits by dispatching to their inner traits (and doing whatever processing is necessary for the specifics of the trait). The `Read` implementation should not downcast.

## `Write` proposal

TODO

TODO OwnedWrite

## `BufRead` proposal

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


### `BufReader`

`BufReader` is a concrete type, it's a utility type for converting objects which implement `Read` into objects which implement `BufRead`. I.e., it encapsulates a reader with a buffer to make a reader with an internal buffer. `BufReader` provides its own buffer and does not let the user customise it.

I think that we don't need a separate `async::BufReader` type, but rather we need to duplicate the `impl<R: Read> BufReader<R>` impl for `R: async::Read` and to implement `async::BufRead` where `R: async::Read`. Similarly, we would duplicate the `impl<R: Seek> BufReader<R>` impl as `impl<R: async::Seek> BufReader<R>`
to provide an async versions of `seek_relative`. This might be an area where async overloading is useful.


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

There are several alternatives to the design proposed above. The first few are presented in separate files and I consider them feasible (though inferior to the above proposal), the later few are clearly sub-optimal and I haven't described them in depth.

### Readiness super trait

[super-ready.md](alternatives/super-ready.md)

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

### Offer only `BufRead`, and not `Read` (likewise for `Write`)

It might be possible to reduce the set of IO traits to only the buffered varieties by using a sophisticated buffer or buffer manager type to abstract over the various ways in which buffers can be managed and passed to read methods. By having the buffer manager supply the buffer, there is no need to wait on readiness and instead the buffer manager creates or provides the buffer when the reader is ready.

The approach would work well with completion based IO as well as readiness. However, this approach adds some complexity for the simple cases of read, would be a radical departure from the existing sync traits, and it's unclear if the lifetimes can be made to work in all cases.

### Add non-async version of readiness and owned read support

Although using the readiness and owned APIs is more strongly motivated in async code, there is no reason it can't be used in non-async code. We might consider adding support for explicit readiness support to std. This would increase the symmetry between sync and async libraries at the expense of increasing the surface are of std.

## Requirements discussion

### The traits should be ergonomic to implement and to use

On the plus side, the primary proposal is simple in the simple case (both to use and define) and is as close to symmetric with the non-async traits as possible (assuming we support specialized IO modes at all). It is always possible to use the specialized traits from the basic one, without requiring multiple versions of functions or data types. On the minus side, 'good citizen' libraries do have some extra work to do (mostly unavoidable if we are to support the specialized modes), in particular, optimally implementing a wrapper type requires some work and that work is not enforced or encouraged by the types (i.e., one can just write a naive `read` impl and there is no warning).

The split trait and polling alternatives are more complex and less ergonomic. Implementing the IO traits in the primary proposal is more complex than in the simple async traits alternative, but that does not support optimal performance. All the other tweaks or minor alternatives make the primary proposal more complex or less symmetric.

I believe the `OwnedRead` trait is the most ergonomic solution for efficient completion reads. It closely matches existing solutions from the community, and allows implementing other models (such as the library managing buffers) using `OwnedRead` as a building block.

### The traits support the same variations as the sync traits

All alternatives support vectored reads and writes and reading into uninitialized memory, in a similar way to the sync traits. There is nothing specific to any alternative which makes this better or worse.

We could remove support for `&mut [u8]` reads and only support `BorrowedBuf` reads. This would be a simpler API and just as flexible, at the cost of a tiny reduction in ergonomics and a reduction in symmetry.

### Generic usage

The primary proposal supports concurrent reading and writing by implementing `Read` and `Write` for reference types in the same manner as for the sync traits.

Other alternatives support concurrent reads and writes similarly, or by polling or using a 'split' trait.

Sometimes it is necessary to permit concurrent reads and writes in generic code. In this case, the reference impls solution does not work: it requires `T: Read + Write` and we can't easily require reference impls for concurrent reads and writes. The primary proposal would work in this case via `ReadyRead`, but not in the simple or owned case (this is similar to the situation in non-async code). The `Split` trait also works here, using `T: Split` rather than `T: Read + Write`.

The traits are usable as trait objects in all variations, assuming that we can support async trait objects in the language.

By using `OwnedBuf` (a concret type), `OwnedRead` is also usable as a trait object (module trait object support for async methods). This would not be the case if using a trait for the buffer type.

### The traits must work performantly with both readiness- (e.g., epoll) and completion-based systems (e.g., io_uring, IOCP)

The primary proposal addresses this requirement well by providing the specialized traits. The only downside is that there is no guarantee (or static checking) of whether the more performant modes are available.

### The traits should permit maximum flexibility of buffers

We do not provide a trait where the IO library manages the buffers (in some previous proposals I called this `ManagedRead`). One could use `BufRead` for this, though the API is optimised for tasks which can't be done without buffering, rather than generic reading. One could also implement such a trait in a third-party crate on top of `OwnedRead`. An alternative would be to include such a trait in std. This could be done later. I opted not to for the sake of simplicity.

`OwnedBuf` is designed to be efficiently interoperable with any reasonable buffer type.

### The traits should work in no_std scenarios

Now that `std::Error` has moved to core, none of the alternatives present any further difficulty in working with no_std crates.
