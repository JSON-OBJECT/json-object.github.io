---
layout: post
title: "Spring Boot, MySQL, 엔티티 스키마 변경 없이 Bulk Insert로 속도 개선하기"
author: "Taehyeong Lee"
tags: [MySQL, Spring Boot]
---
### 개요

-   현재 내가 몸 담고 있는 프로젝트는 세금 계산과 관련되어 한 번에 수십에서 수백만의 엔티티 생성이 발생하는 비지니스 로직이 존재하며, 레코드 생성의 속도가 곧 경쟁력이 되기에 최적화에 있어 아주 중요한 요소이다. 이번 글에서는 기존에 대량의 엔티티가 단건으로 저장되던 것을 **Buik Insert**로 개선하는 과정을 정리하였다.

### MySQL 커넥션 스트링 추가

-   **MySQL**은 반드시 `rewriteBatchedStatements=true` 옵션을 커넥션 스트링에 추가해야 **Bulk Insert**가 활성화된다. **Hikari Pool**을 이용하여 **DataSource**를 생성할 경우 아래와 같이 옵션을 추가하면 된다.

```kotlin
val config = HikariConfig().apply {
    ...
    addDataSourceProperty("rewriteBatchedStatements", true)
}
val datasource = HikariDataSource(config)
```

### FooRepository#saveAll 작성

-   아래는 실제 코드에서 `JdbcTemplate#batchUpdate`를 사용하여 **Foo**라는 엔티티의 목록을 배치로 생성하는 **Bulk Insert**의 실행 예이다.

```kotlin
import org.springframework.jdbc.core.JdbcTemplate
import org.springframework.stereotype.Repository
import org.springframework.transaction.annotation.Isolation
import org.springframework.transaction.annotation.Transactional
import java.math.BigDecimal
import java.sql.Timestamp

@Repository
@Transactional(readOnly = true, isolation = Isolation.READ_COMMITTED)
class FooRepositorySupport(
    private val jdbcTemplate: JdbcTemplate
) {
    @Transactional(isolation = Isolation.READ_COMMITTED)
    fun saveAll(foos: List<Foo>) {
        jdbcTemplate.batchUpdate(
            "INSERT INTO foo (string, long, double, boolean, instant) VALUES (?, ?, ?, ?, ?)",
            foos,
            4096
        ) { ps, foo ->
            foo.string?.let { ps.setString(1, it) } ?: ps.setNull(1, java.sql.Types.NULL)
            foo.long?.let { ps.setLong(2, it) } ?: ps.setNull(2, java.sql.Types.NULL)
            foo.double?.let { ps.setBigDecimal(3, BigDecimal.valueOf(it)) } ?: ps.setNull(3, java.sql.Types.NULL)
            foo.boolean?.let { ps.setBoolean(4, it) } ?: ps.setNull(4, java.sql.Types.NULL)
            foo.instant?.let { ps.setTimestamp(5, Timestamp.from(it)) } ?: ps.setNull(5, java.sql.Types.NULL)
        }
    }
}
```

-   예제 설명을 위해 **Foo** 엔티티가 **String**, **Long**, **Double**, **Boolean**, **Instant** 타입의 필드를 가진 것으로 가정했다. 전달되는 파라메터가 **null**일 경우 그대로 실행하면 오류가 발생하기 때문에 예외 처리 구문을 추가했다.
-   **JdbcTemplate#batchUpdate**의 `batchSize` 파라메터로 하나의 배치 단위에 생성할 엔티티 개수를 지정할 수 있다. 위 예제에서는 **4096**을 지정했는데, 이 것만으로 운영 환경에서 22만건의 엔티티의 생성 소요시간이 10배가 단축된 것을 확인했다.
-   **MySQL**의 경우, 한 번에 실행되는 쿼리의 크기가 클 경우 `max_allowed_packet` 파라메터에 의해 제한이 걸려 **ER\_NET\_PACKET\_TOO\_LARGE** 오류가 발생한다. **max\_allowed\_packet**와 **batchSize** 파라메터를 적절히 튜닝해야 한다.

### 적용 후기

-   **JdbcTemplate#batchUpdate**로 구현한 **Bulk Insert** 성능은 드라마틱하다. 앞서 언급했듯이 운영 환경에서 22만건의 엔티티의 생성 소요시간이 10배가 단축된 것을 확인했다.
-   **Type-Safe**의 장점을 누구보다 잘 알고 추구하기에 평소 코드 레벨에서 **RAW SQL**의 사용은 최대한 지양하려고 노력했지만, 기존 데이터베이스 스키마의 변경 없이 속도 개선이 가능하다는 것은 상당한 이득이었다.

### 참고 글

-   [Using Bulk Insert Statement](https://www.geeksengine.com/database/data-manipulation/bulk-insert.php)
