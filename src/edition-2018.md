# 2019年風のRustに衣替え

これを読んでいるあなたは、**普通の**数当てゲームを作り終えていることでしょう。その数当てゲームは……こんな感じでしたか?

```rust
extern crate rand;

use std::io;
use std::cmp::Ordering;
use rand::Rng;

fn main() {
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
}
```

これを非同期にしようと思いますが、その前に2019年に備えて、これを**Rust2018**に対応させましょう! (Rust2019じゃないよ)

Rust2018はRust2015(現行のRust)にいくつかの非互換な変更を加えています。非互換というと聞こえは悪いですが。既存のRustエコシステムとの互換性はほぼ100%です! 各ライブラリの作者が、気が向いたときにRust2018に切り替えればいいだけです。コンパイラは両方のエディションをサポートしつづけますし、異なるエディションのライブラリも問題なく混在できます。

それでは実際に[edition guideの記述][edition-guide-transitioning]にしたがって、数当てゲームをRust2018に移行しましょう!

[edition-guide-transitioning]: https://rust-lang-nursery.github.io/edition-guide/editions/transitioning-your-code-to-a-new-edition.html

## nightlyを有効化

Rust2018は2018年末に安定化される予定なので、まだnightlyが必要です。[前のセクション](./get-nightly.md)で説明した方法で、nightlyを有効化しましょう。nightlyを新しめのものにするのも忘れずに。

```
$ rustup update nightly
$ echo nightly > rust-toolchain
```

## プレビューモードを有効化

不安定機能を使うにはもう一つ、`feature` を入れる必要がありました。そこで以下の行を先頭に追加します。

```rust
#![feature(rust_2018_preview)]

...
```

この時点ではまだRust2018にはなっていません。あくまで、2018を有効化する準備ができただけです。

## Rust2018対応版に変換

`cargo fix` を使うと、半自動で新しいRustのコードに変換することができます。実際にやってみましょう。

```
$ cargo fix --edition
```

何も起きませんでしたか? 今回は簡単な数当てゲームで、特に非互換な要素もありませんでした。ちなみに、より複雑なプロジェクトでは、非互換な部分が自動で修正されたり、自動修正できない部分については警告が表示されたりします。

この時点では、「Rust2015でもRust2018でも動かせる状態」になっています。

## Rust2018に切り替え

`Cargo.toml` に以下の1行を追加することで、Rust2018への切り替えを指示できます。

```toml
[package]
...
edition = "2018"
```

ところが、この `edition = ...` という機能自体もcargoの不安定機能です。そこで、 `cargo-features = ["edition"]` を**先頭に**追加する必要があります。結果として、 `Cargo.toml` は以下のようになります。

```toml
cargo-features = ["edition"]

[package]
name = "guessing_game"
version = "0.1.0"
authors = ["Foo Bar <foobar@example.com>"]

edition = "2018"

[dependencies]
rand = "0.4.0"
```

## Rust2018らしい書き方にする

Rust2018では推奨されている作法も少し変化しています。これに対応するには `cargo fix --edition-idioms` を実行します。

```
$ cargo fix --edition-idioms
```

こうすると、Rust2018的でない作法が半自動で修正されます。

数当ての場合は、はじめに書いた `#![feature(rust_2018_preview)]` と、 `extern crate rand;` の削除が指示されると思います。

結局、以下のように、 `extern crate` が削除された状態に落ち着くことになりました。

```rust
use std::io;
use std::cmp::Ordering;
use rand::Rng;

fn main() {
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
}
```

この **`extern crate` が要らなくなる**のはRust2018の嬉しい点のひとつです。同じことを2回書かなくてすむようになります。
