`Job`은 백그라운드 작업될 수 있는 코루틴의 실행단위이다. `Job` 은 생명주기에 따라 취소 또는 완료에 이를 수 있다.

Job은 부모-자식간의 계층구조로 이루어질 수 있으며, 부모 Job이 취소되면 하위에 소속된 모든 자식 Job 또한 재귀적으로 즉시 취소된다. 

자식 Job에서 발생한 Exception은 부모 Job에도 영향을 끼친다. (CancellationException은 예외다. 밑에서 설명함)
```kotlin
val parentJob: Job = launch {  
    val child1 = launch {  
        delay(1000)  
        println("Child 1 completed")  
    }  
    val child2 = launch {  
        delay(500)  
        throw CancellationException("child 2 failed")  
    }  
  
    child1.join()  
}  
parentJob.join()
```

> SupervisorJob을 사용하면 예외전파를 자식으로 한정시킬 수 있다.

### 기본적인 Job 구현체들 

* `Coroutine job` : launch 코루틴 빌더를 를 통하여  생성되는 Job. launch블록 안에 있는 로직을 실행하고, 해당 로직이 완료되면 Complete상태에 도달한다.
* `CompletableJob` : 이건 팩토리 메서드인 `Job()`을 통하여 생성할 수 있는 Job이다.이것은 `CompletableJob.complete`를 통해서 완료된다.

일반적으로 Job은 특정 연산의 결과를 리턴하지 않는다. 각각의 Job을 사용하는 목적은 부수효과(Side-Effect)실행 목적이다. (연산 결과를 리턴받고자 하면, Deferred 인터페이스를 참조하면 된다.)

### Job의 생명주기
다음은 각각의 생명주기에 따른 상태 flag값의 변화를 표로 나타낸 것이다.

| **State**                        | [isActive](https://kotlinlang.org/api/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-job/is-active.html) | [isCompleted](https://kotlinlang.org/api/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-job/is-completed.html) | [isCancelled](https://kotlinlang.org/api/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-job/is-cancelled.html) |
| -------------------------------- | ------------------------------------------------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------ |
| _New_ (optional initial state)   | `false`                                                                                                                  | `false`                                                                                                                        | `false`                                                                                                                        |
| _Active_ (default initial state) | `true`                                                                                                                   | `false`                                                                                                                        | `false`                                                                                                                        |
| _Completing_ (transient state)   | `true`                                                                                                                   | `false`                                                                                                                        | `false`                                                                                                                        |
| _Cancelling_ (transient state)   | `false`                                                                                                                  | `false`                                                                                                                        | `true`                                                                                                                         |
| _Cancelled_ (final state)        | `false`                                                                                                                  | `true`                                                                                                                         | `true`                                                                                                                         |
| _Completed_ (final state)        | `false`                                                                                                                  | `true`                                                                                                                         | `false`                                                                                                                        |
일반적으로 Job의 생성은 active 상태에서 진행된다. 그러나 예외적으로 코루틴 시작시점을 Lazy(CoroutineStart.LAZY)하게 설정했을 경우,  `Job.start()`, 혹은  `Job.join()` 메서드를 실행할 때 비로소 active상태가 된다.

active 상태는 코루틴이 동작하고 있거나, CompleteJob이 완료 및 취소될 때 까지 지속된다. 

Exception을 통해 Job을 취소할 경우, Cancelling 상태에 진입하게 된다. 이 상태에서 모든 자식 job들이 취소될 때까지 기다리는 단계며, 모든 자식job이 취소되었다면 canceled 단계가 된다. `cancel()` 메서드를 통해 즉시 cancelling 상태로 즉시 진행시킬 수 있다.

Completing 상태는 모든 자식 Job이 성공할 때까지 기다리며, 모든 자식 Job이 성공하면 Completed상태가 된다.

![[Pasted image 20250224003237.png]]


> `cancel()` 메서드는 CancellationException을 발생시키며, 예외적으로 부모코루틴을 종료시키지 않는다.
> CancellationException은 코루틴의 정상적인 종료를 유도하는데 사용되는 특별한 예외이다.
> CancellationException이 예외인 이유는 Job의 생명주기에 해당 Exception으로 취소여부를 알아야하기 때문이다


## SupervisorJob
예외 전파를 자식으로 한정하는 Job. 
Job은 부모 + 자식 계층 구조의 상태를 가지고 있는 코루틴의 실행 단위인 만큼, 예외 발생시 상하로 전파하도록 되어있다. 따라서 일반적인 CompleteJob 사용시 발생하는 Exception은 부모로의 예외전파까지 포함하므로, 예기치않은 앱종료, Exception이 발생할 수 있다.

이를 방지하기 위해, 예외전파를 자식으로만 한정짓는 방법을 선택할 수 있는데, 그게 SupervisorJob이다.

SupervisorJob은 아래와 같이, childCancelled를 false로 리턴하면서, 부모를 향한 Exception 전파를 막는다.
```kotlin
private class SupervisorJobImpl(parent: Job?) : JobImpl(parent) {  
    override fun childCancelled(cause: Throwable): Boolean = false  
}
```

childCancelled 메서드는 아래와 같이 부모 job을 취소하고자 할 때 사용하는 메서드다.
해당 메서드를 false로 고정 리턴하기 때문에, 부모로의 예외전파는 일어나지않는다.

```kotlin
/**  
 * Child is cancelling its parent by invoking this method. * This method is invoked by the child twice. The first time child report its root cause as soon as possible, * so that all its siblings and the parent can start cancelling their work asap. The second time * child invokes this method when it had aggregated and determined its final cancellation cause. * * @suppress **This is unstable API and it is subject to change.**  
 */@InternalCoroutinesApi  
public fun childCancelled(cause: Throwable): Boolean
```
