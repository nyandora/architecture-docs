[Express公式サイトのベストプラクティス](https://expressjs.com/ja/advanced/best-practice-performance.html)で、以下のコードに遭遇しました。

```javascript
const wrap = fn => (...args) => fn(...args).catch(args[2])

app.get('/', wrap(async (req, res, next) => {
  const company = await getCompanyById(req.query.id)
  const stream = getLogoStreamById(company.id)
  stream.on('error', next).pipe(res)
}))
```

正直 何やってるのか分からず、困ってしまいました。

これがエラーハンドリングのベストプラクティスということでして、まずはアンチパターンの説明をしていこうと思います。

<details><summary>※エラーハンドリングが上記の公式サイトページで解説されている理由</summary>上記のExpress公式サイトのページでは、パフォーマンスと信頼性についてのベストプラクティスが解説されています。その中でなぜエラーハンドリングが解説されているかと言うと、Express（Node.js）では発生したエラーがキャッチされないと異常終了してしまい、Expressがダウンしてしまいます。そうなると、Epxressアプリケーションの信頼性（可用性）が地に落ちてしまいます。信頼性に絡んでくる部分があるので、エラーハンドリングについて解説されています。</details>

# Expressにおけるエラーハンドリング【アンチパターン】

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

しかし、全ての非同期関数がPromiseを返してくれるわけでは無い、という問題が残ります。

そこで、以下の解決策が出てきます。

```javascript
app.get('/', (req, res, next) => {
  (async () => {
    const company = await getCompanyById(req.query.id)
    const stream = getLogoStreamById(company.id)
    stream.on('error', next).pipe(res) // ここは後述。
  })().catch(next)
}))
```

async関数は、暗黙的にPromiseを返しますので、ハンドラ関数の中でPromiseを返さない非同期関数を使っていても、確実にエラーを補足できます。

ただ、このように「async即時関数でcatch」というコードが散在するのはNGでしょう。

ということで、冒頭のパターンがベストプラクティスということなのです。

# Expressにおけるエラーハンドリング【ベストプラクティス】

ベストプラクティスは以下の通りでした（再掲）。

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

以上です。Expressのハンドラ関数で発生するエラーを、確実にキャッチしていきましょう！


