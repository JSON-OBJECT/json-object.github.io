---
layout: post
title: "Spring Boot, Hazelcast를 이용하여 분산 캐시 구현하기"
author: "Taehyeong Lee"
tags: [Server-Sent Event]
---
### 개요

-   **Spring Boot** 프로젝트에서 오픈 소스 분산 캐시 및 인메모리 데이터베이스로 유명한 `Hazelcast`를 **Embedded Cache**로 구현하는 방법을 간단히 정리했다.

### 라이브러리 종속성 추가

-   프로젝트 루트의 `build.gradle.kts`에 **Hazelcast** 사용을 위한 아래 내용을 추가한다.

```gradle
dependencies {
    implementation("com.hazelcast:hazelcast:5.2.3")
    
    // 분산 캐시에 오브젝트 저장시 직렬화 방식으로 Smile을 사용하기 위해 추가
    implementation("com.fasterxml.jackson.dataformat:jackson-dataformat-smile:2.14.3")
}
```

### @Configuration 클래스 작성

-   **Hazelcast** 사용에 필요한 `HazelcastInstance` 빈 생성을 정의하는 **HazelcastConfig** 클래스를 아래와 같이 작성한다.

```kotlin
import com.fasterxml.jackson.annotation.JsonInclude
import com.fasterxml.jackson.databind.DeserializationFeature
import com.fasterxml.jackson.databind.MapperFeature
import com.fasterxml.jackson.databind.ObjectMapper
import com.fasterxml.jackson.databind.SerializationFeature
import com.fasterxml.jackson.dataformat.smile.databind.SmileMapper
import com.fasterxml.jackson.datatype.jsr310.JavaTimeModule
import com.hazelcast.config.Config
import com.hazelcast.core.Hazelcast
import com.hazelcast.core.HazelcastInstance
import com.jocoos.flipflop.api.common.util.CommonUtil
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration

@Configuration
class HazelcastConfig {

    @Bean
    fun hazelcastInstance(): HazelcastInstance {

        return Hazelcast.newHazelcastInstance(
            Config().apply {
                this.clusterName = "foo-dev"
            }
        )
    }
    
    // 분산 캐시에 오브젝트 저장시 직렬화 방식으로 Smile을 사용하기 위해 추가
    @Bean("smileObjectMapper")
    fun smileObjectMapper(): ObjectMapper {

        return SmileMapper().registerModules(JavaTimeModule())
            .configure(SerializationFeature.WRITE_DATES_AS_TIMESTAMPS, true)
            .configure(SerializationFeature.FAIL_ON_EMPTY_BEANS, false)
            .configure(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES, false)
            .setSerializationInclusion(JsonInclude.Include.ALWAYS)
            .disable(MapperFeature.USE_ANNOTATIONS)
    }
}
```

### @Configuration 클래스 작성: Huawei Cloud ECS 환경

-   **Huawei Cloud ECS** 환경은 다른 클라우드와 다르게 **UDP** 멀티캐스트를 이용한 노드 간의 디스커버리를 지원하여 별도의 설정이 필요 없다. 위 설정을 그대로 사용하면 되고 방화벽 포트는 **TCP/5701-5801**, **UDP/54327-54329** 포트 범위를 인바운드 개방하면 바로 작동한다.

### @Configuration 클래스 작성: Amazon ECS 환경

-   **Amazon ECS** 환경에서는 **UDP** 멀티캐스트를 이용한 노드 간의 디스커버리를 지원하지 않기 때문에 별도의 설정이 필요하다. 먼저 해당 **ECS** 태스크 정의에 지정된 **Task Role**과 **Task Execution Role**에 아래 **IAM** 권한이 부여되어 있어야 한다.

```kotlin
ecs:ListTasks
ecs:DescribeTasks
ec2:DescribeNetworkInterfaces
```

-   다음으로 아래와 같이 `HazelcastInstance` 빈을 생성한다.

```kotlin
    @Bean
    fun hazelcastInstance(): HazelcastInstance {

        return Hazelcast.newHazelcastInstance(
            Config().apply {
                this.clusterName = "foo-dev"
                this.networkConfig.join.multicastConfig.isEnabled = false
                this.networkConfig.join.awsConfig.apply {
                    isEnabled = true
                    isUsePublicIp = false
                    setProperty("connection-retries", "2")
                    setProperty("connection-timeout-seconds", "3")                    

                    // 실제 ECS 인스턴스의 서비스명을 기입
                    setProperty("service-name", "foo-dev")
                }
                this.networkConfig.interfaces
                    .setEnabled(true)

                    // 실제 ECS 인스턴스의 VPC CIDR 블록을 기입
                    .addInterface("10.0.*.*")
            }
        )
    }
```

-   마지막으로 방화벽 설정을 추가해야 한다. 따로 설정하지 않을 경우 기본값으로 **TCP/5701-5801** 포트 범위를 인바우드 개방하면 된다.

```bash
# TCP/5701-5801 포트 범위를 인바운드 개방 설정
$ aws ec2 authorize-security-group-ingress \
    --group-id {security-group-id} \
    --ip-permissions '[{"IpProtocol": "tcp", "FromPort": 5701, "ToPort": 5801, "Ipv6Ranges": [{"CidrIpv6": "::/0"}], "IpRanges": [{"CidrIp": "0.0.0.0/0"}]}]'
```

### 분산 캐시 상태 헬스 체크 API 작성

-   현재 분산 캐시 상태를 실시간으로 확인할 수 있는 헬스 체크 **API**를 작성하면 프로덕션 레벨에서의 트러블슈팅에 큰 도움이 된다.

```kotlin
import com.hazelcast.core.HazelcastInstance
import org.springframework.http.ResponseEntity
import org.springframework.web.bind.annotation.*
import java.time.Instant

@RestController
class SystemCacheController(
    private val hazelcastInstance: HazelcastInstance
) {
    @GetMapping("cache-state")
    fun getCacheState(): ResponseEntity<Any> {

        return ResponseEntity.ok(
            mapOf(
                "clusterState" to hazelcastInstance.cluster.clusterState,
                "memberSize" to hazelcastInstance.cluster.members.size,
                "members" to hazelcastInstance.cluster.members.map {
                    mapOf(
                        "host" to it.address.host,
                        "port" to it.address.port,
                        "isIpv4" to it.address.isIPv4,
                        "isIpv6" to it.address.isIPv6,
                        "isLocalMember" to it.localMember(),
                        "isLiteMember" to it.isLiteMember
                    )
                }
            )
        )
    }
}
```

### FooDTO 데이터 클래스 작성

-   분산 캐시 저장 대상으로 예제에 사용할 **FooDTO** 클래스를 아래와 같이 작성한다.

```kotlin
import java.io.Serializable
import java.time.Instant

data class FooDTO(

    var id: Long = 1,
    var name: String? = null,
    var createdAt: Instant = Instant.now()

) : Serializable
```

### FooController 컨트롤러 클래스 작성

-   **FooDTO** 클래스를 분산 캐시에 저장하고 조회할 컨트롤러를 아래와 같이 작성한다.

```kotlin
import com.hazelcast.core.HazelcastInstance
import org.springframework.http.ResponseEntity
import org.springframework.web.bind.annotation.*
import java.util.concurrent.TimeUnit

@RestController
class FooController(
    private val hazelcastInstance: HazelcastInstance,
    private val smileObjectMapper: ObjectMapper
) {
    @GetMapping("/foos/{id}")
    fun getFoo(@PathVariable id: Long): ResponseEntity<Any> {

        val fooMap = hazelcastInstance.getMap<Long, ByteArray>("foo-map")

        fooMap[id]?.let { ResponseEntity.ok(smileObjectMapper.readValue(it, FooDTO::class.java)) } ?: return ResponseEntity.notFound().build()
    }

    @PostMapping("/foos")
    fun setFoo(@RequestBody foo: FooDTO): ResponseEntity<Any> {

        val fooMap = hazelcastInstance.getMap<Long, ByteArray>("foo-map")

        fooMap.set(foo.id, smileObjectMapper.writeValueAsBytes(foo), 2, TimeUnit.HOURS)

        return ResponseEntity.noContent().build()
    }
}
```

### 애플리케이션 실행

-   앞서 작성한 분산 캐시를 테스트하기 위해 2개 인스턴스로 애플리케이션을 실행한다.

```bash
# 1번 인스턴스 실행
$ SERVER_PORT=8080 ./gradlew bootRun

# 2번 인스턴스 실행
$ SERVER_PORT=8081 ./gradlew bootRun
```

### 분산 캐시 작동 테스트

-   1번 인스턴스에서는 캐시를 저장하고, 2번 인스턴스에서는 캐시를 조회하도록 **API**를 호출하여 분산 캐시 작동을 확인할 수 있다.

```bash
# 1번 인스턴스에서 캐시를 저장
$ curl -i -X POST -H "Content-Type:application/json" -d \
'{
  "id": 1,
  "name": "One"
}
' 'http://localhost:8080/foos'

# 2번 인스턴스에서 캐시를 조회
$ curl -i -X GET 'http://localhost:8081/foos/1'
{
    "id": 1,
    "name": "One",
    "createdAt": "2023-04-04T14:34:38.091031743Z"
}
```

### Reliable Topic: 분산 PUB-SUB 메시징 모델

-   `Topic`, `Reliable Topic`은 **Hazelcast**가 제공하는 분산 **PUB-SUB** 메세징 모델이다.
-   **Topic**, **Reliable Topic**의 차이점은 신뢰성에 있다. **PUB** 시점에 **Topic**은 메시지의 수신 여부를 확인하지 않지만(**Fire and Forget**), **Reliable Topic**은 모든 **SUB** 멤버가 메시지를 수신하는 것을 보장한다. 또한, **n**개 노드에서 동시다발적으로 **PUB**이 발생할 경우 **Topic**은 메시지 도착 순서를 발생 순서대로 보장하지 않지만 **Reliable Topic**은 엄격하게 보장한다. (**Reliable Topic**이 내부적으로 `RingBufffer` 자료 구조를 사용하기 때문에 이 것이 가능하며, 그 만큼의 오버헤드 또한 수반되는 것을 감안해야 한다.)
-   내 경우 브라우저로 서비스를 이용 중인 **n**명의 엔드 유저에게 전송되어야 할 **Notification** 정보를 실시간으로 메시지 손실 없이 순서대로 브로드캐스트하고자 할 때 **Reliable Topic** 데이터 구조를 즐겨 사용한다.
-   **Reliable Topic**을 사용하려면 `MessageListener` 인터페이스 구현체를 먼저 작성해야 한다. 아래는 임의의 **FooDTO** 오브젝트를 **PUB-SUB**하는 예이다.

```kotlin
import com.hazelcast.core.HazelcastInstance
import com.hazelcast.topic.Message
import com.hazelcast.topic.MessageListener
import org.springframework.context.annotation.Lazy
import org.springframework.stereotype.Service

@Service
class FooService(
    @Lazy private val hazelcastInstance: HazelcastInstance
) : MessageListener<FooDTO> {

    // [1] 메시지를 동기로 Publish한다.
    fun publishMessage(message: FooDTO) {

        // 멤버가 클러스터 이탈할 상태일 경우 com.hazelcast.core.HazelcastInstanceNotActiveException: Hazelcast instance is not active! 예외 발생
        hazelcastInstance
            .getReliableTopic<FooDTO>("foo-topic")
            .publish(message)
    }

    // [1] 메시지를 비동기로 Publish한다.
    fun publishMessageAsync(message: FooDTO) {

        // 멤버가 클러스터 이탈할 상태일 경우 com.hazelcast.core.HazelcastInstanceNotActiveException: Hazelcast instance is not active! 예외 발생
        hazelcastInstance
            .getReliableTopic<FooDTO>("foo-topic")
            .publishAsync(message)
    }

    // [2] Publish된 메시지를 수신한다.
    // Hazelcast에 의해 관리되는 hz.로 시작하는 쓰레드로 실행됨에 유의한다.
    override fun onMessage(message: Message<FooDTO>) {

        println(message.publishingMember)
        println(message.publishTime)
        println(message.messageObject)
    }
}
```

-   **PUB**된 메시지가 전송되려면 **SUB**된 멤버 노드가 존재해야 한다. **SUB** 시점은 사용 용도에 따라 언제해도 상관 없으나 아래와 같이 **HazelcastInstance** 빈 생성 직후 등록할 수 있다.

```kotlin
@Configuration
class HazelcastConfig(
    private val fooService: FooService
) {
    @Bean
    fun hazelcastInstance(): HazelcastInstance {

        return Hazelcast.newHazelcastInstance(
            Config().apply {
                this.clusterName = "foo-dev"
            }  
        )
        // 클러스터 멤버 등록 즉시 Topic을 Subscribe한다.
        .also { cache ->
            cache
                .getReliableTopic<FooDTO>("foo-topic")
                .addMessageListener(fooService)
        }
    }
}
```

### Graceful Shutdown

-   **Hazelcast**는 데이터를 **JVM** 힙메모리 영역에 저장하기 때문에 애플리케이션의 신규 배포 시점에 종료되는 인스턴스에서의 **Graceful Shutdown** 처리가 매우 중요하다. 대부분의 배포 환경에서는 종료 대상 인스턴스에 대해 **SIGTERM** 신호가 발생하면서 자연스럽게 **Graceful Shutdown** 로직이 실행되고, 삭제될 인스턴스의 파티션이 무중단으로 마이그레이션 처리되고 클러스터 멤버에서 정상적으로 이탈된다.
-   본 예제에서와 같이 **Spring Boot** 기반 프로젝트에서 **Hazelcast**를 **Embedded** 모드로 사용할 경우, **Spring Boot**의 **Graceful Shutdown** 옵션을 활성화하고 **Hazelcast**에서의 해당 옵션은 비활성화하면 **Graceful Shutdown** 설정이 완료된다. 두 옵션을 동시에 활성화하면 **Hazelcast**의 **Graceful Shutdown**이 2번 실행되어 예기치 않은 동작을 야기할 수 있다.

```bash
# [1] 환경 변수로 Spring Boot의 Graceful Shutdown 옵션 활성화
SERVER_SHUTDOWN=graceful
SPRING_LIFECYCLE_TIMEOUT_PER_SHUTDOWN_PHASE=30s

# [2] JVM 파라메터로 Hazelcast의 Graceful Shutdown 옵션 비활성화
# 반드시 SLF4J 로깅 타입 옵션을 활성화해야 Graceful Shutdown 로그가 정상 적재
-Dhazelcast.shutdownhook.enabled=false -Dhazelcast.logging.type=slf4j -Dlogging.level.com.hazelcast=INFO
```

-   **SIGTERM** 신호가 아닌 **SIGKILL** 신호로 애플리케이션을 종료할 경우, **Hazelcast**가 준비할 시간을 주지 않기 때문에 예기치 않은 타임아웃 순단이 발생할 수 있다. 이런 환경이라면 종료 전에 미리 명시적으로 애플리케이션 레벨에서 **Graceful Shutdown**을 실행해야 즉시 안전하게 파티션 마이그레이션을 진행하고 클러스터 멤버에서 이탈할 수 있다.

```kotlin
// 클러스터 멤버의 Graceful Shutdown을 명시적으로 실행
hazelcastInstance.shutdown()
```

-   **Graceful Shutdown**이 실행되면 아래 로그를 콘솔에서 확인할 수 있다.

```bash
[hz.zealous_keldysh.async.thread-2] INFO  c.hazelcast.core.LifecycleService - [192.168.0.2]:5701 [foo-dev] [5.2.3] [192.168.0.2]:5701 is SHUTTING_DOWN
[hz.charming_montalcini.migration] INFO  c.h.i.p.impl.MigrationManager - [192.168.0.2]:5701 [foo-dev] [5.2.3] Repartitioning cluster data. Migration tasks count: 204
[hz.charming_montalcini.migration] INFO  c.h.i.p.impl.MigrationManager - [192.168.0.2]:5701 [foo-dev] [5.2.3] All migration tasks have been completed. (repartitionTime=Fri May 19 01:56:06 UTC 2023, plannedMigrations=204, completedMigrations=204, remainingMigrations=0, totalCompletedMigrations=1425)
[hz.zealous_keldysh.async.thread-2] INFO  com.hazelcast.instance.impl.Node - [192.168.0.2]:5701 [foo-dev] [5.2.3] Shutting down multicast service...
[hz.zealous_keldysh.async.thread-2] INFO  com.hazelcast.instance.impl.Node - [192.168.0.2]:5701 [foo-dev] [5.2.3] Shutting down connection manager...
[hz.zealous_keldysh.async.thread-2] INFO  c.h.instance.impl.NodeExtension - [192.168.0.2]:5701 [foo-dev] [5.2.3] Destroying node NodeExtension.
[hz.zealous_keldysh.async.thread-2] INFO  com.hazelcast.instance.impl.Node - [192.168.0.2]:5701 [foo-dev] [5.2.3] Hazelcast Shutdown is completed in 99 ms.
[hz.zealous_keldysh.async.thread-2] INFO  c.hazelcast.core.LifecycleService - [192.168.0.2]:5701 [foo-dev] [5.2.3] [192.168.0.2]:5701 is SHUTDOWN
```

### Rolling Member Upgrade는 유료 버전에서만 지원

-   **Hazelcast**의 버전을 업그레이드하고 애플리케이션을 실행하면 아래 로그와 같이 기존 구버전으로 실행 중인 클러스터에 조인하지 못해 실행 자체가 실패하게 된다. 운영 환경 도입 이전에 이 부분을 반드시 확인해야 한다. 무료 오픈 소스 버전의 한계인데 엔터프라이즈 버전을 구매하면 이러한 **Rolling Member Upgrade** 기능을 사용할 수 있다.

```bash
[hz.nervous_hodgkin.generic-operation.thread-1] ERROR com.hazelcast.security - [192.168.0.2]:5701 [foo-dev] [5.3.0] Node could not join cluster. Before join check failed node is going to shutdown now!
[hz.nervous_hodgkin.generic-operation.thread-1] ERROR com.hazelcast.security - [192.168.0.2]:5701 [foo-dev] [5.3.0] Reason of failure for node join: Joining node's version 5.3.0 is not compatible with cluster version 5.2 (Rolling Member Upgrades are only supported in Hazelcast Enterprise)
```

### 프로덕션 도입 소감

-   프로덕션 도입 후 2개 멤버로 구성된 클러스터에서 3일간 로그 모니터링 결과, **Map**을 오브젝트 캐시로 사용하여 읽기 성능은 **0ms**가 **53.3%**, **1ms**가 **46.4%**로 측정됬다. (나머지는 0.3%), 쓰기 성능은 **1ms**가 **96.3%**, **2ms**가 **2.29%**로 측정됬다. (나머지는 1.41%) 이 정도면 인메모리 답게 매우 훌륭한 성능이라고 판단된다.

### 참고 글

-   [Hazelcast Overview](https://docs.hazelcast.com/hazelcast/5.2/)
