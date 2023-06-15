---
layout: post
title: "Spring Boot, DynamoDB 테이블 설계 및 CRUD 사용법 정리"
author: "Taehyeong Lee"
tags: [Spring Boot, DynamoDB]
---
### 개요
  * **Spring Boot** 기반 프로젝트에서 **DynamoDB**에 **CRUD**를 수행하기 위한 테이블 설계 및 사용법을 정리했다. **AWS**에서 권장하는 `DynamoDB Enhanced Client` 라이브러리를 사용했다.

### DynamoDB 테이블 설계시 고려할 부분
  * **DynamoDB**는 프로덕션 환경에서 무한히 확장하는 데이터를 저장하기에 적합하다. 그만큼 제약이 따르는데 조인을 지원하지 않는다. 그래서 용도를 정확히 이해하고 전통적인 **RDBMS**와는 다른 접근법이 필요하다.
  * 테이블의 기본 키는 **파티션 키(PK)**와 **소트 키(SK)**의 조합으로 구성된다. 조회 관점에서 파티션 키는 **Hash** 방식이라 값이 정확히 일치해야 하고, 소트 키는 **Range** 방식이라 앞부분부터 부분 일치하는 데이터를 필터링할 수 있다. 파티션 키와 소트 키에 포함되지 않는 필드로 조회하려면 테이블을 풀 스캔하거나 해당 필드에 대해 `LSI` 또는 `GSI`를 생성해야 한다.
  * 일단 **LSI**는 프로덕션 환경에서 제약이 크므로 용도가 명확할 때만 생성해야 한다. **LSI**의 장점은 조회시 언제나 최신 데이터를 보장한다. 반면에 테이블을 생성하는 시점에만 **LSI**를 생성할 수 있고, 이미 생성된 테이블에서는 인덱스 추가, 제거가 불가능하다. 마지막으로 가장 큰 단점은 **LSI**가 1개 이상 존재하는 테이블은 동일 파티션의 최대 크기가 **10GB**로 제한된다. 초과시 **ItemCollectionSizeLimitExceededException** 예외가 발생하는데 프로덕션 환경에서는 치명적일 수 있다.
  * **GSI**는 차지하는 크기 만큼 그대로 비용이 된다. 그래서 항상 **GSI**를 작게 유지해야 한다. 인덱스 생성시 프로젝션 타입을 `KEYS_ONLY`로 설정하면 인덱스 키, 파티션 키, 소트 키 값만 차지하여 저장 공간을 최소화할 수 있다. (`ALL`로 설정하면 기본 테이블과 동일한 복제본을 유지하기에 2배를 차지하게 된다.)

### DynamoDB 테이블 조회시 고려할 부분
  * **Partition Key + Sort Key** 목록으로 이용하여 단 한번의 요청으로 복수개의 아이템을 조회하고 싶다면 `BatchGetItem`를 실행하면 된다. 즉, 단건 조회에 해당하는 **GetItem**의 목록 조회 버전이다. [[관련 링크]](https://stackoverflow.com/a/57232964)
  * **Query**를 실행하면 **Partition Key**와 **Sort Key**의 조합에 의해 물리적으로 필터링된 조회가 발생한다. 추가로 `Filter Expression`을 사용하여 조회 결과를 더욱 좁힐 수 있는데, 앞서와 동일한 조회 결과에서 **DynamoDB** 서버에서 한번 더 필터링하여 클라이언트에 정제된 결과를 응답하는 것이다. 필터식을 사용해도 사용하지 않을 때와 물리적인 조회량이 동일하므로 조회 제약과 과금 또한 동일하다는 것에 유의해야 한다. [[관련 링크]](https://dynobase.dev/dynamodb-filterexpression/)

### build.gradle.kts
  * 프로젝트의 **/build.gradle.kts**에 아래 내용을 추가한다.

```kotlin
dependencies {
    implementation("software.amazon.awssdk:dynamodb-enhanced:2.20.127")
}
```

### 환경 설정
  * 실제 **DynamoDB** 연결시 **@Repository** 빈에서 사용할 `dynamoDbEnhancedClient` 싱글턴 빈을 등록할 차례이다.

```kotlin
import org.springframework.beans.factory.annotation.Qualifier
import org.springframework.beans.factory.annotation.Value
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import software.amazon.awssdk.auth.credentials.AwsBasicCredentials
import software.amazon.awssdk.auth.credentials.StaticCredentialsProvider
import software.amazon.awssdk.enhanced.dynamodb.DynamoDbEnhancedClient
import software.amazon.awssdk.regions.Region
import software.amazon.awssdk.services.dynamodb.DynamoDbClient

@Configuration
class DynamoDbConfig {

    @Bean
    fun dynamoDbClient(): DynamoDbClient {

        return DynamoDbClient.builder()
            .region(Region.AP_NORTHEAST_2)
            .credentialsProvider(
                StaticCredentialsProvider.create(
                    AwsBasicCredentials.create("{accessKey}", "{secretKey}")
                )
            )
            .build()
    }

    @Bean
    fun dynamoDbEnhancedClient(
        @Qualifier("dynamoDbClient") dynamoDbClient: DynamoDbClient
    ): DynamoDbEnhancedClient {

        return DynamoDbEnhancedClient.builder()
            .dynamoDbClient(dynamoDbClient)
            .build()
    }
}
```

### DynamoDB 테이블 생성
  * 빈 설계에 앞서 테이블을 생성할 차례이다. **Hash Key**로 채팅방 ID를 저장할 **pk: String**, **Range Key**로 채팅방 메시지 ID를 저장할 **sk: String**로 아래와 같이 테이블을 설계한다.

```bash
$ aws dynamodb create-table --table-name {table-name} --attribute-definitions AttributeName=pk,AttributeType=S AttributeName=sk,AttributeType=S --key-schema AttributeName=pk,KeyType=HASH AttributeName=sk,KeyType=RANGE --billing-mode PAY_PER_REQUEST
```

### DynamoDB 빈 설계
  * 실제 물리적 테이블에 맵핑되는 빈을 설계할 차례이다. **Single Table Design**을 가정하여 간단하게 설계했다.

```kotlin
import software.amazon.awssdk.enhanced.dynamodb.mapper.annotations.*
import java.math.BigDecimal

@DynamoDbBean
data class MessageDynamoDbBean(

    // ROOM#{room-id}
    @get:DynamoDbPartitionKey
    @get:DynamoDbAttribute("pk")
    var pk: String = "",

    // MESSAGE#{message-id}
    @get:DynamoDbSortKey
    @get:DynamoDbAttribute("sk")
    var sk: String = "",

    // ROOM_MESSAGE
    @get:DynamoDbAttribute("type")
    var type: String = "",

    @get:DynamoDbAttribute("message")
    var message: String = "",

    @get:DynamoDbAttribute("customData")
    @get:DynamoDbConvertedBy(DynamoDbStringMapToJsonAttributeConverter::class)
    var customData: Map<String, String?>? = null,

    // USER#{user-id}#ROOM#{room-id}
    @get:DynamoDbSecondaryPartitionKey(indexNames = ["gsi-pk-user-id-room-id"])
    @get:DynamoDbAttribute("gsk1pk")
    var gsi1pk: String = ""
)
```

  * 각 애트리뷰트와 필드를 맵핑하기 위한 정보로서 `@DynamoDbAttribute`을 명시할 수 있다. 이를 통해 물리적인 애트리뷰트명과 필드명을 다르게 할 수 있다.
  * **pk**, **sk**, **type** 필드는 **Single Table Design**의 최소 필수 구성 필드이다. 하나의 테이블을 복수개의 빈이 같이 사용하는 형태로 모든 빈에 걸쳐서 3개 필드는 동일하게 구성해야 한다.
  * 파티션 키 필드에는 `@DynamoDbPartitionKey`을, 소트 키 필드에는 `@DynamoDbSortey`를 명시한다.
  * 기본 키와 별도로 메시지를 유저 기준으로도 조회할 수 있게 **GSI**를 적용해봤다. **GSI**를 적용할 해당 필드에 `@DynamoDbSecondaryPartitionKey(indexNames = ["{index-name}"])`, `@DynamoDbSecondarySortKey(indexNames = ["{index-name}"])`을 명시하면 된다.
  * **Map<String, String?>?** 타입의 필드를 **JSON** 문자열로 저장하기 위해 아래 설명할 별도로 제작한 **DynamoDbStringMapToJsonAttributeConverter**를 사용했다.
  * **UTC+0** 기준의 **timestamp** 정보를 저장하는 `Instant` 타입은 자동으로 **DynamoDB**의 **String** 타입으로 변환된다. (**Instant** 타입의 `toString()` 값인 **ISO 8601** 형식의 문자열이 그대로 저장된다.) 따라서 만약, **Instant** 타입을 **SK**로 사용할 경우 테이블 생성 시점에 반드시 **String** 타입으로 생성해야 한다.

### 커스텀 애트리뷰트 컨버터 제작: StringMapToJsonAttributeConverter
  * 빈 레벨에서 **Map<String, String?>?** 타입의 필드는 어떻게 저장해야 할까? 여러가지 방법이 있지만 해당 필드가 인덱스 대상이 아니라면 문자열 타입의 필드에 **JSON** 변환된 문자열을 저장하는 것이 가장 이상적이다. 아래와 같이 `AttributeConverter` 구현체를 제작할 수 있다.

```kotlin
import com.fasterxml.jackson.annotation.JsonInclude
import com.fasterxml.jackson.core.JsonProcessingException
import com.fasterxml.jackson.databind.DeserializationFeature
import com.fasterxml.jackson.databind.SerializationFeature
import com.fasterxml.jackson.datatype.jsr310.JavaTimeModule
import com.fasterxml.jackson.module.kotlin.jacksonObjectMapper
import software.amazon.awssdk.enhanced.dynamodb.AttributeConverter
import software.amazon.awssdk.enhanced.dynamodb.AttributeValueType
import software.amazon.awssdk.enhanced.dynamodb.EnhancedType
import software.amazon.awssdk.services.dynamodb.model.AttributeValue

class DynamoDbStringMapToJsonAttributeConverter : AttributeConverter<Map<String, String?>?> {

    override fun transformFrom(input: Map<String, String?>?): AttributeValue {

        return try {
            AttributeValue
                .builder()
                .s(mapper.writeValueAsString(input))
                .build()
        } catch (e: JsonProcessingException) {
            AttributeValue
                .builder()
                .nul(true)
                .build()
        }
    }

    override fun transformTo(input: AttributeValue): Map<String, String?>? {

        return try {
            mapper.readValue(input.s(), Map::class.java) as Map<String, String?>
        } catch (e: JsonProcessingException) {
            null
        }
    }

    override fun type(): EnhancedType<Map<String, String?>?>? {

        return EnhancedType.mapOf(String::class.java, String::class.java)
    }

    override fun attributeValueType(): AttributeValueType {

        return AttributeValueType.S
    }

    companion object {

        private val mapper = jacksonObjectMapper().apply {
            setSerializationInclusion(JsonInclude.Include.ALWAYS)
            configure(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES, false)
            disable(SerializationFeature.FAIL_ON_EMPTY_BEANS, SerializationFeature.WRITE_DATES_AS_TIMESTAMPS)
            registerModules(JavaTimeModule())
        }
    }
}
```

### CRUD: READ
  * 앞서 제작한 빈에 대한 목록 및 단건 조회 예제이다.

```kotlin
import org.springframework.stereotype.Repository
import software.amazon.awssdk.enhanced.dynamodb.DynamoDbEnhancedClient
import software.amazon.awssdk.enhanced.dynamodb.DynamoDbTable
import software.amazon.awssdk.enhanced.dynamodb.Key
import software.amazon.awssdk.enhanced.dynamodb.TableSchema
import software.amazon.awssdk.enhanced.dynamodb.model.QueryConditional
import software.amazon.awssdk.enhanced.dynamodb.model.QueryEnhancedRequest
import software.amazon.awssdk.services.dynamodb.model.ResourceNotFoundException
import java.util.stream.Collectors

@Repository
class MessageDynamoDbRepository(
    private val dynamoDbEnhancedClient: DynamoDbEnhancedClient
) {
    private val table: DynamoDbTable<MessageDynamoDbBean>
        get() = dynamoDbEnhancedClient.table(
            "{table-name}",
            TableSchema.fromBean(MessageDynamoDbBean::class.java)
        )

    // 목록 조회
    fun fetchAllByRoomIdAndMessageId(
        roomId: Long?,
        messageId: String? = "",
        sortBy: String = "lessThan",
        limit: Int = 100
    ): List<MessageDynamoDbBean> {

        roomId ?: return emptyList()

        val queryConditional = when (sortBy) {
            "lessThan" -> QueryConditional
                .sortLessThan(
                    Key.builder()
                        .partitionValue("ROOM#$roomId")
                        .sortValue("MESSAGE#${messageId}")
                        .build()
                )

            else -> QueryConditional
                .sortGreaterThan(
                    Key.builder()
                        .partitionValue("ROOM#$roomId")
                        .sortValue("MESSAGE#${messageId}")
                        .build()
                )
        }

        val queryEnhanceRequest = QueryEnhancedRequest.builder()
            .queryConditional(queryConditional)
            .limit(limit)
            // 정렬 방식 지정
            // true: ASC, false: DESC
            .scanIndexForward(true)
            .build()

        return table
            .query(queryEnhanceRequest)
            .items()
            .stream()
            .collect(Collectors.toList())
    }

    // 단건 조회
    fun fetchOneByRoomIdAndMessageId(
        roomId: Long?,
        messageId: String?
    ): MessageDynamoDbBean? {

        roomId ?: return null
        messageId ?: return null

        val queryConditional = QueryConditional
            .keyEqualTo(
                Key.builder()
                    .partitionValue("ROOM#$roomId")
                    .sortValue("MESSAGE#${messageId}")
                    .build()
            )

        val queryEnhanceRequest = QueryEnhancedRequest.builder()
            .queryConditional(queryConditional)
            .limit(1)
            .scanIndexForward(false)
            .build()

        return try {
            table
                .query(queryEnhanceRequest)
                .items()
                .stream()
                .findFirst()
                .get()
        } catch (ex: ResourceNotFoundException) {
            null
        } catch (ex: NoSuchElementException) {
            null
        }
    }
}
```

### CRUD: BATCH READ
  * `BatchGetItem` 명령으로 복수개의 아이템을 목록으로 조회할 수 있다.

```kotlin
fun fetchAllByRoomIdAndmessageIds(
    roomId: Long?,
    messageIds : List<String> = emptyList()
): List <MessageDynamoDbBean> {

    roomId ?: return emptyList()
    if (messageIds.isNullOrEmpty()) return emptyList()

    val readBatchBuilder = ReadBatch
        .builder(MessageDynamoDbBean::class.java)
        .mappedTableResource(table)

    messageIds.forEach {
        readBatchBuilder.addGetItem(
            Key
                .builder()
                .partitionValue("ROOM#$roomId")
                .sortValue("MESSAGE#${it}")
                .build()
        )
    }

    return dynamoDbEnhancedClient
        .batchGetItem {
            it.addReadBatch(readBatchBuilder.build())
        }
        .resultsForTable(table)
        .stream()
        .collect(Collectors.toList())
}
```

### CRUD: INSERT OR REPLACE
  * `PutItem` 명령은 이름 그대로 아래와 같이 실행할 수 있다. 만약, 동일한 **Primary Key**을 가진 아이템이 이미 존재할 경우, 별도의 예외를 발생시키지 않고 새로운 아이템으로 교체한다.

```kotlin
fun save(message: MessageDynamoDbBean) {
    table.putItem(message)
}
```

### CRUD: INSERT OR UPDATE
  * `UpdateItem`은 **PutItem**과 동작이 비슷하지만 차이점은 이미 아이템이 존재할 경우, 아이템 전체를 교체하지 않고 새로운 아이템의 필드 값을 추가하거나 교체한다.

```kotlin
fun update(message: MessageDynamoDbBean): MessageDynamoDbBean {
    table.updateItem(message)
}
```

### CRUD: Atomic Counter
  * 웹사이트의 방문자 수, 게시물이나 동영상의 조회 수 등을 업데이트하는 것은 원자성의 보장이 핵심이다. 동시 다발적으로 발생하는 여러 요청이 다른 값을 덮어쓰기라도 하면 데이터가 오염되어 쓸 수 없게 된다. **DynamoDB**는 이런 값의 저장을 위해 `Atomic Counter` 기능을 제공한다. 사용법은 아래와 같다.

```kotlin
@get:DynamoDbAtomicCounter(startValue = 0, delta = 1)
@get:DynamoDbAttribute("totalVisitCount")
var totalVisitCount: Long = 0,
```

  * 특정 애트리뷰트에 `@DynamoDbAtomicCounter`를 명시하는 것 만으로 **Atomic Counter**로 작동한다. 하나의 도큐먼트에 1개 이상의 **Atomic Counter**를 명시할 수 있다.
  * `startValue`는 도큐먼트를 최초 생성했을 때 초기값을 지정한다. 기본값은 0이다. `delta`는 도큐먼트에 `updateItem()`을 실행했을 때 증감 또는 차감할 단위이다. 1이면 1씩 증가하고, -1이면 1씩 감소한다. 기본값은 1이다.
  * 하나의 애트리뷰트는 지정된 **delta** 값에 의해서만 값이 업데이트되기 때문에, 1씩 올리다가 어떤 경우에만 1씩 다시 차감한다는 등의 업데이트는 불가능하다. 이런 경우에는 각 필드를 서로 다른 격리된 도큐먼트로 분리해야 한다.
  * 업데이트는 오직 `DynamoDbTable#updateItem` 실행을 통해서만 원자성이 보장된 카운트가 어노테이션으로 명시된 값 만큼 증가 또는 차감된다.

### CRUD: BATCH INSERTS OR REPLACE
  * `BatchWriteItem` 명령은 아래와 같이 실행한다. 2가지 예외를 고려하여 로직을 작성해야 한다.
  * 첫째, 1개 배치 요청 목록의 아이템 개수가 25개를 초과하면 예외가 발생한다.
  * 둘째, 배치 요청시 실패한 아이템에 대해 별도의 예외를 발생시키지 않는다. 대신 배치 응답에 포함된 `unprocessedDeleteItemsForTable()`, `unprocessedPutItemsForTable()`을 통해 실패한 아이템 목록에 대해 재시도 로직을 작성하면 된다.
  * 배치는 **PutItem** 명령에 비하면 로직 작성이 까다롭지만, 대량의 아이템을 다룰수록 처리 속도가 월등히 빠르다. (평균 175.22바이트 크기의 아이템을 **온디맨드** 모드로 배치 생성할 경우 1억건이 약 25분에 완료된다.)

```kotlin
fun saveAll(messages: List<MessageDynamoDbBean>) {

    // 아이템 목록을 최대 25개의 목록으로 분할
    messages.chunked(25).forEach { aChunkOfMessages ->
        val writeBatchBuilder = WriteBatch
            .builder(MessageDynamoDbBean::class.java)
            .mappedTableResource(table)

        aChunkOfMessages.forEach { message ->
            writeBatchBuilder.addPutItem(message)
        }

        val batchWriteItemEnhancedRequest: BatchWriteItemEnhancedRequest = BatchWriteItemEnhancedRequest
            .builder()
            .writeBatches(writeBatchBuilder.build())
            .build()

        val batchWriteResult = dynamoDbEnhancedClient.batchWriteItem(batchWriteItemEnhancedRequest)

        // 삭제 실패한 아이템 목록을 삭제 처리
        batchWriteResult.unprocessedDeleteItemsForTable(table).forEach { key ->
            table.deleteItem(key)
        }

        // 생성 실패한 아이템 목록을 생성 처리
        batchWriteResult.unprocessedPutItemsForTable(table).forEach { item ->
            table.putItem(item)
        }
    }
}
```

### CRUD: DELETE
  * `DeleteItem` 명령은 아래와 같이 실행할 수 있다. 파라메터의 **Primary Key** 필드 값에 해당하는 아이템을 삭제한 후, 삭제된 아이템을 반환한다.

```kotlin
fun delete(message: MessageDynamoDbBean): MessageDynamoDbBean {
    table.deleteItem(message)
}
```

### CRUD 예외 처리: 자동 재시도되는 예외 목록
  * **DynamoDB**는 예외 발생시 **SDK** 레벨에서 자동으로 정해진 최대 횟수까지 재실행을 요청한다. 아래는 재시도 로직이 작동하는 전체 예외 목록이다. (모두 `software.amazon.awssdk.services.dynamodb.model` 패키지에 속한다.)

```bash
# 400
ItemCollectionSizeLimitExceededException
LimitExceededException
ProvisionedThroughputExceededException
RequestLimitExceeded

# 500
InternalServerErrorException
```

### CRUD 예외 처리: 400 ProvisionedThroughputExceededException
  * **DynamoDB** 테이블의 과금 모드를 **Provisioned**로 설정하면, 설정된 한계치를 초과한 읽기/쓰기 요청이 갑작스럽게 발생할 경우 `400 ProvisionedThroughputExceededException`이 발생한다. 이 경우 **SDK**에 설정된 재시도 정책에 따라 하나의 실패 요청이 60초 가까이 소요되는 경우가 발생하는데 프로덕션 환경에서는 치명적일 수 있다. 오토 스케일링 정책이 설정되어 있어도 즉시 적용되지 않기 때문에 요청이 급격하게 증가하는 특정 시간 동안 이 오류는 필연적으로 발생한다. [[관련 링크]](https://medium.com/nerd-for-tech/aws-dynamodb-auto-scaling-is-not-a-magic-bullet-25b2fdc50e5a)
  * 가장 간단한 해결책은 테이블의 과금 모드를 **On-demand**로 변경하는 것이다. 방법은 아래와 같다. (업데이트는 무중단으로 진행되며 시간이 상당히 소요된다. 또한 온디맨드로의 변경은 하루 최대 1회만 허용된다.)

```bash
$ aws dynamodb update-table --table-name {table-name} --billing-mode PAY_PER_REQUEST
```

### CRUD 예외 처리: 500 Internal Server Error
  * 리파지터리 빈에서 **CRUD** 실행시 아래와 같이 **500 Internal Server Error**을 의미하는 `software.amazon.awssdk.services.dynamodb.model.InternalServerErrorException` 예외가 발생할 수 있다. **AWS**의 내부 문제가 원인으로 애플리케이션의 책임은 없다.

```bash
software.amazon.awssdk.services.dynamodb.model.InternalServerErrorException: Internal server error (Service: DynamoDb, Status Code: 500, Request ID: 236OI2K4PJSJK206OMTMV9DQF3VV4KQNSO5AEMVJF66Q9ASUAAJG, Extended Request ID: null)
```

  * 코드에서 점검할 부분은 다음과 같다. **PutItem**, **UpdateItem**, **DeleteItem** 실행시 위 예외가 발생했다면 해당 명령은 완료되었을 수도 있고, 아닐 수도 있다. 따라서 예외 발생시 **GetItem**으로 해당 명령이 정상적으로 실행되었는지 확인한 후, 해당 명령을 재시도하는 보완 로직이 작성되어야 한다. 만약, **TransactWriteItem** 실행시 발생했다면 해당 명령은 완료되지 않은 것으로 바로 재시도 로직을 작성하면 된다.

### 참고 글
  * [DynamoDB Tutorials - Everything You Need To Master It](https://dynobase.dev/dynamodb-tutorials/)
  * [AWS - Using the DynamoDB Enhanced Client in the AWS SDK for Java 2.x](https://docs.aws.amazon.com/sdk-for-java/latest/developer-guide/dynamodb-enhanced-client.html)
  * [AWS - Introducing enhanced DynamoDB client in the AWS SDK for Java v2](https://aws.amazon.com/ko/blogs/developer/introducing-enhanced-dynamodb-client-in-the-aws-sdk-for-java-v2/)
  * [GitHub - DynamoDB Enhanced Client](https://github.com/aws/aws-sdk-java-v2/tree/master/services-custom/dynamodb-enhanced)
  * [Working with AWS DynamoDB and Spring](https://reflectoring.io/spring-dynamodb/)
