## Coroutine Dispatcher
Dispatcher의 주된 역할은 작업을 특정 쓰레드에서 실행되도록 쓰레드풀로 보내는(dispatcher) 작업을 수행한다. 
이게 가능한 이유는 **Continuation Interceptor**에 있는데, 해당 suspend함수의 continuation을 가로채어, 작업이 일시정지되고 재개(resumeWith)하는 시점에 **특정 쓰레드에서 실행하도록 유도하도록 래핑된 Continuation을 반환**하는 것이다.

> Continuation에 대해 짧게 설명한다면, suspend함수 실행시 암시적으로 전달되는 상태객체이다. 이를 통해 일시 정지시 suspension point를 기억하고, resumeWith함수를 통해 다음 로직을 이어서 수행할 수 있도록 해준다.   
> 이 Continuation을 가로채어 resumeWith 함수를 특정 쓰레드에서 실행되도록 커스텀하는게 Dispatcher의 역할이라 보면 된다.


다음은 CoroutineDispatcher 정의가 되어있는 내부코드이다. CoroutineContext면서 ContinuationInterceptor의 동작을 수행하기 위해서 다음과 같이 상속을 받고 있는 것 을 볼 수 있다.
```kotlin
public abstract class CoroutineDispatcher :  
    AbstractCoroutineContextElement(ContinuationInterceptor), ContinuationInterceptor { ... }
```

Continuation은 `interceptContinuation`을 통해서 가로채어진다. 이때 DispatchedContinuation으로 래핑되어 반환되어진다. 
```kotlin
abstract class CoroutineDispatcher : ContinuationInterceptor {
	public final override fun <T> interceptContinuation(continuation: Continuation<T>): Continuation<T> =  
    DispatchedContinuation(this, continuation)
}
```

`DispatchedContinuation`은 다음과 같이 resumeWith을 오버라이드하면서 parent dispatcher의 `dispatch()` 메서드를 호출하도록 유도하고 있다.
```kotlin
internal class DispatchedContinuation<in T>(...) {
    override fun resumeWith(result: Result<T>) {  
        val context = continuation.context  
        if (dispatcher.isDispatchNeeded(context)) {  
            //... 생략
            dispatcher.dispatch(context, this)  
        } else {  
            //... 생략    
        }  
    }
}
```

이 `dispatch()` 함수가 특정 쓰레드풀로 이동하는 역할을 수행하는데, Dispatcher 종류, 플랫폼에 따라 다르게 구현되어 있다.

iO Dispatcher에서는 탄력적으로 쓰레드풀을 조정하는, UnlimitedIoScheduler를 통해 작업이 전달되는 것을 볼 수 있습니다.
```kotlin
internal object DefaultIoScheduler : ExecutorCoroutineDispatcher(), Executor {
	private val default = UnlimitedIoScheduler.limitedParallelism(  
	    systemProp(  
	        IO_PARALLELISM_PROPERTY_NAME,  
	        64.coerceAtLeast(AVAILABLE_PROCESSORS)  
	    )  
	)

	override fun dispatch(context: CoroutineContext, block: Runnable) {  
	    default.dispatch(context, block)  
	}
}
```

Android의 Main Dispatcher에서는 HandlerContext를 통하여 handler.post를 호출하여 MainLooper에서 Runnable을 처리하도록 유도하고 있습니다.
```kotlin
internal class HandlerContext : HandlerDispatcher {
	override fun dispatch(context: CoroutineContext, block: Runnable) {  
	    if (!handler.post(block)) {  
	        cancelOnRejection(context, block)  
	    }  
	}
}
```

그외 설명은 하지 않았지만, 쓰레드풀을 논리 코어수 만큼 제한하는 DefaultDispatcher가 있다. 해당 Dispatcher는 논리코어를 병렬적으로 활용할 수 있으므로, CPU 리소스를 많이 사용하는 작업을 할때 빠르고 효율적으로 이용할 수 있다.

## 마치며
Dispatcher를 활용하면 특정 로직을 특정 쓰레드에서 실행되도록 유도할 수 있다. 
앞서 언급한 Main, IO, Default 디스패처 외에도 특정 쓰레드에서 활용할 수 있도록 커스텀한 Dispatcher를 구현하거나, Continuation Interceptor를 구현해볼 수 있다.
