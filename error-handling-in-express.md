[Express公式サイトのベストプラクティス](https://expressjs.com/ja/advanced/best-practice-performance.html)には、パフォーマンスと信頼性についてのベストプラクティスが解説されています。

その中で、適切なエラーハンドリングのベストプラクティスについて解説されています。

Express（Node.js）では発生したエラーがキャッチされないと、プロセスが異常終了したりハングしてしまいます。そうなると、Epxressアプリケーションの信頼性（可用性）が地に落ちてしまいます。このようにエラーハンドリングの適切さは信頼性に大きく影響するため、エラーハンドリングはとっても重要なのです。

今回の投稿では、Express公式サイト（と、そこからリンクされる参考ページ）で紹介されているエラーハンドリングのベストプラクティスを、できるだけ分かりやすく説明させていだこうと思います。

まずはアンチパターンいついて説明します。

# Expressにおけるエラーハンドリング【アンチパターン】

### その①：コールバック関数で、手動でnextを呼び出す。

非同期関数（以下の例では`queryDb()`, `makeCsv()`）を呼び出した後のエラーハンドリングでは、コールバック関数でnext関数にエラーを渡して実行する必要があります。

```javascript
app.get('/', function (req, res, next) {
  queryDb(function (err, data) {
    if (err) return next(err)
    // handle data

    makeCsv(data, function (err, csv) {
      if (err) return next(err)
      // handle csv

    })
  })
})
app.use(function (err, req, res, next) {
  // handle error
})
```

しかし、それは以下の点で問題があります。

  * 非同期関数でエラーが発生した場合、nextの呼び出しをうっかり忘れてしまう恐れがある。
  * 非同期関数が正常に処理されたとしても、コールバック中にエラーが起きることはある。その場合に、nextの呼び出しをうっかり忘れてしまう恐れがある。

### その②：Promiseを利用する。

そこで、非同期関数を呼び出してPromiseを受け取り、Promiseのcatch()にnext関数を登録すれば良い、ということになります。

```javascript
app.get('/', function (req, res, next) {
  // do some sync stuff
  queryDb()
    .then(function (data) {
      // handle data
      return makeCsv(data)
    })
    .then(function (csv) {
      // handle csv
    })
    .catch(next)
})
app.use(function (err, req, res, next) {
  // handle error
})
```

### その③：async awaitを利用する。

ただ、この書き方は冗長で読みづらいです。そこで、async awaitを使って読みやすくしてみましょう。

```javascript
app.get('/', (req, res, next) => {
  (async () => {
    let data = await queryDb()
    // handle data
    let csv = await makeCsv(data)
    // handle csv
  })().catch(next)
}))
```

async関数は、暗黙的にPromiseを返します。このため、

 * `queryDb()`や`makeCsv()`でエラーがthrowされる
 * async関数内の他の箇所でエラーがthrowされる

といった場合は、throwされたエラーでPromiseがrejectされます。すると、catchに登録された関数`next`にそのエラーが渡されて`next`が実行されます。（これは直前のパターンでも同じことです。）

これで、Promiseのチェーンで書かれたコードよりは、少し読みやすくなりました。

ただ、このように「async即時関数でcatch」というコードが散在するのはNGでしょう。ということで、冒頭のパターンがベストプラクティスということなのです。

先に進む前に、念のため論外なパターンも紹介しておきます。

### その④：【論外】ハンドラ関数をasyncにするだけ。

なお、以下のように、ハンドラ関数をasyncにしただけではNGです。

```javascript
app.get('/', async (req, res) => {
  let data = await queryDb()
  // handle data
  let csv = await makeCsv(data)
  // handle csv
}))
```

この場合、`queryDb()`や`makeCsv()`でエラーがthrowされると、async関数が返すPromiseがrejectされます。しかし、ExpressはrejectされたPromiseをハンドリングしません。

すると、サーバー側の処理は終了となります（UnhandledPromiseRejectionWarningが発生するはずです）。クライアント側（GETメソッドの呼び出し元）には何も返されません。

その結果、多くの場合、呼び出し元はタイムアウトとなります。

Epxressのエラーハンドリング用ミドルウェアでハンドリングするためには、先ほどの例のように、`next`が呼び出されるようにする必要があるのです。


# Expressにおけるエラーハンドリング【ベストプラクティス】

ベストプラクティスは以下の通りです。

```javascript
const wrap = fn => (...args) => fn(...args).catch(args[2])

app.get('/', wrap(async (req, res, next) => {
  const company = await getCompanyById(req.query.id)
  const stream = getLogoStreamById(company.id)
  stream.on('error', next).pipe(res)
}))
```

パーツごとに説明していきます。

まずは以下の部分です。

```javascript
const wrap = fn => (...args) => fn(...args).catch(args[2])
```

wrap関数は、`fn`を受け取り、`(...args) => fn(...args).catch(args[2])`という関数を返します。私がはじめにこれを見たときは「カリー化なのかな・・」と思ったのですが、そうではありませんでした。

今回では、`fn`にはPromiseを返す関数が指定されます。wrap関数の引数に指定されている`async (req, res, next) => …`という関数が、正にこれに当たります。async関数は暗黙的にPromiseを返すのでした。

次に、

```javascript
(...args) => fn(...args).catch(args[2])
```

という関数では何をやっているのでしょうか？

それは以下のとおりです。

* 今回の場合、`fn`にはasync関数が指定されます。`async (req, res, next) => …`のことです。
* 引数`...args`（今回の場合は、expressから渡される`req, res, next`）を`fn`に渡して`fn`を実行します。
* `fn`を実行するとPromiseが返ります。そのPromiseのcatchにargs[2]、つまりexpressから渡される引数の3番目である`next`を指定しています。`next`は次のExpressミドルウェアを呼び出す関数です。Promiseでrejectとなった場合のコールバックとして、`next`を登録しているのです。

`fn`（つまり`async (req, res, next) => …`）の実行中にエラーが発生すると、Promiseがrejectされます。すると、catchに登録したコールバックである`next`が実行されます。発生したエラーは、`next`に引数として渡されます。

その結果、Expressのエラーハンドリング用のミドルウェアが実行され、無事にエラーが捕捉されるというわけです。

この技法を使うことで、「async即時関数でcatch」というコードが散在してしまう、という事態を避けることができます。

また

```javascript
stream.on('error', next).pipe(res)
```

についても説明します。

これはストリームなどを使う場合のみ該当する話なのですが、async関数の中であっても、イベント・エミッター (ストリームなど) により、例外がキャッチされないことがあります。そこでストリームのerrorイベントが発生したときに、確実に`next`が呼び出されるようにしています。

# 注意点

上記のベストプラクティスが通用するのは、async関数内で呼び出す非同期関数がPromiseを返す場合だけです。awaitはあくまでもPromiseがresloveされるかrejectされるのを待つ機能だからです。

ですので、上記のasync関数内でPromiseを返さない非同期関数を呼び出す場合、はじめに説明したアンチパターンと同じことになってしまいます。つまり、コールバック関数でnext(err)の実装を忘れてしまう、というリスクがあります。

今どき、Promiseを返さない非同期関数を使わなきゃいけない、というケースは少ないかなと思いますが、もしそういったケースに当てはまる場合は、以下のうちいずれかを選択しましょう。

 * Promiseを返さない非同期関数から、Promiseを返す関数を自動生成し、後者を呼び出すようにする。Node.js標準の`util.promisify()`を使うことで、自動生成を実現できます。[こちら](http://blog.manaten.net/entry/util-promisify)が参考になります。（以前は、`Bluebird`等のライブラリで実現していましたが、Node.js 8から標準で組み込まれました。）
 * 「Promiseを返さない非同期関数」をラップした関数を定義して、Promiseを返すようにする。[こちら](https://developer.ibm.com/articles/promises-in-nodejs-an-alternative-to-callbacks/#creating-raw-promises)が参考になります。
 * １つ目のアンチパターンの難点を受け入れ、コールバック関数でnext(err)を忘れずに実装する、と堅く誓う（ソースコードレビューが大変…）。

# 参考

https://expressjs.com/ja/advanced/best-practice-performance.html#handle-exceptions-properly

https://strongloop.com/strongblog/async-error-handling-expressjs-es7-promises-generators/

https://developer.ibm.com/articles/promises-in-nodejs-an-alternative-to-callbacks/

http://blog.manaten.net/entry/util-promisify

以上です。Expressのハンドラ関数で発生するエラーを、確実にハンドリングしていきましょう！


