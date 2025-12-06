---
title: "Rust勉強中によく分からなかったこと"
emoji: "🎉"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["rust"]
published: true
---
# はじめに
この記事は自分がZedのExtentionを作りながらRustの勉強をした時に、よく分からない構文が出てきたので備忘録的に書きます。
勉強しながら調べながらこの記事を書くので、たまに覗くと増えてる可能性があります。（12/6現在）

## 対象読者
- Rustを勉強し始めて理解できない構文を見つけた人

# この構文ってどういう意味？
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
// || : 引数がないことを示す
// Path::new("/") : Pathインスタンスとしてルートディレクトリを生成
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
