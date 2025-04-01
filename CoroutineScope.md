## CoroutineScope

새로운 코루틴을 위한 범위를 지정합니다. 모든 코루틴빌더(launch, async 등)는 CoroutineScope의 확장함수고, coroutineContext를 상속하여 context요소와 취소 이벤트를 전파합니다.

스코프의 독립적인 코루틴을 얻는 가장 최고의 방법은 CoroutineScope()와 MainScope() 팩토리 함수를 사용하는 것입니다. 이를 통해 더 이상 사용하지않는 코루틴에 대해서 취소하도록 해줍니다.

CoroutineContext의 `Plus` 연산자를 통해서 추가적인 요소를 Scope에 추가하는 것도 가능합니다.

## 구조화된 동시성에 대한 규칙
일단 CoroutineScope 인터페이스에 대한 수동 구현 대신, delegation을 통한 구현을 권장합니다. 규칙에 따르면, 범위의 Context는 Job의 인스턴스를 가지고있어야하는데, 그 이유가 Job이 취소전파에 대해서 구조화된 동시성 규율을 이행하도록 하기 위함입니다.

모든 코루틴 빌더(launch, async) 혹은 모든 범위함수(coroutineScope, withContext)는 그들 자체의 Job이 포함되어있는 Scope를 내부 코드블럭으로 제공합니다. (이 문서를 통해 withContext는 단순 context변경이 아닌 자체 CoroutineScope를 제공하는 걸로 확인되었음).

규칙에 의하면, 그 함수들은 block안의 코루틴 작업이 모두 완료되기까지 기다립니다. 이것이 Scope의 등장배경이며, 구조화된 동시성입니다.


## 커스텀 사용 사례
`CoroutineScope`는 마치 entity 위에 프로퍼티처럼 선언해야하고, 자식 코루틴을 실행하기에 있어서 최적의 생명주기를 제공해줄 것입니다. 이와 대응하는 CoroutineScope 인스턴스는 `CoroutineScope()`, `MainScope()`를 통해 손쉽게 생성할 수 있습니다.

* CoroutineScope()는 코루틴을 위한 context를 파라미터로 사용하고, Job을 추가합니다.(만약 context에 등록이 안되어있다면)
* MainScope() 는 내부적으로 Dispatchers.Main을 사용하고, SupervisorJob을 포함합니다.

CoroutineScope를 커스텀하기위해 제일 중요한 부분은 범위가 종료되는 시점에 취소해야하는 것입니다.
`CoroutineScope.cancel` 확장함수는 실행했던 코루틴이 더이상 쓸모가 없을 때 취소하는 목적으로 사용됩니다. 이는 모든 하위 코루틴을 취소하게됩니다.

다음은 CustomScope의 예시입니다. 
```kotlin
class MyUIClass {
    val scope = MainScope() // the scope of MyUIClass, uses Dispatchers.Main

    fun destroy() { // destroys an instance of MyUIClass
        scope.cancel() // cancels all coroutines launched in this scope
        // ... do the rest of cleanup here ...
    }

    /*
     * Note: if this instance is destroyed or any of the launched coroutines
     * in this method throws an exception, then all nested coroutines are cancelled.
     */
        fun showSomeData() = scope.launch { // launched in the main thread
           // ... here we can use suspending functions or coroutine builders with other dispatchers
           draw(data) // draw in the main thread
        }
    }
```

