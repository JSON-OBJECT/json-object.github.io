---
layout: post
title: "Spring Boot, Slack 채널에 알림 메시지 전송하기"
author: "Taehyeong Lee"
tags: [Spring Boot]
lastmod : 2023-09-26T10:30:00+09:00
---
### 개요
  * **Slack**은 아주 편리한 엔터프라이즈 협업 도구이다. 애플리케이션에서도 **Slack**을 이용하면 장애와 같은 중요한 상황에서 적절한 메시지를 특정 채널에 전송할 수 있다. 이번 글에서는 **Slack** 연동을 통한 메시지 전송 방법을 소개하고자 한다.

### Slack 채널 생성 및 WebHook URL 획득
  * **Slack**에 메시지를 전송하기 위해서는 먼저 채널 생성과 **WebHook URL** 획득이 선행되어야 한다. **WebHook URL** 획득 방법은 아래와 같다.

```bash
https://api.slack.com/apps 접속

# Your Apps
→ [Create an App] 클릭
 
# Create an apps
→ [From scratch] 클릭
 
# Name apps & choose workspace
→ App Name: (채널 이름 입력)
→ Pick an workspace to develop your apps in: (채널이 속한 워크스페이스 선택)
→ [Create App] 클릭
 
# Basic Information
→ [Incoming Webhooks] 클릭
 
# Incoming Webhooks
→ Activate Incoming Webhooks: On (선택)
→ [Add New Webhooks to Workspace] 클릭
 
→ 어디에 게시해야 합니까? (채널 이름 입력)
→ [허용] 클릭
→ 생성된 WebHook URL을 복사
```

### 환경 설정 추가
  * 프로젝트의 **/src/resources/application.yml** 파일에 아래 내용을 추가한다. 앞서 획득한 **WebHookURL**을 연결하는 작업으로 다양한 프로파일에 대한 복수개의 **WebHookURL**을 설정할 수 있는 장점이 있다.

```bash
slack:
  webhook-url:
    info: {url}
    warn: {url}
    error: {url}
```

### build.gradle.kts
  * 프로젝트의 **/build.gradle** 파일에 아래 내용을 추가한다. **OkHttp**을 이용하여 **WebHookURL**을 대상 주소로 메시지를 전송할 것이다.

```bash
dependencies {
    implementation("com.squareup.okhttp3:okhttp:4.11.0")
    implementation("com.fasterxml.jackson.module:jackson-module-kotlin:2.14.3")
}
```

### OkHttpConfig 작성

```kotlin
@Configuration
class OkHttpConfig {

    @Bean("okHttpClient")
    fun okHttpClient(): OkHttpClient {

        return OkHttpClient()
            .newBuilder().apply {
                connectionSpecs(
                    listOf(
                        ConnectionSpec.CLEARTEXT,
                        ConnectionSpec.Builder(ConnectionSpec.MODERN_TLS)
                            .allEnabledTlsVersions()
                            .allEnabledCipherSuites()
                            .build()
                    )
                )
                connectTimeout(10, TimeUnit.SECONDS)
                writeTimeout(10, TimeUnit.SECONDS)
                readTimeout(10, TimeUnit.SECONDS)
            }.build()
    }
}
```

### JsonConfig 작성
  * 전송할 메시지 오브젝트를 **JSON** 요청 바디로 변환하기 위해 **ObjectMapper** 빈을 아래와 같이 설정한다.

```kotlin
@Configuration
class JsonConfig {

    @Bean("objectMapper")
    fun objectMapper(): ObjectMapper {

        return jacksonObjectMapper().apply {
            setSerializationInclusion(JsonInclude.Include.ALWAYS)
            configure(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES, false)
            disable(SerializationFeature.FAIL_ON_EMPTY_BEANS, SerializationFeature.WRITE_DATES_AS_TIMESTAMPS)
            registerModules(JavaTimeModule())
        }
    }
}
```

### AsyncConfig 작성
  * **Slack** 메시지 전송은 단순 **HTTP** 요청이기에 비동기 전송이 적합하다. (만약 동기 방식으로 전송하면 1분에 10만 건의 장애가 발생했을 때 메시지를 전송하는 행위 자체로 애플리케이션을 셧다운시킬 수 있다.) **Java 19**부터 추가된 **Virtual Thread**를 이용하여 아래와 같이 **AsyncTaskExecutor** 빈을 생성한다.

```kotlin
@Configuration
@EnableAsync
class AsyncConfig {

    @Bean(TaskExecutionAutoConfiguration.APPLICATION_TASK_EXECUTOR_BEAN_NAME)
    fun asyncTaskExecutor(): AsyncTaskExecutor {

        val taskExecutor = TaskExecutorAdapter(Executors.newVirtualThreadPerTaskExecutor())

        return taskExecutor
    }
}
```

### Slack 메시지 송신 레벨 정의
  * 다양한 상황에 대한 메시지를 전송하기 위해 메시지 송신 레벨을 아래와 같이 정의한다.

```kotlin
enum class SlackMessageLevel(val colorCode: String) {

    INFO("#2EB67D"),
    WARN("#ECB22E"),
    ERROR("#E01E5A")
}
```

### Slack 메시지 전송 서비스 작성
  * 이제 메시지 전송 서비스를 작성할 차례이다. 먼저 인터페이스이다.

```kotlin
interface SlackService {

    fun sendMessage(fieldMap: Map<String, String>, messageLevel: SlackMessageLevel = SlackMessageLevel.INFO): Future<Boolean>
}
```

  * 다음은 인터페이스에 대한 구현체를 작성할 차례이다.

```kotlin
@Service
@Async("taskExecutor")
class SlackServiceImpl(
    private val okHttpClient: OkHttpClient
    private val objectMapper: ObjectMapper
) : SlackService {

    @Value("\${slack.webhook-url.info}")
    private lateinit var SLACK_WEBHOOK_URL_INFO: String

    @Value("\${slack.webhook-url.warn}")
    private lateinit var SLACK_WEBHOOK_URL_WARN: String

    @Value("\${slack.webhook-url.error}")
    private lateinit var SLACK_WEBHOOK_URL_ERROR: String

    @Async(TaskExecutionAutoConfiguration.APPLICATION_TASK_EXECUTOR_BEAN_NAME)
    override fun sendMessage(fieldMap: Map<String, String?>, messageLevel: SlackMessageLevel): Future<Boolean> {

        val webHookUrl = when (messageLevel) {
            SlackMessageLevel.INFO -> SLACK_WEBHOOK_URL_INFO
            SlackMessageLevel.WARN -> SLACK_WEBHOOK_URL_WARN
            SlackMessageLevel.ERROR -> SLACK_WEBHOOK_URL_ERROR
        }
        
        val requestBody = objectMapper.writeValueAsString(
            mapOf(
                "attachments" to
                        arrayOf(
                            mapOf(
                                "color" to color,
                                "blocks" to arrayOf(
                                    mapOf(
                                        "type" to "section",
                                        "text" to mapOf(
                                            "type" to "mrkdwn",
                                            "text" to StringBuilder().apply {
                                                fieldMap.forEach {
                                                    if (it.value != null) {
                                                        append("*${it.key}*")
                                                        append("\n    ${it.value}\n\n")
                                                    }
                                                }
                                            }.toString()
                                        ),
                                    )
                                )
                            )
                        )
            )
        )

        val httpResponse = try {
            okHttpClient.newCall(
                Request.Builder()
                    .url(webHookUrl)
                    .post(requestBody.toRequestBody("application/json; charset=utf-8".toMediaType()))
                    .build()
            ).execute()
        } catch (ex: Exception) {
            return CompletableFuture.completedFuture(false)
        }

        val statusCode: Int = httpResponse.code
        val responseBody: String? = httpResponse.body?.string()

        return CompletableFuture.completedFuture(statusCode == 200)
    }
}
```

### Slack 메시지 전송 예
  * 이제 **Slack** 채널에 메시지를 전송할 차례이다. 사용 예는 아래와 같다. 전송할 데이터를 **Key-Value** 구조의 **Map<String, String>** 타입의 오브젝트에 담아 알림 레벨과 함께 파라메터로 전달하면 된다.

```kotlin
slackService.sendMessage(
    mapOf(
        "foo" to "bar",
        "alpha" to "bravo"
    ),
    SlackMessageLevel.INFO
)
```

### Spring Boot 애플리케이션 이벤트 연동
  * 위 제작된 코드를 응용하면 아래와 같이 **Spring Boot** 애플리케이션의 시작 시점에도 메시지를 전송할 수 있다.

```kotlin
@Component
class SlackEventListener(
    private val slackService: SlackService
) {
    @EventListener(ApplicationReadyEvent::class)
    fun afterApplicationReady() {

        slackService.sendMessage(
            mapOf(
                "event" to "APPLICATION_STARTED",
                "node_ip_address" to InetAddress.getLocalHost().hostAddress
            ),
            SlackMessageLevel.INFO
        )
    }

    @EventListener(ApplicationFailedEvent::class)
    fun afterApplicationFailed() {

        slackService.sendMessage(
            mapOf(
                "event" to "APPLICATION_STARTUP_FAILED",
                "node_ip_address" to InetAddress.getLocalHost().hostAddress
            ),
            SlackMessageLevel.ERROR
        )
    }
}
```

### 참고 글
  * [Slack: Post message with Incoming-Webhooks in Slack App](https://medium.com/@life-is-short-so-enjoy-it/slack-post-message-with-incoming-webhooks-in-slack-app-521fe28b0db6)
  * [Incoming Webhooks](https://slack.dev/java-slack-sdk/guides/incoming-webhooks)
  * [Slack Color Palette](https://www.onlinepalette.com/slack/)
