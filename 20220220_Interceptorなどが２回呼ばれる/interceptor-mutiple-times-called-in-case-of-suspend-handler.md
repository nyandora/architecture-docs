バージョン情報

* Spring Boot: 2.6.3

Spring MVCではKotlinのコルーチンがサポートされています。

Controllerのhandler関数にsuspendをつけると、Springがコルーチンを起動してhandler関数を実行してくれます。

```kotlin
@RestController
class DemoController {
    @GetMapping("suspend")
    suspend fun indexSuspend(): String {
        return "suspend handler is executed."
    }
}
```

このようにhandler関数にsuspendをつけると、Controllerの前段にあるInterceptorやControllerAdviceが複数回呼び出されてしまいます。

シーケンスは以下のような感じです。

![](https://raw.githubusercontent.com/nyandora/universal-static-resources/main/seq.svg)

現時点では、複数回呼び出されてしまうこと自体は避けられないようですので、複数回呼び出されても問題ないようにInterceptorやControllerAdviceを作り込んでおくしかなさそうです。

Interceptorの実装例です。

```kotlin
@Component
class DemoHandlerInterceptor(
        private val asyncHandlerUtil: AsyncHandlerUtil
) : HandlerInterceptor {
    override fun preHandle(request: HttpServletRequest, response: HttpServletResponse, handler: Any): Boolean {
        // 非同期処理（Controllerのsuspend fun）の完了後、
        // 後続処理を再開するためにDispatcherServletに再度ディスパッチされる。
        // その時にInterceptorが呼ばれた場合（つまり２回目に呼ばれた場合）は何もしない。
        if (asyncHandlerUtil.isDispatchedToResume(request)) {
            // falseを返してしまうと後続処理の再開も止まってしまうので、trueを返す必要あり。
            return true
        }

        // 非同期処理（Controllerのsuspend fun）を起動するために
        // DispatcherServletにディスパッチされた時のみ（つまり１回目に呼ばれた場合のみ）、
        // 本来の処理を行う。
        println("Interceptorが本来やるべき処理")

        return true
    }
}

@Component
class AsyncHandlerUtil {
    fun isDispatchedToResume(request: HttpServletRequest): Boolean {
        val manager = WebAsyncUtils.getAsyncManager(request)

        // WebAsyncManager.hasConcurrentResult()を利用すれば、
        // 非同期処理（Controllerのsuspend fun）の完了後に再度ディスパッチされたのかどうかを判定できる。
        // see: https://spring.pleiades.io/spring-framework/docs/current/javadoc-api/org/springframework/web/context/request/async/WebAsyncManager.html
        return manager.hasConcurrentResult()
   }
}
```

```kotlin
@RestControllerAdvice
class DemoControllerAdvice() {
    @ModelAttribute("myModel")
    fun addSomeAttributeToModel(request: HttpServletRequest): String? {

        // ２回目にディスパッチされた場合は前回の結果を使う。
        // ２回目にnullを設定するなどもアリかもしれないが、
        // 前回の結果を使う方式の方が堅牢（何回呼ばれても問題は起こり得ない）。
        val cached = request.getAttribute("myModelAttr") as String?
        if (cached != null) {
            return cached
        }

        // 属性値の生成
        val attributeValue = "model!!"

        // ２回目にディスパッチされた時のために、結果をrequestに保存する。
        request.setAttribute("myModelAttr", attributeValue)

        return attributeValue
    }

    @ExceptionHandler
    @ResponseStatus(HttpStatus.INTERNAL_SERVER_ERROR)
    fun handleException(e: Throwable): String {
        // ハンドラ（Controllerのsupend fun）で例外がスローされた場合、
        // ２回目のディスパッチでここにくる。
        return "error occurred."
    }
}
```

コードは以下に置いてあります。

https://github.com/nyandora/spring-handler-executed-multiple-times-demo
