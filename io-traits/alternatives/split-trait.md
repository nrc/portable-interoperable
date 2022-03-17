# Alternative: `Split` trait

```rust
// Ready, Read, and Write are unchanged.

trait Split: Read + Write {
    async fn split(&mut self) -> (impl Read + '_, impl Write + '_)
}
```

An alternative to implementing the `Read` and `Write` traits on reference types to support simultaneous reading and writing, is to have a `Split` trait which is used to split a resource into a reader and writer (which implement `Read` and `Write`, respectively). The advantage of this approach is that it supports reading and writing without simultaneous multiple reads (or writes). However, it is a more complicated API requiring another trait and extra concrete types for many resources, and brings us further from the sync traits.
