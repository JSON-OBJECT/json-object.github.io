---
layout: post
title: "Spring Boot, Redis를 이용하여 API 중복 실행 요청 방지 로직 구현하기"
author: "Taehyeong Lee"
tags: [Spring Boot, Redis]
---
### 개요

-   **API**를 운영하다보면 가장 흔하게 발생하는 이슈가 아주 짧은 찰나에, 동일한 **API** 요청이 거의 동시에 들어오는 것이다. 가장 일반적인 원인은 대개 엔드 유저가 브라우저 상에서 특정 버튼을 아주 빠르게 연속으로 클릭하는 것이고, 크리티컬하게는 특정 엔티티의 상태 변화를 유발하는 아주 미세한 차이의 **Race Condition**이 발생하는 경우도 있다. **API**는 이런 상황에 대비하여 동일 요청에 대해 중복 실행을 방지하는 로직으로 대응할 필요가 있다. 이번 글에서는 **Spring Boot** 프로젝트에서 **Redis**를 이용한 중복 실행 방지 로직을 구현하고 사용하는 예를 설명하고자 한다.

### 운영체제 환경 변수 추가

-   **Redis** 연결을 위한 환경 변수를 아래와 같이 추가한다. (상황에 맞게 **application.yaml** 파일에 추가해도 무방하다. 예제를 실행할 **Redis** 인스턴스가 사전에 셋업되었다고 가정한다.)

```javascript
SPRING_REDIS_HOST={redis-host}
SPRING_REDIS_PORT={redis-port}
SPRING_REDIS_MODE=STANDALONE
```

### 라이브러리 종속성 추가

-   프로젝트 루트의 `build.gradle.kts`에 **Redis** 사용을 위한 아래 내용을 추가한다.

```javascript
dependencies {
    implementation("org.springframework.data:spring-data-redis:3.0.4")
    implementation("io.lettuce:lettuce-core:6.2.3.RELEASE")
}
```

### @Configuration 클래스 작성

-   **Redis** 사용에 필요한 `StringRedisTemplate` 빈 생성에 필요한 `RedisConfig` 클래스를 아래와 같이 작성한다.

```kotlin
import io.lettuce.core.ClientOptions
import io.lettuce.core.SocketOptions
import org.springframework.beans.factory.annotation.Qualifier
import org.springframework.beans.factory.annotation.Value
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import org.springframework.data.redis.connection.RedisClusterConfiguration
import org.springframework.data.redis.connection.RedisStandaloneConfiguration
import org.springframework.data.redis.connection.lettuce.LettuceClientConfiguration
import org.springframework.data.redis.connection.lettuce.LettuceConnectionFactory
import org.springframework.data.redis.core.RedisTemplate
import org.springframework.data.redis.core.StringRedisTemplate
import org.springframework.data.redis.serializer.StringRedisSerializer
import java.time.Duration

@Configuration
class RedisConfig(

    @Value("\${spring.redis.host}")
    private val REDIS_HOST: String,

    @Value("\${spring.redis.port}")
    private val REDIS_PORT: Int,

    @Value("\${spring.redis.mode}")
    private val REDIS_MODE: String
) {
    @Bean("lettuceConnectionFactory")
    fun lettuceConnectionFactory(): LettuceConnectionFactory {

        if (REDIS_MODE == "STANDALONE") {
            return LettuceConnectionFactory(RedisStandaloneConfiguration(REDIS_HOST, REDIS_PORT))
        }

        val clusterConfiguration = RedisClusterConfiguration().apply {
            clusterNode(REDIS_HOST, REDIS_PORT)
        }

        val clientConfiguration = LettuceClientConfiguration.builder()
            .clientOptions(
                ClientOptions.builder()
                    .socketOptions(
                        SocketOptions.builder()
                            .connectTimeout(Duration.ofSeconds(10)).build()
                    )
                    .build()
            )
            .commandTimeout(Duration.ofSeconds(10)).build()

        return LettuceConnectionFactory(clusterConfiguration, clientConfiguration)
    }

    @Bean("stringRedisTemplate")
    fun stringRedisTemplate(
        @Qualifier("lettuceConnectionFactory") lettuceConnectionFactory: LettuceConnectionFactory
    ): StringRedisTemplate {

        return StringRedisTemplate(lettuceConnectionFactory)
    }
}
```

### RequestLockType Enum 클래스 작성

-   **Redis**에 저장할 락의 종류를 아래와 같이 작성한다. (개인적으로 **API** 요청 단위로 락을 식별하는 것을 선호한다.) `lockDuration` 필드에는 락이 걸릴 시간을 명시한다.

```kotlin
import java.time.Duration

enum class RequestLockType(val lockDuration: Duration) {

    CREATE_FOO(Duration.ofSeconds(10)),
    UPDATE_FOO(Duration.ofSeconds(10)),
    DELETE_FOO(Duration.ofSeconds(10))
}
```

### RequestLockService 클래스 작성

-   아래는 실제 **Redis**를 통해 락을 생성하고 해제할 수 있는 **@Service** 클래스를 작성한 것이다.

```kotlin
import org.springframework.data.redis.core.StringRedisTemplate
import org.springframework.stereotype.Service
import java.time.Duration

@Service
class RequestLockService(
    private val stringRedisTemplate: StringRedisTemplate
) {
    fun generateLockKey(requestLockType: RequestLockType, vararg params: Any?): String {

        return "REQUEST_LOCKS/${requestLockType.name}/${params.joinToString("_#_")}"
    }

    fun ifLockedThrowExceptionElseLock(lockKey: String, lockDuration: Duration = Duration.ofMinutes(1)) {

        try {
            stringRedisTemplate
                .opsForValue()
                .setIfAbsent(lockKey, "locked", lockDuration)
                // 중복 실행 요청이 들어왔을 경우 예외 발생 로직 작성
                // 각 애플리케이션 상황에 특화된 부분이므로 상황에 맞게 작성
                ?.also { if (!it) throw CustomException(CustomErrorCode.REQUEST_LOCKED) }

        } catch (ex: Exception) {
            if (ex is CustomException) throw ex
            // Redis 오류 발생시 예외 처리 로직 작성
        }
    }

    fun unlock(key: String?) {

        key ?: return

        try {
            stringRedisTemplate.delete(key)
        } catch (ex: Exception) {
            // Redis 오류 발생시 예외 처리 로직 작성
        }
    }
}
```

### Lock 생성 및 해제 예

-   아래는 앞서 작성한 **RequestLockType**, **RequestLockService**을 이용하여 락을 생성하고 해제하는 예이다.

```kotlin
// 임의에 Foo 오브젝트 업데이트 실행에 대해 Lock을 생성
// 이미 Lock이 생성되었을 경우, REQUEST_LOCKED 예외 발생
val lockKey = requestLockService
    .generateLockKey(RequestLockType.UPDATE_FOO, {foo.id})
    .also { requestLockService.ifLockedThrowExceptionElseLock(it, RequestLockType.UPDATE_FOO.lockDuration) }

try {
    // Lock 생성 대상이 되는 로직 작성
    fooService.update({foo})
}
finally {
    // 실행 종료되면 Lock 해제
    requestLockService.unlock(lockKey)
}
```

-   가장 먼저 고유의 **lockKey** 문자열을 생성한다. 각 **Lock**을 식별할 파라메터는 **Any?** 타입을 **vararg** 형태로 받기 때문에 개별 요청을 유니크하게 식별할 수 있는 **toString()**이 구현된 어떤 오브젝트도 자유롭게 전달이 가능하다. (예제에서는 단순하게 임의의 **foo** 오브젝트의 **id**만 전달했다.)
-   **Lock**이 유지되는 기간은 **RequestLockType**에 사전 정의된 **lockDuration**을 전달하거나, 임의의 **Duration** 타입 오브젝트로 지정이 가능하다. (예제에서는 **Lock** 유지 기간을 10초로 지정했다.)
-   **Lock**을 생성하는 시점에 동일한 **Lock**이 이미 생성되었으면 대상 로직을 실행하지 않고, 사전 정의된 **REQUEST\_LOCKED** 예외를 발생시킨다.
-   **Lock** 생성 대상 로직의 실행이 정상적으로 종료되거나, 실행 중 예외가 발생되는 2가지 경우 모두에 대해 즉시 **Lock**을 해제한다.
