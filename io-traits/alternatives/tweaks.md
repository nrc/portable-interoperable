# Tweaks

These are smaller changes to the proposed design in [README.md](README.md), rather than full alternatives.

They both have the same advantage: that they are arguably more logical designs for the `Read` trait, but also the same disadvantage: they take the async trait design further from the sync version.

## Make vectored IO a separate trait

It was arguably a mistake to add vectored IO to the existing IO traits rather than making them separate traits, the `is_vectored` method indicates this. We could fix that for async IO, at the expense of some similarity to the sync traits.

```rust
pub use std::io::Result;

pub trait Ready {
    async fn ready(&mut self, interest: Interest) -> Result<Readiness>;
}

pub trait Read: Ready {
    fn non_blocking_read_buf(&mut self, buf: &mut ReadBuf<'_>) -> Result<NonBlocking<()>>;

    async fn read(&mut self, buf: &mut [u8]) -> Result<usize> { ... }
    async fn read_buf(&mut self, buf: &mut ReadBuf<'_>) -> Result<())> { ... }
    async fn read_exact(&mut self, buf: &mut [u8]) -> Result<()> { ... }
    async fn read_buf_exact(&mut self, buf: &mut ReadBuf<'_>) -> Result<()> { ... }
    async fn read_to_end(&mut self, buf: &mut Vec<u8>) -> Result<usize> { ... }
    async fn read_to_string(&mut self, buf: &mut String) -> Result<usize> { ... }

    fn by_ref(&mut self) -> &mut Self
    where
        Self: Sized,
    { ... }
}

pub trait ReadVectored: Ready {
    fn non_blocking_read_buf_vectored(&mut self, bufs: &mut ReadBufVec<'_>) -> Result<NonBlocking<usize>>;

    async fn read_vectored(&mut self, bufs: &mut [IoSliceMut<'_>]) -> Result<usize> { ... }
    async fn read_buf_vectored(&mut self, bufs: &mut ReadBufVec<'_>) -> Result<usize> { ... }

    fn by_ref(&mut self) -> &mut Self
    where
        Self: Sized,
    { ... }
}

pub trait Write: Ready {
    fn non_blocking_write(&mut self, buf: &[u8]) -> Result<NonBlocking<usize>>;
    fn non_blocking_flush(&mut self) -> Result<NonBlocking<()>>;

    async fn write(&mut self, buf: &[u8]) -> Result<usize> { ... }
    async fn write_all(&mut self, buf: &[u8]) -> Result<()> { ... }
    async fn write_fmt(&mut self, fmt: Arguments<'_>) -> Result<()> { ... }
    async fn flush(&mut self) -> Result<()> { ... }

    fn by_ref(&mut self) -> &mut Self
    where
        Self: Sized,
    { ... }
}

pub trait WriteVectored: Ready {
    fn non_blocking_write_vectored(&mut self, bufs: &[IoSlice<'_>]) -> Result<NonBlocking<usize>>;
    fn non_blocking_flush(&mut self) -> Result<NonBlocking<()>>;

    async fn write_vectored(&mut self, bufs: &mut [IoSlice<'_>]) -> Result<usize> { ... }
    async fn write_all_vectored(&mut self, bufs: &mut [IoSlice<'_>]) -> Result<()> { ... }
    async fn flush(&mut self) -> Result<()> { ... }

    fn by_ref(&mut self) -> &mut Self
    where
        Self: Sized,
    { ... }
}

// No change to helper types
```

## Only provide `ReadBuf` methods

Rather than providing methods which operate on `ReadBuf` and `&[u8]` we could elide the latter since it is easy to convert from `ReadBuf` to `&[u8]`. Alternatively we could have the methods take `impl Into<ReadBuf>` thus having a single method which can take both types (although this prevents using the IO traits as trait objects).

```rust
pub trait Read: Ready {
    fn non_blocking_read(&mut self, buf: &mut ReadBuf<'_>) -> Result<NonBlocking<()>>;
    fn non_blocking_read_vectored(&mut self, bufs: &mut ReadBufVec<'_>) -> Result<NonBlocking<usize>> { ... }

    async fn read(&mut self, buf: &mut ReadBuf<'_>) -> Result<())> { ... }
    async fn read_exact(&mut self, buf: &mut ReadBuf<'_>) -> Result<()> { ... }
    async fn read_vectored(&mut self, bufs: &mut ReadBufVec<'_>) -> Result<usize> { ... }
    async fn read_to_end(&mut self, buf: &mut Vec<u8>) -> Result<usize> { ... }
    async fn read_to_string(&mut self, buf: &mut String) -> Result<usize> { ... }

    fn is_read_vectored(&self) -> bool { ... }

    fn by_ref(&mut self) -> &mut Self
    where
        Self: Sized,
    { ... }
}

// No change to `Write` or `Ready` trait of utility types.
```

## No impls on reference types

We could encourage implementers not to implement `Read` and `Write` for reference types. This would still allow simultaneous read/write via the `ready` method. However, it forces more users away from the ergonomic path. It is also less symmetric with the sync trait model.
