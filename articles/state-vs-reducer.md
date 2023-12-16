---
title: "useReducerは何者なのか？"
emoji: "❄️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["react", "typescript"]
published: true
publication_name: "openlogi"
---

この記事は、OPENLOGI Advent Calendar 2023の10日目の記事です。
先日、この記事の内容でLTしました。スライドを公開しているので、さっと読みたい方はどうぞ。

@[speakerdeck](dd44f665b5da44c7a2f56992776ff315)

## はじめに

### useReducer って何者？

Reactの学習を始めると、大体 `useState` `useEffect` `useCallback` といった主要なフックの解説は多く見られます。
しかし、あまり使わない`useReducer` についてはサラッと書かれていることが多く「いつどうやって使うのか？」と疑問に思って来ました。
そこでこの記事では、 `useReducer` の使い所について、`useState` の実装と見比べながら考えていこうと思います。

### 注意点

この記事では、フォームを題材にしてuseReducerを使って実装しています。
ですが、フォームの実装でuseReducerを推奨する意図をもって執筆したものではありません[^1]🙏

あくまでユースケースの1つとしてご理解いただけると嬉しいです。

[^1]: 複雑なフォームを実装する場合、[Formik](https://formik.org/) や [React Hook Form](https://www.react-hook-form.com/) などのフォームヘルパーを利用するのが最適だと思います

## 題材

以下のような配送フォームを考えます。

![](/images/state-vs-reducer/form.png)

- 以下を入力項目とする
  - 郵便番号
  - 都道府県（プルダウンで選択）
  - 市区町村
  - 番地
  - 建物名
  - 包装オプション（プルダウンで選択）
- 「送信」ボタンを押すと、入力項目が送信される（今回はコンソールに吐き出す）
- 「内容を確認しました」チェックボックスを設け、チェックが押されている場合のみ送信ボタンを押せる
- 「リセット」ボタンで入力項目とチェックボックスを初期値に戻す

デモ画面を用意したので、よかったらみてください。

https://riku929hr.github.io/state-vs-reducer/

## stateで実装する場合の課題

フォームヘルパーなどのライブラリを使わずvanilla reactで実装する場合、以下のように項目値ごとにstateを設ける方法が考えられます。

```ts
const UseStateForm: FC = () => {
  const [zipcode, setZipcode] = useState<string>('');
  const [prefecture, setPrefecture] = useState<Prefecture>();
  const [city, setCity] = useState<string>('');
  const [address, setAddress] = useState<string>('');
  const [building, setBuilding] = useState<string>('');
  const [wrapping, setWrapping] = useState<keyof typeof wrappingType>('none');
  const [isConfirmed, setIsConfirmed] = useState<boolean>(false);

  const handleSubmit = (e: SyntheticEvent) => {
    e.preventDefault();
    const data = {
      zipcode,
      prefecture,
      city,
      address,
      building,
      wrapping,
    };
    // 以下省略
  };

  const handleReset = (e: SyntheticEvent) => {
    e.stopPropagation();
    setZipcode('');
    setPrefecture(undefined);
    // 以下省略
  };
```

この実装においては、次のようなメリット・デメリットがあります。

### メリット

パッと思いつく限り

- 実装がシンプル
- フォームの入力による再レンダリングコストが小さい（入力したコンポーネントのみレンダーされる）

といったところでしょうか。

### デメリット

フォームの型がない、言い方を変えればstateの凝集度が低いということです。

これはたとえば、次のような問題が起こります。

- formの項目を追加/削除するたびにhandleSubmit, handleResetの変更が必要だが、変更漏れでもlinterやcompilerでエラーにならず、気づきにくい
- `isConfirmed`（チェックボックスの値）はフォームの値ではないが、区別がしづらい

フォームの入力値を1つの変数で扱うことができれば、この問題は回避できます。

## useReducerを使う

上記のデメリットを改善できる解決策の1つが、useReducerです。

### そもそも reducer とは？

ここではreduxの詳細な説明は行いませんが、簡単に説明しておきます！

reducerはreduxにおけるデータ更新関数で、次のように定義されています。

```ts
(prevState, action) => newState;
```

- 更新前のstate（prevState）
- どう更新するかを示すobject（action）

を引数にとり

- 更新後のstate（newState）

を返す純粋関数です。

### なぜ useReducer が有効なのか？

reduxが由来であるということを述べましたが、そもそもreduxとはどういうものだったでしょうか。

端的に言うと、ツリー構造を持つグローバル変数（state）で単方向データフローを保って状態管理する仕組みです。

つまり、useReducerは

```json
{
  "a": 1,
  "b": {
    "b1": 2,
    "b2": 3
  },
  "c": 4
}
```

のような、複雑な構造をもつstateの管理にはうってつけの手段なんです！

## useReducerを利用したリファクタリング

### 方針

公式ドキュメント[state ロジックをリデューサに抽出する](https://ja.react.dev/learn/extracting-state-logic-into-a-reducer)をもとに作成

1. stateを定義し直す
2. actionを抽出する
   - stateをどのように変更するか、そのユースケース
3. reducerを作成する
4. stateのsetをactionのdispatchに置き換える

### 1. stateの定義

フォームのstateとして以下のように定義します。

```ts
type DeliveryForm = {
  zipCode: string;
  prefecture: string;
  city: string;
  address: string;
  building: string;
  wrapping: keyof typeof wrappingType;
};

// 初期値
const initialState: DeliveryForm = {
  zipCode: "",
  prefecture: "",
  city: "",
  address: "",
  building: "",
  wrapping: "none",
};
```

これでフォームの値を取り回すためのstateが一塊になり、`DeliveryForm` という型がつけられます。

### 2. actionの抽出

おさらいになりますが、actionはstateをどのように更新するかを表すオブジェクトです。

今回は、[Flux Standard Action(FSA)](https://github.com/redux-utilities/flux-standard-action) に基づき、actionを次のように定義します。

```ts
const ActionType = {
  reset: "deliveryForm/reset",
  updated: "deliveryForm/updated",
} as const;

type ValueOf<T> = T[keyof T];

type Action = {
  type: ValueOf<typeof ActionType>;
  payload?: Partial<DeliveryForm>;
};
```

- reset
  - stateを初期値に戻す
- updated
  - stateの値を更新する
  - `payload` には次のように更新したい値を渡す

```ts:payloadの例
{
    type: ActionType.updated,
    payload: { zipCode: '1234567' },
}
```

#### actionはsetterではなく、イベントでモデリングする！

これは[Redux 公式スタイルガイド](https://redux.js.org/style-guide/) に書かれているルールの1つでもあります。

actionの定義として、以下のように項目値ごとのsetterとして定義したくなるかもしれません。

```ts:actionをsetterで定義する例
const ActionType = {
    setZipCode: 'setZipCode',
    setPrefecture: 'setPrefecture',
    // ... 省略
```

これでも動くコードはできるのですが、actionがstateの内容を知っていることが前提となるため、stateの項目値の増減とともに、actionを変更する必要があります。
これでは、stateの内容によらず動くロジックを実現できません。

一方、以下のようにイベントベースで定義すると、actionがstateの中身を知る必要がないため、stateの項目値を増減させても、actionに手を入れることなく実装できます。

```ts:actionをイベントで定義する
const ActionType = {
  reset: "deliveryForm/reset",
  updated: "deliveryForm/updated",
} as const;
```

要は、**action と reducer を疎結合にする** ということです。

### 3. reducerの作成

reducerはstateを更新するための関数でした。以下のように定義します。

```ts:reducer
const reducer = (state: DeliveryForm, action: Action): DeliveryForm => {
  switch (action.type) {
    case ActionType.reset: // 初期値を返す
      return initialState;
    case ActionType.updated: // payload で更新する
      return {
        ...state,
        ...action.payload,
      };
    default: {
      const _: never = action.type; // default には何も入らないようにする

      return state;
    }
  }
};
```

そして、reduxの実装ではよく使われていたのですが、action creatorという、 actionを作成するためだけの関数を定義しておきます。
actionの生成は、この関数を呼び出すことで行います。

```ts:action creator
const reset = (): Action => ({
  type: ActionType.reset,
});

const updated = (payload: Partial<DeliveryForm>): Action => ({
  type: ActionType.updated,
  payload,
});
```

:::message
現在のreact docsを見ると、特にaction creatorに関する言及はありません。
個人的な推測ですが、特にjavascriptの環境において、不正なactionが発行されるのを防ぐ手段として使われていたのではないかなと思います。
typescriptで型をつけてあげれば不正なactionが作成されることはなさそうですが、ここではredux実装の慣例に倣っています。
:::

### 4. setStateをdispatchに変更する

最後に、`useState` を `useReducer` に変更し、`setState` を `dispatch(actionCreator())` に置き換えてあげます。

```tsx
const UseReducerForm: FC = () => {
  const [formState, dispatch] = useReducer(reducer, initialState);
  const [isConfirmed, setIsConfirmed] = useState<boolean>(false);

  const handleSubmit = (e: SyntheticEvent) => {
    e.preventDefault();
    console.log(formState);
  };

  const handleReset = (e: SyntheticEvent) => {
    e.stopPropagation();
    setIsConfirmed(false);
    dispatch(reset());
  };

  return (
    <Box p={4} w="md" borderWidth="1px" borderRadius="lg" boxShadow="base">
      <form onSubmit={handleSubmit}>
        <FormLabel htmlFor="zipcode" mt={4}>
          郵便番号
        </FormLabel>
        <Input
          size="md"
          maxLength={7}
          value={formState.zipCode}
          onChange={(e: ChangeEvent<HTMLInputElement>) => {
            dispatch(updated({ zipCode: e.target.value }));
          }}
        />
...以下省略
```

注目してほしいのは、 `handleSubmit`、`handleReset` の中身です。
stateを列挙していた部分がなくなり、スッキリしていることがわかるかと思います。

ここまでのコードはこちら↓

https://github.com/riku929hr/state-vs-reducer/blob/main/src/components/UseReducerForm.tsx

## Redux Toolkit でコード量削減

お気づきかと思いますが、actionやreducerの作成のため、なかなかのコード量を書く必要があります。
が、Redux Toolkitを使うとコード量を減らすことができます。

### Redux Toolkit(RTK) とは？

Reduxのコード量を削減できるRedux公式ライブラリです。

Redux専用ではありますが、useReducerとロジックとしては同じなので、ここで使うこともできます！

### RTKによるリファクタリング

RTKには、sliceという概念があります。これは、

- action
- reducer
- action creator

の3つをまとめたもので、`createSlice` という関数を用いて以下のように生成できます。

```ts
export const deliveryFormSlice = createSlice({
  name: "deliveryForm",
  initialState,
  reducers: {
    reset: () => initialState,
    updated: (state, action: PayloadAction<Partial<DeliveryForm>>) => {
      return { ...state, ...action.payload };
    },
  },
});
```

なんとこれだけで、action、reducer、action creatorが、`deliveryFormSlice` にオブジェクトとして渡されてくるんです。便利ですね。

そして、コンポーネントは以下のようになります。

```tsx
const { reset, updated } = deliveryFormSlice.actions;

const ReduxToolKitForm: FC = () => {
  const [formState, dispatch] = useReducer(
    deliveryFormSlice.reducer,
    initialState,
  );
  const [isConfirmed, setIsConfirmed] = useState<boolean>(false);

  const handleSubmit = (e: SyntheticEvent) => {
    e.preventDefault();
    console.log(formState);
  };

  const handleReset = (e: SyntheticEvent) => {
    e.stopPropagation();
    setIsConfirmed(false);
    dispatch(reset());
  };

  const handleFormChange = (payload: Partial<DeliveryForm>) => {
    dispatch(updated(payload));
  };

  return (
    <Box p={4} w="md" borderWidth="1px" borderRadius="lg" boxShadow="base">
      <form onSubmit={handleSubmit}>
        <FormLabel htmlFor="zipcode" mt={4}>
          郵便番号
        </FormLabel>
        <Input
          size="md"
          maxLength={7}
          value={formState.zipCode}
          onChange={(e: ChangeEvent<HTMLInputElement>) => {
            handleFormChange({
              zipCode: e.target.value,
            });
          }}
        />
// ...以下省略
```

それぞれ個別に定義していたaction creatorやreducerを、`deliveryFormSlice` で作成したものに置き換えているだけです。

最終的なコードはこちら↓
https://github.com/riku929hr/state-vs-reducer/blob/main/src/components/ReduxToolKitForm.tsx

## まとめ

- useStateをたくさん使いすぎると管理が大変になる
- useReducerは構造が複雑な（ツリー構造を持つ）stateの管理に優れている
- actionとreducerは、イベントベースでモデリングすることで、stateと疎結合なロジックにできる
- Redux Tool Kitを使えば、コード量が多い問題を回避できる

今回はわかりやすくフォームを例にとって、useReducerの使い所を考えました。
もちろん、フォーム以外のユースケースはたくさんあると思います。
そんなに頻繁に使うものではないと思っていますが、複数のプロパティを持つstateを管理したい場合、状況に応じてuseReducerの利用を考えてもいいかもしれません。

:::message alert
フォームの入力値を1つのstateで取り回す場合、フォームになにか入力するたびに、フォーム全体がレンダリングされてしまうことになります。
特に項目値が多い場合、パフォーマンスに影響を及ぼしてしまう可能性があります。
その状況に応じて適切な設計、選択をしていくことが重要だと思っています。
:::

## 補足：useStateでもできるよね？

実はuseStateでも似たようなことができちゃいます。

```ts
[state, setState] = useState<T>(initialState);
```

と定義したとき、

```js
// prevState: 更新前のstate
// newState: 更新後のstate
setState((prevState) => newState);
```

とかけるので、

```tsx
const [formState, setFormState] = useState<DeliveryForm>(initialState);

// 省略

return (
    <Box p={4} w="md" borderWidth="1px" borderRadius="lg" boxShadow="base">
      <form onSubmit={handleSubmit}>
        <FormLabel htmlFor="zipcode" mt={4}>
          郵便番号
        </FormLabel>
        <Input
          value={formState.zipCode}
          onChange={(e: ChangeEvent<HTMLInputElement>) => {
            setFormState((prevState) => ({
              ...prevState,
              zipCode: e.target.value,
            }));
          }}
        />
```

のように実装すればuseStateでもstateを一塊にして管理できちゃうんですね…

ただ個人的には、useReducerのメリットの1つとして、コンポーネントからstate管理のロジックを切り離せることが大きいと考えており、同じことをやるならuseReducer使うのがいいのでは？と思っています。

いずれにしても、その場に応じた設計をしていく必要があります。複雑なことをやる前に、もしかしたらstateの定義の仕方や、設計から見直してもいいかもしれません。

## 参考文献

- [react docs](https://ja.react.dev/)
- [りあクト！第4版](https://booth.pm/ja/items/2368045)
- [bulletproof-react docs](https://github.com/alan2207/bulletproof-react)
- [redux](https://redux.js.org/)
- [redux toolkit](https://redux-toolkit.js.org/)
- [recompose](https://github.com/acdlite/recompose/)
