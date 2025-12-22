---
title: "Rust勉強中によく分からなかったこと"
emoji: "📛"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["rust"]
published: true
---
# はじめに
この記事は自分がZedのExtentionを作りながらとか、Advent Of Codeを解きながらRustの勉強をした時に、よく分からない構文が出てきたので備忘録的に書きます。
勉強しながら調べながらこの記事を書くので、たまに覗くと増えてる可能性があります。（12/22現在）

## 対象読者
- Rustを勉強し始めて理解できない構文を見つけた人

# この構文ってどういう意味？
## use std::io::{self, Read};
この構文を見たのは、[Advent of Code - 2025](https://adventofcode.com/2025)の問題を解いている時です。
問題から与えられる入力を読み取るための標準入力ライブラリの`use`つまりインポートを宣言していますが、`self`ってなんなの？となりました。
```rust
use std::io::{self, Read};
```
どうやらこの`self`はPythonのクラスでいう`self`と同じ働きをするそうです。つまり、インポートしているモジュール自身を指します。
公式ドキュメントの記載の通り、このプログラムは`std::io`モジュールと`std::io::Read`モジュールを同時にインポートするようです。
```rust
use std::io;
use std::io::Read;
```
https://doc.rust-lang.org/std/keyword.self.html

## (start..end).contains(&point)
ちょっとしたコーディング問題を解いてる時に、ある値が特定の範囲に含まれるかどうかを調べるマクロを探している時に知りました。
```rust
let contains = if (start..end).contains(&point);
```
`..`や`contains`、`&`の意味が分からなかったので記事にします。
まず`..`は演算子（Operator）のひとつらしいです。
`..`, `expr..`, `..expr`, `expr..expr`のような使い方により、指定した範囲の未満範囲を判定します。
よって、`start..end`は`start`以上`end`未満の範囲を表します。
```rust
(start..end) == { start <= point || point < end }
```

`contains`は`Range`構造体に付随するメソッドのひとつです。
https://doc.rust-lang.org/std/ops/struct.Range.html#method.contains

```rust
<Range>.contains(&item)
```
このような書き方をすることで、引数に与えられた`item`が`<Range>`オブジェクトの中に存在するか判定し、`<T>`または`<F>`を返します。

じゃあこの`&`ってなんなんだという話ですが、これはRustのコア思想の"所有権"・"借用"に関係するオペレーターのようです。
所有権とは、「すべてのメモリ上のデータには"持ち主"がいる」ことを決定する仕組みであり、プログラム中で発生するメモリ上のデータはすべてこのルールに基づきます。
あるデータに対して所有権を持つ"持ち主"は一人で、"持ち主"が消えるとデータは自動で破棄されます。
借用とは、「"所有権"を渡さずにデータを参照・利用する仕組み」のことです。
多くのプログラムにおいてデータとは所有権を移動させずにアクセスだけもらえれば充分であることが多いです。そこで、この"借用"の仕組みを使ってデータに対して閲覧権限を得るわけです。
Rustはメモリ安全性が保証されたモダン言語だと言われていますが、その理由のひとつがこれです。
"借用"には不変借用と可変借用の2種類があります。
```rust
&point      -> 参照先を変更せず利用する
&mut point  -> 参照先を変更して利用する
```
しかし、今回の場合入力で指定されたポイントが指定の範囲に含まれているか調べるだけなので、不変借用で充分です。

さて、ここまでの話をまとめると次のようになります。
```rust
// let contains: ifから返される<T>または<F>を格納する変数の宣言
// if {...}: 条件分岐
// (start..end): start<= .. < end を満たす<Range>オブジェクトの生成
// contains(&point): 引数に渡された変数が含まれているか判定する<Range>メソッド
// &point: 入力から渡された値を読み取るための"不変借用"

let contains = if (start..end).contains(&point);
```

## || Path::new("")
この構文を見たのはワークスペースのディレクトリ構造を操作するStructメソッドを記述している時です。
```rust
fn upsert_file_items(&mut self, file: &Path, items: Vec<TodoItem>) {
        let dir = file
            .parent()
            .unwrap_or_else(|| Path::new("/"))
            .to_path_buf();
        let files = self.data.entry(dir).or_default();
        files.insert(file.to_path_buf(), items);
    }
```

このうちの`.unwrap_or_else(|| Path::new("/"))`という部分。
これはPythonでいうところのLambda関数のようです。RustだとClosure（クロージャー）。
```rust
// 基本構文: |引数| { 処理ブロック }
// ※処理が単一の式の場合は{}を省略できる
let add_one = |x| x + 1; // |x| { x + 1 } と同じ意味

println!("{}", add_one(5)); // 出力: 6
```
これを前提にさっきの箇所を見てみましょう。
Closureの条件をもとに分解すると以下のようになります。
```rust
// || : 引数がないことを示す // Path::new("/") : Pathインスタンスとしてルートディレクトリを生成
// 処理が単一式なので{}を省略
unwrap_or_else(|| Path::new("/"))
```
つまり、関数`upsert_file_items`とは、Pathインスタンスである`file`を受け取り、`.parent`メソッドでその親ディレクトリを探索します。
`file.parent()`の返り値は結果の有無を示す`Option<&Path>`です。
- `Some(&Path)`: 親ディレクトリが存在する場合
- `None`: 親ディレクトリが存在しない場合

`.unwrap_or_else`は`Option`が`None`だった場合に**引数として渡されたClosureを実行**し、代替の値としてルートディレクトリ`/`が返されるわけです。


# おわりに
この備忘録がRust初心者の皆さんの役に立てば嬉しいです。
「これ本当にそうなの？」「ここ間違ってると思う！」などなどコメントしてください。
