# 7. Non-Blocking Sleep

코루틴에서 현재 쓰레드를 Blocking하는 Thread.sleep()를 사용하면 안됩니다. 대신, Java의 ScheduledThreadpoolExecutor를 사용하여 직관적이고 Non-Blocking한 `delay` 를 대신 사용할 수 있습니다.
```kotlin
private val executor = Executors.newSingleThreadScheduledExecutor {
	Thread(it, "scheduler").apply { isDaemon = true }
}

suspend fun delay(time: Long, unit: TimeUnit = TimeUnit.MILLISECONDS): Unit = suspendCoroutine { cont --> 
	executor.schedule({ cont.resume(Unit), time, unit })
}
```

해당 delay함수를 통해 별도의 scheduler 쓰레드에서 코루틴을 resume하도록 수현할 수 있습니다.  
대신 사용하는 Context에 따라서, 해당 쓰레드에서만 머무르지 않는 케이스도 존재합니다. Swing과 같은 interceptor를 사용하는 코루틴은, 인터셉터를 통해 적절한 쓰레드 풀을 찾아서 작업을 전달해주기 때문에 하나의 쓰레드에서만 머무르는 것을 기대할 수 없습니다.

그렇기 때문에 위에서 보여준건 편의를 위한 목적으로 보여준 것이지 모든 환경에서 효율적인 방법은 아닙니다. interceptor환경에서도 적절한 non-blocking sleep을 네이티브로 구현하려면, SwingTimer를 사용한다면 다음과 같이 표현할 수 있습니다.

```kotlin
suspend fun Swing.delay(mills: Int) = suspendCoroutine { cont ->
	Timer(mills) { cont.resume(Unit) }.apply {
		isRepeats = false
		start()
	}
}
```
> 실제로 delay함수는 여러 interceptor를 고려하여, 자동으로 적절한 메서드로 리턴해줍니다.

한마디로 정리하자면 Coroutines는 각각의 사용하는 Context, 플랫폼에 맞는 Delay함수가 직접 구현되어있습니다. 위에서 보여줬던 예시에 추가로, Android는 HandlerContext에 Delay를 상속하여 `handler.poseDelayed`를 호출하도록 구현하고 있습니다.

코루틴의 내부코드를 보면 효율적인 Non-Blocking한 Delay를 위해 고민한 흔적이 보여집니다.
