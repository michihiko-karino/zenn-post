---
title: "vue composition apiにおけるpropsのリアクティブ性について知っておくべきこと" # 記事のタイトル
emoji: "📝" # アイキャッチとして使われる絵文字（1文字だけ）
type: "tech" # tech: 技術記事 / idea: アイデア記事
topics: ["vue"] # タグ。["markdown", "rust", "aws"]のように指定する
published: true # 公開設定（falseにすると下書き）
---

# この記事で伝えたいこと

- vue composition apiにおけるコンポーネントの`setup`で受け取れる`props`は`reactive`で定義した変数の**ような**リアクティブ性をもちます
- `props`のプロパティにリアクティブ性を求めるパターンと求めないパターンごとの使い分け

:::message
**ような**という曖昧な表現にしているのは、挙動は似ているが同一のものか私には判断できないからです。
:::

# リアクティブをもたらすAPI `ref`, `reactive`, `computed`

composition apiではリアクティブをもたらすAPIとして`ref`, `reactive`, `computed`などがあり、特に`ref`と`reactive`に関してはどちらをつかうべきかという記事が多い印象をもちます。
それぞれの比較としては他開発者さまの記事ですがこちらが大変参考になると思います。
https://zenn.dev/azukiazusa/articles/ref-vs-article

私自身振り返って考えてみると`ref`と`computed`を多用しています。
`ref`はTypeScriptを使っていればリアクティブな変数かどうかわかりやすいし、
`computed`は他のリアクティブな変数に依存するリアクティブな変数を簡潔に表現でき、かつ`ref`のように別のロジックから変更される恐れがないので使いやすいです。

```ts
const isOpen = ref(true);
const message = computed(() => isOpen.value ? '開いてるよ！' : '閉まっているよ！');
// messageに入る値は上記の２パターンのみであり更新はされない。下記を実行すると `Write operation failed: computed value is readonly` というワーニングが表示される
message.value = '全然関係ないメッセージ';
```

UIロジックなどを表現する際には`ref`と`computed`でほぼ十分でありますが、storeを利用する場合などでは`reactive`を用います。
しかし`reactive`には**リアクティブの消失**という注意ポイントがあります。

# `reactive`のリアクティブの消失について簡単に言うと

リアクティブの消失については[リアクティブな状態の分割代入](https://v3.ja.vuejs.org/guide/reactivity-fundamentals.html#%E3%83%AA%E3%82%A2%E3%82%AF%E3%83%86%E3%82%A3%E3%83%95%E3%82%99%E3%81%AA%E7%8A%B6%E6%85%8B%E3%81%AE%E5%88%86%E5%89%B2%E4%BB%A3%E5%85%A5)にある通りで分割代入によってリアクティブを消失します。
分割代入の例としては下記のようなパターンもあります。

```ts
const state = reactive({ name: 'karino' });

// 1. リンクにある通りの分割代入を行うパターン
const { name } = state;

// 2. プロパティを直接参照して代入するパターン
const name = state.name;

// 3. プロパティを直接参照して他の処理に渡すパターン
const { length } = useLength(state.name);
// useLengthの戻り値のlengthはリアクティブな変数のはずですが、state.nameが更新されてもその値が変わることはないです

// 一種のcomposable
const useLength = (name: string) => {
  const length = computed(() => name.length);
  return { length };
};
```

これらは分割代入によってリアクティブが消失しているといえますが、別の言い方をすると **「`reactive`で定義された変数自体はリアクティブだが、そのプロパティはリアクティブではない」** という表現もできるかなと思います。

実験1: どんなタイプのプロパティでもリアクティブは消失する
https://stackblitz.com/edit/vue-eysy8a?embed=1&file=src/App.vue&hideDevTools=1&hideExplorer=1&hideNavigation=1

実験2: プロパティがrefによるものでも分割代入でリアクティブは消失する
https://stackblitz.com/edit/vue-kjypvk?embed=1&file=src/App.vue&hideDevTools=1&hideExplorer=1&hideNavigation=1

# 本題1: `props`は何にあたるのか

コンポーネントの`setup`の第一引数に渡される`props`変数もリアクティブな変数です。
そして`reactive`で作られた変数のプロパティがリアクティブを消失するパターンを見ましたが、
同様に`props`も同じ操作でそのプロパティのリアクティブを消失します。

例: propsは分割代入でリアクティブを失う
https://stackblitz.com/edit/vue-cnb8my?embed=1&file=src/App.vue&hideDevTools=1&hideExplorer=1&hideNavigation=1

上記実験1でのリンク先では`reactive`で作成した変数の分割代入された変数がリアクティブを失っているのと同じように、`props`のプロパティも分割代入によってリアクティブを失っています。
このことから`props`も **`reactive`で定義された変数と同じような** リアクティブ性を持つと考えられます。

# 本題2: `props`のプロパティにリアクティブを求めるパターンと求めないパターンの使い分け

`props`のプロパティがリアクティブを失うパターンについて見ました。
では`props`のリアクティブを生かした状態でロジックを書きたい時と、そうでない時ではそれぞれどのように使い分ける必要があるのかを書き方ごとにまとめてみます。

## `props`のプロパティにリアクティブを求めるパターン

### `props`をそのまま使える場合はリアクティブのまま

上記までで`props`のプロパティがリアクティブを失うパターンを見ましたが、いずれも分割代入の実行タイミングが`setup`の直下でした。
分割代入の実行タイミングが変われば、その時のプロパティの値が取得できます。

下記例ではボタンを押すとその時の`id`の値を用いて`someAPI`を呼びます。

```vue
<template>
  <button @click="callAPI">call api</button>
</template>

<script>
export default {
  props: { id: { type: Number } },
  setup(props) {
    const callAPI = () => {
      someAPI(props.id);
    };

    return { callAPI };
  },
};
</script>
```

### `props` + `toRefs`, `toRef`

https://v3.ja.vuejs.org/api/refs-api.html#toref

`toRefs`, `toRef`はいずれも`reactive`な変数に使えますし`props`にも使えます。

例: `props`に`toRef`を使い、親・子からそれぞれ更新してみる実験
https://stackblitz.com/edit/vue-rcqnny?embed=1&file=src/App.vue&hideNavigation=1

親から子の変更は伝わりますが、子で取得したリアクティブな変数を更新しようとすると`Set operation on key "〇〇" failed: target is readonly.`というワーニングが出て更新されません。
これは`props`の子側からの変更を許さないVueの方針によるものです。

この方法のメリットは`props`のプロパティを独立したリアクティブな変数として宣言できるところにあります。
これを行うことでプロパティのみを`composable`なメソッドに与えロジックの分割ができるようになります。

しかしTypeScriptを用いている場合、`toRefs`, `toRef`によって取得した変数は`Ref`型になり、`〇〇.value = △△`で**型エラーになりません**。
もしそのようなコードがある場合、実行時不具合になりエラーログではなくアラートログが出るだけなので、発見や修正が遅れるかも…しれません。ちょっと怖いですね。
今後のVue自体のアップデートで対応されたり、もしかしたらLintの設定などで対処できるのか分かりませんが、一先ず注意ポイントですね。

### `props` + `computed`
`computed`を使うことでも`props`のプロパティをリアクティブな変数として独立させることができます。
個人的には`toRef`, `toRefs`を使うよりもいいのではないかと思っています。

```ts
export default {
  props: { name: { type: String } },
  setup(props) {
    // プロパティに連動した別のリアクティブな変数を作るのに便利
    const nameLength = computed(() => props.name.length);
    // プロパティとまったく同じ意味合いのリアクティブな変数を作ることも可能
    const name = computed(() => props.name);
  },
};
```

`computed`をおすすめする理由は２つあります。
1つ目はTypeScript上では`ComputedRef`型になる点です。
`ComputedRef`型はReadOnlyな型なので`〇〇.value = △△`で**型エラーになります**。実行前に不備を気づけるようになります。

２つ目は覚えるAPIの数を減らせるという点です。
`computed`は便利な特性をもつので積極的に使い覚えたいAPIですが、`toRef`,`toRefs`は比較的使用頻度は高くないです。
`toRefs`は一気に複数のプロパティを取り出せる使い方もできますが、少なくとも`toRef`に限って言えば`computed`でいいのではないでしょうか？


## `props`のプロパティにリアクティブを求めないパターン

### `const 〇〇 = props.〇〇`

`setup`の直下でこの書き方をするということは、コンポーネントがマウントされて以降更新されない値だということになります。

### `props` + `ref`

```ts
const input = ref(props.initialValue);
const onUpdate = (inputValue: string) => input.value = inputValue;
```

上記例のように`ref`の中で`props`のプロパティを使う場合は、その`ref`の変数の初期値として`props`のプロパティを使うという意味になります。
`props`のプロパティと連動して`ref`の変数が変わることはありません。
`props`のプロパティに対応する変数が親のコンポーネント側で更新されても、子のコンポーネントの`ref`は更新されませんし、逆に子の`ref`が更新されても親の値は変わりません。

https://stackblitz.com/edit/vue-qyqwmo?embed=1&file=src/Child.vue&hideDevTools=1&hideExplorer=1&hideNavigation=1

# 最後に

なぜこのような当たり前のような記事を今更書いたかというと、実は私は`props`がリアクティブじゃなくなる条件をよく分からないまま使っていたからです。
しかし考えてみれば`reactive`と同じような動きをすることに気づき、自分の鈍感さを痛感しました。
あまり良くないロジックを量産していたような気がします。恥ずかしい〜

以上
