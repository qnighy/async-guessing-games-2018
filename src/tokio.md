# tokioのランタイムを起動する

ようやく非同期化を開始しましょう。非同期I/Oには [`tokio`][tokio] というライブラリを使います。

[tokio]: https://tokio.rs/

そもそもRustは昔、Go言語と同じようなグリーンスレッドを持っていました。これを捨て、紆余曲折あって再整備されつつあるのが今のfutures/tokioエコシステムです。グリーンスレッドを捨ててまで得たこの仕組みは以下のような利点があると考えられます。

- オプトインなので、使わないときはランタイムの空間的・時間的オーバーヘッドがかからない。
- スタックを丸ごと積み降ろしするオーバーヘッドがかからない。
- スタックの積み降ろしのような、プラットフォームに極端に依存する部分がない。Cの基本的な制御フローから逸脱しないため、他の処理との噛み合わせの問題も起きにくい。
- FFIの邪魔をしない。
- 所有権やライフタイムとの相性がよい。

非同期処理をどう組込むかはあとで考えるとして、tokioランタイムを起動してみましょう。

## tokioを依存に追加する

`Cargo.toml` に以下の行を追加します。

```toml
[dependencies]
...
tokio = "0.1.7"
```

対応する `extern crate tokio;` を `main.rs` に…… おっと、これはもうRust2018では要らないのでした。

## ランタイムを起動する

tokioを使うときには `tokio::prelude::*` を全てuseしておくと便利です。

```rust
use tokio::prelude::*;
```

さて、とりあえず既存のコードをtokioランタイムの中に入れてしまいましょう。mainの中身を丸ごと、以下のように囲んでしまいましょう。

```rust
// ...

fn main() {
    tokio::run(future::lazy(|| {
        println!("Guess the number!");

        // ...

        future::ok(())
    }));
}
```

実行して動作を確認してみてください。

ブロッキング処理をtokio内で実行しているので行儀は悪いですが、この状態でも一見非同期的に動作します。やってみましょう。

## 非同期的に実行する

`println!("Guess the number!");` の直前に、以下のようなコードをはさみます。

```rust
use std::time::{Duration, Instant};
use tokio::timer::Interval;

// ...

        let interval = Interval::new(Instant::now(), Duration::from_millis(1000));

        tokio::spawn(interval.for_each(|_| {
            println!("foo");
            future::ok(())
        }).map_err(|_| panic!("timer error")));

        println!("Guess the number!");

        // ...
```

もしあなたのPCがマルチコアなら、鬱陶しいメッセージが1秒おきに表示されつつ、数当てゲームが進行しているのが見えると思います。行儀の悪いコードなので、特定の場合だけ期待通りの動作をします。

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
tokio = "0.1.7"
```

`main.rs`

```rust
use tokio::prelude::*;

use std::io;
use std::cmp::Ordering;
use std::time::{Instant, Duration};
use rand::Rng;
use tokio::timer::Interval;

fn main() {
    tokio::run(future::lazy(|| {
        let interval = Interval::new(Instant::now(), Duration::from_millis(1000));

        tokio::spawn(interval.for_each(|_| {
            println!("foo");
            future::ok(())
        }).map_err(|_| panic!("timer error")));

        println!("Guess the number!");

        let secret_number = rand::thread_rng().gen_range(1, 101);

        loop {
            println!("Please input your guess.");

            let mut guess = String::new();

            io::stdin().read_line(&mut guess)
                .expect("Failed to read line");

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

        future::ok(())
    }));
}
```
