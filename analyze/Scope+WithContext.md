# withContext는 별도의 Coroutine을 생성한다?

<img width="381" alt="image" src="https://github.com/user-attachments/assets/5f4d78fe-b9e9-4f8c-bc38-48cd8ef8b9c9" />


withContext는 내부 suspend block안의 로직을 특정 Context 환경에서 실행하도록 구성해주는 메서드이다.  
단순히 이 기능만 수행한다면, 내부에서 생성된 Job의 부모는 상위에서 생성한 CoroutineScope가 되어야할 것이다.

그러나 withContext는 내부적으로 **Coroutine을 자체 생성**하는 케이스가 있다는 사실을 알았고, 내부에서 생성된 Job의 에러전파가 상위 CoroutineScope에 전달되지 않던 이슈를 발견했다. 

아래 코드를 보자. 이슈를 재현하기 위해 supervisorScope 내부에 withContext 를 활용하여 코드를 구성하였다. 예상대로라면, Job B에서 Exception이 발생하였어도, 부모인 supervisorScope가 상위로의 에러전파를 막아주기 때문에 Job A가 정상 실행되어야할 것이다.

> **supervisorScope**는 에러전파를 자식코루틴으로 한정시키기 위해 사용된다. 
> 일반적으로 코루틴에서의 예외전파는 **부모 -> 자식**, **자식 -> 부모**로 전파되는 **양방향 전파**가 이루어지는데, 예외 전파를 자식으로만 한정하여, 자식 Job의 죽음으로 인한 의도치않은 앱의 크래시를 예방할 수 있다.

```kotlin
import kotlinx.coroutines.*

fun main() {
    runBlocking {
        println("start")
        supervisorScope {
            withContext(this.coroutineContext) {
                launch(CoroutineName("Job A")) {
                    delay(200L)
                    println("Job A completed.")
                }
                launch(CoroutineName("Job B")) {
                    delay(100L)
                    throw Exception("something went wrong")
                }
            }
        }
        println("end")
    }
}
```

위 코드의 예상 구조도는 다음과 같다. B에서 에러가 발생해도, 부모인 supervisorScope가 에러전파를 막아주기 때문에 Job A 뿐 만 아니라, runBlocking을 포함한 모든 프로세스가 정상적으로 마무리된다.

![image](https://github.com/user-attachments/assets/47a6f861-4bb4-4646-bb0d-727289956572)

 그러나 위 코드의 결과값은 다음과 같다.
```
start
Exception in thread "main" ...
```

Job A 마저 취소되었고, `supervisorScope` 외부에 있는 `println("end")` 로직까지 실행되지않았다.


## 왜 그랬을까?
우선 결론부터 말하자면, withContext에서 생성된 Job들이 supervisorScope소속일 것이라는 추측이 틀렸다.

withContext 내부에서는 어떤 스코프를 사용하는지 다음과 같이 로그를 찍어보았다.
```kotlin
fun main() {
    runBlocking {
        supervisorScope {
            println("super : ${this@supervisorScope}")
            withContext(this.coroutineContext) {
                println("child : ${this@withContext}")
                //...
            }
        }
    }
}
```

당연히 같은 스코프를 리턴할 줄 알았지만, 결과는 다음과 같았다. 
```
super : "coroutine#1":SupervisorCoroutine{Active}@75412c2f 
child : "coroutine#1":UndispatchedCoroutine{Active}@2fc14f68 <-- 누구세요?
```

supervisorScope가 부모로서 자식의 예외 전파를 막아주는 줄 알았지만, **남의 자식**이었던 것이다.
그림으로 표현해보면 다음과 같이 동작했었다.

![image](https://github.com/user-attachments/assets/6430aa5b-d41d-4b44-b04f-baa7659c7447)

#### 수정 (03/12)
위 구조도도 사실상 틀린 내용이었다.  
그러나 논리적으로는 이 구조도대로 구조가 이루어질 것 같았지만, 사실은 withContext는 supervisorScope의 자식이 맞는걸로 확인되었다.  
GDG Korea Android 오픈채팅방의 **주간개발자님**께서 해당 근거를 코드로 전달주셨다. (감사합니다.)  
<img width="360" alt="image" src="https://github.com/user-attachments/assets/5433f2be-9496-4cb7-b23c-535acb1a664d" />  

그 근거는 아래 사진에서 확인할 수 있었다.  
<img width="700" alt="image" src="https://github.com/user-attachments/assets/03b37ab2-df3b-4129-962f-2ad589a092fb" />  

이 근거에 기반하여 구조도를 수정하면 다음과 같다.  
![image](https://github.com/user-attachments/assets/a81c8010-40bf-43fc-a666-d84f49ef84b2)

### 의문점
* 그렇다면 withContext 내부에서 발생한 에러는 supervisorJob을 건너뛰고 전파되는건지 모르겠다.

## withContext 내부 구조
withContext가 어떻게 UndispatchedCoroutine을 생성하게 되었는지 알기위해선 withContext의 내부 동작을 알아야한다. 다음은 withContext의 내부 코드를 나타낸다. 내부 코드에서는 크게 `FAST PATH #1`, `FAST PATH #2`, `SLOW PATH`의 세가지 케이스로 나누어 처리하고 있다.

```kotlin
public suspend fun <T> withContext(
    context: CoroutineContext,
    block: suspend CoroutineScope.() -> T
): T {
    contract {
        callsInPlace(block, InvocationKind.EXACTLY_ONCE)
    }
    return suspendCoroutineUninterceptedOrReturn sc@ { uCont ->
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
            withCoroutineContext(coroutine.context, null) {
                return@sc coroutine.startUndispatchedOrReturn(coroutine, block)
            }
        }
        // SLOW PATH -- use new dispatcher
        val coroutine = DispatchedCoroutine(newContext, uCont)
        block.startCoroutineCancellable(coroutine, coroutine)
        coroutine.getResult()
    }
}
```

withContext는 `suspendCoroutineUninterceptedOrReturn` 를 통해 함수를 일시정지 하고 재개될 때 intercepted되지 않은 순수한 continuation을 람다를 통해 전달한다. 왜냐하면, 외부에서 Dispatcher등 통해 내부적으로 Continuation을 가로채어 커스텀하는 케이스를 방지하기 위함이다.

이때, 전달되는 Continuation의 context와, 인자로 전달받은 context를 기반으로 새로 생성된 Context에 대해 동일성, 동등성 비교를 실시한다.

결과적으로 세가지 케이스 모두, 새로운 코루틴을 생성하여 시작한다. 

* Path 1 : 동일성 비교, 만약 두 객체가 동일(같은 주소)할 시 SocopeCoroutine 생성 
* Path 2 : 동등성 비교, 두 객체가 바라보는 Dispatcher 가 같다면 UndispatchedCoroutine 생성
* Slow Path : 그 외 두 객체가 논리적으로 다를 경우 DispatchedCoroutine 생성
	* 이 곳을 통해 Dispatcher가 전환된다.


실제 위 코드에서는 어떤 Path를 타는지 확인하기 위해 커스텀 메서드를 작성하였다.
```kotlin
@OptIn(InternalCoroutinesApi::class)
suspend fun test(context: CoroutineContext) {
    return suspendCoroutineUninterceptedOrReturn { uCont ->
        println("cont = $uCont")

        val old = uCont.context
        val new = old.newCoroutineContext(addedContext = context)
        new.ensureActive()
        println("PATH1 --> old == new : ${new[ContinuationInterceptor] == old[ContinuationInterceptor]}")
        println("PATH2 --> old === new : ${old === new}")
    }
}

fun main() {
    runBlocking {
        supervisorScope {
            test(coroutineContext)
        }
    }
}
```

이때, 결과적으로 다음과 같은 결과를 얻을 수 있었다.
```
PATH1 --> old === new : false 
PATH2 --> old == new : true
```

supervisorScope의 context를 그대로 넘겨도 두 콘텍스트는 동일하지 않다고 출력되었다.
`newCoroutineContext` 통해 기존 Context에 새로운 Context를 추가하여 새로운 객체를 생성하기에 당연한 결과라고 생각한다. 그렇다면, PATH1을 타는 로직는 제거해도 되지않는가 하는 의문점은 있다.

다만 PATH2 비교로직인 두 ContextDispatcher가 동일한지에 대한 비교에서 true가 나왔다. 같은 콘텍스트를 넘겼기에 어찌보면 당연하다. 결과적으로 이곳을 타게되어 UndispatchedCoroutine으로 생성하게 되었다.

만약 인자로 Dispatcher와 같은 추가적인 인자를 넣었다면, Slow Path를 탔었을 것이다.


## 마치며 
코루틴을 적용하면서, withContext는 내부적으로 새로운 코루틴을 생성한다는 점에서 의도와 다르게 동작한다는 것을 알았다. withContext를 사용함으로서 부모 코루틴은 외부에서 감싸고 있던 supervisorJob이 아닌, UnDispatchedCoroutine이 되어버리면서 내부에러가 runBlocking까지 흘러가게 되었던 것이다.
이를 통해 withContext를 사용하면 exception이 의도치않게 전파되지 않는다라는 것을 알았고, 이것을 안티패턴으로 치부할 것인지, 의도된 동작으로 치부할 것인지는 아직 의문으로 남아있다. 코루틴에서 안정적인 exception 처리를 위한다면 ExceptionHandler와 같은 별도의 처리방법을 사용해야할 것 같다.
