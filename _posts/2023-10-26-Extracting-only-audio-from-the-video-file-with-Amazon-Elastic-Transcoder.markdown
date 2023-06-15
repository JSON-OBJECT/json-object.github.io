---
layout: post
title: "Kotlin, Amazon Elastic Transcoder로 원본 동영상에서 오디오만 추출하기"
author: "Taehyeong Lee"
tags: [Spring Boot]
lastmod : 2023-10-26T10:30:00+09:00
---
### 개요
  * 대용량의 원본 비디오 파일을 다양한 네트워크 대역폭과 디바이스에 대응하기 위해 다양한 프리셋으로 변환하는 요건은 리얼 월드에서 빈번하게 발생한다. 이 경우 오픈 소스에서 보편적으로 사용되는 툴은 `FFmpeg`, `HandBrake CLI`가 대표적이다. 이번 글에서는 오픈 소스를 사용하지 않고 AWS의 트랜스코더 상품인 `Amazon Elastic Transcoder`를 사용하여 원본 비디오 파일에서 오디오 파일을 추출하는 방법을 정리했다. (프리셋만 변경하면 오디오 뿐만 아니라 다양한 형식의 비디오로 변환할 수 있다.)

### build.gradle.kts
  * 프로젝트의 **/build.gradle.kts** 파일에 아래 내용을 추가한다.

```bash
dependencies {    
    implementation("software.amazon.awssdk:elastictranscoder:2.21.37")
}
```

### ElasticTranscoderClient 생성
  * 트랜스코더 사용을 위한 첫 단계로 **ElasticTranscoderClient** 오브젝트를 아래와 같이 생성한다.

```kotlin
val transcoderClient = ElasticTranscoderClient
    .builder()
    // 대한민국에서 가장 가까운 도쿄 리전으로 설정
    .region(Region.AP_NORTHEAST_1)
    .build()
```

  * **Amazon Elastic Transcoder**는 서울 리전을 제공하지 않는다. 그래서 대한민국에서 가장 가까운 도쿄 리전을 사용했다.

### 시스템 프리셋 목록 조회
  * 트랜스코딩을 하려면 트랜스코딩을 어떻게 할지를 정의한 **Preset ID**를 지정해야 한다. 트랜스코딩을 위한 커스텀 프리셋을 직접 생성할 수도 있고, 사전에 제공되는 시스템 프리셋을 사용할 수도 있다.

```kotlin
// 현재 사용 가능한 프리셋 목록을 조회
transcoderClient.listPresets().presets().forEach {
    // Id=1351620000001-100141, Arn=arn:aws:elastictranscoder:ap-northeast-1:776875668468:preset/1351620000001-100141, Name=System preset: Audio AAC - 64k, Description=System preset: Audio AAC - 64 kilobits/second, Container=mp4, Audio=AudioParameters(Codec=AAC, SampleRate=44100, BitRate=64, Channels=1, CodecOptions=AudioCodecOptions(Profile=auto)), Type=System
    println("$it")
}
```

  * 본 예제에서는 **AAC**로 트랜스코딩할 것이라 **AAC** 관련 시스템 프리셋 목록을 추리면 아래와 같다.

```kotlin
Id=1351620000001-100110, Arn=arn:aws:elastictranscoder:ap-northeast-1:776875668468:preset/1351620000001-100110, Name=System preset: Audio AAC - 256k, Description=System preset: Audio AAC - 256 kilobits/second, Container=mp4, Audio=AudioParameters(Codec=AAC, SampleRate=44100, BitRate=256, Channels=2, CodecOptions=AudioCodecOptions(Profile=AAC-LC)), Type=System
Id=1351620000001-100120, Arn=arn:aws:elastictranscoder:ap-northeast-1:776875668468:preset/1351620000001-100120, Name=System preset: Audio AAC - 160k, Description=System preset: Audio AAC - 160 kilobits/second, Container=mp4, Audio=AudioParameters(Codec=AAC, SampleRate=44100, BitRate=160, Channels=2, CodecOptions=AudioCodecOptions(Profile=AAC-LC)), Type=System
Id=1351620000001-100130, Arn=arn:aws:elastictranscoder:ap-northeast-1:776875668468:preset/1351620000001-100130, Name=System preset: Audio AAC - 128k, Description=System preset: Audio AAC - 128 kilobits/second, Container=mp4, Audio=AudioParameters(Codec=AAC, SampleRate=44100, BitRate=128, Channels=2, CodecOptions=AudioCodecOptions(Profile=AAC-LC)), Type=System
Id=1351620000001-100141, Arn=arn:aws:elastictranscoder:ap-northeast-1:776875668468:preset/1351620000001-100141, Name=System preset: Audio AAC - 64k, Description=System preset: Audio AAC - 64 kilobits/second, Container=mp4, Audio=AudioParameters(Codec=AAC, SampleRate=44100, BitRate=64, Channels=1, CodecOptions=AudioCodecOptions(Profile=auto)), Type=System
```

### 트랜스코딩 Job 요청 및 결과 확인
  * 아래는 **S3**에 업로드된 원본 비디오 파일에서 오디오만 **AAC**로 트랜스코딩하여 추출하여 **S3**에 결과물을 업로드하는 예제이다.

```kotlin
// 트랜스코딩을 요청할 파이프라인 ID
val pipelineId = "{elastic-transcoder-pipeline-id}"

// 원본 파일의 S3 Key를 지정
// S3 Bucket은 파이프라인 생성 시점에 사전 지정
val inputFileKey = "{input-s3-key}"

// 대상 파일의 S3 Key를 지정
// S3 Bucket은 파이프라인 생성 시점에 사전 지정
val outputFileKey = "{output-s3-key}"

// [Audio AAC - 128k] 시스템 프리셋을 지정
val presetId = "1351620000001-100130"

val jobInput = JobInput.builder()
    .key(inputFileKey)
    .aspectRatio("auto")
    .container("auto")
    .frameRate("auto")
    .resolution("auto")
    .build()

val createJobOutput = CreateJobOutput
    .builder()
    .key(outputFileKey)
    .presetId(presetId)
    .build()

val createJobRequest = CreateJobRequest
    .builder()
    .pipelineId(pipelineId)
    .input(jobInput)
    .output(createJobOutput)
    .build()

// 트랜스코딩 Job을 파이프라인에 요청
val createJobResponse = transcoderClient.createJob(createJobRequest)
```

  * 존재하지 않는 **Preset ID**를 지정하면 아래와 같이 `ResourceNotFoundException` 예외가 발생함에 유의한다.

```kotlin
ResourceNotFoundException: The specified preset was not found: account={account}, presetId={preset-id}
```

  * 요청된 **Job**의 실행 결과를 아래와 같이 확인할 수 있다.

```kotlin
repeat((1..1000).count()) {

    // 10초마다 트랜스코딩 Job 처리 결과 확인
    // JVM 19 이상에서는 Virtual Thread를 사용
    Thread.sleep(1000 * 10)

    val readJobResponse = transcoderClient
        .readJob(
        	ReadJobRequest
        	    .builder()
        	    .id(createJobResponse.job().id())
        	    .build()
        )

    when (readJobResponse.job().status().uppercase()) {

    	  // 트랜스코딩 완료 결과를 처리
        "COMPLETE" -> {
            val queuedAt: Instant = Instant.ofEpochMilli(readJobResponse.job().timing().submitTimeMillis())
            val startedAt: Instant = Instant.ofEpochMilli(readJobResponse.job().timing().startTimeMillis())
            val endedAt: Instant = Instant.ofEpochMilli(readJobResponse.job().timing().finishTimeMillis())
            val outputFileSize: Long = readJobResponse.job().output().fileSize()
        }

        // 트랜스코딩 에러 결과를 처리
        "ERROR" -> {}
    }
}
```
