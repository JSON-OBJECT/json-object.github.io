---
layout: post
title: "Server-Sent Event(SSE), JavaScript, Android, iOS에서 클라이언트 연결 및 메시지 수신하기"
author: "Taehyeong Lee"
tags: [Server-Sent Event]
---
### 개요
  * `Server-Sent Event`(**SSE**)의 클라이언트 입장에서 EventSource 연결을 생성하고 이벤트를 수신하는 예제를 대표적인 각 환경에 따라 정리했다.

### curl 명령어로 SSE 연결 및 이벤트 수신
  * 테스트 및 디버깅 목적으로 `curl` 명령어를 이용하여 아래와 같이 **EventSource** 연결 생성 및 이벤트 수신이 가능하다. 명령을 실행하면 종료 전까지 연결이 유지되며 새로운 이벤트가 수신된다.

```bash
curl -N --http2 \
    -H "Accept:text/event-stream" \
    -H "Authorization:Bearer {token}" \
  '{server-url}'
```

### Browser JavaScript에서의 SSE 연결 및 이벤트 수신
  * 브라우저 환경에서는 **HTML5** 표준 **EventSource** 오브젝트틀 생성하여 **SSE** 연결 이벤트 수신이 가능하다. 다만, 표준 API로는 인증 등을 위한 커스텀 요청 헤더를 전송할 수 없는데, 아래와 같이 써드파티 라이브러리를 사용하면 이 것이 가능하다.

```html
// 바닐라 EventSource에서 불가능한 커스텀 헤더 요청을 가능하게 해주는 Yaffle EventSource Polyfill 라이브러리를 사용
<script src="https://raw.githubusercontent.com/Yaffle/EventSource/master/src/eventsource.min.js"></script>
<script>
    // EventSource 오브젝트를 생성
    const eventSource = new EventSourcePolyfill('{server-url}', {
        // 커스텀 요청 헤더를 명시
        headers: {
            'Authorization': 'Bearer {token}'
        },
        // QueryString으로 전달할 Last-Event-ID의 이름을 명시
        lastEventIdQueryParameterName: 'Last-Event-ID',
        // 최대 연결 유지 시간을 ms 단위로 설정, 서버에 설정된 최대 연결 유지 시간보다 길게 설정
        heartbeatTimeout: 600000
    })

    // 특정 Channel의 Message 도착 이벤트를 처리하는 리스너 작성
    eventSource.addEventListener("{channel}", function(event) {
        console.log(event.data)
    })
</script>
```

### Android Kotlin에서의 SSE 연결 및 이벤트 수신
  * 안드로이드 및 **JVM** 환경에서는 아래와 같이 써드파티 라이브러리를 사용하여 구현할 수 있다.
  * 먼저 프로젝트 루트의 **build.gradle.kts** 에 아래 내용을 추가한다.

```kotlin
dependencies {
    // SSE 기반의 서버 푸시 노티로 알려진 실리콘 밸리의 LaunchDarkly가 제작한 OkHttp 기반의 EventSource 라이브러리를 사용
    implementation("com.launchdarkly:okhttp-eventsource:4.1.0")
}
```

  * 특정 **Channel**로의 **Message** 도착 이벤트를 처리하는 리스너를 작성한다.

```kotlin
class SseEventHandler : BackgroundEventHandler {

    override fun onOpen() {
        // SSE 연결 성공시 처리 로직 작성
    }

    override fun onClosed() {
        // SSE 연결 종료시 처리 로직 작성
    }

    override fun onMessage(event: String, messageEvent: MessageEvent) {
        // SSE 이벤트 도착시 처리 로직 작성
        
        // event: String = 이벤트가 속한 채널 또는 토픽 이름
        // messageEvent.lastEventId: String = 도착한 이벤트 ID
        // messageEvent.data: String = 도착한 이벤트 데이터
    }

    override fun onComment(comment: String) {
    }

    override fun onError(t: Throwable) {
        // SSE 연결 전 또는 후 오류 발생시 처리 로직 작성
    	
        // 서버가 2XX 이외의 오류 응답시 com.launchdarkly.eventsource.StreamHttpErrorException: Server returned HTTP error 401 예외가 발생
        // 클라이언트에서 서버의 연결 유지 시간보다 짧게 설정시 error=com.launchdarkly.eventsource.StreamIOException: java.net.SocketTimeoutException: timeout 예외가 발생
        // 서버가 연결 유지 시간 초과로 종료시 error=com.launchdarkly.eventsource.StreamClosedByServerException: Stream closed by server 예외가 발생
    }
}
```

  * 마지막으로 **EventSource** 오브젝트를 생성하는 코드를 아래와 같이 구현한다.

```kotlin
// EventSource 오브젝트 생성
val eventSource: BackgroundEventSource = BackgroundEventSource
    .Builder(
        SseEventHandler(),
        EventSource.Builder(
            ConnectStrategy
                .http(URL("{server-url}"))
                // 커스텀 요청 헤더를 명시
                .header(
                    "Authorization",
                    "Bearer {token}"
                )
                .connectTimeout(3, TimeUnit.SECONDS)
                // 최대 연결 유지 시간을 설정, 서버에 설정된 최대 연결 유지 시간보다 길게 설정
                .readTimeout(600, TimeUnit.SECONDS)
        )
    )
    .threadPriority(Thread.MAX_PRIORITY)
    .build()

// EventSource 연결 시작
eventSource.start()
```

### iOS Swift에서의 SSE 연결 및 이벤트 수신
  * iOS Swift 환경에서는 아래와 같이 써드파티 라이브러리를 사용하여 구현할 수 있다.
  * 먼저 프로젝트 환경에 따라 아래와 같이 라이브러리 종속성을 추가한다. (안드로이드 예제와 동일한 **LaunchDarkly**가 제작한 **EventSource** 라이브러를 사용했다.)

```swift
// CocoaPods 환경에서는 Podfile에 아래 내용을 추가
pod 'LDSwiftEventSource', '~> 3.1'

// Carthage 환경에서는 Cartfile에 아래 내용을 추가
github "LaunchDarkly/swift-eventsource" ~> 3.1

// Swift Package Manager 환경에서는 Package.swift에 아래 내용을 추가
dependencies: [
    .package(url: "https://github.com/LaunchDarkly/swift-eventsource.git", .upToNextMajor(from: "3.1.1"))
]
```

  * 특정 Channel로의 Message 도착 이벤트를 처리하는 리스너를 작성한다.

```swift
class SseEventHandler: EventHandler {

    func onOpened() {
        // SSE 연결 성공시 처리 로직 작성
    }

    func onClosed() {
        // SSE 연결 종료시 처리 로직 작성
    }

    func onMessage(eventType: String, messageEvent: MessageEvent) {
        // SSE 이벤트 도착시 처리 로직 작성
        
        // eventType: String = 이벤트가 속한 채널 또는 토픽 이름
        // messageEvent.lastEventId: String = 도착한 이벤트 ID
        // messageEvent.data: String = 도착한 이벤트 데이터
    }

    func onComment(comment: String) {
    }

    func onError(error: Error) {
        // SSE 연결 전 또는 후 오류 발생시 처리 로직 작성
        
        // error.responseCode: Int = 오류 응답 코드
    }
}
```

  * 마지막으로 EventSource 오브젝트를 생성하는 코드를 아래와 같이 구현한다.

```swift
// EventSource 오브젝트 생성
var config = EventSource.Config(handler: SseEventHandler(), url: URL(string: "{server-url}")!)
// 커스텀 요청 헤더를 명시
config.headers = ["Authorization": "Bearer {token}"]
// 최대 연결 유지 시간을 설정, 서버에 설정된 최대 연결 유지 시간보다 길게 설정
config.idelTimeout = 600.0
let eventSource = EventSource(config: config)

// EventSource 연결 시작
eventSource.start()
```

### 참고 글
  * [GitHub - EventSource Polyfill](https://github.com/Yaffle/EventSource)
  * [GitHub - okhttp-eventsource](https://github.com/launchdarkly/okhttp-eventsource)
  * [GitHub - swift-eventsource](https://github.com/launchdarkly/swift-eventsource)
