## CPS(Continuation-Passing-Style)
코루틴의 특징은 suspend 키워드를 통해 특정 함수에서의 실행을 일시 정지 하고, 특정 작업 이후, 일시정지했던 시점으로부터 작업을 재개할 수 있는 것이다.

이 suspend 함수는 CPS(Continuation-Passing-Style)기법을 통해 구현된다. 모든 suspend 함수와 suspend 람다는 추가적인 Continuation 파라미터가 주어지는데, 이는 suspend 함수가 불려질때 암시적으로 전달된다.

suspend 함수를 선언하면 내부적으로 Continuation을 구현하는 익명객체가 생성되고, 해당 객체에서 내부적으로 label이라는 state를 두어 suspension point를 기억하고 재개할 수 있는것이다.

await() 함수를 살펴보면 다음과 같다.
```kotlin
suspend fun <T> CompletableFuture<T>.await(): T
```
그러나, CPS 기법이 적용된 실제 구현체에서는 suspend 키워드 대신, 다음과 같이 continuation이 전달된다.
```kotlin
fun <T> CompletableFuture<T>.await(continuation: Continuation<T>): Any?
```

기존의 반환 타입이었던 T는 continuation의 type argument 자리로 이동되었다. 
CPS로 구현된 함수에서 `Any?` 타입은  suspend 함수에 대한 동작을 표현하기위해 설계되었다. suspend 함수가 코루틴을 정지시킬 때, `COROUTINE_SUSPENDED`라는 특별한 marker value를 반환한다. 이 값은 아래에 State Machine할 때 다뤄볼 것이다.

코루틴의 실행은 계속되나 현재 코루틴을 정지하지 않는 경우에는 다이렉트로 Result를 반환함과 동시에 Exception을 던진다. 

이 방법으로 해당 구현된 함수에서의 `Any?` 반환 타입은 사실상 `COROUTINE_SUSPENDED`와 T 타입에 대한 모음이라고 봐도 무방하다.

앞에서 설명했던 바와 같이, 실제 suspend 함수의 구현은 continuation을 스택 프레임에서 직접 호출하도록 허락하지 않았다. 왜냐하면 이는 오래 점유하는 Coroutines에 대해서 Stack-Overflow의 위험성이 있기 때문이다. 대신 `suspendCoroutine` 함수를 통해 suspend 기능을 구현할 수 있는다. 이는 개발자로부터 continuation 호출들을 직접 관리해야하는 복잡도를 줄여줄 수 있고, continuation이 언제 실행되는지와는 상관없이 의도된 규약을 따르도록 보장할 수 있다.
### State Machine (상태 기계)
코루틴을 효율적으로 구현하는 것은 상당히 중요하며, 클래스나 객체생성을 최소화해야한다. 그래서 많은 언어들이 Coroutine을 설계할 때 State Machine을 사용하였고, Kotlin에서도 동일한 방법을 사용하였다. 그 결과로 컴파일러는 하나의 suspend 함수당 suspend 지점에 대한 랜덤한 숫자를 가진 하나의 클래스만 생성할 수 있도록 최적화할 수 있다.

메인 아이디어 : suspend 함수는 State Machine으로 컴파일 된다. 그 state는 각각의 suspend 지점에 대한 정보를 담고 있다. 아래 코드를 보자 :

```kotlin
val a = a()
val y = foo(a).await() // suspension point #1
b()
val z = bar(a, y).await() // suspension point #2
c(z)
```

위의 함수에는 3가지 상태가 존재한다.
1. initial (suspend 지점 이전)
2. 첫 번째 suspend 지점 이후
3. 두 번째 suspend 지점 이후

각각의 상태는 continuation에서의 시작 지점(entryPoint)이 되는 것이다.
저 코드는 State Machine을 구현하는 익명 클래스로 컴파일 되며, 현재의 상태를 기억하고 있고 로컬 변수를 통해 각각의 state간에 상태를 공유할 수 있다.(Closure를 통해 기억하고 활용하도록 되어있다.)

아래 코드는 실제 await() 함수의 내부동작을 CPS + 상태 머신을 통하여 구현해본 의사코드이다.
```kotlin
class <anonymous_for_state_machine> extends SuspendLambda<...> {
    // The current state of the state machine
    int label = 0
    
    // local variables of the coroutine
    A a = null
    Y y = null
    
    void resumeWith(Object result) {
        if (label == 0) goto L0
        if (label == 1) goto L1
        if (label == 2) goto L2
        else throw IllegalStateException()
        
      L0:
        // result is expected to be `null` at this invocation
        a = a()
        label = 1
        result = foo(a).await(this) // 'this' is passed as a continuation 
        if (result == COROUTINE_SUSPENDED) return // return if await had suspended execution
      L1:
        // external code has resumed this coroutine passing the result of .await() 
        y = (Y) result
        b()
        label = 2
        result = bar(a, y).await(this) // 'this' is passed as a continuation
        if (result == COROUTINE_SUSPENDED) return // return if await had suspended execution
      L2:
        // external code has resumed this coroutine passing the result of .await()
        Z z = (Z) result
        c(z)
        label = -1 // No more steps are allowed
        return
    }          
}    
```

bytecode에서 어떤 일이 일어나는지 나타내기 위해 goto문을 사용함.

Coroutine이 시작된 시점에는 label은 초기값인 0이 되고, resumeWith 함수에 따라서 L0로직으로 향하게된다. label은 다음 State값인 1을 기리킵니다. 그리고 await함수를 호출하게 되고, 해당 함수는 suspend 함수이기 때문에 result에서 `COROUTINE_SUSPENDED`를 리턴하여 함수를 일시 정지한다. (suspend 함수의 원리)

이때, label은 자신의 suspended지점을 기억하고 있기에, 특정 작업 이후 해당 로직을 다시 태우기 위해서는 resumeWith을 다시 호출하게 되는데, 이때 L1부터 이어서 호출한다.

반복문을 사용하여 suspend 함수를 반복호출하게 될 경우, state는 하나만 사용한다.
```kotlin
var x = 0
while (x < 10) {
	x += nextNumber().await()
}
```

이 함수는 내부적으로 다음과 같이 형성됩니다.
```java
class <anonymous_for_state_machine> extends SuspendLambda<...> {
    // The current state of the state machine
    int label = 0
    
    // local variables of the coroutine
    int x
    
    void resumeWith(Object result) {
        if (label == 0) goto L0
        if (label == 1) goto L1
        else throw IllegalStateException()
        
      L0:
        x = 0
      LOOP:
        if (x >= 10) goto END
        label = 1
        result = nextNumber().await(this) // 'this' is passed as a continuation 
        if (result == COROUTINE_SUSPENDED) return // return if await had suspended execution
      L1:
        // external code has resumed this coroutine passing the result of .await()
        x += ((Integer) result).intValue()
        label = -1
        goto LOOP
      END:
        label = -1 // No more steps are allowed
        return 
    }          
}
```

