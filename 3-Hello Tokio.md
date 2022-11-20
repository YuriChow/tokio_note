我们将从编写一个非常基本的 Tokio 应用程序开始。它将连接到 Mini-Redis 服务器，设置键 `hello` 的值为 `world` 。然后它将读回密钥。这将使用 Mini-Redis 客户端库完成。          


# 代码
## 生成一个新箱
让我们从生成一个新 ***Rust*** app 开始：
>cargo new my-redis
>cd my-redis

## 添加依赖
接下来，打开 *Cargo.toml* 并添加以下内容至 `[dependencies]` 下面：
```TOML
tokio = { version = "1", features = ["full"] }
mini-redis = "0.4"
```

## 编写代码
接下来，打开 *main.rs* 并且使用以下内容替换文件原内容：
```Rust
use mini_redis::{client, Result};

#[tokio::main]
async fn main() -> Result<()> {
    // 打开一个到 mini-redis 地址的链接.
    let mut client = client::connect("127.0.0.1:6379").await?;

    // 用 value "world" 设置 key "hello" 
    client.set("hello", "world".into()).await?;

    // 获取 key "hello"
    let result = client.get("hello").await?;

    println!("got value from the server; result={:?}", result);

    Ok(())
}
```
确保 Mini-Redis 服务器正在运行。在一个单独的终端中，运行：
>mini-redis-server

如果你还没安装 mini-redis，您可以使用以下命令安装：
>cargo install mini-redis

现在，运行 `my-redis` 应用：
>cargo run 

终端将输出：
>got value from the server; result=Some(b"world")

成功！          
您可以在 [这里](https://github.com/tokio-rs/website/blob/master/tutorial-code/hello-tokio/src/main.rs) 找到完整代码。


# 拆解
让我们花些时间来仔细检查我们刚刚做了什么。代码不多，但是发生的事很多。
```Rust
let mut client = client::connect("127.0.0.1")
```
[`client::connect`](https://docs.rs/mini-redis/0.4/mini_redis/client/fn.connect.html) 函数由 `mini-redis` 箱提供。它异步地建立与指定远程地址的 TCP 连接。连接建立后，将返回一个 `client` 句柄。即使操作是异步执行的，我们编写的代码 ***看起来*** 也是同步的。操作是异步的 的唯一标识 是 `.await` 运算符。

## 什么是异步编程？
大多数计算机程序的执行顺序与它们的编写顺序相同。第一行执行，然后执行下一行，依此类推。在同步编程中，当程序遇到无法立即完成的操作时，它将阻塞，直到操作完成。例如，建立 TCP 连接需要通过网络与另一 ***peer(端)*** 进行交换，这可能需要相当长的时间。在此期间，线程被阻塞。             
使用异步编程，无法立即完成的操作将挂起到后台。线程没有被阻塞，可以继续运行其他东西。一旦操作完成，任务将不再挂起，并从停止的位置继续处理。我们之前的示例只有一个任务，因此挂起时不会发生任何事情，但异步程序通常有许多这样的任务。                
虽然异步编程可以编写更快的应用程序，但它通常会导致更复杂的程序。一旦异步操作完成，程序员需要跟踪 恢复工作所需的所有状态。从历史上看，这是一项乏味且容易出错的任务。            

## ***Compile-time(编译时) Green-threading(绿色线程)***
***Rust*** 使用名为 [`async/await`](https://en.wikipedia.org/wiki/Async/await) 的功能实现异步编程。执行异步操作的函数用 `async` 关键字标记。在我们的示例中，`connect` 函数定义类似于下面：
```Rust
use mini_redis::Result;
use mini_redis::client::Client;
use tokio::net::ToSocketAddrs;

pub async fn connect<T: ToSocketAddrs>(addr: T) -> Result<Client> {
   // ...
}
```
`async fn` 定义看起来像常规的同步函数，但操作是异步地。***Rust*** 在 ***编译时*** 将 `async fn` 转换为异步地运行的 一套例行操作。`async fn` 中对 `.await` 的任何调用都会将控制权返回给线程。当操作在后台处理时，线程可以执行其他工作。          
>尽管其他语言也实现了 [`async/await`](https://en.wikipedia.org/wiki/Async/await)，但 ***Rust*** 采用了一种独特的方法。首先，***Rust*** 的异步操作是 ***lazy(懒惰)*** 的。这导致与其他语言不同的运行时语义。

如果这还不太合理，不要担心。在本指南中，我们将进一步探讨 `async/await`。

## 使用 `async/await`
异步函数的调用与任何其他 ***Rust*** 函数一样。但是，调用这些函数不会导致函数体执行。替代的，调用一个 `async fn` 将返回一个表示操作的值。这在概念上类似于一个零参数闭包。要实际运行该操作，应在返回值上使用 `.await` 运算符。              
例如，给定程序：
```Rust
async fn say_world() {
   println!("World");
}

#[tokio::main]
async fn main() {
   // 调用 `say_world()` 不会置顶它的函数体
   let op = say_world();

   // 这个首先输出
   println!("hello");

   // 在 `op` 上调用 `.await` 开始执行 `say_world` 
   op.await;
}
```
输出：
>hello
>World

`async fn` 的返回值是实现 [`Future`](https://doc.rust-lang.org/std/future/trait.Future.html) 特征的匿名类型。

## 异步 `main` 函数
用于启动应用程序的主函数 与 在大多数 ***Rust*** 箱中找到的常见的 不同。
1. 它是一个 `async fn` 
2. 它用 `#[tokio::main]` 标注

当我们想要输入异步上下文时，使用 `async fn`。但是，异步函数必须由一个 [runtime](https://docs.rs/tokio/1/tokio/runtime/index.html) 执行。运行时包含异步任务调度程序，提供事件 I/O、计时器等。运行时不会自动启动，因此主函数需要启动它。             
`#[tokio::main]` 函数是一个宏。它将 `async fn main()` 转换为同步 `fn main()`，它初始化一个运行时实例 并 执行异步 main 函数。                
例如，下面：
```Rust
#[tokio::main]
async fn main() {
   println!("hello");
}
```
转化为：
```Rust
fn main() {
   let mut rt = tokio::runtime::Runtime::new().unwrap();
   rt.block_on(async {
      println!("hello");
   })
}
```
Tokio 运行时的细节将在后面介绍。

## Cargo 特征
当为了此教程依赖于 Tokio 时，`full` 特征标志位被打开：
```TOML
tokio = { version = "1", features = ["full"] }
```
Tokio 有很多功能(TCP、UDP、Unix 套接字、计时器、同步实用工具、多种调度程序类型，等)。并非所有应用程序都需要所有功能。当试图优化编译时间 或 最终应用程序占用时，应用程序可以决定 ***只*** 选择它使用的功能。              
目前，在依赖 Tokio 时使用 "full" 特征。