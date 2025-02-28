## Continuation Interceptor
Coroutines은 일반적으로 특정 Thread에 종속되지않고 독립적으로 실행가능한 실행단위이다.(Non-Blocking 특성)  
그러나, Android의 UI 쓰레드와 같이 특정 작업을 특정 Thread에서만 동작하도록 제한을 해야하는 상황이 있을 수 있다.  
이러한 문제를 해결하기 위해 Continuation Interceptor를 활용해야한다.

Continuation Interceptor는 해당 suspend 함수의 Continuation을 가로채어 함수가 정지하고 재개할 때 의도한 동작으로 수행하도록 유도할 수 있는 기능이다.
이는 **Dispatcher Context에서 사용하는 핵심 개념**이다. 

본격적인 설명에 앞서 코루틴의 생명주기에 대해 자세히 알 필요가 있다. 
아래 launch를 이용한 스니펫을 참고하여 생각해보자.
```kotlin
launch(Swing) {
	initialCode() // execution of initial code
	f1.await() // suspend point #1
	block1() // execution #1
	f2.await() // suspend point #2
	block2() // execution #2
	
}
```
코루틴은 초기에 initiaCode()부터 다음 suspension point까지 작업을 실행한다. suspension point에서 재게되는 시점에는 block1()을 이어서 실행하고, 다시 정지되고, 다시 재게되면서 block2()를 실행하고 완료한다.

여기서 Continuation Interceptor는 suspenion point에서 다시 일반 execution으로 재개할 시점에 continuation을 가로채서 특정 작업을 수행하는 Continuation으로 래핑하는 역할을 한다.

```kotlin
interface ContinuationInterceptor : CoroutineContext.Element {
    companion object Key : CoroutineContext.Key<ContinuationInterceptor>
    fun <T> interceptContinuation(continuation: Continuation<T>): Continuation<T>
    fun releaseInterceptedContinuation(continuation: Continuation<*>)
}
```
* interceptContinuation : continuation을 가로채서 새로운 Continuation으로 리턴함.
* releaseInterceptedContinuation : 가로채어진 continuation이 더 이상 필요없을 때 반환

이제 Swing Context를 활용한 예시를 들어보면, Swing은 내부적으로 SwingContinuation으로 래핑하여 반환하고 있다.
```kotlin
object Swing : AbstractCoroutineContextElement(ContinuationInterceptor), ContinuationInterceptor {
    override fun <T> interceptContinuation(continuation: Continuation<T>): Continuation<T> =
        SwingContinuation(continuation)
}
```

SwingContinuation의 세부 구현을 보면, 재개(resultWith)시점에 UI Thread에서 실행하도록 처리되어 있는 것을 볼 수 있다.
```kotlin
private class SwingContinuation<T>(val cont: Continuation<T>) : Continuation<T> {
	override val context: CoroutineContext = cont.contxt
	override fun resumeWith(result: Result<T>) {
		SwingUtilities.invokeLater { cont.resultWith(result) }
	}
}
```

이로써 이제 Swing Context 환경에서 동작하는 execution 들은 resume 시점에 UI쓰레드에서 동작한다.
