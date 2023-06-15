---
layout: post
title: "Spring Boot, Amazon S3에 최대 5TB 대용량 파일 Multipart 업로드 구현하기"
author: "Taehyeong Lee"
tags: [Spring Boot, AWS]
---
### 개요

-   클라이언트-서버 관계에서 파일 업로드 구현시 파일의 최종 저장 위치가 `Amazon S3`일 경우, 서버는 클라이언트에게 제한된 시간을 가지는 업로드 전용의 `Presigned URL`을 제공하여 파일 업로드 처리를 서버가 직접 부담하지 않고 **S3**에게 전가할 수 있다. 이를 통해 서버 입장에서는 보안과 서버 자원 절약이라는 2마리 토끼를 모두 잡을 수 있다.
-   한가지 이슈는 **Presigned URL**로 업로드 가능한 최대 파일 크기가 **5GB**로 이 것을 초과하는 대용량 파일은 업로드가 불가능하다. **AWS**가 권장하는 해결책은 `Multipart` 기능으로 원본 대용량 파일을 복수개의 작은 단위로 쪼개어 업로드하는 것이다. 이번 글에서는 **Spring Boot**에서 **Amazon S3**의 **Multipart** 기능을 이용하여 클라이언트가 최대 **5TB**의 대용량 파일을 업로드할 수 있는 방법을 소개하고자 한다.

### S3 멀티파트 업로드 흐름

-   클라이언트는 서버에게 업로드할 파일 정보를 제공한다. (ex: 파일 크기, 파일 이름)
-   서버는 클라이언트가 제공한 파일 크기를 기반으로 적절한 파트 개수를 계산하여 각 파트가 가지는 **partNumber**, **uploadUrl**의 목록을 클라이언트에게 응답한다. (이 때 **uploadUrl**은 특정 기간만 업로드가 가능한 **Presigned URL**을 제공한다.) 서버는 해당 멀티 파트 업로드를 식별하는 **uploadId**를 잘 저장해둔다.
-   클라이언트는 획득한 파트 목록을 이용하여 차례로 또는 병렬로 상황에 맞게 모든 파트의 업로드를 완료한다. (이 때 클라이언트가 브라우저라면 **File#slice()**를 이용하여 파일을 파트 개수만큼 논리적으로 분할하여 각 **chunk**를 개별 업로드할 수 있다.)
-   클라이언트가 모든 파트의 업로드를 완료하면 각 파트의 업로드 성공시 응답 헤더에서 획득한 **ETag** 값을 각 파트의 **partNumber** 값과 맵핑하여 서버에게 최종 업로드 완료를 요청한다.
-   서버는 클라이언트에게 받은 모든 파트의 **ETag**, **partNumber**와 최초 멀티 파트 생성시 획득한 **uploadId**를 **AWS**에 요청하여 멀티 파트 업로드를 최종 완료한다.

### S3 멀티파트 업로드를 써야 하는 이유

-   **Presigned URL**을 이용하여 단일 파일 업로드시 최대 **5GB** 이상만 업로드가 허용되는데, 멀티파트 업로드를 이용하면 최대 **5TB**까지 업로드할 수 있다.
-   **100MB**를 초과하는 동영상 등의 대용량 파일을 최대 **10,000개**로 **n**개로 쪼개어 병렬로 동시에 업로드할 수 있어 단일 파일 업로드 대비 처리 속도가 빠르다.
-   네트워크 이슈 등으로 업로드 실패시 문제가 발생한 해당 파트만 재업로드하면 되어 오류로 인한 영향 범위가 적다.

### S3 멀티파트 업로드시 고려할 점

-   **Amazon S3**에서 권장하는 **Multipart** 사용 여부의 판단 기준은 업로드 대상 오브젝트가 **100MB** 이상일 경우이다. [\[관련 링크\]](https://docs.aws.amazon.com/AmazonS3/latest/userguide/mpuoverview.html)
-   **Multipart**로 합해지는 **Object**의 최대 허용 업로드 파일은 **5TB = 5,497,558,138,880 bytes**이다.
-   **Multipart** 업로드시 개별 파트의 최대 업로드 허용 파일 크기는 **5GB = 5,368,709,120 bytes**이고, 최소 허용 크기는 **5MB = 5,242,880 bytes**이다.
-   **Multipart**의 최대 허용 개수는 10,000개이다.
-   대부분의 모던 브라우저에서 개별 파트의 최대 업로드 허용 크기는 제한이 없다. 예외적으로 **IE8**, **Firefox**는 **2GB = 2,147,483,648 bytes**로 제한이 있다. 이를 고려하여 개별 파트의 최대 업로드 크기를 **2GB**로 제한하면 모든 상황에 대응할 수 있다.
-   브라우저 레벨에서 개별 파트를 업로드할 경우 **S3** 버킷의 **CORS** 정책에서 아래 내용이 추가되어야 한다. 개별 파트 업로드시 **S3**에서 내려주는 `ETag` 응답 헤더를 브라우저가 획득해야 하므로 매우 중요하다.

```json
// Amazon S3 콘솔 로그인 > 버킷 > 권한 > CORS(Cross-origin 리소스 공유) > 편집
[
  {
    "AllowedHeaders": [
      "*"
    ],
    "AllowedMethods": [
      "POST",
      "GET",
      "HEAD",
      "PUT"
    ],
    "AllowedOrigins": [
      "*"
    ],
    "ExposeHeaders": [
      "ETag"
    ]
  }
]
```

### build.gradle.kts

-   프로젝트 루트의 **build.gradle.kts**에 아래 내용을 추가한다.

```bash
dependencies {
    implementation("software.amazon.awssdk:s3:2.19.13")
}
```

### AmazonS3Util 작성

-   코드 레벨에서 **Multipart Upload**를 실행하기 위한 유틸리티 클래스를 아래와 같이 작성한다.

```kotlin
import software.amazon.awssdk.core.sync.RequestBody
import software.amazon.awssdk.regions.Region
import software.amazon.awssdk.services.s3.S3Client
import software.amazon.awssdk.services.s3.model.*
import software.amazon.awssdk.services.s3.presigner.S3Presigner
import software.amazon.awssdk.services.s3.presigner.model.GetObjectPresignRequest
import software.amazon.awssdk.services.s3.presigner.model.PutObjectPresignRequest
import software.amazon.awssdk.services.s3.presigner.model.UploadPartPresignRequest
import java.io.File
import java.io.InputStream
import java.time.Duration

object AmazonS3Util {
    private fun s3ClientV2(): S3Client {

        return S3Client
            .builder()
            .region(Region.of("{region}"))
            .build()
    }

    private fun s3PresignerV2(): S3Presigner {

        return S3Presigner
            .builder()
            .region(Region.of("{region}"))
            .build()
    }


    fun createMultipartUploadV2(
        bucket: String,
        key: String
    ): CreateMultipartUploadResponse {

        return s3ClientV2().createMultipartUpload(
            CreateMultipartUploadRequest.builder().bucket(bucket).key(key).build()
        )
    }

    private fun generateWriteOnlyMultipartPresignedUrlV2(
        bucket: String,
        key: String,
        duration: Duration,
        uploadId: String,
        partNumber: Int
    ): String {

        return s3PresignerV2().presignUploadPart { request: UploadPartPresignRequest.Builder ->
            request.signatureDuration(duration)
                .uploadPartRequest { uploadPartRequest: UploadPartRequest.Builder ->
                    uploadPartRequest.bucket(bucket)
                        .key(key)
                        .partNumber(partNumber)
                        .uploadId(uploadId)
                }
        }.url().toString()
    }

    fun generateWriteOnlyMultipartPresignedUrlsV2(
        bucket: String,
        key: String,
        duration: Duration,
        uploadId: String,
        partSize: Int
    ): List<FileMultipartUploadUrlDTO> {

        val multipartPresignedUrls = mutableListOf<FileMultipartUploadUrlDTO>()
        (1..partSize).forEach { partNumber ->
            multipartPresignedUrls.add(
                FileMultipartUploadUrlDTO(
                    partNumber, generateWriteOnlyMultipartPresignedUrlV2(bucket, key, duration, uploadId, partNumber)
                )
            )
        }

        return multipartPresignedUrls
    }

    fun completeMultipartUploadV2(
        bucket: String,
        key: String,
        uploadId: String,
        parts: List<CompletedPart>
    ): CompleteMultipartUploadResponse {

        return s3ClientV2().completeMultipartUpload { request ->
            request
                .bucket(bucket)
                .key(key)
                .uploadId(uploadId)
                .multipartUpload(CompletedMultipartUpload.builder().parts(parts).build())
        }
    }
    
    fun abortMultipartUploadV2(
        bucket: String,
        key: String,
        uploadId: String
    ): AbortMultipartUploadResponse {

        return s3ClientV2().abortMultipartUpload { request ->
            request
                .bucket(bucket)
                .key(key)
                .uploadId(uploadId)
        }
    }

    fun calculateMultipartCount(originalFileSize: Long, requestCount: Long = 10): Int {

        val minPartSize: Long = 5242880
        val maxPartSize: Long = 2147483648
        val recommendedMinOriginalFileSize: Long = 104857600
        val maxPartCount: Long = 10000
        val correctedPartCount: Long = if (requestCount > maxPartCount) {
            maxPartCount
        } else {
            requestCount
        }

        if (originalFileSize < recommendedMinOriginalFileSize) return 1
        if (originalFileSize / correctedPartCount < minPartSize) {
            return (originalFileSize / minPartSize).toInt()
        }
        if (originalFileSize / correctedPartCount > maxPartSize) {
            return (originalFileSize / maxPartSize).toInt()
        }

        return requestCount.toInt()
    }
}

data class FileMultipartUploadUrlDTO(

    var partNumber: Int = 1,
    var uploadUrl: String = ""

) : Serializable
```

### 1\. 멀티파트 업로드 목록 요청

-   앞서 작성한 유틸리티를 이용하여 멀티파트 업로드 대상 **URL** 목록을 아래와 같이 생성할 수 있다.

```kotlin
// 특정 Bucket의 Key에 멀티파트 업로드를 하기 위한 uploadId 값을 요청
val uploadId = AmazonS3Util.createMultipartUploadV2("{bucket}", "{key}").uploadId()

// 멀티파트 업로드시 분할 업로드 개수를 계산
val multipartCount = AmazonS3Util.calculateMultipartCount({fileSize})

// 멀티파트 업로드 URL 목록을 생성
val multipartUploadUrls = AmazonS3Util.generateWriteOnlyMultipartPresignedUrlsV2(
    "{bucket}",
    "{key}",
    Duration.ofMinutes(60),
    uploadId,
    multipartCount
)
```

-   위에서 생성된 멀티파트 업로드 **URL**을 **JSON** 문자열로 표현하면 아래와 같다. **partNumber**와 페어링된 **uploadUrl**은 요청한 분할 개수 만큼 생성된다.

```json
[
    {
        "partNumber": 1,
        "uploadUrl": "{url}"
    },
    {
        "partNumber": 2,
        "uploadUrl": "{url}"
    }
]
```

### 2-1. 멀티파트 업로드 실행: 리눅스 사이드

-   앞서 확보한 **uploadUrl** 목록을 이용하여 아래와 같이 각 파트를 업로드할 수 있다. 가장 대중적인 `curl`을 사용했다.

```bash
$ curl -v -k -T multipart.mp4.1 "{url}"
< HTTP/1.1 100 Continue
* We are completely uploaded and fine
* Mark bundle as not supporting multiuse
< HTTP/1.1 200 OK
< x-amz-id-2: /lZyTJxaI8ZWoEUDNpY7AHzSUhaZegPw2/Fg3Riy2EZwQHNFbIOIuGfCuIufbwu0MAgLmrzx5Yw=
< x-amz-request-id: 9K40M0MSTVB0ZWDK
< Date: Wed, 21 Sep 2022 06:11:39 GMT
< ETag: "0b41b1b5c7228c08597fe7ae9ea06abc"
< Server: AmazonS3
< Content-Length: 0
```

-   각 파트의 업로드가 성공하면 `200 OK`을 응답하고 응답 헤더에 `ETag` 값으로 파트 고유의 해시 값을 내려준다. 마지막으로 모든 파트의 업로드를 완료하려면 **partNumber**에 이 **ETag** 값을 페어링하여 제공해야 한다.

### 2-2. 멀티파트 업로드 실행: 브라우저 사이드

-   브라우저에서는 `File#slice()`를 이용하여 아래와 같이 하나의 파일을 **n개**의 **chunk**로 분리하여 개별 파트로 업로드할 수 있다. (아래 예제는 회사 동료인 프론트 엔지니어 **Jun** 님의 도움을 받았다. [\[Jun 님 GitHub 링크\]](https://github.com/Jay-WKJun))
-   앞서 업로드 대상 버킷의 **CORS** 정책 설정으로 개별 파트 업로드의 결과로 내려주는 `ETag` 응답 헤더를 획득할 수 있다.

```javascript
const chunkInterval = Math.floor(file.size / {서버가 응답한 파트 개수});
let chunkedStart = 0;

const chunkWithUrlList = {서버가 응답한 파트 목록}.map(({
    partNumber,
    uploadUrl
}, i) => {
    if (i === {서버가 응답한 파트 개수}.length - 1) {
        chunkEnd = file.size;
    } else {
        chunkEnd = chunkedStart + chunkInterval;
    }

    const chunk = file.slice(chunkedStart, chunkEnd);
    chunkedStart = chunkEnd;

    return {
        uploadUrl,
        partNumber,
        chunk,
    }
});

const fulfilledList = [];
const rejectedList = [];

await Promise.allSettled(chunkWithUrlList.map(
    ({
        uploadUrl,
        partNumber,
        chunk
    }) => fetch(
        uploadUrl, {
            method: 'PUT',
            body: chunk,
        }).then((res) => {
        console.log(`partNumber : ${partNumber} / ETag : ${res.headers.get('ETag')}`)
        return {
            partNumber,
            eTag: res.headers.get('ETag').replace(/"/g, ''),
        }
    })
)).then((res) => {
    console.log(`upload result : ${res}`)
    res.forEach((el) => {
        if (el.status === 'fulfilled') {
            fulfilledList.push(el.value);
            return;
        }

        rejectedList.push(el.value);
    });
});

// 각 파트에 대한 fetch는 ETag와 partNumber를 기억하고, 모든 파트에 대한 fetch 완료 시점에 그대로 결과 목록을 서버에 요청
```

### 3-1. 멀티파트 업로드 완료 요청

-   **partNumber**와 개별 파트 업로드로 획득한 **ETag**을 페어링하여 아래와 같이 업로드 완료 요청을 실행하면 최종적으로 멀티파트 업로드가 완료된다.

```kotlin
// 멀티파트 업로드 완료 요청
AmazonS3Util.completeMultipartUploadV2(
    "{bucket}",
    "{key}",
    "{uploadId}",
    parts = listOf(CompletedPart.builder().partNumber({partNumber}).eTag("{eTag}").build())
)
```

-   실제로 모든 멀티 파트가 업로드되지 않은 상태에서 완료 요청을 하면 `software.amazon.awssdk.services.s3.model.S3Exception` 예외가 발생하므로 적절히 예외 처리하면 된다.
-   만약, 올바르지 않은 **ETag** 값을 요청하면 어떤 일이 일어날까? `software.amazon.awssdk.services.s3.model.NoSuchUploadException` 예외가 발생한다. 역시 적절히 예외 처리하면 된다.

### 3-2. 멀티파트 업로드 취소 요청

-   멀티파트 업로드 진행을 위해 **uploadId**을 획득한 순간부터, **S3**에 각 파트를 업로드하게 되는데 이 것이 완전한 업로드 완료로 이어지면 큰 문제가 없다. 하지만, 리얼 월드에서는 업로드 도중 취소와 같은 다양한 사례가 나올 수 있다. 가장 큰 문제는 업로드 중 완료하지 않은채 방치하게 되면 그 과정에서 업로드된 각 파트의 용량이 고스란히 **S3** 운영 비용으로 남게 된다. 이런 경우를 대비하여 아래와 같이 업로드 취소를 실행할 수 있다. 보통 1일 단위의 배치 스케쥴러를 작성하여 업로드 완료되지 않은 건을 취소 처리하면 된다.

```kotlin
// 멀티파트 업로드 취소 요청
AmazonS3Util.abortMultipartUploadV2(
    "{bucket}",
    "{key}",
    "{uploadId}"
)
```

### 참고 글

-   [Amazon S3 User Guide - Uploading and copying objects using multipart upload](https://docs.aws.amazon.com/AmazonS3/latest/userguide/mpuoverview.html)
-   [AWS S3 Multipart Uploads — Avoiding Hidden Costs from Unfinished Uploads](https://www.doit.com/aws-s3-multipart-uploads-avoiding-hidden-costs-from-unfinished-uploads/)
