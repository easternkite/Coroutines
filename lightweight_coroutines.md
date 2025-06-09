## Coroutine은 왜 LightWeight 할까?

### Thread는 OS 자원이 필요하나, Coroutine은 그렇지 않다.
Thread는 운영체제 수준에서 쓰레드 테이블, 스택 메모리(수백 KB ~ 1MB)를 부여받는다.
반면, Coroutine은 스택을 직접 소유하지않아서 메모리상 가벼우며, 스케쥴링도 라이브러리 수준이라 가볍다.

### Context Switching 비용이 적다.
Thread는 커널모드로 진입하여 레지스터 저장/복원, 캐시 무효화 등 무거운 작업이 발생한다.
반면, Coroutines은 단순히 코틀린의 continuation 상태 저장 및 복원만 수행하면 되므로 가볍고 빠르다.

### Coroutines는 필요할 때만 메모리를 점유한다.
동시 실행될 경우에만 Thread를 점유하고, suspend되면 즉시 반납하기 때문에 효율적이다.

