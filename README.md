> A coroutine — is an instance of suspendable computation

앱 개발시 Coroutines를 중요하게 여기지 않았던 나에게 내리는 벌

### 학습 경로 제한 
올바르고 정확한 지식 습득을 위해, 학습 루트를 제한한다.
1. [공식 문서](https://kotlinlang.org/docs/coroutines-overview.html)
2. [API 문서](https://kotlinlang.org/api/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/launch.html)
3. [Kotlin KEEP 문서](https://github.com/Kotlin/KEEP/blob/master/proposals/coroutines.md)

### 로드 맵
1. 기본 용어들에 대한 각각의 쓰임새와 활용방법 숙지
2. 내부 동작 숙지

### 분석 개념
* [Job](https://github.com/easternkite/Coroutines/blob/main/1.Job/Job.md)
* [CPS + State Machine](https://github.com/easternkite/Coroutines/blob/main/CPS%2BStateMachine.md)
* [Continuation_Interceptor](https://github.com/easternkite/Coroutines/blob/main/Continuation_Interceptor.md) - Dispatcher의 핵심 개념
* [Coroutine Dispatcher](https://github.com/easternkite/Coroutines/blob/main/Dispatcher.md) - Continuation Interceptor 활용 심화편
* [CoroutineContext](https://github.com/easternkite/Coroutines/blob/main/CoroutineContext.md)
* [Non-Blocking Sleep](https://github.com/easternkite/Coroutines/blob/main/Non-Blocking-Sleep.md) - 쓰레드를 Block시키지않고 delay를 구현하는법
* [Restricted Suspension](https://github.com/easternkite/Coroutines/blob/main/Restricted_suspansion.md) - 의도된 suspend 함수만 허용해야할 때
* [CoroutineScope + StructuredConcurrency](https://github.com/easternkite/Coroutines/blob/main/CoroutineScope.md)
* [withContext 내부 구조 이해 및 컨셉](https://github.com/easternkite/Coroutines/blob/main/withContext.md)


## 이슈
* [supervisorScope가 내부 withContext에서 생성한 Job의 예외전파를 막아주지 못하는 현상](https://github.com/easternkite/Coroutines/blob/main/analyze/Scope%2BWithContext.md)
    * https://github.com/Kotlin/kotlinx.coroutines/issues/4385 이슈 등록
