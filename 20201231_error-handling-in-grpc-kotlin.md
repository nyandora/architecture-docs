タイトル：gRPCにおけるエラーハンドリング(Kotlin版)

gRPCサーバーでエラーとなった場合、どのようにハンドリングすれば良いのか調べましたので、まとめておこうと思います。なお、Unary RPCs（SimpleRPC)の場合についてのみ記載します。Streamingを利用する場合については考慮しません。

ポイントは、Googleが公式にオススメしているメッセージ構造を利用することです。

具体的には、[gRPCの公式サイト](https://grpc.io/docs/guides/error/#richer-error-model)でオススメされている[error_details.proto](https://github.com/googleapis/googleapis/blob/master/google/rpc/error_details.proto)のメッセージ構造を利用しましょう。このprotoファイルはproto-google-common-protos-{version}.jarに同梱されています。このprotoファイルには様々なメッセージタイプが定義されており、それらに対応するJavaクラスも同じJARに同梱されています。ですので、クライアント側／サーバー側の双方のアプリケーションコードで、これらのメッセージタイプを利用することができます。

独自のメッセージ構造を定義することはもちろんできますが、Googleがこれまでの経験をもとに考え抜いてくれたものを使えば、「車輪を再発明するムダ」を回避できます。使わない手はありません。

今回ご紹介するソースコードの全量は、[こちら](https://github.com/nyandora/grpc-kotlin-trial)にあります。

# サーバー側

```kotlin
private class HelloWorldService : GreeterGrpcKt.GreeterCoroutineImplBase() {
  override suspend fun sayHello(request: HelloRequest): HelloReply {
    //　バリデーションを実行します。
    validate(request)

    return HelloReply.newBuilder().setMessage(request.name).build()
  }

  private fun validate(request: HelloRequest) {
    // nameプロパティのドメインは「10文字未満の文字列」とします。
    if (request.name.length >= 10) {
      val nameFieldError = BadRequest.FieldViolation.newBuilder()
            .setField("name")
            .setDescription("More than 10 characters are not allowed.").build()

      val badRequestError = BadRequest.newBuilder()
            .addFieldViolations(nameFieldError).build()

      val errorDetail = Metadata()
      errorDetail.put(ProtoUtils.keyForProto(badRequestError), badRequestError)

      throw StatusException(Status.INVALID_ARGUMENT, errorDetail)
    }
  }
}
```

## ポイント

アプリケーションコードで`io.grpc.StatusException`をスローすると、gRPCのKotlinコードジェネレータにより生成されたスタブコードがそれをハンドリングし、クライアントに適切に応答してくれます。ここで想定するコードジェネレータは、gRPC公式サイトのチュートリアルで利用されているものです。具体的には、[こちらのgithubリポジトリ](https://github.com/grpc/grpc-kotlin)にある「protoc-gen-grpc-kotlin」をKotlinコードジェネレータとして使っています。

`StatusException`には[googleが定義しているgRPCステータスコード](https://cloud.google.com/apis/design/errors#handling_errors)を設定します。上記の例のように入力値に問題がある場合は、`Status.INVALID_ARGUMENT`を使いましょう。

`StatusException`にはエラーの詳細情報も詰めます。以下の目的を達成できる、必要最小限の情報だけを設定しましょう。

* クライアント側の開発者が適切な対応をとれるようにする。
* クライアント側のコードが適切にハンドリングできるようにする。

上記の例のように入力値に問題がある場合は、`com.google.rpc.BadRequest`を使いましょう。[error_details.proto](https://github.com/googleapis/googleapis/blob/master/google/rpc/error_details.proto)を見ると分かるのですが、この`BadRequest`にはネストされたメッセージタイプとして`FieldViolation`が定義されています。`FieldViolation`にはフィールドごとに発生したエラー情報を詰めることができます。`BadRequest`には複数の`FieldViolation`を設定できるので、入力値の全量をチェックし、発生した全てのエラーを一気にクライアントに返す、ということもできます。

`StatusException`にエラーの詳細情報を詰める場合は、`io.grpc.Metadata`を利用します。`Metadata`オブジェクトにエラーの詳細情報を設定し、それを`StatusException`に詰めます。

# クライアント側

```kotlin
class HelloWorldClient(private val channel: ManagedChannel) : Closeable {
  private val stub: GreeterGrpcKt.GreeterCoroutineStub = GreeterGrpcKt.GreeterCoroutineStub(channel)

  suspend fun greet(name: String) {
    val request = HelloRequest.newBuilder().setName(name).build()
    try {
      val response = stub.sayHello(request)
      println("Received: ${response.message}")
    } catch (e: StatusException) {
      if (e.status == Status.INVALID_ARGUMENT) {
        val badReqErrDetail = e.trailers[ProtoUtils.keyForProto(BadRequest.getDefaultInstance())]
        badReqErrDetail?.fieldViolationsList?.forEach {
          println("error occurred at ${it.field}")
          println("error description: ${it.description}")
        }
      }
    }
  }

  // 略
}
```

## ポイント

gRPCコールの結果、エラー応答がかえってきた場合、クライアントコードでは`StatusException`がスローされます。クライアントではステータスコードを判定し、`StatusException`からエラーの詳細情報を取り出すことができます。

# バリデーションエラー以外のケース

上記の例では、バリデーションエラーが発生した場合に`com.google.rpc.BadRequest`を利用しています。しかし発生するエラーはバリデーションエラーばかりではありません。エラーの内容に応じて適切なメッセージ構造を使いましょう。[error_details.proto](https://github.com/googleapis/googleapis/blob/master/google/rpc/error_details.proto)には、エラー時に利用できる様々なメッセージ構造が定義されています。適宜参照して使い分けをしましょう。

# 参考
GoogleのAPI設計ガイド：　https://cloud.google.com/apis/design/errors

