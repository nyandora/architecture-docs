Spring WebFluxでは、Node.jsのようなイベントループによるノンブロッキングな動きが実現される。
呼び出し元がSubscriber, 呼び出し先がPublisher（Mono/Flux）を即時で呼び出し元に応答する。
呼び出し元がPublisherをsubscribeし、Publisherは呼び出し元が指定するコールバックを呼び出す。
呼び出し元はPublisherからの通知（onNext, onCompleteなどのコールバック）を受けて、後続の処理を行う。
応答を受けてから通知を受けるまでの間は、スレッドはブロックされず、他のことをやれる。

これをCoroutineを使って実現するのが`kotlinx-coroutines-reactor`。
位置付けとしては、Reactive Streamsを実装するReactorから利用されるユーティリティ。
Reactorはこのユーティリティを使ってPublisher（Mono/Flux）を生成する。
FWはがPublisherをsubscribeすると、Publisherはコルーチンを起動し、そのコルーチン内でControllerを呼び出される。
Contollerの処理が終わると、PublisherはSubscriber（FW側）に通知(onNextなど)を行い、FWは呼び出し元にレスポンスする。
Controller側はメソッドに`suspend`をつけるだけで、この流れに乗ることができる。


[コード](https://github.com/Kotlin/kotlinx.coroutines/blob/master/reactive/kotlinx-coroutines-reactor/src/Mono.kt#L34)によると、GlobalScopeでコルーチンが起動される。Jobとして起動する。このためワーカースレッドはブロックされない。

Controller内でasyncで子コルーチンが起動し、CoroutineContextに特定のDispatcherが指定されていない場合、Dispatchers.Default、つまりスレッドプールから取得された別のスレッドがその子コルーチンを実行する。
参考：http://tateisu.hatenablog.com/entry/2018/11/09/045759
参考：https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/async.html

では、`kotlinx-coroutines-reactor`を使って起動されたコルーチンには、Dispatcherが指定されるのだろうか？


async内でIO待ちになると中断される。IO待ちが終わると、空いてるスレッドが処理を再開する。
IOが終わったことって、どうやって検知するのだろうか・・？
async内の処理が、IO待ちが発生するようなものならDispathers.IOを指定すべきでは？
CPU負荷の高いものについては、Dispatchers.Defaultが適切っぽい。
参考：https://developer.android.com/kotlin/coroutines/coroutines-adv?hl=ja
参考！！：https://stackoverflow.com/questions/59039991/difference-between-usage-of-dispatcher-io-and-default

しかし、Dispatchers.Default/IOを使うのはアンチパターン。。
https://www.techyourchance.com/coroutines-dispatchers-default-and-dispatchers-io-considered-harmful/


# 何が嬉しいのか？
* サーバ側アプリケーションにとって
  * メモリの消費を抑えて並列処理を実行できる。
    * 少ないスレッドで多数の並列処理が可能。スレッドは1本あたり2MB程度を要するので、多数のスレッド起動は負荷が高い。
  * CPUを余すところなく有効に活用できる。
    * DBアクセスや通信処理などの「待ち」が発生する時に、スレッドに他の仕事をさせることができる。
* スマホアプリケーションにとって（Android）
  * ユーザーを待たせないUIを実現できる。
    * 重い処理をバックグランドで起動することで、メインスレッドがブロックされるのを回避できる。



GlobalScopeの使い所は具体的には分からず。Strucured Concurrencyの場合のメリットが得られず
管理する手間やバグるリスクが増えるだけなので、通常は使わないでいるべき。いつか使い所が分かる日が来るのかも。。

asyncなどはCoroutineScopeの拡張関数。
CoroutineScopeがレシーバ(this)となっている時しか呼び出せないということ。




produceとかを調べたい。