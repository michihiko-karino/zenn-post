---
title: "vue3で実装するFlashMessage（テストもあるよ）(typescriptだよ)"
emoji: "🔦"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["vue3", "vite", "vitest", "typescript"]
published: true
published_at: 2022-10-02 10:00
---

# 動機

vue3を利用してライブラリ単体でも実装しやすくて、いろんな機能を使える練習問題にちょうどよさそうな題材ということでFlashMessage(ToastMessageとも言われているアレ)を思いつきました。

同様の実装は軽くググっただけでもライブラリとして溢れています。自前で実装するよりそちらを利用することも実業務では多いかもしれません。

しかしある程度切り出しやすい粒度の機能であれば自前で実装した方が長く見てもコスト安の場合もあるのでこれを機に実装してみました。


# 作ったものと各種ライブラリバージョン

@[stackblitz](https://stackblitz.com/edit/github-kzg6pu?embed=1&file=src/App.vue&view=preview)

github: https://github.com/michihiko-karino/vue3-flash-message

```
vue: 3.2.37
vite: 3.1.0
vitest: 0.23.2
typescript: 4.6.4
```


# 実装ポイント解説

作りましただけでは技術記事としてアレなので工夫点などを解説します。

基本的なことばかりですが誰かのためになれば幸いです。


## `teleport`を使おう！

[`teleport`](https://v3.ja.vuejs.org/guide/teleport.html)はVue3から追加された機能で、Vue2ではPortalVueなどのプラグインで実現されていました。

`teleport`はコンポーネントをHTML構造からテレポートさせる機能であり、モーダルなどがよく例にだされます。

FlashMessageもモーダルと同じように画面のFixedな位置に、色々なコンポーネントから呼び出される可能性のあるコンポーネントのため、同様に`teleport`を使いましょう。

https://github.com/michihiko-karino/vue3-flash-message/blob/main/src/components/FlashMessage.vue#L18-L26

`teleport`を使うとSpecで実際に表示されているかの検証がちょっとめんどくさくなってしまいます。

https://test-utils.vuejs.org/guide/advanced/teleport.html#interacting-with-the-teleported-component

しかしこれには回避方法があります。テストの段落で説明します。

## 主なロジックはcomposablesにしてしまおう！

https://github.com/michihiko-karino/vue3-flash-message/blob/main/src/composables/useMessage.ts#L36-L62

リアクティブ変数とそれを変化させるメソッドをVueコンポーネントから別のモジュールに切り出すことで、テストしやすさと型をシンプルにします。

FlashMessageという機能自体がシンプルなおかげでもありますが、リアクティブ変数が3つ、メソッドが2つだけで実装できてしまうのは嬉しいですね。

## `provide/inject`の活用と注意

FlashMessageを各コンポーネントに使ってもらうために、メッセージを表示するメソッドと消すメソッドを各コンポーネントから参照できるようにする必要があります。

そのために[`provide/inject`](https://v3.ja.vuejs.org/guide/component-provide-inject.html)を使いました。

`provide/inject`はコンポーネントの親子関係を部分的に無視し、親が提供したデータを子孫全体で利用できるようにする機能です。

簡易ストア実装や、アプリコンテキストなどで使われますが、今回のような閉じた機能を作る際にも使えるものだと思っています。

### `inject`するときの型引数を工夫しよう

`inject`はその`inject`が何を返すかを指定できる型引数を渡せます。

https://github.com/michihiko-karino/vue3-flash-message/blob/main/src/composables/useMessage.ts#L15-L24

FlashMessageを使う側のコンポーネントにはリアクティブ変数を参照されても困るので、メソッドだけが定義された`Mutations`型を使い、

https://github.com/michihiko-karino/vue3-flash-message/blob/main/src/composables/useMessage.ts#L26-L34

FlashMessageコンポーネント自体は、自分を表示するor消すメソッドに関心がないので`MessageState`型を使います。

https://github.com/michihiko-karino/vue3-flash-message/blob/main/src/components/FlashMessage.vue#L7

こんな風にprovideされた値全てを取り出すのではなく必要なものだけにするといいでしょう。

### 注意: `provide`した値は`provide`したコンポーネント自身は`inject`できない

`provide/inject`には辛いな〜と思う仕様があります。

`provide/inject`のキーにSymbolが使えること、もしくはその値自身がある程度の **機能** をもつ場合、それらをまとめて別のモジュールとして定義したくなります。そして`provide`するだけのメソッドを定義したくなります。

当然ですが`provide`するコンポーネント自体がその値を利用したい場合もあるでしょう。しかしながらモジュールの中で`provide`された場合コンポーネントは値への参照を失います。

この仕様のため私は`provide/inject`を利用する場合は`app level provide`を推奨します。これは文字通り`createApp`の返却値でprovideする方法です。

今回の例でも`app level provide`を使っています。

https://github.com/michihiko-karino/vue3-flash-message/blob/main/src/messagePlugin.ts#L5-L10

:::message
余談1:
vue3の`provide/inject`の実装です↓
https://github.com/vuejs/core/blob/main/packages/runtime-core/src/apiInject.ts#L53-L56

ちなみにこの「`provide`した値は`provide`したコンポーネント自身は`inject`できない」仕様は公式ドキュメントのどこにも書いておらず、最終的に実装を確認しました。
:::

:::message
余談2:
実はvue2向けの[`vuejs/composition-api`](https://github.com/vuejs/composition-api)プラグインが提供する`provide/inject`ではこの制約はありません。
Vue作者はこの挙動を[プラグイン側のバグ](https://github.com/vuejs/vue/issues/12678#issuecomment-1189775474)だと表現しましたが、個人的にはプラグインの方がイケてるのにな〜と思いました。
:::


## テストを書く

### `teleport`のSpecにハマった

`teleport`を使っている時`wrapper.html()`では意味のある内容が表示されません。詳しくは↓のリンクを見てほしいです。

https://test-utils.vuejs.org/guide/advanced/teleport.html#interacting-with-the-teleported-component

ですが内部のDOM的にはちゃんと描画されているので`document.〇〇`でHTMLを直接見てあげれば問題ありません。

https://github.com/michihiko-karino/vue3-flash-message/blob/main/test/components/FlashMessage.spec.ts#L12-L39


### composablesのテストは書きやすいし、モックしやすい

`teleport`や`provide/inject`が登場するモジュールのSpecはセットアップが必要になります。

しかし`useMessage.ts`のようにVue実装でないモジュールに切り出してしまえばシンプルなSpecになりますし、セットアップも必要ありません。

https://github.com/michihiko-karino/vue3-flash-message/blob/main/test/composables/useMessage.spec.ts

網羅的なSpecはこちらで書き、コンポーネントのSpecでは代表的な例だけを検証すれば良さそうです。

またモジュールに切り出すことでモジュールモックもしやすくなります。

https://github.com/michihiko-karino/vue3-flash-message/blob/main/test/components/MeesageControlls.spec.ts#L7-L12

↑のようにモジュールごとモックすることで、実際にメッセージがHTML上で表示されているかを確認せず、メソッドが呼ばれたかを検証するだけで済むようになり、セットアップコードをへらすことができます。

https://github.com/michihiko-karino/vue3-flash-message/blob/main/test/components/MeesageControlls.spec.ts#L19-L25


# 最後に

CompositionAPIやVue３からの各種機能があるおかげでVue単体で実装できるものも増え、テストもしやすくなりました。

今回実装したFlashMessageは文字列を表示するだけの単純なものですが、ライブラリ標準機能を使いこなす、テストしやすいモジュールを考えることは普遍的な技術です。

定期的にこういうことをやっていこうと思いました。

以上
