# まずはnightlyを入れる

みなさん、[TRPLの2章][trpl-ch2]は読みおえましたか? 今日やる内容を終えてしまって暇ですか? それではこの本で**フューチャーな**数当てゲームを作っていきましょう!

[trpl-ch2]: https://doc.rust-lang.org/book/second-edition/ch02-00-guessing-game-tutorial.html

Rustの未来へ駆けぬけるために、まずは**nightlyコンパイラ**のインストールをしましょう。 nightlyは[夜ごと生成されるナウでヤングな最先端バージョン][trpl-nightly]のことです。

[trpl-nightly]: https://doc.rust-lang.org/book/second-edition/appendix-07-nightly-rust.html

```
$ rustup toolchain install nightly
```

## nightlyツールチェインを使う

nightlyツールチェインを使うには3種類の方法があります。

### `+nightly` を指定する

皆さんが触っている `rustc` や `cargo` などのコマンドは rustup によってラップされており、 `+version` オプションにより指定のツールチェインで起動することができます。

```
$ rustc +nightly --version
rustc 1.30.0-nightly (33b923fd4 2018-08-18)
$ rustc --version
rustc 1.28.0 (9634041f0 2018-07-30)
```

### `rust-toolchain` ファイルを使う

`rust-toolchain` ファイルを置いておくと、その指定が使われます。

```
$ echo nightly > rust-toolchain
$ rustc --version
rustc 1.30.0-nightly (33b923fd4 2018-08-18)
$ rm rust-toolchain
$ rustc --version
rustc 1.28.0 (9634041f0 2018-07-30)
```

### `rustup override` を使う

特定ディレクトリのバージョンを固定する別の方法です。バージョン情報は `~/.rustup/settings.toml` に集約されます。

```
$ rustup override set nightly
$ rustc --version
rustc 1.30.0-nightly (33b923fd4 2018-08-18)
$ rustup override unset
$ rustc --version
```

## nightlyツールチェインを更新する

`rustup update` によりnightlyツールチェインも更新されます。ほぼ毎日更新されるので、特定バージョンを使い続けたい場合は `nightly-2018-08-05` のように日付つきのツールチェインを指定することもできます。(存在しない日付もあります)

## nightlyツールチェインでできること

nightlyツールチェインではRustの**最新機能** (不安定機能) を使うことができます。たとえば以下のコードは安定版コンパイラでは意図的に禁止されていますが、nightlyツールチェインではコンパイルできます。

```rust
// vec_resize_default という不安定機能を使うことを宣言する
#![feature(vec_resize_default)]

fn main() {
    let mut v = vec![1, 2, 3];
    // この関数は不安定
    v.resize_default(5);
    assert_eq!(v, [1, 2, 3, 0, 0]);
}
```

なぜ禁止されているのか? それは、この機能を**この形で提供しつづける(安定化する)合意がまだとれていない**からです。つまり、このような最新機能は[設計が変更されたり][#53227]、ときには[その機能が丸ごと削除されてしまう][#48333]かもしれないということです。

[#53227]: https://github.com/rust-lang/rust/pull/53227
[#48333]: https://github.com/rust-lang/rust/pull/48333

とはいっても、既存のコードが全く立ち行かなくなるような根本的なちゃぶ台返しはさすがに行われないでしょう。基本的には、新しい仕様への追従を迫られたり、今まで便利にやっていた方法が使えなくなったりするだけです。

これら不安定機能は[The Unstable Book][the-unstable-book]で解説されているほか、その安定化の状況を[State of Rust][state-of-rust]で確認できます。

[the-unstable-book]: https://doc.rust-lang.org/unstable-book/index.html
[state-of-rust]: https://forge.rust-lang.org/state-of-rust.html
