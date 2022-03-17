# Alternative: async traits

The simplest and most ergonomic alternative is to copy `io::{Read, Write}` and add `async` to most of the methods. I think this would work and would be optimal from the point of view of symmetry with the sync traits. However, it requires always pre-allocating a buffer per-read, which is unacceptable for some applications. The current proposal can be thought of as a super-set of the async-method-only proposal, from the perspective of the trait user.

```rust
pub use std::io::Result;

pub trait Read {
    async fn read(&mut self, buf: &mut [u8]) -> Result<usize>;
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
    async fn write(&mut self, buf: &[u8]) -> Result<usize>;
    async fn write_vectored(&mut self, bufs: &mut [IoSlice<'_>]) -> Result<usize> { ... }
    async fn write_all(&mut self, buf: &[u8]) -> Result<()> { ... }
    async fn write_all_vectored(&mut self, bufs: &mut [IoSlice<'_>]) -> Result<()> { ... }
    async fn write_fmt(&mut self, fmt: Arguments<'_>) -> Result<()> { ... }
    async fn flush(&mut self) -> Result<()>;

    fn is_write_vectored(&self) -> bool { ... }

    fn by_ref(&mut self) -> &mut Self
    where
        Self: Sized,
    { ... }
}
```

## Discussion

Usage of this design matches the ergonomic usage of the proposed design. For implementers, things here are easier since they do not have to consider the `Ready` trait.

The downside of this design is the constraints on how buffers must be managed. Lets look in detail at how `read` is used (the same arguments apply to the other reading and writing methods). The caller passes a borrowed buffer to `read`. Since `read` is async it may take some time to execute and the buffer is only used at the end of that time (when the IO read is ready to write data into the buffer), but the buffer must be allocated and borrowed by `read` for the whole duration of the call. That means that the buffer memory must be allocated and not used for that duration. In contrast, with the proposed approach, the buffer does not need to be passed into the IO system until the read is ready (when calling `non_blocking_read`, as the name suggests, this call should be quick). Whilst waiting for the IO operation to be ready (by calling `ready`), the buffer is not required. That means the buffer can be allocated just in time, or can be used for something else.

Lets consider why this is important. Imagine some kind of HTTP server waiting on many requests. With this alternative, the server must allocate a lot of fairly large buffers - a significant usage of memory which is sitting uselessly while waiting for IO. With the `Ready` trait, there are multiple memory management strategies the server could use:

* Allocate a buffer just in time for the read (no memory sits uselessly waiting); memory could be allocated on the stack for temporary usage or on the heap if the results of the read should be stored.
* Use a single buffer for all reads, allocated in long-living memory or even statically. Not only is memory not sitting uselessly, but the server doesn't need to allocate on each call. The memory can only be used temporarily however since it will be required for the next read.
* Use a buffer pool or similar strategy. Similar advantages to using a single buffer, but the buffers can be kept semi-permanently by clients.
* Using a buffer which is part of an external data structure (this could be done under this alternative and I mention it since it is a scenario with different lifetimes which must be accommodated. One advantage of the `Ready` trait design is that even if the buffer is owned by an external object the memory itself can be allocated later leading to lower memory usage, e.g., if the buffer is a `Vec`, it can be owned by the external data structure but can be minimally sized until a read is ready when it can be expanded).

### Alternative buffer management designs

Given that memory usage is important, how could we accommodate late allocation without the `Ready` trait (in particular, whilst keeping symmetry with the sync traits)?

#### Don't

We could just do nothing. However, at least one major runtime (Tokio) view supporting the memory-optimal path as a hard requirement, so if we choose not to support it then the async IO traits are unlikely to see widespread use. The requirement is also highlighted by end users (at AWS and Microsoft, at least) and by library authors (Hyper). Therefore, I believe some solution is necessary.

#### Use a different buffer type

We could either change the signature of `read` to use a different type or add methods to the `Read` trait. In either case, there are several possibilities for the buffer:

* a concrete buffer type would include API for allocating the memory as well as writing into that memory so that the read method could allocate as late as possible, thus avoiding having memory 'sitting around'. Allocation could mean literally allocating or could mean checking out a buffer from a buffer pool.
* A concrete buffer could refer to memory elsewhere (for example a single shared buffer, or buffers in other data structures).
* The buffer type could be a trait permitting any of the previous strategies. However, if the trait is used generically it prevents using the `Read` trait as a trait object, and if the trait is used as a trait object then it is less efficient to use.

Whether we use a trait or a concrete type, we must make a choice about whether the buffer is borrowed or owned. I don't think there is a single type that can be used which supports all buffer use cases and satisfies all other requirements for the `Read` trait design. In any case, we're still adding some API compared with the sync trait.

This has the major downside that it is less ergonomic than the main proposal *and* is not symmetric with the sync system.

#### Use `BufRead` and supply a growable buffer or buffer pool

We could support only the straightforward `Read` trait and require using `BufRead` for memory-optimal use. That allows the buffer to be passed to the reader as the concrete type rather than in generic code which frees the implementation to handle buffers however it likes. The trouble with this approach is that it just pushes the difficult (impossible?) task of designing the buffer type to the IO library rather than std. For a general purpose IO library (or runtime), the task is no easier because there is no smaller scope.

This approach has the additional downside that generic code which wants to support memory-optimal performance must accept `BufRead` instead of `Read` (or duplicate the functionality), even where `Read` would give a better experience for most users. On the other hand, it is the only solution which is truly symmetric with sync IO. I don't think that is enough to offset the downsides though.

#### Use `Ready` trait in addition to the straightforward trait design

In this alternative we'd have the straightforward `Read` and `Write` traits and have the `Ready` trait and `ReadyRead: Ready` and `ReadyWrite: Ready` as well. This would allow for a symmetric system and a separate memory-optimal system. However, it still means there is not symmetry between sync and async designs, and seems worse than just one set of IO traits.

