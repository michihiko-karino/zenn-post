---
title: "yarnのメジャーバージョン１を使いつつcorepackによるバージョン制御はできない"
emoji: "🧶"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ['yarn', 'nodejs']
published: true
published_at: 2022-07-12 12:00
---

# 確認したバージョン

```
yarn 1系統: 1.22.19 (以降v1と記す)
yarn 3系統:   3.2.1 (以降v3,最新stableと記す)
```

# 伝えたいこと

1. yarnのインストール方法や設定ファイルなどのv1とv3で大きく差がある
1. dependabotがまだv3に対応できていない
1. v1を使いつつ少しでも設定を最新stableに寄せられないかな〜？→できません🙅‍♀️

# はじめに

yarnはすでにメジャーバージョンが3になっています。それどころか[4のrc版](https://github.com/yarnpkg/berry/tree/%40yarnpkg/shell/4.0.0-rc.9)も出ています。

またv1とv3の公式サイトやGithubリポジトリも分かれており、v1には`The 1.x line is frozen`や機能改善は行わない旨などが記されています。
これらのことから積極的に最新stableに更新したいなと思いましたが、一時断念しました。
その調査で得た気づきがあったので共有したいと思います。

# yarnのバージョンに関連する前提知識

- バージョンを確認するコマンド: `yarn -v`
  - 他にも`yarn install`を実行すると一番最初のログとして表示されます
- バージョンを設定するコマンド: `yarn set version 〇〇`
  - 〇〇には`stable`, `classic`, もしくはバージョン番号を指定することができます
- yarnのバージョンはコマンドを実行するディレクトリで変化する
  - yarnにはグローバルインストールされているバージョンとは別に、プロジェクト（ディレクトリ）が依存するバージョンを指定できます
  - ディレクトリの`.yarn/releases/`以下に指定したバージョンのyarnスクリプトが配置され使われる感じです。便利ですね

# yarn v1とv3のインストール方法や設定ファイルの違い

上記の`前提知識`はv1,v3は関係ありませんが、機能以外にもインストール方法や設定ファイルも明確に違ってきます。

## インストール方法

- v1のインストール方法:

```
npm install --global yarn
```
https://classic.yarnpkg.com/en/docs/install#mac-stable

- v3のインストール方法:

```
corepack enable
yarn set version stable
```
https://yarnpkg.com/getting-started/install

### 余談1: corepackとは？

nodejsの`v16.9.0, v14.19.0`から搭載された機能です。
`package.json`の`packageManager`フィールドでそのプロジェクトが何のパッケージマネージャのどのバージョンに依存するか指定することができます。
https://nodejs.org/api/corepack.html

```package.json
  "packageManager": "yarn@3.2.1"
```

と指定されていれば`pnpm`でパッケージをインストールしようとするとエラーになるという機能です。（`corepack enable`のコマンド実行が必要になります。）

しかし残念ながら`npm`はデフォルトでは`corepack`の管理下に置かれないので`npm install`を実行してもエラーにならないという罠があります。
`corepack enable npm`を実行することで`npm`も管理に追加されます。

## 設定ファイル

- v1: `.yarnrc`
- v3: `.yarnrc.yml`

これらのファイルはファイル名だけでなく、フォーマットにも違いがあります。

プロジェクトルートにどちらのファイルが設置されていたとしてもyarnバージョンによって使われるファイルが切り替わります。

## `yarn.lock`ファイル

`yarn.lock`ファイルは人間が更新するファイルではありませんが、このファイルのフォーマットがバージョンによって変化します。
v3に更新してから`yarn install`を実行することで勝手にマイグレーションしてくれます。
ファイルを消す必要はありません。

# 以上のことから…

yarnはv1とv3を同じプロジェクトで混ぜて使うことができないことが分かりました。
同じメジャーバージョンの間でマイナーバージョンが変わるのとは違い、そのプロジェクトごとにマイグレーション対応が必要なんですね。
詳しいマイグレーションガイドは[v3公式サイト](https://yarnpkg.com/getting-started/migration)から確認できます。

しかしながら次のことから私は一旦対応を諦めました。

# dependabotがまだv3に対応できていない

dependabotとは、Githubで利用できる依存関係更新の仕組みです。
これを有効にしているとnodejsプロジェクトだけでなく様々なエコシステムの依存関係更新のPRを作ってくれます。
便利なので有効にしていることも多いかと思われますが、残念ながらv1の`yarn.lcok`のフォーマットしか対応していません。

[サポート追加要望Issue](https://github.com/dependabot/dependabot-core/issues/1297)を見るといくつか対処方法がありました。

- 同様のサービスである[renovatebot/renovate](https://github.com/renovatebot/renovate)に乗り換える
- dependabotが作るPRをトリガーにして差分を更新するGithub Actionを設定する

`renovate`も良さそうですが、プロジェクトによっては同じリポジトリの中で複数のエコシステムを含んでいたりしてnodejsだけ別の仕組みを使い始める気持ちにはあまりなれませんでした。
Github Action追加が現実的でしたがまだ試せていなかったです。

そこでv1を使いながら`corepack`の仕組みを寄せておけば将来的にv3に上げられたとき対応が楽になるかな〜と思い、調査してみました。

# 結論: yarnのv１を使いつつ`corepack`によるバージョン制御はできない

v1の公式サイトには`corepack`を使える・使えないは一切書かれていないので、もしかしたら使えるかもな〜ぐらいの気持ちで調査してみました。
結果としては「`corepack`関連を更新するのはv3を使いつつ`yarn set version`するときだけでv1では更新されない」ということが分かったので、v1利用時では`corepack`を利用するのは諦めました。
試したコマンドについては下記に書いておきます。ただしグローバルインストールされているバージョンやどのディレクトリでコマンド実行するかで挙動が変わる恐れがあるため、再現性は保証できません。

:::details `corepack`関連を更新するのはv3を使いつつ`yarn set version`するときだけでv1では更新されない

```
yarn -v
> 1.22.xx # グローバルインストールされているyarnはv1

# corepackを有効にしておく
corepack enable yarn

mkdir test && cd test
# package.jsonを作る。v1の状態では`packageManager`フィールドは指定されない
yarn init -y

# versionをv1に明示的に設定する
yarn set version 1.22.19
# v1にsetしてもpackage.jsonは更新されない。ただし設定ファイルである`.yarnrc`と`.yarn`ディレクトリが作られる

yarn -v
> 1.22.19

# 最新系にsetするとpackage.jsonが更新される
# なおこの際に`yarn.lock`ファイルが必須。v3からはlockファイルがないとそのディレクトリをプロジェクトルートと認識せずlockファイルが見つかるまで上位ディレクトリを探しにいってしまう
touch yarn.lock
yarn set version 3.2.1
# `.yarn/releases/`が更新され、`.yarnrc.yml`が追加される

# またpackage.jsonに`packageManager`フィールドが追加される
cat package.json
> {...
> "packageManager": "yarn@3.2.1"
> ...}

yarn -v
> 3.2.1

# 逆にv3からv1に戻してみる
yarn set version 1.22.19

yarn -v
> 1.22.19

# package.jsonの`packageManager`フィールドはv1に更新される
# v3用の設定ファイル`.yarnrc.yml`の設定も更新される
cat package.json
> {...
> "packageManager": "yarn@1.22.19"
> ...}
cat .yarnrc.yml
> yarnPath: .yarn/releases/yarn-1.22.19.cjs

# アレ？v1でもcorepack使えてるっぽい:thinking:と思うが…
# ここでv1の別のバージョンに設定してみる
yarn set version 1.22.18

# `yarn -v`は`1.22.19`何故か更新されない！！！
# package.jsonの`packageManager`は`1.22.19`のまま更新されない
# `.yarnrc.yml`は`1.22.19`のまま更新されない
# `.yarnrc`は`1.22.18`で更新されている！？
```

以上のことから分かることとして…

- `yarn set version`を実行したときのyarnバージョンによって更新されるファイルは変わる
- v1を使いつつ`corepack`の仕組みに寄せても挙動が直感的でない。v1を使いつつ`corepack`によるバージョン制御は意図された挙動でない可能性がある

ここで注意してほしいのが`yarn set version`コマンドがそのディレクトリがyarnが管理すべきディレクトリと判断するまでどんどん上位のディレクトリを探しにいって、最終的にユーザルートディレクトリに設定ファイルを残したりすることがあります。
上位のディレクトリに変な設定ファイルがあると下位ディレクトリにも影響が出たりするので、なんかへんだなと思ったら上位ディレクトリを疑ってほしいです。
こまったらyarnを再installしてみるといいです。
:::


# 終わりに

yarnのバージョンアップはdependabotの対応を待ってからで一旦はいいかなという気持ちになりました。
上記で紹介した[dependabotのv2サポート追加Issue](https://github.com/dependabot/dependabot-core/issues/1297)は最近（2022/7）いい方向に進んでいそうなので案外すぐ更新できるかもしれませんね。

以上
