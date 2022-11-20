我们将换挡，并开始在 Redis 服务器上工作。                  
首先，将客户端 `SET`/`GET` 代码从上一节移动到一个示例文件。这样，我们可以在服务器上运行它。
>mkdir -p examples
>mv src/main.rs examples/hello-redis.rs

然后创建一个新的、空的 *src/main.rs* 然后继续。


# 接收 套接字
我们的 Redis 服务器需要做的第一件事是 接受入站 TCP 套接字。这是通过 [`tokio::net::TcpListener`](https://docs.rs/tokio/1/tokio/net/struct.TcpListener.html) 完成的。                
>Tokio 的许多类型的命名与 ***Rust*** 标准库中的 其同步地等价类型相同。当其有意义的情况下，Tokio 公开与 `std` 相同的 API，但使用 `async fn`。

一个 `TcpListener` 绑定到端口 **6379**，然后在循环中接受套接字。处理每个套接字，然后关闭。现在，我们将读取命令，将其打印到 stdout，并以错误响应。
```Rust
use tokio::net::{TcpListener, TcpStream};
use mini_redis::{Connection, Frame};

#[tokio::main]
async fn main() {
    // 绑定监听器到地址
    let listener = TcpListener::bind("127.0.0.1:6379").await.unwrap();

    loop {
        // 第二个项目包含新链接的 IP 和 端口
        let (socket, _) = listener.accept().await.unwrap();
        process(socket).await;
    }
}

async fn process(socket: TcpStream) {
    // `Connection` 让我们 read/write redis **frames** 而不是 byte streams. `Connection` 类型是由 mini-redis 定义的.
    let mut connection = Connection::new(socket);

    if let Some(frame) = connection.read_frame().await.unwrap() {
        println!("GOT: {:?}", frame);

        // 以一个 error 来响应
        let response = Frame::Error("unimplemented".to_string());
        connection.write_frame(&response).await.unwrap();
    }
}
```
现在运行这个接收循环：
>cargo run

在另一个终端中，运行 `hello-redis` 示例(上一节中的 `SET`/`GET` 命令)：
>cargo run --example hello-redis

输出应该为：
>Error: "unimplemented"

在服务器终端中，输出应该为
>GOT: Array([Bulk(b"set"), Bulk(b"hello"), Bulk(b"world")])


# 并发
我们的服务器有一个小问题(除了只响应错误)。它一次处理一个入站请求。当连接被接受时，服务器将停留在接受循环块中，直到响应完全被写入套接字。                 
我们希望我们的 Redis 服务器能够处理 **许多** 并发请求。为此，我们需要添加一些并发性。
>***Concurrency(并发)*** 和 ***Parallelism(并行)*** 不是一回事。如果您在两个任务之间交替，那么您正 ***concurrently(并发地)*** 处理这两个任务，但不是 ***parrllel(并行地)*** 处理。要使其成为并行的，您需要两个人，专门负责每个任务。              
>使用 Tokio 的优点之一是异步代码允许您并发地处理许多任务，而不必使用普通线程并行地处理它们。事实上，Tokio 可以在一个线程上同时运行许多任务！

要并发地处理连接，将为每个入站连接生成一个新任务。此任务将处理连接。          
接受循环变为：
```Rust
use tokio::net::TcpListener;

#[tokio::main]
async fn main() {
    let listener = TcpListener::bind("127.0.0.1:6379").await.unwrap();

    loop {
        let (socket, _) = listener.accept().await.unwrap();
        // 为每个入站连接生成一个新任务。套接字被 moved 到新的任务，并在那里被处理。
        tokio::spawn(async move {
            process(socket).await;
        });
    }
}
```

## ***Tasks(任务)***
一个 Tokio 任务是一个异步的绿色线程。它们是通过将 `async` 块传递给 `tokio::spawn` 来创建的。`tokio::spawn` 函数返回一个 `JoinHandle`，调用方可以使用它与生成的任务进行交互。`async` 块可以具有返回值。调用方可以在 `JoinHandle` 上使用 `.await` 获得返回值。                  
例如：
```Rust
#[tokio::main]
async fn main() {
    let handle = tokio::spawn(async {
        // 做些异步工作
        "return value"
    });

    // 做些其他工作

    let out = handle.await.unwrap();
    println!("GOT {}", out);
}
```
Awaiting on `JoinHandle` 返回一个 `Result`。当任务在执行过程中遇到错误时，`JoinHandle` 将返回 `Err`。当任务 panics，或者 运行时关闭强制取消任务时，都会发生这种情况。                      
任务 是由 ***scheduler(调度程序)*** 管理的 执行单元。生成任务将其提交给 Tokio 调度程序，然后确保任务在有工作要做时执行。生成的任务可能在与其生成的线程相同的线程上执行，也可能在不同的运行时线程上执行，任务也可能在生成后在线程之间 moved。                 
Tokio 的任务非常轻。幕后，它们只需要一次分配 和 64字节的内存。应用程序应该可以自由地生成数千(如果不是数百万)个任务。

## `'static` 约束
在 Tokio 运行时上 生成一个任务时，其类型的生命周期必须为 `'static`。这意味着生成的任务不能包含对任务外部拥有的数据的任何引用。            
>`'static` 总是意味着“永远存在”，***这是一种常见的误解***，但事实并非如此。仅仅因为值是 `'static` 并不意味着内存泄漏。您可以阅读更多：[Common Rust Lifetime Misconceptions](https://github.com/pretzelhammer/rust-blog/blob/master/posts/common-rust-lifetime-misconceptions.md#2-if-t-static-then-t-must-be-valid-for-the-entire-program)。

例如，以下内容将无法编译：
```Rust
use tokio::task;

#[tokio::main]
async fn main() {
    let v = vec![1, 2, 3];

    task::spawn(async {
        println!("Here's a vec: {:?}", v);
    });
}
```
尝试编译这个代码会导致以下错误：
```
error[E0373]: async block may outlive the current function, but
              it borrows `v`, which is owned by the current function
 --> src/main.rs:7:23
  |
7 |       task::spawn(async {
  |  _______________________^
8 | |         println!("Here's a vec: {:?}", v);
  | |                                        - `v` is borrowed here
9 | |     });
  | |_____^ may outlive borrowed value `v`
  |
note: function requires argument type to outlive `'static`
 --> src/main.rs:7:17
  |
7 |       task::spawn(async {
  |  _________________^
8 | |         println!("Here's a vector: {:?}", v);
9 | |     });
  | |_____^
help: to force the async block to take ownership of `v` (and any other
      referenced variables), use the `move` keyword
  |
7 |     task::spawn(async move {
8 |         println!("Here's a vec: {:?}", v);
9 |     });
  |
```

这是因为默认情况下，变量不会 ***moved*** 到异步块中。`v` 向量仍由 `main` 函数拥有。`println!` 行借用了 `v`。***Rust*** 编译器帮助我们解释了这一点，甚至提供了修复建议！将第 7 行更改为 `task::spawn(async move{` 将指示编译器将 `v` 移到生成的任务中。现在，任务拥有其所有数据，使其 `'static`。            
如果单个数据段必须能够被 同时从多个任务访问，则必须使用同步原语(如 `Arc`)共享该数据段。                   
请注意，错误消息讨论了 实参类型 ***outliving*** `'static` 生命周期。这个术语可能相当令人困惑，因为 `'static` 生命周期 持续到程序结束，所以如果它超过了它，你不会有内存泄漏吗？解释是，必须超过 `'static` 生命周期 的是 ***类型***，而不是 ***值***，并且该值可能在其类型不再有效之前被销毁。                   
当我们说一个值是 `'static` 时，这意味着永远保持该值是正确的。这一点很重要，因为编译器无法推断新生成的任务会停留多长时间，因此确保任务不会活得太久的唯一方法是确保它可能永远存在。                
信息框之前链接到的文章使用术语 “bounded by `'static`(由 `'static` 约束)”，而不是 “its type outlives `'static`(它的类型声明周期超过 `'static`)" or "the value is `'static`(值是 `'static`)” 来指 `T: 'static`。这些都是相同的意思，但不同于 `&'static T` 中的 “annotated with `'static`(标识为 `'static`)”。

## `Send` 约束
`tokio::spawn` 生成的任务必须实现 `Send`。这允许 Tokio 运行时在线程之间移动任务，当它们在 `.await` 挂起时。                
当任务 ***across(跨越)*** `.await` 时所持有的 ***全部*** 数据都是 `Send` 的 时候，任务也是 `Send` 的。这有点微妙。当调用 `.await` 时，任务 让步 给调度程序。下一次执行任务时，它将从上次 让步 的点恢复。要使这能够成功，任务必须保存 在 `.await` 之后使用的所有状态。如果此状态为 `Send`，即可以跨线程移动，则任务本身可以跨线程进行移动。相反，如果状态不是 `Send`，则任务也不是。                   
例如，这是有效的：
```Rust
use tokio::task::yield_now;
use std::rc::Rc;

#[tokio::main]
async fn main() {
    tokio::spawn(async {
        // 域强制 `rc` 在 `.await` 前 drop 掉.
        {
            let rc = Rc::new("hello");
            println!("{}", rc);
        }

        // `rc` 不再被使用。它 ***不会*** 持续到当任务让步给调度程序的时候
        yield_now().await;
    });
}
```
这个不行：
```Rust
use tokio::task::yield_now;
use std::rc::Rc;

#[tokio::main]
async fn main() {
    tokio::spawn(async {
        let rc = Rc::new("hello");

        // `rc` 在 `.await` 之后还被使用. 它必持续到任务的状态中。
        yield_now().await;

        println!("{}", rc);
    });
}
```
尝试编译，错误片段：
```
error: future cannot be sent between threads safely
   --> src/main.rs:6:5
    |
6   |     tokio::spawn(async {
    |     ^^^^^^^^^^^^ future created by async block is not `Send`
    | 
   ::: [..]spawn.rs:127:21
    |
127 |         T: Future + Send + 'static,
    |                     ---- required by this bound in
    |                          `tokio::task::spawn::spawn`
    |
    = help: within `impl std::future::Future`, the trait
    |       `std::marker::Send` is not  implemented for
    |       `std::rc::Rc<&str>`
note: future is not `Send` as this value is used across an await
   --> src/main.rs:10:9
    |
7   |         let rc = Rc::new("hello");
    |             -- has type `std::rc::Rc<&str>` which is not `Send`
...
10  |         yield_now().await;
    |         ^^^^^^^^^^^^^^^^^ await occurs here, with `rc` maybe
    |                           used later
11  |         println!("{}", rc);
12  |     });
    |     - `rc` is later dropped here
```

我们将在 [[5-共享的状态]] 更深入地讨论这种错误的一个特殊情况。


# 存储值
现在我们将实现 `process` 函数来处理传入的命令。我们将使用 `HashMap` 来存储值。`SET` 命令将插入 `HashMap`，`GET` 值将加载它们。此外，我们将使用循环接受每个连接的多个命令。
```Rust
use tokio::net::TcpStream;
use mini_redis::{Connection, Frame};

async fn process(socket: TcpStream) {
    use mini_redis::Command::{self, Get, Set};
    use std::collections::HashMap;

    // 一个用于存储数据的 hashmap
    let mut db = HashMap::new();

    // 链接，由 `mini-redis` 提供，控制 解析来自于套接字的帧
    let mut connection = Connection::new(socket);

    // 使用 `read_frame` 来从链接接收一个命令
    while let Some(frame) = connection.read_frame().await.unwrap() {
        let response = match Command::from_frame(frame).unwrap() {
            Set(cmd) => {
                // 值被存为 `Vec<u8>` 格式
                db.insert(cmd.key().to_string(), cmd.value().to_vec());
                Frame::Simple("OK".to_string())
            }
            Get(cmd) => {
                if let Some(value) = db.get(cmd.key()) {
                    // `Frame::Bulk` 期望数据为类型 `Bytes`。这种类型在教程后面会讲。目前为止，`&Vec<u8>` 使用 `into()` 方法被转换为 `Bytes`。
                    Frame::Bulk(value.clone().into())
                } else {
                    Frame::Null
                }
            }
            cmd => panic!("unimplemented {:?}", cmd),
        };

        // 写 到客户端的响应
        connection.write_frame(&response).await.unwrap();
    }
}
```
现在，开启服务器：
>cargo run

并且，在另一个终端中，运行 `hello-redis` 示例：
>cargo run --example hello-redis

然后，输出将为：
>got value from the server; result=Some(b"world")

我们现在可以获取和设置值，但存在一个问题：连接之间不共享值。如果另一个套接字连接并尝试 `GET` `hello` 键，它将找不到任何东西。                 
你可以在 [这里](https://github.com/tokio-rs/website/blob/master/tutorial-code/spawning/src/main.rs) 找到完整的代码。                      
在下一节中，我们将为所有套接字实现持久化数据。
