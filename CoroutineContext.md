# Coroutine Context
Coroutine Context는 특정 코루틴에 적용할 수 있는 일종의 지속적인 사용자 정의 환경이다.
Context에는 코루틴 쓰레딩 정책, 로깅, 보안 및 실행 관련 트랜잭션, 코루틴 이름 등을 포함하고 있다. 

코루틴은 경량화된 쓰레드라는 특성으로 비추어보았을 때, Coroutine Context는 일종의 Thread-Local 변수들의 집합이라 봐도 무방하다. 실제 Thread 로컬 변수와의 차이점은 mutability에 있다.Coroutine Context는 변경이 불가능하지만, 일반적은 쓰레드 로컬변수는 변경가능하다는 점이다. 

이는 사실상 코루틴 측면에서 크리티컬하지는 않다. 왜냐하면 코루틴은 경량화된 쓰레드라는 측면에서 특정 코루틴의 context를 직접 수정하는 것보다 새로운 컨텍스트 씌워서 별도의 코루틴으로 실행하는것이 더 심플하면서 쓰레드보다 성능적으로 좋기 때문이다.

standard library에는 기본적으로 context 항목에 대한 구체적인 구현이 없다. 그렇지만 interface와 추상 클래스로 정의되어있으며 이를 활용해 더 유연하고 확장성있는 방법으로 정의할 수 있도록 하였다. 그래서 다른 라이브러리에서 구현된 Context일지라도 안정적으로 함께 사용할 수 있도록 설계하였다.

그렇기에 코루틴은 의도적으로 유니크한 키를 index 갖는 Set 형태의 Element로 구성되도록 설계하였다. 자료구조에서 Value값만을 활용하지만 각각의 유니크한 인덱스가 있다는 점에서 Set과 Map 개념의 혼합이라고 보면 된다. 각각의 유니크한 키로 접근할 수 있는 부분은 Map과 같지만, 해당 키는 Value와 직접적으로 연관되어있는 요소기 때문에 Set의 특성도 가지고 있는 것이다.

```kotlin
interface CoroutineContext {
    operator fun <E : Element> get(key: Key<E>): E?
    fun <R> fold(initial: R, operation: (R, Element) -> R): R
    operator fun plus(context: CoroutineContext): CoroutineContext
    fun minusKey(key: Key<*>): CoroutineContext

    interface Element : CoroutineContext {
        val key: Key<*>
    }

    interface Key<E : Element>
}
```

위 코드에서 보면, CoroutineContext는 4가지의 연산이 가능하도록 설계되어있다.
* `get` : Key에 맞는 Element를 리턴합니다. Type Safe한 방식으로 조회할 수 있습니다.
* `fold` : 일반적인 `Collection.fold` 와 같은 연산을 하며, Element를 순회하면서 필요한 연산을 할 수 있도록 설계되어있다.
* `plus` : 두개의 context를 하나로 Set 모음으로 합치기 위해서 사용된다. 같은 계열의 Context는 오른편에 있는 Context가 왼편에있는 Context를 대체한다.
* `minusKey` : 특정 Key의 Element를 삭제하고, 해당 Element가 삭제된 상태의 Context를 반환한다.

하나의 Element는 Coroutine Context 그자체이다. 이는 하나의 Element만 가지고 있는 싱글톤 Context인 것이다. 이를 통해 +(plus) 연산을 통해 다른 Context를 조합할 수 있는 것이다. 예를들어서특정 라이브러리에서 사용자 인증 정보를 위해서 `auth` 라는 element를 정의하고, 또다른 라이브러리에서는 `threadPool` 오브젝트를 실행 콘텍스트 정보로서 정의를하였다면, launch 빌더를 통해서 손쉽게 두가지 Context를 합칠 수 있는 것이다. 
```kotlin
launch(auth + threadPool) {...}
```

> kotlinx.coroutines 패키지에는 특정 Context Element를 제공하고 있다. 예를 들면 특정 작업이 백그라운드 쓰레드의 공유 쓰레드풀 위에서 동작하도록 실행을 전달하는 `Dispatchers.Default` 가 있다.

Standard library에서는 어떠한 Element를 포함하고있지 않은 EmptyCoroutineContext를 제공하고있다.

`AbstractCoroutineContextElement` 를 사용하면 손쉽게 Element를 정의할 수 있다. 이는 Element를 상속 받는 추상클래스이기 때문에, Key값만 생성자로 넘겨주면 Context에 대한 구현은 자동으로 해준다. 다음과 같이 사용자의 이름을 저장하는 커스텀 Context를 정의할 수 있다.
```kotlin
class AuthUser(val name: String) : AbstractCoroutineContextElement(AuthUser) {
	companion object Key : CoroutineContext.Key<AuthUser>
}
```

특정키로 유연하게 Access를 하기위해 Companion Object로 Key값을 정의하였으며, 이는 다음과 같이 손쉽게 Context를 조회하고, 예외처리를 할 수 있다.

```kotlin
suspend fun doSomething() {
	val currentUserName = coroutineContext[AuthUser]?.name ?: throw SecurityException("unauthorized")
}
```

coroutineContext는 현재 코루틴에서 사용중인 Context에 접근할 수 있는 property이다.
