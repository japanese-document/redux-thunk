# Redux Thunk

ReduxのThunk [middleware](https://redux.js.org/advanced/middleware)

## 2.xへUpdateする際の注意点

今あるチュートリアルはReact Thunk 1.xを対象にしている物がほとんどです。React Thunk 2.xでそれを試すと問題が生じるかもしれません。**CommonJSでRedux Thunk 2.xを
[インポートする時は`.default`を付けることを忘れないでください。](https://github.com/reduxjs/redux-thunk/releases/tag/v2.0.0)**

```diff
- const ReduxThunk = require('redux-thunk')
+ const ReduxThunk = require('redux-thunk').default
```

ES modulesを使っている場合、変更する必要はありません。

```js
import ReduxThunk from 'redux-thunk'; // 変更なし 😀
```

2.xから[UMD build](https://unpkg.com/redux-thunk/dist/redux-thunk.min.js)をサポートします。

```js
const ReduxThunk = window.ReduxThunk.default;
```

見ての通り、これも末尾に`.default`が必要です。

## なぜRedux Thunkが必要なのか？

素のRedux storeはactionをdispatchすることによるシンプルで同期的な更新しかできません。middlewareはstoreの可能性を拡張します。そして、dispatchとstoreの間で非同期ロジックを書くことを可能にします。

[Redux Thunk](https://github.com/reduxjs/redux-thunk)は、storeにアクセスする必要がある複雑な同期ロジックや単純なAjaxリクエストような非同期ロジックが含まれる基本的なRedux side effectロジックに適したmiddlewareです。

Redux Thunkの有用性に関する詳しい内容は以下の記事にあります。

- **Stack Overflow: Dispatching Redux Actions with a Timeout**  
  http://stackoverflow.com/questions/35411423/how-to-dispatch-a-redux-action-with-a-timeout/35415559#35415559  
  Dan Abramovが段階的に改善過程(inline async calls, async action creators, thunk middleware)を説明しながらReduxでの非同期処理の基本を説明しています。 

- **Stack Overflow: Why do we need middleware for async flow in Redux?**  
  http://stackoverflow.com/questions/34570758/why-do-we-need-middleware-for-async-flow-in-redux/34599594#34599594  
  Dan AbramovがThunkと非同期middlewareを使う理由とThunkを使った便利なパターンを説明しています。

- **What the heck is a "thunk"?**  
  https://daveceddia.com/what-is-a-thunk/  
  "thunk"の一般的な意味とそれのReduxを使った具体的な説明

- **Thunks in Redux: The Basics**  
  https://medium.com/fullstack-academy/thunks-in-redux-the-basics-85e538a3fe60  
  thunkとは何か、何を解決するのか、その使い方の詳解


**[Redux FAQの非同期middlewareの選択](https://redux.js.org/faq/actions#what-async-middleware-should-i-use-how-do-you-decide-between-thunks-sagas-observables-or-something-else)** を読むことをお勧めします。

このRedux Thunk middlewareはReduxコアライブラリに含まれていません。私たちが提供している **[`@reduxjs/toolkit`](https://github.com/reduxjs/redux-toolkit)** ではデフォルトで使われています。

## React Thunkを使う理由

Redux Thunk [middleware](https://redux.js.org/advanced/middleware)を使うと、actionの代わりに関数を返すaction creatorを書くことができるようになります。React Thunkを使うとactionがdispatchされるタイミングを遅らせたり、条件に応じてdispatchすることができます。action creatorの内側の関数はstoreのメソッドである`dispatch`と`getState`を引数として受け取ります。

非同期でdispatchを行う関数を返すaction creator

```js
const INCREMENT_COUNTER = 'INCREMENT_COUNTER';

function increment() {
  return {
    type: INCREMENT_COUNTER,
  };
}

function incrementAsync() {
  return (dispatch) => {
    setTimeout(() => {
      // `dispatch`でsync actionとasync actionを呼び出すことができます。
      dispatch(increment());
    }, 1000);
  };
}
```

条件に応じてdispatchする関数を返すaction creator

```js
function incrementIfOdd() {
  return (dispatch, getState) => {
    const { counter } = getState();

    if (counter % 2 === 0) {
      return;
    }

    dispatch(increment());
  };
}
```

## thunkって何？

[thunk](https://en.wikipedia.org/wiki/Thunk)は評価を遅延させるために式をラップする関数です。

```js
// 1 + 2の計算はすぐに実行されます。
// x === 3
let x = 1 + 2;

// 1 + 2の計算は遅延します。
// あとでfooを呼び出した時に
// fooがthunkです。
let foo = () => 1 + 2;
```

この用語の[語源](https://en.wikipedia.org/wiki/Thunk#cite_note-1)は"think"の過去形のおどけた表現です。

## インストール

```bash
npm install redux-thunk
```

Redux Thunkを有効にするために[`applyMiddleware()`](https://redux.js.org/api/applymiddleware)を使います。

```js
import { createStore, applyMiddleware } from 'redux';
import thunk from 'redux-thunk';
import rootReducer from './reducers/index';

// このAPIはredux@>=3.1.0で使えます。
const store = createStore(rootReducer, applyMiddleware(thunk));
```

## 非同期のコントロールフローの構築

thunk形式のaction creatorの内側の関数で、その引数である`dispatch`の戻り値を戻り値にするとします。こうするとthunk形式のaction creator内で別のthunk形式のaction creatorをdispatchして、それの戻り値のPromiseの完了を待つような非同期のコントロールフローを構築することが容易になります。

```js
import { createStore, applyMiddleware } from 'redux';
import thunk from 'redux-thunk';
import rootReducer from './reducers';

// このAPIはredux@>=3.1.0で使えます。
const store = createStore(rootReducer, applyMiddleware(thunk));

function fetchSecretSauce() {
  return fetch('https://www.google.com/search?q=secret+sauce');
}

// これらは普段使っている普通のaction creatorです。
// これらが返すactionはmiddlewareなしでdispatchすることができます。
// そのactionは単に“facts”を表しているのみであり、“async flow”を表していません。

function makeASandwich(forPerson, secretSauce) {
  return {
    type: 'MAKE_SANDWICH',
    forPerson,
    secretSauce,
  };
}

function apologize(fromPerson, toPerson, error) {
  return {
    type: 'APOLOGIZE',
    fromPerson,
    toPerson,
    error,
  };
}

function withdrawMoney(amount) {
  return {
    type: 'WITHDRAW',
    amount,
  };
}

// middlewareなしでも、actionをdispatchすることができます。
store.dispatch(withdrawMoney(100));

// では、APIの呼び出しやrouterのトランジションのようなasync actionを開始する必要がある場合はどうしますか？

// そこで、thunkの登場です。
// ここでいうthunkとは非同期処理を実行するためにdispatchされる関数でactionをdispatchすることができてstateを読み込むことができます。
// これはthunkを返すaction creatorです。
function makeASandwichWithSecretSauce(forPerson) {
  // "thunk"形式の関数を返すことによって制御を別の物に変更することができます。
  // この関数が`dispatch`に渡されると、React Thunk middlewareは通常とは別ルートの処理を行います。
  // その処理はその関数に`dispatch`と`getState`を引数として渡して実行する処理です。
  // これによって、thunk関数内でロジックを実行したり、storeを操作することが可能になります。
  return function(dispatch, getState) {
    return fetchSecretSauce().then(
      (sauce) => dispatch(makeASandwich(forPerson, sauce)),
      (error) => dispatch(apologize('The Sandwich Shop', forPerson, error)),
    );
  };
}

// React Thunk middlewareを使うと、thunk形式のasync actionを普通のactionのようにdispatchすることができます。

store.dispatch(makeASandwichWithSecretSauce('Me'));

// dispatchはthunkの戻り値を返すようになっています
// だから、Promiseをthunkが返した場合、それのチェインをすることができます。

store.dispatch(makeASandwichWithSecretSauce('My partner')).then(() => {
  console.log('Done!');
});

// 実際には、他のaction creatorから生成した普通のactionやasync actionをdispatchするaction creatorを書くことができます。
// そして、Promiseを使ってコントロールフローを構築することができます。

function makeSandwichesForEverybody() {
  return function(dispatch, getState) {
    if (!getState().sandwiches.isShopOpen) {
      // Promiseを返す必要はありませんが、これは便利なやり方です。
      // これで、呼び出し元はasync actionをdispatchした結果に対して常に`.then()`を実行することができます。

      return Promise.resolve();
    }

    // 素のactionオブジェクトとthunkをdispatchすることができます。複数のasync actionで1つのフローを構成することができます。

    return dispatch(makeASandwichWithSecretSauce('My Grandma'))
      .then(() =>
        Promise.all([
          dispatch(makeASandwichWithSecretSauce('Me')),
          dispatch(makeASandwichWithSecretSauce('My wife')),
        ]),
      )
      .then(() => dispatch(makeASandwichWithSecretSauce('Our kids')))
      .then(() =>
        dispatch(
          getState().myMoney > 42
            ? withdrawMoney(42)
            : apologize('Me', 'The Sandwich Shop'),
        ),
      );
  };
}

// データが利用可能になるまで待ってから、同期的にアプリケーションをレンダリングすることができるので、これはとても便利です。

store
  .dispatch(makeSandwichesForEverybody())
  .then(() =>
    response.send(ReactDOMServer.renderToString(<MyApp store={store} />)),
  );

// propsが変更された時は必要なデータを取得するために、コンポーネントからthunk形式のasync actionをdispatchすることができます。

import { connect } from 'react-redux';
import { Component } from 'react';

class SandwichShop extends Component {
  componentDidMount() {
    this.props.dispatch(makeASandwichWithSecretSauce(this.props.forPerson));
  }

  componentDidUpdate(prevProps) {
    if (prevProps.forPerson !== this.props.forPerson) {
      this.props.dispatch(makeASandwichWithSecretSauce(this.props.forPerson));
    }
  }

  render() {
    return <p>{this.props.sandwiches.join('mustard')}</p>;
  }
}

export default connect((state) => ({
  sandwiches: state.sandwiches,
}))(SandwichShop);
```

## カスタム引数の注入

2.1.0から、`withExtraArgument`関数を使うとカスタム引数の注入ができるようになります。

```js
const store = createStore(
  reducer,
  applyMiddleware(thunk.withExtraArgument(api)),
);

// 後で
function fetchUser(id) {
  return (dispatch, getState, api) => {
    // カスタム引数の`api`は、ここで使うことができます。
  };
}
```

複数の値を渡したい場合は、それらを1つのオブジェクトにまとめます。ES2015のshorthand property namesを使うとより簡潔に書くことができます。

```js
const api = "http://www.example.com/sandwiches/";
const whatever = 42;

const store = createStore(
  reducer,
  applyMiddleware(thunk.withExtraArgument({ api, whatever })),
);

// 後で
function fetchUser(id) {
  return (dispatch, getState, { api, whatever }) => {
    // カスタム引数の`api`とその他は、ここで使うことができます。
  };
}
```

## License

### Japanese part

Attribution-NonCommercial-NoDerivatives 4.0 International (CC BY-NC-ND 4.0)

Copyright (c) 2021 38elements

### Other

The MIT License (MIT)

Copyright (c) 2015-present Dan Abramov

Permission is hereby granted, free of charge, to any person obtaining a copy of this software and associated documentation files (the "Software"), to deal in the Software without restriction, including without limitation the rights to use, copy, modify, merge, publish, distribute, sublicense, and/or sell copies of the Software, and to permit persons to whom the Software is furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM, OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE SOFTWARE.
