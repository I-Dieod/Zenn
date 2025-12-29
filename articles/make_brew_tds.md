---
title: "TODO検索するCLIツールを作ってHomebrewで配布した備忘録"
emoji: "📌"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["rust","homebrew"]
published: true
---
# はじめに
この記事は自分がTodoコメント行を抽出するCLIツールを作成した話を、配布時の備忘録的に書くものです。
以下の内容が含まれます。
- Rustのちょっとした解説
- Homebrewで配布する方法

## 制作したもの
https://github.com/I-Dieod/TodoScanner

## 対象読者
- Homebrewで自作のツールを配布してみたい人
- Rustでファイル読み取り、コメント抽出の方法が知りたい人

# 開発背景
Zedエディターの拡張機能としてVSCodeにある[TodoTree](https://marketplace.visualstudio.com/items?itemName=Gruntfuggly.todo-tree)のようなものを作りたいと思い、`"TODO:"`コメント行を抽出するロジックをRustでコーディングしていました。
しかしやりたいことが予想よりも高難易度だったことや[似たような拡張機能](https://github.com/thedadams/zed-comment)がすでに存在したこと、`zed_extension_api`がアプリケーションのUIに干渉できないなどの理由によりプロジェクトを無期限凍結にし、副産物としてできたロジックをどうせだしHomebrewで配布してみようと思いついたのが発端です。
それすらも[先行プロジェクト](https://github.com/tsoding/snitch)が存在しますが、Rustで高速に動くかもしれないとか、4年前で更新が停止しているとか、まあとにかくやってみました。

# ツールの内容
やってることはシンプルです。 
`args`でコマンドに対する引数を受け取ると同時に、実行時のカレントディレクトリを`env`ライブラリで取得、引数が空であればカレントディレクトリからスキャンすることを宣言します。

```rust
let args: Vec<String> = env::args().collect();
let cdu: String = env::current_dir().unwrap().to_string_lossy().to_string();
let root_path = if args.len() > 1 { &args[1] } else { &cdu };

println!("🔍 Scanning for COMMENT items in: {}", root_path);
```

ディレクトリのスキャンは`Path`オブジェクトにある`is_dir()`メソッドでディレクトリか否か判定し、再帰的に引数以下のファイルを探索します。
`path.extension()`は拡張子の判定です。

```rust
if path.is_dir() {
            scan_directory(&path, todos)?;
        } else if let Some(ext) = path.extension().and_then(|e| e.to_str()) {
            // Check if it's a supported file type
            match ext {
                "rs" | "js" | "ts" | "jsx" | "tsx" | "py" | "java" | "go" | "c" | "cpp" | "h"
                | "hpp" | "php" | "rb" | "swift" | "kt" | "scala" | "html" | "css" | "vue"
                | "md" | "txt" | "yaml" | "yml" | "json" | "toml" | "xml" | "sh" | "bash"
                | "zsh" => {
                    scan_file(&path, todos)?;
                }
                _ => {}
            }
        }
```

合致する拡張子を持つファイルを発見したら、`scan_file`関数でファイルの中身を読み取り、`//`や`#`などで始まるコメント行の内、`TODO:`や`FIX:`, `NOTE:`で始まる行の内容を抽出し、構造体に加えます。
ディレクトリのスキャンが終われば、構造体からコメントの種類ごとにアイテムを並べるための`BTreeMap`オブジェクトを宣言し、アイテムを一つずつ取り出してプッシュします。
最後に、特定の順序でコメントを表示するために用意した順序配列から`for`文で要素を取り出し、合致するコメントの種類ごとにそれぞれのアイテムの内容を`println!`します。

```rust
for comment_type in &type_order {
            if let Some(files) = by_type.get(*comment_type) {
                println!("🏷️  {} Items:", comment_type);
                println!("   {}", "=".repeat(comment_type.len() + 7));

                for (file, file_todos) in files {
                    println!("    📁 {}", file);
                    for todo in file_todos {
                        println!("        📌 Line {}: {}", todo.line + 1, todo.text);
                    }
                    println!();
                }
            }
        }
```

# Homebrewでの配布
作成したツールをHomebrewで配布するために、新しく`homebrew-tap`というリポジトリを作成し、そこに`Ruby`で記述される`Formula`ファイルを配置します。
https://github.com/I-Dieod/homebrew-tap
この時、`Cargo.toml`で定義したパッケージ名と、作成する`Formula`ファイル名と記述するクラス名は同じになるように気をつけてください。
一致していないと後述する`brew tap`のタイミングでエラーが出ます。
作成する`Formula`ファイルはClaude Sonnet 4.5に任せました。
```rb
# ！ここのクラス名！
class Tds < Formula
  desc "Fast TODO comment scanner for your codebase"
  homepage "https://github.com/I-Dieod/TodoScanner"
  url "https://github.com/I-Dieod/TodoScanner/archive/refs/tags/v0.2.0.tar.gz"
  sha256 "f2afe11e469036e8af3344e546d85df70fbe6340241c156309b6b904f520befa"
  license "MIT"

  depends_on "rust" => :build

  def install
    system "cargo", "install", *std_cargo_args
  end

  test do
    # Create a test file with a TODO comment
    (testpath/"test.rs").write <<~EOS
      // TODO: Test comment
      fn main() {}
    EOS

    output = shell_output("#{bin}/tds #{testpath}")
    assert_match "Found 1 TODO items", output
  end
end
```
具体的な記述内容について、`url`にはリポジトリ上でリリースした際に作成されるソースコードの`tarball`のリンク、`sha256`には`tarball`リンクから得られる`SHA256`ハッシュを記述してください。
ハッシュについては`Nushell`, `Bash`であれば以下のコマンドで取得できると思います。
```sh
curl -L [tarball_url] | shasum -a 256
```
:::message
#### ハッシュやURLの扱いについて
ハッシュは検証用の情報であり、秘密情報ではありません。
むしろ公開することで、ユーザーがダウンロードしたファイルが改ざんされていないことを確認できます。
これはセキュリティを強化する仕組みです。

#### 公開してはいけないもの
逆に、Formulaに絶対に含めてはいけないもの：
❌ APIキー、トークン
❌ パスワード
❌ プライベートリポジトリのアクセストークン
❌ 秘密鍵
:::
作成した`Formula`ファイルをリモートリポジトリにPushしたら、お手元のCLIで以下のコマンドを実行してください。
```sh
brew tap [your_user_name]/tap
brew install [your_user_name]/[package_name_you_want]
# または
brew install [your_user_name]/tap/[package_name_you_want]
```
これで自作したツールが`brew`経由でお手元のCLI上で使えるようになると思います。
![](/images/make_brew_tds/tds_run.png)
*今回配布したツールが動作する様子*

:::message
#### brewパッケージのアップデートについて
基本的には`Formula`ファイルのうち`url`と`sha256`を新しいバージョンでリリースしたものに置き換えるだけでOKです。
:::

# おわりに
ほんとはZedの拡張機能を作りたかったんですが、すでに存在したことが悔しいです。アイデア負け。
これを機に色々、ツール制作とかOSSコントリビューションとかやっていけたらなと思いながら、まぁとにかくやってみました。
この記事がHomebrewで何か配布してみたい人の一助になってくれると嬉しいです。
ツールに対する文句とかPRとか、何か間違った箇所等あればコメントで教えてくださると助かります。
