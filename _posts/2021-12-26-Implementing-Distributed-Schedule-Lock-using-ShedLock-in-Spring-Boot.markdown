---
layout: post
title: "Spring Boot, ShedLock으로 멀티 노드 환경에서 스케쥴 중복 실행 방지하기"
author: "Taehyeong Lee"
tags: [Spring Boot]
lastmod : 2023-09-26T09:50:00+09:00
---
### 개요
  * **Spring Boot**에서 특정 빈의 메써드에 지정된 `@Scheduled`는 특정 시간 또는 주기로 애플리케이션 로직이 실행될 수 있도록 해준다. **@Scheduled**에서 발생할 수 있는 주요 이슈로, 분산 시스템의 개념이 없기 때문에 애플리케이션을 **n**개의 멀티 노드에 배포하면 같은 스케쥴 작업이 **n**개 노드에서 동시에 실행된다는 단점이 있다. 어떤 상황에서는 특정 스케쥴 작업이 반드시 1개 노드에서만 실행되어야 하는 경우가 있는데, 이번 글에서 소개할 `SchedLock` 라이브러리를 이용하면 특정 작업 실행시 락을 생성하여 다른 인스턴스에서는 해당 작업의 실행을 무시하도록 설정할 수 있다.

### DynamoDB 테이블 생성
  * **ShedLock**은 락 정보를 저장할 테이블로 여러 저장소를 지원한다. 이번 글에서는 **DynamoDB**를 저장소로 지정하여 예제를 작성해보겠다. (**DynamoDB**를 예제에서 선택한 이유는 **AWS** 클라우드 환경에서 셋업이 가장 쉽고 빠르기 때문이다.) **AWS** 콘솔에 접속하여 아래와 같이 테이블을 생성한다. (테이블 이름은 자유롭게 해도 상관없지만, 파티션 키는 반드시 `_id`이어야 한다.)

```bash
DynamoDB 콘솔 접속
→ [테이블 생성]

# 테이블 생성
→ 테이블 이름: shedlock (입력)
→ 파티션 키 이름: _id (입력)
→ 파티션 키 타입: [문자열] 선택
→ 설정: [설정 사용자 지정] 선택
→ 테이블 클래스 선택: [DynamoDB Standard] 선택
→ 용량 모드: [온디맨드] 선택
→ [테이블 생성] 클릭
```

### build.gradle.kts
  * 프로젝트의 **/build.gradle.kts**에 아래 내용을 추가한다.

```kotlin
dependencies {
    implementation("net.javacrumbs.shedlock:shedlock-spring:5.10.2")
    implementation("net.javacrumbs.shedlock:shedlock-provider-dynamodb2:5.10.2")
    implementation("software.amazon.awssdk:dynamodb-enhanced:2.22.12")
}
```

### 환경 변수
  * 프로젝트 또는 운영체 환경 변수에 아래 내용을 추가한다.

```bash
# JDK 21 에서 Virtual Thread를 활성화
SPRING_THREADS_VIRTUAL_ENABLED=true
# ShedLock에서 사용할 테이블 이름을 지정
SHEDLOCK_TABLE=shedlock-dev
```

### @EnableScheduling 빈 작성
  * 스케쥴 작업 실행을 전담할 총 `taskScheduler` 빈을 아래와 같이 생성한다. (**JDK** 버전에 따라 3개의 선택지가 존재한다.)

```kotlin
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import org.springframework.scheduling.TaskScheduler
import org.springframework.scheduling.annotation.EnableScheduling
import org.springframework.scheduling.concurrent.ConcurrentTaskScheduler
import org.springframework.scheduling.concurrent.SimpleAsyncTaskScheduler
import java.util.concurrent.Executors

@Configuration
@EnableScheduling
class SchedulingConfig {

    @Bean
    fun taskScheduler(): TaskScheduler {

        // [옵션 1] JDK 21 에서 Virtual Thread를 활성화
        return SimpleAsyncTaskScheduler().apply {
            this.setVirtualThreads(true)
            this.setTaskTerminationTimeout(30 * 1000)
        }
        
        // [옵션 2] JDK 19/20 에서 Virtual Thread를 활성화
        return ConcurrentTaskScheduler(
            Executors.newScheduledThreadPool(0, Thread.ofVirtual().factory())
        )
        
        // [옵션 3] JDK 17 이하에서 Platform Thread의 개수를 10개로 설정
        return ThreadPoolTaskScheduler().apply {
            this.poolSize = 10
        }
    }
}
```

### @EnableSchedulerLock 빈 작성
  * **ShedLock** 활성화와 함께 데이터 저장소로 **DynamoDB**를 지정할 차례이다. 클래스 작성에 앞서 `dynamoDbClient` 빈의 생성은 본 블로그의 [이 글](https://jsonobject.tistory.com/597)을 따라 진행한다.

```kotlin
import net.javacrumbs.shedlock.core.LockProvider
import net.javacrumbs.shedlock.provider.dynamodb2.DynamoDBLockProvider
import net.javacrumbs.shedlock.spring.annotation.EnableSchedulerLock
import org.springframework.beans.factory.annotation.Qualifier
import org.springframework.beans.factory.annotation.Value
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import software.amazon.awssdk.services.dynamodb.DynamoDbClient

@Configuration
@EnableSchedulerLock
class SchedulerShedLockConfig(
    @Value("\${shedlock.table}") val shedLockTable: String
) {
    @Bean
    fun lockProvider(@Qualifier("dynamoDbClient") dynamoDbClient: DynamoDbClient): LockProvider {

        return DynamoDBLockProvider(dynamoDbClient, shedLockTable)
    }
}
```

### @SchedulerLock 중복 실행 방지 기능 적용

```kotlin
import net.javacrumbs.shedlock.spring.annotation.SchedulerLock
import org.springframework.scheduling.annotation.Scheduled
import org.springframework.stereotype.Component

@Component
class ScheduleService {

    // 매일 KST 04:30:00에 중복 실행 없이 단 1개의 노드에서만 스케쥴 작업을 실행
    @Scheduled(cron = "0 30 4 * * *", zone = "Asia/Seoul")
    @SchedulerLock(name = "fooSchedule", lockAtLeastFor = "30s", lockAtMostFor = "1m")
    fun fooSchedule() {
        // 스케쥴로 실행될 내용 작성
    }
}
```

  * 기본 설정에서 락이 설정된 작업은 실행이 종료되면 즉시 락이 해제된다. 하지만 예외 발생 등의 이유로 작업이 완료되지 못했을 때, `lockAtMostFor`를 설정하여 작업 실행 중 여부와 관계 없이 지정된 시간이 지난 후에는 강제로 락을 해제할 수 있다. 이 값은 반드시 실제 작업의 예상 소요 시간보다 길게 설정해야 한다.
  * 노드마다 시스템 시간의 미세한 차이가 있을 수 있기 때문에 작업이 아주 빨리 끝나는 경우, 락이 설정되었는데도 작업이 중복 실행되는 경우가 있다. 이 경우, `lockAtLeastFor` 를 설정하여 중복 실행을 방지할 수 있다. 작업이 지정된 시간보다 빨리 끝날 경우에도 지정된 시간까지 락을 보장한다. (작업이 지정된 시간을 초과했을 경우에는 이 값이 무시된다.)

### 런타임에서 다이나믹하게 스케쥴러 실행 예약 적용
  * **@Scheduled** 어노테이션을 명시한 스케쥴러 실행은 프로젝트 빌드 전에만 가능하다. 런타임에서도 코드 레벨로 스케쥴러에 특정 작업의 실행을 예약할 수 있다. (이 경우 예약을 실행한 노드에서만 해당 작업이 실행된다.)

```kotlin
import net.javacrumbs.shedlock.spring.annotation.SchedulerLock
import org.springframework.scheduling.annotation.Scheduled
import org.springframework.stereotype.Component

@Service
class FooService(
    private val taskScheduler: TaskScheduler
) {
    ...
    taskScheduler.schedule(
        { // 스케쥴러로 예약 실행할 로직 작성 },
        // 현재 시간에서 10초 뒤에 예약 실행 설정
        Instant.now().plusSeconds(10)
    )
}
```

### 참고 글
  * [GitHub - ShedLock](https://github.com/lukas-krecan/ShedLock)
  * [Lock @Scheduled Tasks With ShedLock and Spring Boot](https://rieckpil.de/lock-scheduled-tasks-with-shedlock-and-spring-boot/)
