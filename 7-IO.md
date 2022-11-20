Tokio 中的 I/O 操作方式与 `std` 中的基本相同，但是异步地。有读特性([`AsyncRead`](https://docs.rs/tokio/1/tokio/io/trait.AsyncRead.html)) 和 写特性([`AsyncWrite`](https://docs.rs/tokio/1/tokio/io/trait.AsyncWrite.html))。特定类型根据需要实现这些特性([`TcpStream`](https://docs.rs/tokio/1/tokio/net/struct.TcpStream.html)，[`File`](https://docs.rs/tokio/1/tokio/fs/struct.File.html)，[`Stdout`](https://docs.rs/tokio/1/tokio/io/struct.Stdout.html))。[`AsyncRead`](https://docs.rs/tokio/1/tokio/io/trait.AsyncRead.html) 和  [`AsyncWrite`](https://docs.rs/tokio/1/tokio/io/trait.AsyncWrite.html) 也由许多数据结构实现，如 `Vec<u8>` 和 `&[u8]`。这允许在需要 读取器 或 写入器 的地方使用字节数组。                         
本页将介绍使用 Tokio 进行的基本 I/O 读写，并实践一些示例。下一页将介绍更高级的 I/O 示例。                  


# `AsyncRead` 和 `AsyncWrite`
这两个特性提供了异步读取和写入字节流的功能。这些特性上的方法通常不会直接调用，类似于您不手动调用 `Future` 特性中的 `poll` 方法。相反，您将通过 [`AsyncReadExt`](https://docs.rs/tokio/1/tokio/io/trait.AsyncReadExt.html) 和 [`AsyncWriteExt`](https://docs.rs/tokio/1/tokio/io/trait.AsyncWriteExt.html) 提供的实用工具方法使用它们。                 
让我们简要地看一下其中的一些方法。所有这些函数都是 `async` 的，必须与 `.await` 一起使用。              

## `async fn read()`
[`AsyncReadExt::read`](https://docs.rs/tokio/1/tokio/io/trait.AsyncReadExt.html#method.read) 提供了一个异步方法，用于将数据读入缓冲区，返回读取的字节数。                
***注意***：当 `read()` 返回 `Ok(0)` 时，这表示流已关闭。对 `read()` 的任何进一步调用将立即完成，并返回 `Ok(0)`。对于 [`TcpStream`](https://docs.rs/tokio/1/tokio/net/struct.TcpStream.html) 实例，这意味着套接字的读取部分已关闭。 
```Rust
use tokio::fs::File;
use tokio::io::{self, AsyncReadExt};

#[tokio::main]
async fn main() -> io::Result<()> {
    let mut f = File::open("foo.txt").await?;
    let mut buffer = [0; 10];

    // 读取至多 10 字节
    let n = f.read(&mut buffer[..]).await?;

    println!("The bytes: {:?}", &buffer[..n]);
    Ok(())
}
```

## `async fn read_to_end()` 
[`AsyncReadExt::read_to_end`](https://docs.rs/tokio/1/tokio/io/trait.AsyncReadExt.html#method.read_to_end) 将 stream 中的所有字节全部读入，直到 EOF：
```Rust
use tokio::io::{self, AsyncReadExt};
use tokio::fs::File;

#[tokio::main]
async fn main() -> io::Result<()> {
    let mut f = File::open("foo.txt").await?;
    let mut buffer = Vec::new();

    // 读取整个文件
    f.read_to_end(&mut buffer).await?;
    Ok(())
}
```

## `async fn write()`
[`AsyncWriteExt::write`](https://docs.rs/tokio/1/tokio/io/trait.AsyncWriteExt.html#method.write) 将一个 buffer 中的内容写入到 ***writer(写入器)*** 中，返回 写入的字节数：
```Rust
use tokio::io::{self, AsyncWriteExt};
use tokio::fs::File;

#[tokio::main]
async fn main() -> io::Result<()> {
    let mut file = File::create("foo.txt").await?;

    // 给字节字符串写一些前缀，但不必须是其所有？(but not necessarily all of it)
    let n = file.write(b"some bytes").await?;

    println!("Wrote the first {} bytes of 'some bytes'.", n);
    Ok(())
}
```

## `async fn write_all()`
[`AsyncWriteExt::write_all`](https://docs.rs/tokio/1/tokio/io/trait.AsyncWriteExt.html#method.write_all) 将整个 buffer 写入到 writer：
```Rust
use tokio::io::{self, AsyncWriteExt};
use tokio::fs::File;

#[tokio::main]
async fn main() -> io::Result<()> {
    let mut file = File::create("foo.txt").await?;

    file.write_all(b"some bytes").await?;
    Ok(())
}
```
这两个特性都包括许多其他有用的方法。请参阅 API 文档以获取全面列表。


# 辅助函数
此外，与 `std` 一样，[`tokio::io`](https://docs.rs/tokio/1/tokio/io/index.html) 模块包含许多有用的实用函数以及用于处理 [standard input](https://docs.rs/tokio/1/tokio/io/fn.stdin.html)、[standard output](https://docs.rs/tokio/1/tokio/io/fn.stdout.html) 和 [standard error](https://docs.rs/tokio/1/tokio/io/fn.stderr.html) 的 API。例如，[`tokio::io::copy`](https://docs.rs/tokio/1/tokio/io/fn.copy.html) 异步地将读取器的全部内容复制到写入器中。
```Rust
use tokio::fs::File;
use tokio::io;

#[tokio::main]
async fn main() -> io::Result<()> {
    let mut reader: &[u8] = b"hello";
    let mut file = File::create("foo.txt").await?;

    io::copy(&mut reader, &mut file).await?;
    Ok(())
}
```
注意，这使用了字节数组也实现 `AsyncRead` 的事实。


# Echo server
让我们练习一些异步 I/O。我们将编写一个 ***echo server(回声服务器)***。          
回声服务器绑定一个 `TcpListener` 并在一个循环中接收入站连接。对于每个入站连接，从套接字读取数据，并立即将其写回套接字。客户端向服务器发送数据，并接收完全相同的数据。                   
我们将使用稍微不同的策略实现两次回声服务器。

# 使用 `io::copy()`
首先，我们将使用 [`io::copy`](https://docs.rs/tokio/1/tokio/io/fn.copy.html) 实用程序实现 echo 逻辑。                 
您可以在新的二进制文件中编写此代码：
>touch src/bin/echo-server-copy.rs

您可以使用以下命令启动(或者 就是编译检查下)：
>cargo run --bin echo-server-copy

您可以使用标准的命令行工具(如 `telnet`) 或 编写简单的客户端(如 [`tokio::net::TcpStream`](https://docs.rs/tokio/1/tokio/net/struct.TcpStream.html#examples) 文档中的客户端)来尝试服务器。                  
这是一个TCP服务器，需要一个接受循环。生成一个新任务来处理每个接受的套接字。
```Rust
use tokio::io;
use tokio::net::TcpListener;

#[tokio::main]
async fn main() -> io::Result<()> {
    let listener = TcpListener::bind("127.0.0.1:6142").await?;

    loop {
        let (mut socket, _) = listener.accept().await?;

        tokio::spawn(async move {
            // 在这里复制数据
        });
    }
}
```
如前所述，该实用程序函数接收一个读取器和一个写入器，并将数据从一个复制到另一个。然而，我们只有一个 `TcpStream`。此单个值 ***同时实现了*** `AsyncRead` 和 `AsyncWrite`。由于 `io::copy` 对读取器和写入器都需要 `&mut`，因此套接字不能同时用于两个实参。
```Rust
// 这将编译失败
io::copy(&mut socket, &mut socket).await
```

## 拆分一个 读取器 + 写入器
要解决这个问题，我们必须将套接字拆分为 读取器句柄 和 写入器句柄。拆分读写器组合的最佳方式取决于特定类型的。                 
可以使用 [`io::split`](https://docs.rs/tokio/1/tokio/io/fn.split.html) 实用工具拆分任何读写器类型。此函数接受单个值并返回分离的读取器句柄和写入器句柄。这两个句柄可以独立使用，包括从单独的任务中使用。               
例如，回声客户端 可以像这样处理并发读写：
```Rust
use tokio::io::{self, AsyncReadExt, AsyncWriteExt};
use tokio::net::TcpStream;

#[tokio::main]
async fn main() -> io::Result<()> {
    let socket = TcpStream::connect("127.0.0.1:6142").await?;
    let (mut rd, mut wr) = io::split(socket);

    // 在后台写入数据
    tokio::spawn(async move {
        wr.write_all(b"hello\r\n").await?;
        wr.write_all(b"world\r\n").await?;

        // 有时 Rust 类型推断需要一点帮助
        Ok::<_, io::Error>(())
    });

    let mut buf = vec![0; 128];

    loop {
        let n = rd.read(&mut buf).await?;

        if n == 0 {
            break;
        }

        println!("GOT {:?}", &buf[..n]);
    }

    Ok(())
}
```
由于 `io::split` 支持实现 `AsyncRead + AsyncWrite` 并返回单独句柄的 ***任何值***，因此在内部 `io::split` 使用 `Arc` 和 `Mutex`。`TcpStream` 可以避免这种额外开销。`TcpStream` 提供了两个专门的拆分功能。                    
[`TcpStream::split`](https://docs.rs/tokio/1/tokio/net/struct.TcpStream.html#method.split) 接收 stream 的一个 ***引用*** 并返回读写器句柄。因为使用了引用，所以两个句柄必须保持在与调用 `split()` 的同一任务上。这种专门的 `split` 是零成本的。不需要 `Arc` 或 `Mutex`。`TcpStream` 还提供了 [`into_split`](https://docs.rs/tokio/1/tokio/net/struct.TcpStream.html#method.into_split)，它支持仅以一个 `Arc` 的开销为代价跨任务移动的句柄。                        
由于 `io::copy()` 是在拥有 `TcpStream` 的同一任务上调用的，因此我们可以使用 [`TcpStream::split`](https://docs.rs/tokio/1/tokio/net/struct.TcpStream.html#method.split)。在服务器中处理回声逻辑的任务变为：
```Rust
tokio::spawn(async move {
    let (mut rd, mut wr) = socket.split();
    
    if io::copy(&mut rd, &mut wr).await.is_err() {
        eprintln!("failed to copy");
    }
});
```
你可以在 [这里](https://github.com/tokio-rs/website/blob/master/tutorial-code/io/src/echo-server-copy.rs) 找到完整的代码

## 手动复制
现在，让我们看看如何通过手动复制数据来编写回声服务器。为此，我们使用[`AsyncReadExt::read`](https://docs.rs/tokio/1/tokio/io/trait.AsyncReadExt.html#method.read) 和 [`AsyncWriteExt::write_all`](https://docs.rs/tokio/1/tokio/io/trait.AsyncWriteExt.html#method.write_all)。                   
完整的 回声服务器 如下：
```Rust
use tokio::io::{self, AsyncReadExt, AsyncWriteExt};
use tokio::net::TcpListener;

#[tokio::main]
async fn main() -> io::Result<()> {
    let listener = TcpListener::bind("127.0.0.1:6142").await?;

    loop {
        let (mut socket, _) = listener.accept().await?;

        tokio::spawn(async move {
            let mut buf = vec![0; 1024];

            loop {
                match socket.read(&mut buf).await {
                    // 返回 值 `Ok(0)` 表示远程已被关闭
                    Ok(0) => return,
                    Ok(n) => {
                        // 复制数据回套接字
                        if socket.write_all(&buf[..n]).await.is_err() {
                            // 预期外的套接字错误。这里我们没什么能做的，所以就停止处理就行
                            return;
                        }
                    }
                    Err(_) => {
                        // 预期外的套接字错误。这里我们没什么能做的，所以就停止处理就行
                        return;
                    }
                }
            }
        });
    }
}
```
(您可以将此代码放入 *src/bin/echo-server.rs* 中，并使用 `cargo run --bin echo-server` 启动它)。                     
让我们把它分解。首先，由于使用了 `AsyncRead` 和 `AsyncWrite` 实用工具，因此必须将扩展特性纳入范围。                
```Rust
use tokio::io::{self, AsyncReadExt, AsyncWriteExt};
```

## 分配一个缓冲区
策略是从套接字读取一些数据到缓冲区，然后将缓冲区的内容写回套接字。
```Rust
let mut buf = vec![0; 1024];
```
明确避免 栈缓冲区。回想 [[4-Spawning-生成]] 中 `Send` 约束节，我们注意到，调用 `.await` 时存在的所有任务数据都必须由任务存储。在这种情况下，`buf` 在跨越 `.await` 调用时被使用。所有任务数据都存储在单个分配中。您可以将其视为一个 `enum`，其中每个变体都是特定调用 `.await` 时需要存储的数据。                        
如果缓冲区由 栈数组 表示，则每个接受套接字生成的任务的内部结构体可能类似于：
```Rust
struct Task {
    // 这里是内部任务字段
    task: enum {
        AwaitingRead {
            socket: TcpStream,
            buf: [BufferType],
        },
        AwaitingWriteAll {
            socket: TcpStream,
            buf: [BufferType],
        }

    }
}
```
如果 栈数组 用作缓冲区类型，它将 ***inline(内联)*** 存储在任务结构体中。这将使任务结构体非常大。此外，缓冲区大小通常是页面大小。这反过来会使 `Task` 的大小变得尴尬：`$page-size + a-few-bytes`。                        
编译器比一个基本 `enum` 更进一步优化异步块的布局。在实践中，变量不会在变体之间移动，这是 `enum` 所需要的。然而，任务结构体大小至少与最大变量一样大。            
因此，使用专用的缓冲区分配通常更有效。                

## EOF 的处理
当 TCP 流的读取部分关闭时，调用 `read()` 返回 `Ok(0)`。此时退出读取循环很重要。忘记中断 EOF 上的读取循环是常见的错误源。
```Rust
loop {
    match socket.read(&mut buf).await {
        // 返回 值 `Ok(0)` 表示远程已被关闭
        Ok(0) => return,
        // ... 处理其他情况
    }
}
```
忘记中断读取循环通常会导致 100% CPU 无限循环的情况。当套接字关闭时，`socket.read()` 立即返回。然后循环永远重复。               
完整的代码可以在 [这里](https://github.com/tokio-rs/website/blob/master/tutorial-code/io/src/echo-server.rs) 找到。