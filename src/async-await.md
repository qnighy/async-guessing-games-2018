# async/awaitを使う

[前の章](./tokio.md) のコードはブロッキング処理をtokioに食わせているのでとても行儀が悪いです。これに対処するには3つの方法があります。

- ノンブロッキングな処理に置き換える。 (正攻法)
- [`tokio_threadpool::blocking`](https://docs.rs/tokio-threadpool/0.1/tokio_threadpool/fn.blocking.html)を使う。 (ブロッキングI/Oをどうしても使う必要があるときの方法)
- [`futures_cpupool`](https://docs.rs/futures-cpupool/0.1.8/futures_cpupool/index.html)を使う。 (CPU中心の処理をイベントループから退避させる場合に最適な方法)

今回は正攻法を使いましょう。

とは言っても、このまま安定版の方法を使っていると[コールバック地獄][callback-hell]に突入してしまいます。そこで、近未来的な不安定版機能のasync/awaitを使いましょう!

[callback-hell]: http://callbackhell.com/

## futures-previewを使う

さて、ここで悲しいお知らせがあります。現行のエコシステムで使われている `future::Future` 型と、async/awaitが使う `std::future::Future` には互換性がありません。

後者を扱うためのライブラリは `futures-preview` の0.3系列です。現行のシステムは `futures` の0.1系列で、0.2は色々な理由があって欠番になりました。

0.1系列の `Future` と 0.3系列の `Future` に互換性はありませんが、互いを行き来するための手段が `futures-preview` に実装されているので、それを使うことにしましょう。

`Cargo.toml` に以下の2行を追加します。

```toml
[dependencies]
...
pin-utils = "0.1.0-alpha.1"
# 互換性機能 "tokio-compat" はデフォルト無効なので有効化しておく
futures-preview = { version = "0.3.0-alpha.3", features = ["tokio-compat"] }
```

そういえば`futures-preview` のAPIドキュメントがありませんね。 `cargo doc` で生成しておくといいでしょう。

```
$ cargo doc --open
```

## futures-previewを使う(2)

futures-previewを使いたいので、以下のインポートを追加しておきます。

```rust
use futures::prelude::*;
use futures::compat::{Future01CompatExt, Stream01CompatExt, TokioDefaultSpawn};
```

逆に、 `tokio::prelude::*` のインポートはfutures-preview 0.3系列と衝突してしまうため、名前を変えておきます。

```rust
// use tokio::prelude::*;
use tokio::prelude::{Stream as Stream01, Future as Future01, future as future01};
```

あわせて、 `future::ok` と `future::lazy` も `future01::ok` と `future01::lazy` に変えておきます。

## asyncブロックを使う

async/awaitを使うために、ソースコードの先頭にfeature宣言を足します。

```rust
#![feature(async_await, await_macro, futures_api, pin)]

// ...
```

`tokio::run` で囲った部分を以下のように書き換えます。

```rust
fn main() {
    tokio::run(async {
        // ...

        Ok(())
    }.boxed().compat());
}
```

`async` ブロックが出てきたので、これで `await!( ... )` が使える状態になりました。

## awaitを使ってみる

ブロッキングI/Oをなくすにはもう少し手間がかかるので、まずは簡単なawaitを入れてみましょう。

```rust

// ...

use tokio::timer::{Delay, Interval};

        // ...

        loop {
            await!(Delay::new(Instant::now() + Duration::from_millis(500)).compat());

            println!("Please input your guess.");

            // ...
```

この `await!` の位置でプログラムは0.5秒停止したかのように動作するはずです。実際には、この位置でいったん呼び出し元に復帰し、0.5秒後にここに戻ってきて処理を再開するという処理が動いています。

## 標準入力をasyncにする

標準入力を非同期にするには [`tokio-stdin-stdout`][tokio-stdin-stdout] や [`tokio-file-unix`][tokio-file-unix] が使えます。Windowsユーザーのことも考えてここでは `tokio-stdin-stdout` を使ってみます。

[tokio-stdin-stdout]: https://crates.io/crates/tokio-stdin-stdout
[tokio-file-unix]: https://crates.io/crates/tokio-file-unix

`Cargo.toml` に以下の行を追加します。

```toml
[dependencies]
...
tokio-stdin-stdout = "0.1.4"
```


標準入力をあらかじめ非同期入力に変換しておく必要があります。

```rust
use std::io::BufReader;


        // ...
        }).map_err(|_| panic!("timer error")));

        let stdio = tokio_stdin_stdout::stdin(0);
        let stdio = BufReader::new(stdio);
        let lines = tokio::io::lines(stdio).compat();

        println!("Guess the number!");
        // ...
```

`StreamExt` の `next` メソッドで 1行ずつ取り出すことができます。これはFutureを返すので、 `await!` で包みます。

```rust
            // ...
            println!("Please input your guess.");

            let guess = await!(lines.next())
                .expect("End of input")
                .expect("I/O error");

            let guess: u32 = match guess.trim().parse() {
            // ...
```

この状態でコンパイルすると巨大なエラーが出力されます。

```
error[E0277]: the trait bound `(dyn futures::task_impl::std::Unpark + 'static): std::marker::Unpin` is not satisfied in `futures_util::compat::compat::Compat<tokio_io::lines::Lines<std::io::BufReader<tokio_stdin_stdout::ThreadedStdin>>, ()>`
  --> src/main.rs:35:38
   |
35 |             let guess = await!(lines.next())
   |                                      ^^^^ within `futures_util::compat::compat::Compat<tokio_io::lines::Lines<std::io::BufReader<tokio_stdin_stdout::ThreadedStdin>>, ()>`, the trait `std::marker::Unpin` is not implemented for `(dyn futures::task_impl::std::Unpark + 'static)`
   |
   = note: required because it appears within the type `std::marker::PhantomData<(dyn futures::task_impl::std::Unpark + 'static)>`
   = note: required because it appears within the type `std::sync::Arc<(dyn futures::task_impl::std::Unpark + 'static)>`
   = note: required because it appears within the type `futures::task_impl::std::TaskUnpark`
   = note: required because it appears within the type `futures::task_impl::Task`
   = note: required because it appears within the type `std::option::Option<futures::task_impl::Task>`
   = note: required because it appears within the type `futures::sync::mpsc::ReceiverTask`
   = note: required because it appears within the type `std::cell::UnsafeCell<futures::sync::mpsc::ReceiverTask>`
   = note: required because it appears within the type `std::sync::Mutex<futures::sync::mpsc::ReceiverTask>`
   = note: required because it appears within the type `futures::sync::mpsc::Inner<std::boxed::Box<[u8]>>`
   = note: required because it appears within the type `std::marker::PhantomData<futures::sync::mpsc::Inner<std::boxed::Box<[u8]>>>`
   = note: required because it appears within the type `std::sync::Arc<futures::sync::mpsc::Inner<std::boxed::Box<[u8]>>>`
   = note: required because it appears within the type `futures::sync::mpsc::Receiver<std::boxed::Box<[u8]>>`
   = note: required because it appears within the type `tokio_stdin_stdout::ThreadedStdin`
   = note: required because it appears within the type `std::io::BufReader<tokio_stdin_stdout::ThreadedStdin>`
   = note: required because it appears within the type `tokio_io::lines::Lines<std::io::BufReader<tokio_stdin_stdout::ThreadedStdin>>`
   = note: required because it appears within the type `futures_util::compat::compat::Compat<tokio_io::lines::Lines<std::io::BufReader<tokio_stdin_stdout::ThreadedStdin>>, ()>`

error[E0277]: the trait bound `(dyn futures::task_impl::UnsafeNotify + 'static): std::marker::Unpin` is not satisfied in `futures_util::compat::compat::Compat<tokio_io::lines::Lines<std::io::BufReader<tokio_stdin_stdout::ThreadedStdin>>, ()>`
  --> src/main.rs:35:38
   |
35 |             let guess = await!(lines.next())
   |                                      ^^^^ within `futures_util::compat::compat::Compat<tokio_io::lines::Lines<std::io::BufReader<tokio_stdin_stdout::ThreadedStdin>>, ()>`, the trait `std::marker::Unpin` is not implemented for `(dyn futures::task_impl::UnsafeNotify + 'static)`
   |
   = note: required because it appears within the type `*mut (dyn futures::task_impl::UnsafeNotify + 'static)`
   = note: required because it appears within the type `futures::task_impl::NotifyHandle`
   = note: required because it appears within the type `futures::task_impl::core::TaskUnpark`
   = note: required because it appears within the type `futures::task_impl::std::TaskUnpark`
   = note: required because it appears within the type `futures::task_impl::Task`
   = note: required because it appears within the type `std::option::Option<futures::task_impl::Task>`
   = note: required because it appears within the type `futures::sync::mpsc::ReceiverTask`
   = note: required because it appears within the type `std::cell::UnsafeCell<futures::sync::mpsc::ReceiverTask>`
   = note: required because it appears within the type `std::sync::Mutex<futures::sync::mpsc::ReceiverTask>`
   = note: required because it appears within the type `futures::sync::mpsc::Inner<std::boxed::Box<[u8]>>`
   = note: required because it appears within the type `std::marker::PhantomData<futures::sync::mpsc::Inner<std::boxed::Box<[u8]>>>`
   = note: required because it appears within the type `std::sync::Arc<futures::sync::mpsc::Inner<std::boxed::Box<[u8]>>>`
   = note: required because it appears within the type `futures::sync::mpsc::Receiver<std::boxed::Box<[u8]>>`
   = note: required because it appears within the type `tokio_stdin_stdout::ThreadedStdin`
   = note: required because it appears within the type `std::io::BufReader<tokio_stdin_stdout::ThreadedStdin>`
   = note: required because it appears within the type `tokio_io::lines::Lines<std::io::BufReader<tokio_stdin_stdout::ThreadedStdin>>`
   = note: required because it appears within the type `futures_util::compat::compat::Compat<tokio_io::lines::Lines<std::io::BufReader<tokio_stdin_stdout::ThreadedStdin>>, ()>`

error[E0277]: the trait bound `(dyn futures::task_impl::std::EventSet + 'static): std::marker::Unpin` is not satisfied in `futures_util::compat::compat::Compat<tokio_io::lines::Lines<std::io::BufReader<tokio_stdin_stdout::ThreadedStdin>>, ()>`
  --> src/main.rs:35:38
   |
35 |             let guess = await!(lines.next())
   |                                      ^^^^ within `futures_util::compat::compat::Compat<tokio_io::lines::Lines<std::io::BufReader<tokio_stdin_stdout::ThreadedStdin>>, ()>`, the trait `std::marker::Unpin` is not implemented for `(dyn futures::task_impl::std::EventSet + 'static)`
   |
   = note: required because it appears within the type `std::marker::PhantomData<(dyn futures::task_impl::std::EventSet + 'static)>`
   = note: required because it appears within the type `std::sync::Arc<(dyn futures::task_impl::std::EventSet + 'static)>`
   = note: required because it appears within the type `futures::task_impl::std::UnparkEvent`
   = note: required because it appears within the type `futures::task_impl::std::UnparkEvents`
   = note: required because it appears within the type `futures::task_impl::Task`
   = note: required because it appears within the type `std::option::Option<futures::task_impl::Task>`
   = note: required because it appears within the type `futures::sync::mpsc::ReceiverTask`
   = note: required because it appears within the type `std::cell::UnsafeCell<futures::sync::mpsc::ReceiverTask>`
   = note: required because it appears within the type `std::sync::Mutex<futures::sync::mpsc::ReceiverTask>`
   = note: required because it appears within the type `futures::sync::mpsc::Inner<std::boxed::Box<[u8]>>`
   = note: required because it appears within the type `std::marker::PhantomData<futures::sync::mpsc::Inner<std::boxed::Box<[u8]>>>`
   = note: required because it appears within the type `std::sync::Arc<futures::sync::mpsc::Inner<std::boxed::Box<[u8]>>>`
   = note: required because it appears within the type `futures::sync::mpsc::Receiver<std::boxed::Box<[u8]>>`
   = note: required because it appears within the type `tokio_stdin_stdout::ThreadedStdin`
   = note: required because it appears within the type `std::io::BufReader<tokio_stdin_stdout::ThreadedStdin>`
   = note: required because it appears within the type `tokio_io::lines::Lines<std::io::BufReader<tokio_stdin_stdout::ThreadedStdin>>`
   = note: required because it appears within the type `futures_util::compat::compat::Compat<tokio_io::lines::Lines<std::io::BufReader<tokio_stdin_stdout::ThreadedStdin>>, ()>`

error[E0277]: the trait bound `(dyn futures::task_impl::std::Unpark + 'static): std::marker::Unpin` is not satisfied in `futures_util::compat::compat::Compat<tokio_io::lines::Lines<std::io::BufReader<tokio_stdin_stdout::ThreadedStdin>>, ()>`
  --> src/main.rs:35:25
   |
35 |             let guess = await!(lines.next())
   |                         ^^^^^^^^^^^^^^^^^^^^ within `futures_util::compat::compat::Compat<tokio_io::lines::Lines<std::io::BufReader<tokio_stdin_stdout::ThreadedStdin>>, ()>`, the trait `std::marker::Unpin` is not implemented for `(dyn futures::task_impl::std::Unpark + 'static)`
   |
   = note: required because it appears within the type `std::marker::PhantomData<(dyn futures::task_impl::std::Unpark + 'static)>`
   = note: required because it appears within the type `std::sync::Arc<(dyn futures::task_impl::std::Unpark + 'static)>`
   = note: required because it appears within the type `futures::task_impl::std::TaskUnpark`
   = note: required because it appears within the type `futures::task_impl::Task`
   = note: required because it appears within the type `std::option::Option<futures::task_impl::Task>`
   = note: required because it appears within the type `futures::sync::mpsc::ReceiverTask`
   = note: required because it appears within the type `std::cell::UnsafeCell<futures::sync::mpsc::ReceiverTask>`
   = note: required because it appears within the type `std::sync::Mutex<futures::sync::mpsc::ReceiverTask>`
   = note: required because it appears within the type `futures::sync::mpsc::Inner<std::boxed::Box<[u8]>>`
   = note: required because it appears within the type `std::marker::PhantomData<futures::sync::mpsc::Inner<std::boxed::Box<[u8]>>>`
   = note: required because it appears within the type `std::sync::Arc<futures::sync::mpsc::Inner<std::boxed::Box<[u8]>>>`
   = note: required because it appears within the type `futures::sync::mpsc::Receiver<std::boxed::Box<[u8]>>`
   = note: required because it appears within the type `tokio_stdin_stdout::ThreadedStdin`
   = note: required because it appears within the type `std::io::BufReader<tokio_stdin_stdout::ThreadedStdin>`
   = note: required because it appears within the type `tokio_io::lines::Lines<std::io::BufReader<tokio_stdin_stdout::ThreadedStdin>>`
   = note: required because it appears within the type `futures_util::compat::compat::Compat<tokio_io::lines::Lines<std::io::BufReader<tokio_stdin_stdout::ThreadedStdin>>, ()>`
   = note: required because of the requirements on the impl of `core::future::future::Future` for `futures_util::stream::next::Next<'_, futures_util::compat::compat::Compat<tokio_io::lines::Lines<std::io::BufReader<tokio_stdin_stdout::ThreadedStdin>>, ()>>`
   = note: required by `std::future::poll_in_task_cx`
   = note: this error originates in a macro outside of the current crate (in Nightly builds, run with -Z external-macro-backtrace for more info)

error[E0277]: the trait bound `(dyn futures::task_impl::UnsafeNotify + 'static): std::marker::Unpin` is not satisfied in `futures_util::compat::compat::Compat<tokio_io::lines::Lines<std::io::BufReader<tokio_stdin_stdout::ThreadedStdin>>, ()>`
  --> src/main.rs:35:25
   |
35 |             let guess = await!(lines.next())
   |                         ^^^^^^^^^^^^^^^^^^^^ within `futures_util::compat::compat::Compat<tokio_io::lines::Lines<std::io::BufReader<tokio_stdin_stdout::ThreadedStdin>>, ()>`, the trait `std::marker::Unpin` is not implemented for `(dyn futures::task_impl::UnsafeNotify + 'static)`
   |
   = note: required because it appears within the type `*mut (dyn futures::task_impl::UnsafeNotify + 'static)`
   = note: required because it appears within the type `futures::task_impl::NotifyHandle`
   = note: required because it appears within the type `futures::task_impl::core::TaskUnpark`
   = note: required because it appears within the type `futures::task_impl::std::TaskUnpark`
   = note: required because it appears within the type `futures::task_impl::Task`
   = note: required because it appears within the type `std::option::Option<futures::task_impl::Task>`
   = note: required because it appears within the type `futures::sync::mpsc::ReceiverTask`
   = note: required because it appears within the type `std::cell::UnsafeCell<futures::sync::mpsc::ReceiverTask>`
   = note: required because it appears within the type `std::sync::Mutex<futures::sync::mpsc::ReceiverTask>`
   = note: required because it appears within the type `futures::sync::mpsc::Inner<std::boxed::Box<[u8]>>`
   = note: required because it appears within the type `std::marker::PhantomData<futures::sync::mpsc::Inner<std::boxed::Box<[u8]>>>`
   = note: required because it appears within the type `std::sync::Arc<futures::sync::mpsc::Inner<std::boxed::Box<[u8]>>>`
   = note: required because it appears within the type `futures::sync::mpsc::Receiver<std::boxed::Box<[u8]>>`
   = note: required because it appears within the type `tokio_stdin_stdout::ThreadedStdin`
   = note: required because it appears within the type `std::io::BufReader<tokio_stdin_stdout::ThreadedStdin>`
   = note: required because it appears within the type `tokio_io::lines::Lines<std::io::BufReader<tokio_stdin_stdout::ThreadedStdin>>`
   = note: required because it appears within the type `futures_util::compat::compat::Compat<tokio_io::lines::Lines<std::io::BufReader<tokio_stdin_stdout::ThreadedStdin>>, ()>`
   = note: required because of the requirements on the impl of `core::future::future::Future` for `futures_util::stream::next::Next<'_, futures_util::compat::compat::Compat<tokio_io::lines::Lines<std::io::BufReader<tokio_stdin_stdout::ThreadedStdin>>, ()>>`
   = note: required by `std::future::poll_in_task_cx`
   = note: this error originates in a macro outside of the current crate (in Nightly builds, run with -Z external-macro-backtrace for more info)

error[E0277]: the trait bound `(dyn futures::task_impl::std::EventSet + 'static): std::marker::Unpin` is not satisfied in `futures_util::compat::compat::Compat<tokio_io::lines::Lines<std::io::BufReader<tokio_stdin_stdout::ThreadedStdin>>, ()>`
  --> src/main.rs:35:25
   |
35 |             let guess = await!(lines.next())
   |                         ^^^^^^^^^^^^^^^^^^^^ within `futures_util::compat::compat::Compat<tokio_io::lines::Lines<std::io::BufReader<tokio_stdin_stdout::ThreadedStdin>>, ()>`, the trait `std::marker::Unpin` is not implemented for `(dyn futures::task_impl::std::EventSet + 'static)`
   |
   = note: required because it appears within the type `std::marker::PhantomData<(dyn futures::task_impl::std::EventSet + 'static)>`
   = note: required because it appears within the type `std::sync::Arc<(dyn futures::task_impl::std::EventSet + 'static)>`
   = note: required because it appears within the type `futures::task_impl::std::UnparkEvent`
   = note: required because it appears within the type `futures::task_impl::std::UnparkEvents`
   = note: required because it appears within the type `futures::task_impl::Task`
   = note: required because it appears within the type `std::option::Option<futures::task_impl::Task>`
   = note: required because it appears within the type `futures::sync::mpsc::ReceiverTask`
   = note: required because it appears within the type `std::cell::UnsafeCell<futures::sync::mpsc::ReceiverTask>`
   = note: required because it appears within the type `std::sync::Mutex<futures::sync::mpsc::ReceiverTask>`
   = note: required because it appears within the type `futures::sync::mpsc::Inner<std::boxed::Box<[u8]>>`
   = note: required because it appears within the type `std::marker::PhantomData<futures::sync::mpsc::Inner<std::boxed::Box<[u8]>>>`
   = note: required because it appears within the type `std::sync::Arc<futures::sync::mpsc::Inner<std::boxed::Box<[u8]>>>`
   = note: required because it appears within the type `futures::sync::mpsc::Receiver<std::boxed::Box<[u8]>>`
   = note: required because it appears within the type `tokio_stdin_stdout::ThreadedStdin`
   = note: required because it appears within the type `std::io::BufReader<tokio_stdin_stdout::ThreadedStdin>`
   = note: required because it appears within the type `tokio_io::lines::Lines<std::io::BufReader<tokio_stdin_stdout::ThreadedStdin>>`
   = note: required because it appears within the type `futures_util::compat::compat::Compat<tokio_io::lines::Lines<std::io::BufReader<tokio_stdin_stdout::ThreadedStdin>>, ()>`
   = note: required because of the requirements on the impl of `core::future::future::Future` for `futures_util::stream::next::Next<'_, futures_util::compat::compat::Compat<tokio_io::lines::Lines<std::io::BufReader<tokio_stdin_stdout::ThreadedStdin>>, ()>>`
   = note: required by `std::future::poll_in_task_cx`
   = note: this error originates in a macro outside of the current crate (in Nightly builds, run with -Z external-macro-backtrace for more info)

error: aborting due to 6 previous errors

For more information about this error, try `rustc --explain E0277`.
error: Could not compile `guessing_game`.

To learn more, run the command again with --verbose.
```

ここでポイントは ``the trait bound `~~~: std::marker::Unpin` is not satisfied`` という部分です。 `Unpin` はasync/awaitのために新たに導入されたトレイトで、ムーブできない特殊な状況のために使われます。

この場合、 `lines` がムーブされないことを保証する必要があるので、 `pin_mut!` マクロで固定してしまいます。

```rust
#[macro_use]
extern crate pin_utils;

        // ...
        let lines = tokio::io::lines(stdio).compat();

        pin_mut!(lines);

        println!("Guess the number!");
        // ...
```

これで標準入力が非同期になりました。

## ここまでのソースコード

`Cargo.toml`

```toml
cargo-features = ["edition"]

[package]
name = "guessing_game"
version = "0.1.0"
authors = ["Foo Bar <foobar@example.com>"]

edition = "2018"

[dependencies]
rand = "0.4.0"
pin-utils = "0.1.0-alpha.1"
futures-preview = { version = "0.3.0-alpha.3", features = ["tokio-compat"] }
tokio = "0.1.7"
tokio-stdin-stdout = "0.1.4"
```

`main.rs`

```rust
#![feature(async_await, await_macro, futures_api, pin)]

#[macro_use]
extern crate pin_utils;

use futures::prelude::*;
use tokio::prelude::{Stream as Stream01, Future as Future01, future as future01};

use std::io::BufReader;
use std::cmp::Ordering;
use std::time::{Instant, Duration};
use rand::Rng;
use tokio::timer::{Delay, Interval};
use futures::compat::{Future01CompatExt, Stream01CompatExt, TokioDefaultSpawn};

fn main() {
    tokio::run(async {
        let interval = Interval::new(Instant::now(), Duration::from_millis(1000));

        tokio::spawn(interval.for_each(|_| {
            println!("foo");
            future01::ok(())
        }).map_err(|_| panic!("timer error")));

        let stdio = tokio_stdin_stdout::stdin(0);
        let stdio = BufReader::new(stdio);
        let lines = tokio::io::lines(stdio).compat();

        pin_mut!(lines);

        println!("Guess the number!");

        let secret_number = rand::thread_rng().gen_range(1, 101);

        loop {
            await!(Delay::new(Instant::now() + Duration::from_millis(500)).compat());

            println!("Please input your guess.");

            let guess = await!(lines.next())
                .expect("End of input")
                .expect("I/O error");

            let guess: u32 = match guess.trim().parse() {
                Ok(num) => num,
                Err(_) => continue,
            };

            println!("You guessed: {}", guess);

            match guess.cmp(&secret_number) {
                Ordering::Less => println!("Too small!"),
                Ordering::Greater => println!("Too big!"),
                Ordering::Equal => {
                    println!("You win!");
                    break;
                }
            }
        }

        Ok(())
    }.boxed().compat(TokioDefaultSpawn));
}
```
