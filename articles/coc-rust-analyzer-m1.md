---
title: "M1 Macでcoc-rust-analyzerが一部機能しない場合"
emoji: "🐡"
type: "tech"
topics: ["NeoVim", "Tech", "Vim", "Rust"]
published: false
---
# 現象
coc.nvim + coc-rust-analyzerでRustを書いていたら、一部のコードで定義ジャンプできない現象が発生しました。
例:
```rust
#[derive(Debug, StructOpt)]
#[structopt(global_settings = &[AppSettings::ColoredHelp])]
pub struct Args {
    #[structopt(short="j", long="json")]
    pub json: bool,
}

fn main() {
  // このfrom_argsから定義ジャンプしようとしてもnot foundになる
  let mut args = Args::from_args();
}
```
この時点で使用しているrust-analyzerはcoc-rust-analyzerを入れるとデフォルトでダウンロードされるやつを使っています。
VSCodeで、VSCode上からダウンロードされたrust-analyzerを使うと定義ジャンプは問題なくできました。

`:CocCommand rust-analyzer.serverVersion`でrust-analyzerのバージョンを見てみると
```
[coc.nvim] rust-analyzer 65fbe0a8d 2022-04-18 stable
```
となっています。

# 原因の特定
## デフォルトインストールされたrust-analyzer以外のrust-analyzerをcocから使う
VSCode拡張から入れるとrust-analyzerは`~/.vscode/extensions/matklad.rust-analyzer{バージョン名}/server/rust-analyzer`のパスに入っています。
これをcoc.nvimから使ってみます。

```json:coc-settings.json
{
  "rust-analyzer.server.path": "~/.vscode/extensions/matklad.rust-analyzer-0.2.1022-darwin-arm64/server/rust-analyzer"
}
```
この設定で試みると、問題なく定義ジャンプができました。

同じように、[Releases · rust-lang/rust-analyzer](https://github.com/rust-lang/rust-analyzer/releases)から上記のcoc-rust-analyzerからインストールしたrust-analyzerと同じバージョンのrust-analyzer-aarch64-apple-darwin.gzをダウンロードして解凍し、cocから読ませてもうまくいきます。

## coc-rust-analyzerの中身を見てみる
どうやらrust-analyzerをインストールしているのは[このファイル](https://github.com/fannheyward/coc-rust-analyzer/blob/master/src/downloader.ts)っぽいです。
この中でM1 Macに何か影響しそうなものを探すと、
```ts: downloader.ts
function getPlatform(): string | undefined {
  const platforms: { [key: string]: string } = {
    'ia32 win32': 'x86_64-pc-windows-msvc',
    'x64 win32': 'x86_64-pc-windows-msvc',
    'x64 linux': 'x86_64-unknown-linux-gnu',
    'x64 darwin': 'x86_64-apple-darwin',
    'arm64 win32': 'aarch64-pc-windows-msvc',
    'arm64 linux': 'aarch64-unknown-linux-gnu',
    'arm64 darwin': 'aarch64-apple-darwin',
  };

  let platform = platforms[`${process.arch} ${process.platform}`];
  if (platform === 'x86_64-unknown-linux-gnu' && isMusl()) {
    platform = 'x86_64-unknown-linux-musl';
  }
  return platform;
}
```
この関数がめちゃくちゃ怪しいので、とりあえずnodeのprocess.archがM1 Macで何を返すのか検証してみます。

```sh
$ node -p process.arch
x64
```
...原因っぽいものが分かりました。上記`getPlatform()`関数で`x86_64-apple-darwin`が返されることになりそうです。
当然ですがシェルでarchコマンドを叩くと正しいアーキテクチャが表示されます。
```
$ arch
arm64
```
`process.arch`が何を返すかはnodeのバージョンにもよりそうなので、nodeのバージョンを変えてみます。
ちなみに上記のコマンド時点ではv14.19.1を使っていました。
次はnode16に上げてから実行します。

```sh
$ node -v
v16.14.2

$ node -p process.arch
arm64
```
予想通り、nodeのバージョンが原因になってそうです。

# 対処法
グローバルのバージョンをnode16以上に固定し、rust-analyzerをcoc-rust-analyzerを再インストールします。
インストール済みのものは`~/.config/coc/extensions/coc-rust-analyzer-data/rust-analyzer`に入っているのでこれを消します。
```sh
rm ~/.config/coc/extensions/coc-rust-analyzer-data/rust-analyzer
```
その後、`:CocRestart`や`:CocCommand rust-analyzer.reload`、Vim再起動などでcoc-rust-analyzerが以下のようなメッセージを出すので、あとはこれに従ってインストールします。
```
rust-analyzer is not found, download from GitHub release?:
1. Yes
2. Cancel
Type number and <Enter> or click with the mouse (q or empty cancels):
```
インストール後、定義ジャンプは正常に機能するようになりました。
この現象に対応するPRをcoc-rust-analyzer出そうかと思いましたが、process.arch使わないと汚いコードで対応しなきゃいけなくなりそうな予感がしたので、該当現象と対処法をドキュメントに残すissueでも立てて検討してもらおうかと思います。

# 備考
- [node 15.5.1](https://github.com/nodejs/node/blob/master/doc/changelogs/CHANGELOG_V15.md#15.5.1)以降のバージョンなら問題なく動きそうです。
- 私はnodeバージョンマネージャとして[volta](https://github.com/volta-cli/volta)を使っているので、グローバルはv16にしてv14とか使いたいときは`volta pin node@14`でローカルのnodeバージョン固定することにしました。

