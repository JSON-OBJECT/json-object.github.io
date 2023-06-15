---
layout: post
title: "Kotlin + Spring Boot, Virtual Thread 적용하기"
author: "Taehyeong Lee"
tags: [Spring Boot, Virtual Thread]
---
### 개요
  * **Java 19**부터 `Virtual Thread` 개념이 **Preview Feature**로 새롭게 추가되었고 **Java 21 LTS**부터 정식 기능으로 전환되었다. 기존의 전통적인 **Platform Thread**가 **OS**의 쓰레드와 직접 맵핑되는 개념이었다면 **Virtual Thread**는 **JVM**에 의해 추상화되어 작동하는 경량의 가상 쓰레드로서 훨씬 낮은 메모리를 소모하는 것이 장점이다. 가상 스레드는 **JVM**의 스케쥴러에 의해 자동으로 관리되므로 개발자는 비지니스 로직에 더욱 집중하면서 성능의 이점까지 누릴 수 있다.
  * **Spring Boot 3**, **Spring Framework 6**는 가상 쓰레드를 공식으로 지원한다. 이번 글에서는 **Spring Boot** 기반 프로젝트에서 **Spring Web MVC** 요청, **@Async**, 코루틴 실행을 처리하는 플랫폼 쓰레드를 가상 쓰레드로 대체하는 방법을 정리했다. (아래 모든 내용은 프로덕션 환경에서 검증을 완료했다.)

### Virtual Thread 특징
  * **Java 19/20**은 **Preview Feature**로 `Virtual Thread`를 제공한다. **LTS**에 해당하는 **Java 21**부터는 정식 기능으로 제공한다.
  * 기존의 **Platform Thread**는 운영체제의 쓰레드를 직접 랩핑한 것이라 네트워크, 디비 실행 등의 이유로 **IO**, 또는 컴퓨팅으로 인한 블록킹이 발생하면 그 시간 만큼 쓰레드를 사용할 수 없었다. **Virtual Thread**는 물리적인 **OS** 쓰레드에서 격리되어 **JVM**에 의해 생명주기가 관리되는 논리적인 단위로 훨씬 가벼우며, 결정적으로 블록킹 발생시 사용 중이던 **OS** 쓰레드를 다른 **Virtual Thread**가 사용할 수 있어 동시성 처리가 비약적으로 향상된다. 이를 통해 개발자는 **Reactive** 방식이 아닌 전통적인 순차적 프로그래밍에서 큰 변화 없이 비약적인 성능 향상이 가능해진다. (**JVM** 진영에서는 수년 만의 큰 혁신에 해당한다.)
  * **Java** 진영은 세심한 노력으로 **Virtual Thread** 도입에 있어 하위 호환성을 최대한 유지했다. `Executors.newVirtualThreadPerTaskExecutor()` 실행 만으로 **Executor** 오브젝트를 쉽게 생성할 수 있다. 이 것을 서블릿 컨테이너, **Kotlin** 코루틴에 연동하여 사용하면 기존 코드의 변화 없이 **Virtual Thread**를 즉시 사용할 수 있다.
  * **Virtual Thread**를 사용하는 방법은 2가지이다. **Java 19/20**은 프로젝트 빌드 시점에 `--release 19 --enable-preview` 옵션을 추가해야 하고, 실행 시점에는 `--enable-preview` 옵션을 추가하면 된다. 반면에 **Java 21**은 해당 옵션을 추가할 필요가 없다.

### OpenJDK 21 설치
  * 개발 환경에 `OpenJDK 21`을 아래와 같이 설치한다. (편의를 위해 `SDKMAN`을 사용했다.)

```bash
# SDKMAN 설치
$ curl -s "https://get.sdkman.io" | bash
$ source "$HOME/.sdkman/bin/sdkman-init.sh"

# Amazon Corretto 21 설치 및 기본 JDK 지정
$ sdk i java 21.0.1-amzn
$ sdk default java 21.0.1-amzn

$ sdk current java
Using java version 21.0.1-amzn

# 설치된 버전 확인
$ java --version
openjdk 21.0.1 2023-10-17 LTS
OpenJDK Runtime Environment Corretto-21.0.1.12.1 (build 21.0.1+12-LTS)
OpenJDK 64-Bit Server VM Corretto-21.0.1.12.1 (build 21.0.1+12-LTS, mixed mode, sharing)
```

### IntelliJ IDEA에서 JDK 21 활성화
  * **IntelliJ IDEA**에서는 아래와 같이 앞서 설치된 **JDK 21**을 프로젝트 레벨에서 활성화한다.

```bash
Settings → Project Structure
→ SDK: [corretto-21] 선택
→ Language Level: [21 (Preview) - String templates, unnamed classes and instance main methods etc.] 선택
```

### build.gradle.kts
  * 프로젝트 루트의 **build.gradle.kts** 에 아래 내용을 추가하여 **JDK 21**을 빌드 레벨에서 활성화한다.

```kotlin
// BEFORE: java.sourceCompatibility = JavaVersion.VERSION_17
java.sourceCompatibility = JavaVersion.VERSION_21

tasks.withType<KotlinCompile> {
    kotlinOptions {
        // BEFORE: freeCompilerArgs = listOf("-Xjsr305=strict")
        // BEFORE: jvmTarget = "17"
        // AFTER: 컴파일 단계에 --release 21 --enable-preview 옵션을 추가
        freeCompilerArgs = listOf("-Xjsr305=strict -Xlint:preview --release 21 --enable-preview")
        jvmTarget = "21"
    }
}

tasks.withType<JavaExec> {
    // AFTER: 런타임 단계에 --enable-preview 옵션을 추가
    jvmArgs = listOf("--enable-preview")
}
```

### HTTP 요청 처리를 Virtual Thread로 전환
  * **Java** 생태계에서 서블릿 컨테이너는 단위 요청마다 **OS**의 쓰레드를 랩핑한 물리적인 플랫폼 쓰레드를 통해 요청을 처리한다. 아래 코드를 통해 **Spring Boot**에 임베디드된 서블릿 구현체인 **Apache Tomcat**에게 모든 요청에 대해 플랫폼 쓰레드가 아닌 가상 쓰레드로 처리하도록 지정할 수 있다.

```kotlin
import org.apache.coyote.ProtocolHandler
import org.springframework.boot.web.embedded.tomcat.TomcatProtocolHandlerCustomizer
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import java.util.concurrent.Executors

@Configuration
class TomcatConfig {

    @Bean
    fun protocolHandlerVirtualThreadExecutorCustomizer(): TomcatProtocolHandlerCustomizer<*>? {
        return TomcatProtocolHandlerCustomizer<ProtocolHandler> { protocolHandler: ProtocolHandler ->
            protocolHandler.executor = Executors.newVirtualThreadPerTaskExecutor()
        }
    }
}
```

### 비동기 실행을 Virtual Thread로 전환
  * **Spring Boot** 생태계에서 비동기 로직을 실행하는 가장 편리한 방법 중 하나인 `@Async`는 전통적인 플랫폼 쓰레드 풀 기반의 **AsyncTaskExecutor** 구현체에 의해서 쓰레드를 할당 받아 실행된다. 이 것을 아래와 같이 가상 쓰레드로 바꿀 수 있다.

```kotlin
import org.slf4j.MDC
import org.springframework.boot.autoconfigure.task.TaskExecutionAutoConfiguration
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import org.springframework.core.task.AsyncTaskExecutor
import org.springframework.core.task.TaskDecorator
import org.springframework.core.task.support.TaskExecutorAdapter
import org.springframework.scheduling.annotation.EnableAsync
import java.util.concurrent.Executors

@Configuration
@EnableAsync
class AsyncConfig {

    @Bean(TaskExecutionAutoConfiguration.APPLICATION_TASK_EXECUTOR_BEAN_NAME)
    fun asyncTaskExecutor(): AsyncTaskExecutor {

        val taskExecutor = TaskExecutorAdapter(Executors.newVirtualThreadPerTaskExecutor())
        taskExecutor.setTaskDecorator(LoggingTaskDecorator())

        return taskExecutor
    }
}

class LoggingTaskDecorator : TaskDecorator {

    override fun decorate(task: Runnable): Runnable {

        val callerThreadContext = MDC.getCopyOfContextMap()

        return Runnable {
            callerThreadContext?.let {
                MDC.setContextMap(it)
            }
            task.run()
        }
    }
}
```

### 스케쥴러 실행을 Virtual Thread로 전환
  * **Spring Boot** 생태계에서 정해진 규칙에 의해 특정 시간에 실행되는 `@Scheduled`는 전통적인 플랫폼 쓰레드 풀 기반의 **ThreadPoolTaskExecutor** 구현체에 의해서 쓰레드를 할당 받아 실행된다. 이 것을 아래와 같이 가상 쓰레드로 바꿀 수 있다.

```kotlin
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import org.springframework.scheduling.TaskScheduler
import org.springframework.scheduling.annotation.EnableScheduling
import org.springframework.scheduling.concurrent.ConcurrentTaskScheduler
import java.util.concurrent.Executors
 
@Configuration
@EnableScheduling
class SchedulingConfig {
 
    @Bean
    fun taskScheduler(): TaskScheduler {

        return ConcurrentTaskScheduler(
            Executors.newScheduledThreadPool(0, Thread.ofVirtual().factory())
        )
    }
}
```

### Kotlin Coroutine 실행을 Virtual Thread로 전환
  * **Kotlin**은 **Virtual Thread**가 없을 때부터 코루틴을 통해 개발자들에게 유사한 **suspend** 기능을 일찌감치 제공해왔다. 아래 코드를 작성 후 기존 코루틴 실행시 사용하던 **Dispatchers.IO** 대신 가상 쓰레드를 적용한 **Dispatchers.LOOM**을 사용하면 코루틴이 가상 쓰레드로 실행된다.

```kotlin
import kotlinx.coroutines.CoroutineDispatcher
import kotlinx.coroutines.Dispatchers
import kotlinx.coroutines.asCoroutineDispatcher
import java.util.concurrent.Executors

val Dispatchers.LOOM: CoroutineDispatcher
    get() = Executors.newVirtualThreadPerTaskExecutor().asCoroutineDispatcher()
```

### Amazon ECS 컨테이너 기동을 위한 Dockerfile 제작
  * **Amazon ECS** 환경에서 **JVM 21**이 적용된 컨테이너를 기동하기 위해 `Dockerfile`을 아래와 같이 작성할 수 있다. (현재 글 작성 시점에 마땅히 사용할만한 **OpenJDK 21** 기반의 도커 베이스 이미지가 존재하지 않아 직접 **OpenJDK 21**을 설치하는 형태로 예제를 작성했다. 현재 프로덕션 레벨에서 직접 검증한 방법이다.)

```bash
# Rate Limit 제한을 예방하기 위해 DockerHub을 사용하지 않고 AWS Public ECR을 이용, 그리고 Amazon ECS와의 호환성이 검증된 베이스 이미지를 사용했다.
FROM public.ecr.aws/ews-network/amazoncorretto:17-debian
ENV SPRING_OUTPUT_ANSI_ENABLED=ALWAYS \
      HTTP_PROXY=http:... \
      HTTPS_PROXY=http:...
EXPOSE 8080
USER root
# OpenJDK 21 설치, 자신의 환경에 맞게 21 기반의 베이스 이미지를 사용하면 된다.
RUN apt update -y
RUN apt install wget gnupg -y
RUN update-ca-certificates
RUN wget https://apt.corretto.aws/corretto.key
RUN apt-key add corretto.key
RUN echo 'deb https://apt.corretto.aws stable main' | tee /etc/apt/sources.list.d/corretto.list
RUN apt-get update -y
RUN apt-get install java-21-amazon-corretto-jdk -y
# 앞서 Gradle 설정을 통해 빌드된 .jar 파일의 경로가 build/libs/app.jar 라고 가정, 자신의 환경에 맞게 수정하면 된다.
COPY build/libs/app.jar /app.jar
COPY buildspec/entrypoint.sh /
ENTRYPOINT ["sh", "/entrypoint.sh"]
```

  * `entrypoint.sh`를 아래와 같이 작성한다. 핵심은 런타임에서 `--enable-preview` 옵션을 명시한 것이다.

```bash
#!/bin/sh
export ECS_INSTANCE_IP_TASK=$(curl --retry 5 -connect-timeout 3 -s ${ECS_CONTAINER_METADATA_URI})
export ECS_INSTANCE_HOSTNAME=$(cat /proc/sys/kernel/hostname)
export ECS_INSTANCE_IP_ADDRESS=$(echo ${ECS_INSTANCE_IP_TASK} | jq -r '.Networks[0] | .IPv4Addresses[0]')
echo "${ECS_INSTANCE_IP_ADDRESS} ${ECS_INSTANCE_HOSTNAME}" | sudo tee -a /etc/hosts
exec java ${JAVA_OPTS} -server -XX:+UseZGC -XX:+ZGenerational --enable-preview -jar /app.jar
```

  * `AWS CodePipeline` 연동을 위해 추가로 `buildspec.xml`을 아래와 같이 작성할 수 있다. (**{region}, {repository-uri}, {image-name}, {container-name}** 파라메터 부분은 적절히 프로젝트 환경에 변경하여 사용하면 된다.)

```bash
version: 0.2

phases:
  install:
    runtime-versions:
      java: corretto21
    run-as: root
    commands:
      - update-ca-trust
      - javac --version
  pre_build:
    commands:
      - REGION={region}
      - REPOSITORY_URI={repository-uri}
      - IMAGE_NAME={image-name}
      - IMAGE_TAG=latest
      - DEPLOY_TAG=dev
      - COMMIT_HASH=$(echo $CODEBUILD_RESOLVED_SOURCE_VERSION | cut -c 1-7)
      - BUILD_TAG=${COMMIT_HASH:=dev}
      - CONTAINER_NAME={container-name}
      - DOCKERFILE_PATH=buildspec/Dockerfile
      - echo Logging in to Amazon ECR...
      - aws --version
      - aws ecr get-login-password --region $REGION | docker login -u AWS --password-stdin $REPOSITORY_URI
  build:
    commands:
      - echo Building the Docker image...
      - chmod +x ./gradlew
      - ./gradlew build -x test
      - docker build -f $DOCKERFILE_PATH -t $IMAGE_NAME .
      - docker tag $IMAGE_NAME:$IMAGE_TAG $REPOSITORY_URI/$IMAGE_NAME:$DEPLOY_TAG
  post_build:
    commands:
      - echo Pushing the Docker images...
      - docker push $REPOSITORY_URI/$IMAGE_NAME:$DEPLOY_TAG
      - printf '[{"name":"%s","imageUri":"%s"}]' $CONTAINER_NAME $REPOSITORY_URI/$IMAGE_NAME:$DEPLOY_TAG > imagedefinitions.json
      - cat imagedefinitions.json

cache:
  paths:
    - '/root/.m2/**/*'
    - '/root/.gradle/caches/**/*'

artifacts:
  files:
    - imagedefinitions.json
```

### 참고 글
  * [Virtual Threads: New Foundations for High-Scale Java Applications](https://www.infoq.com/articles/java-virtual-threads/)
  * [Embracing Virtual Threads](https://spring.io/blog/2022/10/11/embracing-virtual-threads)
  * [Running Kotlin coroutines on Project Loom's virtual threads](https://kt.academy/article/dispatcher-loom)
  * [Boost Your Application’s Performance with Virtual Threads in Java and Spring: Exploring Project Loom](https://medium.com/@knowledge.cafe/spring-boot-virtual-threads-52e28bb0ca5)
