---
title: "bytes"
author: ["zhangjun"]
lastmod: 2024-08-04T17:09:16+08:00
tags: ["bytes", "rust"]
categories: ["rust"]
draft: false
series: ["rust crate"]
series_order: 4
---

[crate bytes](https://docs.rs/bytes/latest/bytes/) 主要功能：

1.  提供了低开销的只读 `连续内存` 的共享和可修改访问(引用计数, 例如 clone() 方法只是增加引用计数)；
2.  支持按照 le/ne 方式来读取数据，特别适合网络 IO 场景；

主要使用场景是网络数据流, 等效为 Arc&lt;Vec&lt;u8&gt;&gt; (但是是连续内存).

struct Bytes/BytesMut：

-   Bytes：提供了高效的 `只读的` zero-copy 的内存区域 clone/slice/split 操作, 并且支持 Send/Sync, 所以可以在多线程中高效共享数据; split_XX/slice_XX 都是引用计数方式产生新 Bytes 的操作, 高效共享数据; 可以当作 Arc&lt;Vec&lt;u8&gt;&gt;
    来使用；
-   BytesMut：提供了高效的 `可读写` 的内存区域操作。

Buf/BufMut trait：

-   Buf trait：支持 get_u16_le 等大小端读取，同时 get_XX 操作自动更新内部 buffer 的读 cursor 指针，从而实现连续读取相邻内存。
-   BufMut trait：支持 put_u16_le 等大小端写入，同时 put_XX 操作自动更新内部 buffer 的写 cursor 指针，从而实现连续写入相邻内存。

struct Bytes 实现了 Buf trait，struct BytesMut 同时实现了 Buf 和 BufMut trait。

struct Bytes 为底层连续内存区域提供了只读的 zeor copy（基于引用计数）的 slice/split/clone 操作, 主要用于高效共享读取访问，它实现了 bytes::Buf trait, 实现了 AsRef&lt;[u8]&gt;, Borrow&lt;[u8]&gt;, Deref&lt;Target=[u8]&gt; 的 trait，所以
`Bytes 可以当作 [u8] 来使用` ：

-   Bytes 实现了各种 From&lt;T&gt; trait, T 可以是 &amp;[u8]/&amp;str/Bytes/BytesMut/String/Vec&lt;u8&gt;

<!--listend-->

```rust
use bytes::Bytes;

let b = Bytes::new();
assert_eq!(&b[..], b""); // b[..] 是调用 Deref 到 slice [u8] 的 index 操作

let b = Bytes::from_static(b"hello");
assert_eq!(&b[..], b"hello");

let b = Bytes::from(&b"hello"[..]);
assert_eq!(b.len(), 5);

let mut mem = Bytes::from("Hello world");
// pub fn slice(&self, range: impl RangeBounds<usize>) -> Self
let a = mem.slice(0..5); // 没有内存拷贝, 只是增加引用计数，返回一个新的 Bytes 对象.
assert_eq!(a, "Hello");

let mem2 = mem.clone(); // 没有内存拷贝, 只是增加引用计数. 返回一个新的 Bytes 对象

// Returns a slice of self that is equivalent to the given subset.
let bytes = Bytes::from(&b"012345678"[..]);
let as_slice = bytes.as_ref();
let subset = &as_slice[2..6];
let subslice = bytes.slice_ref(&subset); // subset 使用 Bytes 生成的, 然后又使用它生成一个 Bytes.
assert_eq!(&subslice[..], b"2345");

// split_off 将 Bytes 拆为两个, self 包含 [0, at)，返回 Bytes 包含 [at, len)
let mut a = Bytes::from(&b"hello world"[..]);
let b = a.split_off(5);
assert_eq!(&a[..], b"hello");
assert_eq!(&b[..], b" world");

// split_to 将 Bytes 拆为两个， self 包含 [at, len), 返回 Bytes 包含 [0, at).
let b = mem.split_to(6); // 内存 zero copy
assert_eq!(mem, "world");
assert_eq!(b, "Hello ");

let mut buf = Bytes::from(&b"hello world"[..]);
buf.truncate(5);
assert_eq!(buf, b"hello"[..]);

let mut buf = Bytes::from(&b"hello world"[..]);
buf.clear();
assert!(buf.is_empty());
```

struct Bytes 实现了 bytes::Buf trait，当调用 bytes::Buf 方法, 例如 get_u8/get_u16 时， `会自动更新内部的 cursor`
，从而在连续调用时依次返回下一个 u8/u16 内存：

-   由于 Bytes Deref&lt;Target=[u8]&gt;, 所有 Bytes 可以调用 [u8] 的方法, 如 slice[..] 索引操作;
-   bytes::Buf.len() 返回内存区域总长度（不随 get_XX 而变化）;
-   &amp;bytes::Buf[..] 返回的是内存区域总长度的 slice;

<!--listend-->

```rust
use bytes::Buf;
use bytes::Bytes;

let mut buf = Bytes::from("hello world");
// 使用 bytes::Buf 的 get_XX() 方法读取时，自动更新内部 cursor 指针，这样连续调用时返回下一个内存区域内容。
assert_eq!(b'h', buf.get_u8());
assert_eq!(b'e', buf.get_u8());
assert_eq!(b'l', buf.get_u8());

buf.remaining(); // 返回未读取的数据长度
println!("{}, {:#?}", buf.len(), &buf[..]); // buf.len() 为 8, &buf[..] 获得所有元素的引用

let mut rest = [0; 8];
buf.copy_to_slice(&mut rest); // 将未读取的内容 copy 到传入的 slice
assert_eq!(&rest[..], &b"lo world"[..]);

// &[u8], Box<T>, &mut T, VecDeque<u8>, Cursor<T>， Bytes， BytesMut 等都实现了 bytes::Buf
let mut buf = &b"hello world"[..];
assert_eq!(buf.remaining(), 11);
buf.get_u8();
assert_eq!(buf.remaining(), 10);
```

struct BytesMut 提供了一个可以高效共享和读写的连续内存区域，对象的 owners 可以对它进行修改。在向
BytesMut 写入时自动扩充底层内存区域。

```rust
use bytes::{BytesMut, BufMut};

// BytesMut 实现了 BufMut trait， 这里调用调用 BufMut Trait 的方法时，内部自动更新写 cursor，所以连续的写操作
// 会依次更新内存区域。
let mut buf = BytesMut::with_capacity(64);
buf.put_u8(b'h');
buf.put_u8(b'e');
buf.put(&b"llo"[..]);
assert_eq!(&buf[..], b"hello"); // 由于内部有读写 cursor，所以 &buf[..] 不需要指定 range
```

struct BytesMut 同时实现了 bytes::Buf 和 bytes::BufMut trait, 可以 AsMut&lt;[u8]&gt;, BorrowMut&lt;[u8]&gt;,
DerefMut&lt;Target=[u8]&gt;;

-   读 cursor, 如使用 Buf trait 的 get_XX() 方法, 从数据区域的 self.ptr 开始, 可以继续读的有效内容为 self.len,
    读取一段长度 cnt 后, 增加 self.ptr+cnt, 同时减少 self.len-cnt;
-   写 cursor, 如使用 BufMut trait 的 put_XX() 方法, 从数据区域的 self.ptr+self.len 开始, 可以继续写的空间是无限大( u32::MAX - self.len), 写一段长度 cnt 后, 增加 self.len, 同时减少 self.capacity-cnt;
-   在写内容时, 会看当前实际分配的 Vec 的 capacity-len 是否满足 reserve 的需求, 同时看当前 Vec 前面已经读的内存区域长度是否满足 reserve 的需求: 具体参考<https://docs.rs/bytes/1.5.0/src/bytes/bytes_mut.rs.html#507>
    1.  如果满足, 不需要 increase Vec 长度, 而是将 Vec 当前读取 cursor 到 len 的 `内容移动到 Vec 开始` 的位置;
    2.  如果不满足,则需要 increase Vec 长度;

<!--listend-->

```rust
use bytes::{BytesMut, BufMut};

let mut buf = BytesMut::with_capacity(64);

// 实现了 bytes::Buf trait, 故可以使用 Buf 的 put_XX 方法, 自动更新内部写 cursor, 所以连续的 put_XX()会写先后
// 连续的内存区域.
buf.put_u8(b'h');
buf.put_u8(b'e');
buf.put(&b"llo"[..]); // 返回已经写入的所有内容
assert_eq!(&buf[..], b"hello");

// Freeze the buffer so that it can be shared Converts self into an immutable Bytes.
let a = buf.freeze();
// This does not allocate, instead `b` points to the same memory.
let b = a.clone(); // 共享同一个连续内存区域
assert_eq!(&a[..], b"hello");
assert_eq!(&b[..], b"hello");
```

由于 BufMut 是自动增长的, 而且内部自动维护读写 cursor 和 buff 内存区域, 所以非常适合循环读写场景：

```rust
// https://tokio.rs/tokio/tutorial/framing
use tokio::io::AsyncReadExt;
use bytes::Buf;
use mini_redis::Result;

pub async fn read_frame(&mut self) -> Result<Option<Frame>>
{
    loop {
        // Attempt to parse a frame from the buffered data. If enough data has been buffered, the frame is
        // returned.
        if let Some(frame) = self.parse_frame()? {
            return Ok(Some(frame));
        }

        // There is not enough buffered data to read a frame.  Attempt to read more data from the socket.
        //
        // On success, the number of bytes is returned. `0` indicates "end of stream".
        //
        // 每次 read_buf 都从上次写入的地址开始
        if 0 == self.stream.read_buf(&mut self.buffer).await? {
            // The remote closed the connection. For this to be a clean shutdown, there should be no data in
            // the read buffer. If there is, this means that the peer closed the socket while sending a frame.
            if self.buffer.is_empty() {
                return Ok(None);
            } else {
                return Err("connection reset by peer".into());
            }
        }
    }
}
```

如果使用 Vec 的不带写 cursor 的 buffer 实现, 会复杂很多:

1.  使用 Vec 来作为 buffer, 并用 0 值初始化;
2.  手动维护 cursor 的位置, 当 cursor 打到 Vec len 时, 需要扩容 Vec;
3.  向 Vec buff 写数据时, read(&amp;mut[buf]) 方法需要知道传入的的 buff 的 length；
4.  缺点: Vec 会一直在增长, 缺少缩容的机制.( 但是 Bytes/BytesMut 写操作在检查剩余空间时, 会看内部 Vec
    前面已读的空间是否满足增量需求, 如果可以会将未读取内容移动到 Vec 开头位置, 从而不需要扩容).

<!--listend-->

```rust
use tokio::net::TcpStream;

pub struct Connection {
    stream: TcpStream,
    buffer: Vec<u8>,
    cursor: usize,
}

impl Connection {
    pub fn new(stream: TcpStream) -> Connection {
        Connection {
            stream,
            // Allocate the buffer with 4kb of capacity.
            buffer: vec![0; 4096], // len() 和 capacity() 均为 4096
            cursor: 0,
        }
    }
}

use mini_redis::{Frame, Result};

pub async fn read_frame(&mut self) -> Result<Option<Frame>>
{
    loop {
        if let Some(frame) = self.parse_frame()? {
            return Ok(Some(frame));
        }

        // Ensure the buffer has capacity。 刚开始 self.buffer.len() 为 4096, self.cursor 为 0，resize() 会自
	// 动扩容 Vec 的容量.
        if self.buffer.len() == self.cursor {
            // Grow the buffer. 将 buff len 增加为指定的长度, 新增的元素用 0 填充
            self.buffer.resize(self.cursor * 2, 0);
        }

        // Read into the buffer, tracking the number of bytes read, 传入的 &mut[u8] 的 len 是 Vec 的 len -
	// cursor, read() 确保读取的直接长度 n < length
        let n = self.stream.read(&mut self.buffer[self.cursor..]).await?;
        if 0 == n {
            if self.cursor == 0 {
                return Ok(None);
            } else {
                return Err("connection reset by peer".into());
            }
        } else {
            // Update our cursor
            self.cursor += n;
        }
    }
}
```

Buf 和 BufMut 都是 trait, 而 Bytes/BytesMut 实现了这两个 trait, 类型也实现了这两个 trait, 通过 Buf/BufMut
的 get_XX/put_XX 方法读写这些类型时, 内部也维护了读写 cursor:

-   BytesMut 和 &amp;mut[u8] 同时实现了 Buf 和 BufMut trait;
-   &amp;[u8] 实现了 Buf，&amp;mut [u8] 和 Vec&lt;u8&gt; 实现了 BufMut；

<!--listend-->

```rust
// 实现了 Buf 的类型
impl Buf for &[u8]
impl<T: Buf + ?Sized> Buf for &mut T
impl<T: Buf + ?Sized> Buf for Box<T>
impl Buf for VecDeque<u8>
impl<T: AsRef<[u8]>> Buf for Cursor<T>

impl Buf for Bytes
impl Buf for BytesMut
impl<T, U> Buf for Chain<T, U> where T: Buf, U: Buf
impl<T: Buf> Buf for Take<T>

// 实现了 BufMut 的类型
impl BufMut for Vec<u8>
impl BufMut for &mut [MaybeUninit<u8>]
impl<T: BufMut + ?Sized> BufMut for &mut T
impl<T: BufMut + ?Sized> BufMut for Box<T>

impl BufMut for &mut [u8]
impl BufMut for BytesMut
impl<T, U> BufMut for Chain<T, U>where T: BufMut, U: BufMut
impl<T: BufMut> BufMut for Limit<T>
```

注意：使用 &amp;mut[u8] 作为 BufMut trait 的实现时有问题(解决办法是使用能自动扩容的 Vec 作为 buff):

1.  put_XX() 操作不会自动扩充底层的 &amp;mut[u8], 当写入的内容超过它的 length 时会 panic;
2.  put_XX() 操作会更新底层 &amp;mut[8] 的内存起始地址, 导致 buf[..] 返回的是 `还剩的没有写入的` 的内存内容;

<!--listend-->

```rust
use bytes::BufMut;

fn main() {
    let mut v = vec![1,2]; // 长度和容量为 2;
    let mut buf :&mut [u8] = &mut v;
    println!("{} {}", buf.remaining_mut(), buf2.remaining());

    buf.put_u8(b'h');
    println!("len: {}, content: {:#?}", buf.len(), String::from_utf8(buf[..].to_vec()));

    buf.put_u8(b'e');
    println!("len: {}, content: {:#?}", buf.len(), String::from_utf8(buf[..].to_vec()));

    buf.put(&b"llo"[..]); // panic
    println!("len: {}, content: {:#?}", buf.len(), String::from_utf8(buf[..].to_vec()));
}

// 执行失败:
// Exited with status 101

// Standard Error

//    Compiling playground v0.0.1 (/playground)
//     Finished dev [unoptimized + debuginfo] target(s) in 0.37s
//      Running `target/debug/playground`
// thread 'main' panicked at /playground/.cargo/registry/src/index.crates.io-6f17d22bba15001f/bytes-1.5.0/src/buf/buf_mut.rs:201:9:
// assertion failed: self.remaining_mut() >= src.remaining()
// note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace

// Standard Output

// remaining_mut: 2
// len: 1, remaining_mut: 1, content: Ok(  // buf[..] 返回的是剩下的可写内容
//     "\u{2}",
// )
// len: 0, remaining_mut: 0, content: Ok(
//     "",
// )
```

使用 Vec 作为 BufMut 没问题, 但是它没有实现 Buf trait, 所以 `只能写不能读` :

-   buf 和 buf2 不共享读写 cursor 也会导致问题.

<!--listend-->

```rust
use bytes::BufMut;
use bytes::Buf;

fn main() {
    let mut buf  = vec![1,2];

    println!("remaining_mut: {}", buf.remaining_mut());

    buf.put_u8(b'h');
    println!("len: {}, remaining_mut: {}, content: {:#?}", buf.len(), buf.remaining_mut(), String::from_utf8(buf[..].to_vec()));

    buf.put_u8(b'e');
    println!("len: {}, remaining_mut: {}, content: {:#?}", buf.len(), buf.remaining_mut(),String::from_utf8(buf[..].to_vec()));


    buf.put(&b"llo"[..]);
    println!("len: {}, remaining_mut: {}, content: {:#?}", buf.len(), buf.remaining_mut(), String::from_utf8(buf[..].to_vec()));

    let mut buf2 : &[u8]= &buf;
    println!(" buf2 len: {}, content: {:#?}", buf2.len(),String::from_utf8(buf2[..].to_vec()));

    buf2.get_u8();
    println!(" buf2 len: {},content: {:#?}", buf2.len(), String::from_utf8(buf2[..].to_vec()));

    buf.put_u8(b'!');
    println!("len: {}, remaining_mut: {}, content: {:#?}", buf.len(), buf.remaining_mut(),String::from_utf8(buf[..].to_vec()));
    // 报错: cannot borrow `buf` as mutable because it is also borrowed as immutable
    //println!(" buf2 len: {},content: {:#?}", buf2.len(), String::from_utf8(buf2[..].to_vec()));

    //      输出:
    //     remaining_mut: 9223372036854775805
    // len: 3, remaining_mut: 9223372036854775804, content: Ok(
    //     "\u{1}\u{2}h",
    // )
    // len: 4, remaining_mut: 9223372036854775803, content: Ok(
    //     "\u{1}\u{2}he",
    // )
    // len: 7, remaining_mut: 9223372036854775800, content: Ok(
    //     "\u{1}\u{2}hello",
    // )
    //  buf2 len: 7, content: Ok(
    //     "\u{1}\u{2}hello",
    // )
    //  buf2 len: 6,content: Ok(
    //     "\u{2}hello",
    // )
    // len: 8, remaining_mut: 9223372036854775799, content: Ok(
    //     "\u{1}\u{2}hello!",
    // )
}
```

相比之下, BytesMut 同时实现了 Buf/BufMut, 所以可以同时读写, `是建议的使用 bytes Crate 的类型` ：

-   使用 Buf/BufMut trait 提供的方法才会更新内部读写 cursor, 使用继承自 slice [u8] 的方法并不会更新内部 cursor；
-   BytesMut Deref&lt;target=[u8]&gt;, 所以 BytesMut 支持 index slice 操作, 例如 &amp;buf[..], 该操作会感知内部读指针（返回当前未读的所有内容），但是不会更新它。

<!--listend-->

```rust
use bytes::BufMut;
use bytes::Buf;
use bytes::BytesMut;

fn main() {
    let mut buf = BytesMut::with_capacity(64);
    buf.put_u8(b'h');
    buf.put_u8(b'e');
    buf.put(&b"llo"[..]);
    println!("len: {}, content: {:#?}", buf.len(), String::from_utf8(buf[..].to_vec())); // 5 和 hello

    buf.get_u8();  // 读一个字节后, buf.len() 减少, buf[..] 返回剩余未读的内容
    println!("len: {}, content: {:#?}", buf.len(), String::from_utf8(buf[..].to_vec())); // 4 和 ello

    buf.put_u8(b'!');  // 继续上次写的位置写入
    buf.put_u8(b'!');
    println!("len: {}, content: {:#?}", buf.len(), String::from_utf8(buf[..].to_vec())); // 6 和 ello!!

    let bytes = buf.copy_to_bytes(5); // Buf trait 的 copy_to_bytes/copy_to_slice 都会读消耗缓存
    println!("len: {}, content: {:#?}", buf.len(), String::from_utf8(buf[..].to_vec())); // 1 和 !
}
```

tokio::io::AsyncReadExt trait 提供了使用 BufMut 作为 buff 的 `read_buf() 方法`, 内部可以自动管理读写 cursor:

```rust
fn read_buf<'a, B>(&'a mut self, buf: &'a mut B) -> ReadBuf<'a, Self, B>
where
    Self: Unpin,
    B: BufMut + ?Sized,

// 例子
use tokio::fs::File;
use tokio::io::{self, AsyncReadExt};
use bytes::BytesMut;

#[tokio::main]
async fn main() -> io::Result<()> {
    let mut f = File::open("foo.txt").await?;
    let mut buffer = BytesMut::with_capacity(10);
    assert!(buffer.is_empty());
    assert!(buffer.capacity() >= 10);

    // note that the return value is not needed to access the data that was read as `buffer`'s internal cursor
    // is updated.
    //
    // this might read more than 10 bytes if the capacity of `buffer` is larger than 10.
    f.read_buf(&mut buffer).await?;  // 不需要返回读入的字节数, 因为 BytesMut 内部自动维护读写 cursor
    println!("The bytes: {:?}", &buffer[..]); // 返回读到的所有内容

    Ok(())
}


// 如果使用普通的 slice buffer, 则需要返回读到的字节数
use tokio::fs::File;
use tokio::io::{self, AsyncReadExt};

#[tokio::main]
async fn main() -> io::Result<()> {
    let mut f = File::open("foo.txt").await?;
    let mut buffer = [0; 10];

    // read up to 10 bytes
    let n = f.read(&mut buffer[..]).await?; // read() 方法使用 &'a mut [u8] 类型的 Buff

    println!("The bytes: {:?}", &buffer[..n]);
    Ok(())
}
```
