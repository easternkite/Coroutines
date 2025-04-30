# withContext

지정된 코루틴 컨텍스트로 주어진 일시 중단 블록을 호출하고, 그것이 완료될 때까지 일시 중단한 뒤 결과를 반환합니다.

그 block의 context는 현재의 coroutineContext와 직접 명시한 context가 합쳐진 형태로 제공됩니다.
`coroutineContext + context`. 이 정지함수는 cancellable합니다. 이는 context의 cancellation을 즉시 체크하고 active 상태가 아닐경우 CancellationException을 던집니다. (context가 취소되면 바로 감지하고 Exception 던진다는 말)

> 여기서 active하지 않다는 말은 코루틴 콘텍스트가 취소되었거나, 이미 완료된 상태일 경우에 대한 상태다.


만약 withContext로 다른 Dispatcher를 전달하였을 때, 필수적으로 추가적인 dispatch 작업이 수행됩니다. ㅠblock은 그자체로 즉시 실행될 순 없고, 작업 실행을 위해 다른 ThreadPool로 dispatch된 상태에서 실행됩니다. 그 후에 block이 최종적으로 완료되었을 경우, 현재 dispatcher는 원래의 dispatcher로 자동 전환됩니다.

여기서 알아두어야 할것은 withContext의 결과가 되는 값은 cancellable한 방법으로 원래의 context로 dispatch됩니다. > 즉, `withContext`가 끝나고 **다시 원래의 coroutineContext로 돌아가려는 시점에**  
그 원래의 컨텍스트가 **이미 취소된 상태라면**,  `withContext`의 결과는 **버려지고** `CancellationException`이 던져집니다.

앞서 설명한 cancellation 동작은 dispatcher가 변경된 경우에만 활성화됩니다. 예를들어, `withContext(NonCancellable)`을 사용한다고 한다면 이는 dispatcher의 변화가 없고 block의 진입시점이나 종료시점에서 해당 호출은 취소되지 않습니다.

withContext의 내부를 살펴보면, 다음과 같이 dispatcher가 변경되지 않은 케이스에 대해서  빠르게 분기하도록 FAST PATH를 정의하고 있다. 두 루트는 결과적으로 Dispatcher가 변경되었는지 체크하기 위해서 분기된 코드이며, 실질적으로 Dispatcher가 변경되지 않았을 땐 UndispatchedCoroutine을 리턴하는 것을 볼 수 있다.
1. context 자체가 같을 경우
2. ContinuationInterceptor 가 같을 경우
```kotlin
suspendCoroutineUninterceptedOrReturn sc@ { uCont ->  
    // compute new context  
    val oldContext = uCont.context  
    // Copy CopyableThreadContextElement if necessary  
    val newContext = oldContext.newCoroutineContext(context)  
    // always check for cancellation of new context  
    newContext.ensureActive()  
    // FAST PATH #1 -- new context is the same as the old one  
    if (newContext === oldContext) {  
        val coroutine = ScopeCoroutine(newContext, uCont)  
        return@sc coroutine.startUndispatchedOrReturn(coroutine, block)  
    }  
    // FAST PATH #2 -- the new dispatcher is the same as the old one (something else changed)  
    // `equals` is used by design (see equals implementation is wrapper context like ExecutorCoroutineDispatcher)    
    if (newContext[ContinuationInterceptor] == oldContext[ContinuationInterceptor]) {  
        val coroutine = UndispatchedCoroutine(newContext, uCont)  
        // There are changes in the context, so this thread needs to be updated  
        withCoroutineContext(newContext, null) {  
            return@sc coroutine.startUndispatchedOrReturn(coroutine, block)  
        }  
    }  
    // SLOW PATH -- use new dispatcher  
    val coroutine = DispatchedCoroutine(newContext, uCont)  
    block.startCoroutineCancellable(coroutine, coroutine)  
    coroutine.getResult()  
}
```

## startUndispatchedOrReturn?
요건 코루틴을 ContinuationInterceptor를 무시하고, 현재 쓰레드에서 즉시 실행하도록 하는 메서드이다.
이 함수의 리턴값은 만약에 중간에 suspend가 일어났으면 `COROUTINE_SUSPEND`를 리턴하고, 그게 아니라면 coroutine의 결과, 혹은 Exception을 던지게 된다. 

일반적인 startCoroutineCancellable과의 차이점은, 코루틴 시작시점에 새로운 ContinuationInterceptor를 무시하고, 기존 ContinuationInterceptor로 동작한다는 점이다. (suspension point에서 resume할 때는 새로운 ContinuationInterceptor를 사용한다)

즉, 새롭게 Dispatcher를 갱신하더라도 기존 쓰레드에서 실행되게 된다.
withContext에서 Dispatcher가 바뀌지 않았을 때 이 함수를 실행하는 이유도, Dispatcher로 인한 Context 스위칭을 최소화하기 위해서 사용된다. 그래서 코드상에 FAST PATH라고 지정해 놓았다.


## ScopeCoroutine리턴 이유
withContext에서 oldContext와 newContext가 같은 객체라면 바로 ScopeCoroutine을 사용하여 코루틴을 시작하고 있다.

내부 코드를 보면 unintercepted continuation객체를 받고 있다.
즉 intercept 되지않은 부모의 순수 continuation으로 로직을 실행하도록 구현되어있으며, 빠르게 이전 continuation 기반으로 로직을 실행시키기 위함이다. (새로운 context를 전환해줘야하는 수고를 덜어준다.)
```kotlin
internal open class ScopeCoroutine<in T>(  
    context: CoroutineContext,  
    @JvmField val uCont: Continuation<T> // unintercepted continuation  
) : AbstractCoroutine<T>(context, true, true), CoroutineStackFrame
```

### ScopeCoroutine은 SupervisorScope안에서 시작될 때 에러전파가 막히지 않는다.
ScopeCoroutine은 `isScopedCoroutine`이 true로 세팅되어있기 때문이다.
`isScopedCoroutine`이 무슨 이유일까? 생각해보자.

supervisorScope에서 동작하는 SupervisorCoroutine을 보면, childCancelled() = false로 되어있다.
```kotlin
private class SupervisorCoroutine<in T>(  
    context: CoroutineContext,  
    uCont: Continuation<T>  
) : ScopeCoroutine<T>(context, uCont) {  
    override fun childCancelled(cause: Throwable): Boolean = false  
}
```
이는 jobSupport의 `cancelParent()` 함수에서 취소(에러) 전파를 부모에게 할것인가를 결정하게 되는데, 
이때 `childCancelled()` 를 활용하게 된다. `cancelParent()`가 true로 리턴하면, 에러 처리권을 부모에게 넘겨주게 되고, false면 부모에게 넘겨주지 않고 자체적으로 처리하게 된다. 

여기서 주목해야할건 `cancelParent()` 의 첫번째 줄에 `isScopeCoroutine`인지 체크하고 true를 얼리리턴하는 케이스가 있다.
```kotlin
private fun cancelParent(cause: Throwable): Boolean {  
    // Is scoped coroutine -- don't propagate, will be rethrown  
    if (isScopedCoroutine) return true
    //...
    return parent.childCancelled(cause) || isCancellation
}
```
여기서 결론적으로 ScopeCoroutine은 `isScopedCoroutine`이 true이기 때문에 에러 처리권을 더 상위 구조로 전달하게 된다.


> withContext에서 실행하는 모든 코루틴은, ScopeCoroutine을 상속받고 있기 때문에,
> supervisorScope 에서 사용하게 될 경우, 에러전파가 상위로 가게 되는 것이다.
> 아래 이슈에 대해서 궁금한점이 풀리던 순간이다.
> https://github.com/easternkite/Coroutines/blob/main/analyze/Scope%2BWithContext.md

### UnDispatchedCoroutine vs DispatchedCoroutine
둘의 차이는 continuation을 사용할 때, intercepted 되기 전의 continuation을 사용하느냐, 그 후의 continuation을 사용하느냐의 차이다.

전자는 UnDispatchedCoroutine으로 앞서 설명했듯, intercepted 되기전의 continuation을 사용한다. 그래서 부모의 dispatcher로 동작하도록 설계되어있다.

후자의 DispatchedCoroutine은 intercepted 된 상태로 동작하기 때문에 새로운 Dispatcher가 반영된 상태로 싫행된다.

이렇게 두개의 루트로 나눠진 이유는 withContext를 사용할 때 ContinuationInterceptor가 같을 경우, DispatchedCoroutine은 오버헤드가 될 수 있다. 그렇기에 대안으로 FAST PATH인 UnDispatchedCoroutine으로 실행하도록 유도하는 것이다.
