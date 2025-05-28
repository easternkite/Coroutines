# Restricted Suspension
`sequence{}` 스코프 내부는 suspend 블럭임에도 `yield()`메서드 외에는 다른 suspend 함수 호출을 제한한다.

`sequence{}`, `yield()`를 구현할 때는 다른 Coroutine Builder를 사용한다. 
아래 `sequence{}` 빌더를 구현한 코드를 보자
```kotlin
fun <T> sequence(block: suspend SequenceScope<T>.() -> Unit): Sequence<T> = Sequence {
    SequenceCoroutine<T>().apply {
        nextStep = block.createCoroutine(receiver = this, completion = this)
    }
}
```

위 코드를 보면, 코루틴을 생성하기 위해 `startCoroutine` 대신, `createCoroutine`을 사용하였다.
두 메서드는 비슷한 역할을 하지만, `createCoroutine`은 코루틴을 시작하지 않고 초기 `Continuation` 참조를 리턴하고 있다는 차이점이 있다.

```kotlin
fun <T> (suspend () -> T).createCoroutine(completion: Continuation<T>): Continuation<Unit>
fun <R, T> (suspend R.() -> T).createCoroutine(receiver: R, completion: Continuation<T>): Continuation<Unit>
```

또 다른 차이점은, 해당 suspend 람다 블럭은, `SequenceScope<T>`를 receiver로 사용하는 extension 람다 블럭이라는 점이다. `SequenceScope` 인터페이스는 아래와 같이 시퀀스를 생성하기 위한 스코프를 제공한다. 
```kotlin
interface SequenceScope<in T> {
    suspend fun yield(value: T)
}
```
불필요한 여러 객체 생성을 막기 위해서, `sequence{}` 에서는 내부적으로 `SquenceScope`, `Continuation<Unit>` 을 구현하는  `SequenceCoroutine<T>`를 정의하였다. 그렇기에 `createCoroutine` 을 위한 receiver 파라미터로도 활용할 수 있고. completion 파라미터로 Continuation을 제공할 수 있다.

> 둘다 따로따로 생성하는 방법보다는, 하나의 객체로 두개의 인터페이스를 구현하는 방식이 더 효율적이라는 말을 하고 있다.

다음은 `SequenceCoroutine<T>`을 간단히 나타낸 코드다.
```kotlin
private class SequenceCoroutine<T>: AbstractIterator<T>(), SequenceScope<T>, Continuation<Unit> {
    lateinit var nextStep: Continuation<Unit>

    // AbstractIterator implementation
    override fun computeNext() { nextStep.resume(Unit) }

    // Completion continuation implementation
    override val context: CoroutineContext get() = EmptyCoroutineContext

    override fun resumeWith(result: Result<Unit>) {
        result.getOrThrow() // bail out on error
        done()
    }

    // Generator implementation
    override suspend fun yield(value: T) {
        setNext(value)
        return suspendCoroutine { cont -> nextStep = cont }
    }
}
```

그러나,  위에서 언급된 `sequence{}` 와 `yield()` 는 임의의 suspend 함수와 함께 사용될 케이스에 대한 제한사항이 있다. Scope는 suspend 람다를 사용하고 있지만, Scope 내부에서 임의의 suspend 함수가 호출한다면 Iterator의 next() 호출에 대한 예측이 불가능해지는 Side-Effect 문제가 있기 때문이다. 해당이유로 Sequence는 동기적으로 작동해야하며, 그렇기에 SequenceScope는 `yield()` 함수를 통해서만 suspension되고 continuation이 캡쳐되도록 **제한**해야한다.

이 문제를 해결하기 위해 클래스나 인터페이스에 `@RestrictsSuspension` 어노테이션을 사용하면된다.
```kotlin
@RestrictsSuspension
interface SequenceScope<in T> {
    suspend fun yield(value: T)
}
```
이 어노테이션은 suspend 함수에 특정 제한사항을 강제하는 옵션입니다. restricted suspension scope에 해당하는 람다 혹은 함수는 `제한된 정지함수(restricted suspending function)`이라 불린다. 제한된 정지함수에서는 해당 스코프(같은 인스턴스여야 함)에 해당하는 멤버만 호출할 수 있다. 

한마디로 `@RestrictsSuspension` 옵션으로 설정된 인터페이스나 클래스의 멤버 외에는 `suspendContinuation` 과 같은 suspend 함수를 호출할 수 없다는 뜻입니다. `SequenceScope` 을 예시로 든다면, `yield()` 만이 함수를 일시정지하고 재개할 수 있는 유일한 메서드이다.

제한된 정지함수 개념은 sequence와 같은 부수효과없이 예측가능한 동작 실행을 기대하기 위해, suspend 함수 사용에 대해 제한을 두기위해 사용된 개념이다. 실제로, sequenceBuilder 에서 delay와 같은 외부 suspend함수를 사용하려고 하면 ide에서 컴파일 에러가 노출된다. 

```kotlin
val seq = sequence<Int> {  
    yield(1)  
    delay(1000) // Compilation Error!
    yield(2)  
}
```

Coroutines와 Compose는 컴파일러 설계 측면에서 비슷한 부분이 많은 듯 싶다.
Coroutines의 suspend-resume, Compose의 Recomposition 둘다 Compiler 수준에서 처리해주는 함수 처리 방식이라는 측면에서, 부수효과의 발생, 그리고 해결방안에 대해서 비슷한 성향을 띤다.
