---
title: "ZshからNushellに平和的に移行した話"
emoji: "🤖"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Nushell", "Zsh", "Shell"]
published: true
---
# はじめに
ZshよりもNushellはモダンで扱いやすいシェルだと聞いたので、元のZshで利用していた設定をなるべく破壊することなく移行することを試みた記録です。
設定ファイルの分割インポートなどは以下の記事を引用しました。
https://zenn.dev/estra/articles/nu-typed-shell

## 対象読者
- Nushellのことが気になっているけど移行が不安で手を出せない人
- Nushellの設定管理方法を知りたい人
- モダンなシェル環境に興味がある人

# Shellとは何か
Shellは、ユーザーとオペレーティングシステム（OS）の間を仲介するプログラムです。コマンドを入力すると、Shellがそれを解釈してOSに伝え、結果を返してくれます。
私たちが普段ターミナルでlsやcdといったコマンドを実行しているとき、実際にはShellがそれらのコマンドを解釈して実行しています。Shellには様々な種類があり、macOSやLinuxでは以下のようなものがよく使われています：

- Bash：多くのLinuxディストリビューションでデフォルト
- Zsh：macOS Catalina以降のデフォルトシェル。Bashの拡張版として人気
- Fish：ユーザーフレンドリーな補完機能が特徴
- Nushell：本記事で紹介する、構造化データを扱える新世代のシェル

従来のShellは主にテキスト処理を前提としていますが、現代のデータは構造化されていることが多く、JSONやCSVなどを扱う際に不便さを感じることがあります。そこで登場したのがNushellです。

# NuShellについて
Nushellは2019年に登場した比較的新しいシェルで、Rust言語で書かれています。従来のShellとは異なり、すべてのデータを構造化データとして扱うという革新的なアプローチを取っています。
## Nushellの主な特徴
### 構造化データの第一級サポート
従来のShellではテキストを行単位で処理していましたが、Nushellではテーブル形式のデータとして扱います。例えばlsコマンドの出力は、テキストではなく構造化されたテーブルとして返されます。
```sh
nu> ls | where size > 1mb | sort-by modified
```
このように、SQLライクな操作でデータをフィルタリング・ソートできます。
### 型システム
Nushellは型を持っており、文字列、数値、日付、バイナリなどを明確に区別します。これにより、予期しないエラーを減らし、より安全なスクリプト記述が可能になります。
### 豊富な組み込みコマンド
JSON、CSV、YAML、TOMLなどの形式を直接パースして操作できる組み込みコマンドが多数用意されています。jqやawkのような外部ツールに頼る必要が減ります。
### 優れたエラーメッセージ
Rustの影響を受けた分かりやすいエラーメッセージで、問題の特定が容易です。

# 段階的移行
## インストール
まずは`brew`経由でNushellをインストールします。
```sh
brew install nushell
```
念の為、インストール完了後は`brew list`できちんとNushellがインストールされていることを確認してください。
インストールが出来ていれば、そのままターミナルで以下のコマンドを実行してください。
```sh
nu -l
```
`-l`オプションを付けることでNushellをログインシェルとして起動します。この挙動は一時的なもので、デフォルトシェルを変更するものではありません。
次のようなログが確認できれば大丈夫です。
```sh
     __  ,
 .--()°'.' Welcome to Nushell,
'|, . ,'   based on the nu language,
 !_-(_\    where all data is structured!

Version: 0.107.0 (aarch64-apple-darwin)
Please join our Discord community at https://discord.gg/NtAbbG
Our GitHub repository is at https://github.com/nushell/nushell
Our Documentation is located at https://nushell.sh
And the Latest Nushell News at https://nushell.sh/blog/
Learn how to remove this at: https://nushell.sh/book/configuration.html#remove-welcome-message

It's been this long since Nushell's first commit:
6yrs 4months 24days 11hrs 25mins 32secs 591ms 584µs

Startup Time: 14ms 979µs 917ns
```
この時、コンフィグファイルが存在しないため新規に作成したことを報告するログも表示されます。
作成先はデフォルトで`/users/user/.config/nushell`であり、`env.nu`, `config.nu`の二つが作成されます。
```sh
~/.config/nushell/
├── env.nu
└── config.nu
```

## コンフィグの機能別分割
設定の管理をやりやすくするために、設定フォルダを分割してそれぞれエクスポートし、最終的に`config.nu`でインポートする形式をとります。
コンフィグフォルダにそれぞれプラグイン用、デフォルト設定用、自作関数用のフォルダを作成してください。`+`マークは新規作成するフォルダです。
```sh
~/.config/nushell/
├── + conf.d/
├── + functions/
├── env.nu
└── config.nu
```

以下で各フォルダの構成について記述します。

### conf.d
`conf.d`フォルダではこのように3つのファイルを作成してください。
```sh
~/.config/nushell/
├── conf.d/
│  ├── + alias.nu
│  ├── + index.nu
│  └── + theme.nu
├── functions/
├── env.nu
└── config.nu
```

`index.nu`ファイルは、他の2ファイルをエクスポートする記述を書きます。
```sh
export use theme.nu *
export use alias.nu *
```
`export use`という記述は、Nushellのドキュメント[export use](https://www.nushell.sh/commands/docs/export_use.html)に説明があります。
末尾の`*`は、該当ファイル内の全てのコマンドや関数をエクスポートするための記述です。

`alias.nu`ファイルは、エイリアスを定義する記述を書きます。
```sh
export alias g = git
export alias nv = nvim
```
`export alias`と記述することで、エイリアスを定義します。
:::message
`exort`がないと、外部フォルダで定義を読み込めないので注意してください。
:::

### functions
`functions`フォルダでは、自分でカスタムコマンドを作成することができます。
例えば、以下のコマンドはghqとfzfを組み合わせて、設定したルートディレクトリより以下にある`.git`を持つディレクトリを検索し、ローカルリポジトリの一覧を提示してくれます。
```sh
# ghq-fzf
export def --env "ghq fzf" [] {
  let src = (ghq list | fzf --preview $"bat --color=always --style=header,grid --line-range :80 (ghq root)/{}/README.*")
  if $src != "" {
      cd $"(ghq root)/($src)"
  }
}
```
:::message alert
関数名(`def`の後ろ)とファイル名は必ず一致させてください。一致していないとエラーが発生します。
:::
作成した関数は、`mod.nu`でエクスポートします。
```sh
export use ghq-fzf.nu *
```
そして、`config.nu`の適切な場所に以下のようなコードを追加することで、この関数を`Ctrl+G`のキーバインドで呼び出せるようにします。
```sh
$env.config.keybindings ++= [{
    name: ghq-fzf
    modifier: control
    keycode: char_g
    mode: emacs
    event: { send: executehostcommand, cmd: "ghq fzf" }
}]
```

ここまでが初期設定および段階的移行時の設定になります。
これらを行った上で、しばらくはzshからnuを起動して使用するということを続けました。

# 本格移行
ある程度Nushell環境でも問題なく扱えることが分かってきたので、本格的にデフォルトのシェルをNushellに置き換えます。
まず、Zsh上で以下のコマンドを実行し、nuコマンドが何を呼び出しているのか確認します。
```sh
which nu
```
Apple siliconであれば`/opt/homebrew/bin/nu`となっていると思います。
これを、`/etc/shells`に追加します。
```sh
echo "/opt/homebrew/bin/nu" | sudo tee -a /etc/shells
```
追加されているかどうかは次のコマンドで確認できます。
```sh
cat /etc/shells
```
```sh
# List of acceptable shells for chpass(1).
# Ftpd will not allow users to connect who are not using
# one of these shells.

/bin/bash
/bin/csh
/bin/dash
/bin/ksh
/bin/sh
/bin/tcsh
/bin/zsh
/opt/homebrew/bin/nu
```
最後に、`chsh`コマンドを使ってデフォルトシェルを変更します。
```sh
chsh -s /opt/homebrew/bin/nu
```
:::message alert
シェル変更後、段階的移行時では有効だったコマンドが呼び出せなくなった場合、`config nu`を実行して、Nushellが新しく生成したディレクトリに今までの設定ファイルを全て移動してください。`echo $nu.data-dir | path join "config.nu"`を実行すればフルパスが得られるはずです。
2025/12/8時点で、この参照先を変える方法は見つかっていません。
:::
:::message alert
シェル変更直後であれば、`chsh -s /bin/zsh`を実行し、ターミナルを再起動して`nu`を呼び出せば段階的移行時と同じ挙動に戻ります。
:::

# おわりに
この記事がデフォルトシェルをモダンなツールに乗り換えたい全ての人の役に立ってくれると幸いです。
何か間違った解説をしている部分、「貸せ、シェル移行はこうする」といった部分があればコメントで教えてくれると嬉しいです。
あとは設定参照先変えられるようになって欲しい。（ハードコーディングしなきゃいけない関係上難しそうだけど...）
