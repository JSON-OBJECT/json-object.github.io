---
layout: post
title: "Kotlin, 코루틴 사용법 정리"
author: "Taehyeong Lee"
tags: [Kotlin, Spring Boot]
---
### 개요

-   백엔드 엔지니어로서 프로덕션 레벨에서 쓰레드 풀을 이용한 비동기 및 병렬 처리에 충분히 만족하고 있었기 때문에, 코루틴을 따로 공부하지 않고 않았는데 코루틴을 이용하려 병목 현상을 효과적으로 해소하는 지인을 보고 코루틴 공부를 시작하게 되었다. 코루틴에 대한 첫 인상은 완전히 새로 발명된 비장의 무기라는 느낌보다는 쓰레드 사용을 정말 쉽게 해준다는 것이었다. 의식적으로 콜백 지옥에서 해방된 느낌마저 받았다. 이번 글에서는 **Kotlin/JVM**을 이용한 서버 사이드 관점에서 **Kotlin**의 코루틴의 기본적인 사용법을 정리하였고 지속적으로 업데이트할 예정이다.

### build.gradle.kts

-   프로젝트 루트의 **build.gradle.kts**에 아래 내용을 추가한다.

```javascript
dependencies {
    implementation("org.jetbrains.kotlinx:kotlinx-coroutines-core:1.6.0")
    runtimeOnly("org.jetbrains.kotlinx:kotlinx-coroutines-core-jvm:1.6.0")
    implementation("org.jetbrains.kotlinx:kotlinx-coroutines-slf4j:1.6.0")
}
```

### runBlocking

-   코루틴의 시작은 `runBlocking` 블록의 작성이다. 현재 쓰레드에서 코루틴 블록을 작성하기 위한 첫 관문 역할을 한다.

```kotlin
fun main() {

    // [1] 현재 쓰레드에서 코루틴을 실행할 수 있는 상태로 전환
    runBlocking {

        // 블록 내에서는 여전히 현재 쓰레드 사용
        // [2] 0개 이상의 코루틴 블록 작성
    }

    // [3] 앞의 runBlocking의 코루틴을 포함한 블록 내 모든 로직이 종료된 후 현재 쓰레드 재개
}
```

-   또는, 특정 함수의 블록으로 바로 사용할 수도 있다.

```kotlin
fun doSomeCoroutine() = runBlocking {

    // 0개 이상의 코루틴 블록 작성
}
```

-   **runBlocking** 블록에서도 현재 쓰레드는 그대로 진행된다. 알고 있어야할 점은 **runBlocking** 블록 내 어떠한 코루틴 실행이 병렬이나 비동기로 실행되더라도 해당 블록들이 완전히 종료되어야 **runBlocking** 블록이 종료된다는 것이다.
-   위 예제만 봐서는 **runBlocking** 블록이 큰 의미가 없어보이지만 이 블록 안에서만 비로소 `coroutineScope`, `launch`, `async`와 같은 코루틴 블록을 작성할 수 있다. 아래 이어서 설명한다.

### launch

```kotlin
runBlocking {

    // [1] 현재 쓰레드를 그대로 사용하는 코루틴 블록 작성
    launch {
        // 실행 로직 작성
    }

    // [2] IO 요청에 최적화된 새로운 쓰레드를 사용하는 코루틴 블록 작성
    launch(Dispatchers.IO) {
        // DefaultDispatcher-worker-{thread-number} 이름의 쓰레드로 실행
        // 실행 로직 작성
    }

    // [3] CPU 연산에 최적화된 새로운 쓰레드를 사용하는 코루틴 블록 작성
    launch(Dispatchers.Default) {
        // DefaultDispatcher-worker-{thread-number} 이름의 쓰레드로 실행
        // 실행 로직 작성
    }

    // [4] 응용하면 특정 로직을 서로 다른 쓰레드에서 병렬 실행이 가능
    (1..1000).forEach {
        launch(Dispatchers.IO) {
            // DefaultDispatcher-worker-{thread-number} 이름의 쓰레드로 실행
            doSomething(it)
        }
    }

    // [5] 각 launch 블록의 실행 순서는 랜덤하게 결정
}
```

### suspend

-   `suspend`는 코루틴을 빛나게 하는 진정한 주인공이자 이 글을 쓰게 된 이유라고 할 수 있다. 일반 함수의 앞에 `suspend` 키워드만 추가하면 현재 쓰레드를 쉬지 않고 실행시킬 수 있게 된다. 개발자가 특별히 신경쓰지 않아도, 실제 실행은 시스템 상황에 따라 여러 쓰레드가 처리하게 되는 획기적인 방식이다.

```kotlin
// fun 앞에 suspend 키워드를 추가
suspend fun doSomething(number: Int): String {

    // [1] 로직 1
    // [2] 100ms 걸리는 IO 로직 2, 이 시점에 놀게 되는 현재 쓰레드를 다른 suspend 함수가 가져갈 수 있다.
    // [3] 로직 3, 이 것을 실행하는 쓰레드는 다른 쓰레드가 될 확률이 높다.

    return "something"
}

// 호출한 쓰레드의 MDC 필드를 보존하려면 withContext(MDCContext()) 추가
suspend fun doSomething(number: Int): String = withContext(MDCContext()) {
    ...
    return@withContext "foobar"
}
```

-   확실히 알아야할 것은 **suspend**가 해당 함수의 실행 속도를 빠르게 해주지는 않는다. 함수 자체의 실행은 여느 일반 함수와 다를 것 없다. 하지만 **suspend**의 진정한 강점은 해당 함수가 원격지의 데이터베이스 조회와 같은 쓰레드를 블록킹시키는 로직을 실행할 때 있다. 이 경우, 일반 함수는 현재 쓰레드가 블록킹되어 로직이 종료될 때까지 기다리게 된다. 결국, 일시적으로 시스템에 놀고 있는 쓰레드가 생겨버리는 것이다. 반면에 **suspend** 함수는 쓰레드가 놀게 되는 즉시, **Dispatcher**에 의해 쓰레드가 다른 일에 투입된다. 즉, 해당 함수의 실행 속도에는 변함이 없지만, 같은 시간 실행되어야할 다른 중요한 함수에게 쓰레드를 빌려줄 수 있는 것이다. 결론적으로 **suspend** 함수를 많이 사용할수록 시스템 전체의 퍼포먼스를 비약적으로 향상시킬 수 있게 된다.
-   **suspend**가 의도한대로 여러 쓰레드에 의해 실행되려면 아래와 같이 상위 코루틴 블록에 **Dispatcher**가 선언되어야 한다.

```kotlin
// 현재 쓰레드만 doSomething()을 위해 일한다.
runBlocking {
    doSomething(100)
}

// 여러 IO 쓰레드가 doSomething()를 위해 일한다.
runBlocking(Dispatchers.IO) {
    doSomething(100)
}

// 여러 Default 쓰레드가 doSomething()를 위해 일한다.
runBlocking(Dispatchers.Default) {
    doSomething(100)
}
```

### Spring MVC에서 suspend 함수 호출

-   코루틴을 사용하기 가장 좋은 환경은 **Spring WebFlux** 환경이다. **Spring MVC**라고 불가능한 것은 아니며, 아래와 같이 **suspend** 함수를 호출할 수 있다.

```kotlin
import kotlinx.coroutines.Dispatchers
import kotlinx.coroutines.async
import kotlinx.coroutines.runBlocking
import kotlinx.coroutines.slf4j.MDCContext
import org.springframework.http.ResponseEntity
import org.springframework.web.bind.annotation.GetMapping
import org.springframework.web.bind.annotation.RestController

@RestController
class FooController(
    private val fooService: FooService
) {
    @GetMapping("/foos")
    fun getFoo(): ResponseEntity<Foo> {

        return runBlocking(Dispatchers.IO + MDCContext()) {
            // @Service 스프링 빈이자 suspend 함수인 FooService#getFoo() 호출
            val result = async { fooService.getFoo({id}) }
            ResponseEntity.ok(result.await())
        }
    }
}
```

### @Scheduled에서 suspend 함수 호출

-   아래는 **@Scheduled** 로직에서 **suspend** 함수를 호출하는 예이다.

```kotlin
import kotlinx.coroutines.Dispatchers
import kotlinx.coroutines.async
import kotlinx.coroutines.runBlocking
import kotlinx.coroutines.slf4j.MDCContext
import net.javacrumbs.shedlock.spring.annotation.SchedulerLock
import org.springframework.context.annotation.Profile
import org.springframework.scheduling.annotation.Scheduled
import org.springframework.stereotype.Component

@Profile("batch")
@Component
class BatchScheduledService(
    private val fooService: FooService
) {
    @Scheduled(cron = "0 0 * * * *")
    @SchedulerLock(name = "batchUpdateFoos", lockAtLeastFor = "30s", lockAtMostFor = "1h")
    fun batchUpdateFoos() {

        runBlocking(Dispatchers.IO + MDCContext()) {

            val result = async { fooService.getFoo({id}) }
            result.await()
        }
    }
}
```

### 참고 글

-   [GeeksforGeeks - Dispatchers in Kotlin Coroutines](https://www.geeksforgeeks.org/dispatchers-in-kotlin-coroutines/)
-   [GeeksforGeeks - runBlocking in Kotlin Coroutines with Example](https://www.geeksforgeeks.org/runblocking-in-kotlin-coroutines-with-example)
-   [GeeksforGeeks - Launch vs Async in Kotlin Coroutines](https://www.geeksforgeeks.org/launch-vs-async-in-kotlin-coroutines/)
-   [GeeksforGeeks - Suspend Function In Kotlin Coroutines](https://www.geeksforgeeks.org/suspend-function-in-kotlin-coroutines/)
-   [StackOverflow - Is it better to use a thread or coroutine in Kotlin?](https://stackoverflow.com/a/58255417/17742933)
-   [StackOverflow - What does the suspend function mean in a Kotlin Coroutine?](https://stackoverflow.com/a/54561432/17742933)
-   [StackOverflow - How important is it to specify dispatchers/context in Kotlin coroutines? What happens if you don’t specify them?](https://stackoverflow.com/a/69853768/17742933)
-   [StackOverflow - Kotlin Coroutines vs CompletableFuture](https://stackoverflow.com/a/58477365/17742933)
-   [Andrey Minogin - Kotlin coroutines real life example](https://medium.com/@minogin/kotlin-coroutines-real-life-example-d940cc170217)
-   [Kotlin coroutines vs RxJava: an initial performance test](https://proandroiddev.com/kotlin-coroutines-vs-rxjava-an-initial-performance-test-68160cfc6723)
-   [Custom Kotlin Coroutine Context Uses Cases](https://codingwithmohit.com/coroutines/custom-coroutine-context-uses-cases/)