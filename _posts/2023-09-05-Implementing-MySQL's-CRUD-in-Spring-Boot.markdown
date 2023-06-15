---
layout: post
title: "Kotlin + Spring Boot, JPA와 Querydsl 기본 설정 방법 정리"
author: "Taehyeong Lee"
tags: [Spring Boot, JPA, Querydsl, MySQL]
---

### 개요
  * **Kotin + Spring Boot + JPA** 기반 프로젝트에서 **MySQL**에 **CRUD**를 수행하기 위한 기본 설정 방법을 정리했다. **Spring Data JPA**, **Infobip Spring Data Querydsl**, **AWS JDBC Driver for MySQL**을 사용했다.

### build.gradle.kts
  * 프로젝트의 **/build.gradle.kts**에 아래 내용을 추가한다. (**Spring Initializr**에서 **Gradle - Kotlin** 선택 후 **Spring Data JPA** 의존성만 추가한 프로젝트를 기준으로 추가 내용만 작성했다.)

```kotlin
val springBootVersion by extra { "3.2.1" }

buildscript {
    val kotlinVersion = "1.9.21"
    dependencies {
        classpath("gradle.plugin.com.ewerk.gradle.plugins:querydsl-plugin:1.0.10")
        classpath("org.jetbrains.kotlin:kotlin-gradle-plugin:$kotlinVersion")
        classpath("org.jetbrains.kotlin:kotlin-allopen:$kotlinVersion")
        classpath("org.jetbrains.kotlin:kotlin-noarg:$kotlinVersion")
    }
}

plugins {
    val kotlinVersion = "1.9.21"
    kotlin("plugin.jpa") version kotlinVersion
    kotlin("kapt") version kotlinVersion
    idea
}

allOpen {
    annotation("jakarta.persistence.Entity")
    annotation("jakarta.persistence.MappedSuperclass")
    annotation("jakarta.persistence.Embeddable")
}

dependencies {
    // 로컬 환경 또는 원격지의 MySQL 연결시
    implementation("com.mysql:mysql-connector-j:8.2.0")
    // 원격지의 Amazon Aurora MySQL 연결시
    implementation("software.aws.rds:aws-mysql-jdbc:1.1.12")
    implementation("org.springframework.boot:spring-boot-starter-data-jpa:$springBootVersion")
    implementation("com.vladmihalcea:hibernate-types-60:2.21.1")
    implementation("io.hypersistence:hypersistence-utils-hibernate-60:3.7.0")
    implementation("com.infobip:infobip-spring-data-jpa-querydsl-boot-starter:9.0.2")
    kapt("com.querydsl:querydsl-apt:5.0.0:jakarta")
}

idea {
    module {
        val kaptMain = file("build/generated/source/kapt/main")
        sourceDirs.add(kaptMain)
        generatedSourceDirs.add(kaptMain)
    }
}
```

### 환경 변수
  * 아래 작성할 환경 설정 빈에 주입할 운영체제 환경 변수를 아래와 같이 작성한다.

```bash
# 로컬 환경 또는 원격지의 MySQL을 이용할 경우
SPRING_DATASOURCE_DRIVER_CLASS_NAME=com.mysql.cj.jdbc.Driver
SPRING_DATASOURCE_URL_READ_WRITE=jdbc:mysql://{read-write-url}:{port}/{database}?useUnicode=true&characterEncoding=utf8&useLegacyDatetimeCode=false&serverTimezone=UTC
SPRING_DATASOURCE_URL_READ_ONLY=jdbc:mysql://{read-only-url}:{port}/{database}?useUnicode=true&characterEncoding=utf8&useLegacyDatetimeCode=false&serverTimezone=UTC

# 원격지의 Amazon Aurora MySQL을 이용할 경우 (순정 MySQL에도 사용 가능)
SPRING_DATASOURCE_DRIVER_CLASS_NAME=software.aws.rds.jdbc.mysql.Driver
SPRING_DATASOURCE_URL_READ_WRITE=jdbc:mysql:aws://{read-write-url}:{port}/{database}?useUnicode=true&characterEncoding=utf8&useLegacyDatetimeCode=false&serverTimezone=UTC
SPRING_DATASOURCE_URL_READ_ONLY=jdbc:mysql:aws://{read-only-url}:{port}/{database}?useUnicode=true&characterEncoding=utf8&useLegacyDatetimeCode=false&serverTimezone=UTC

SPRING_DATASOURCE_USERNAME={username}
SPRING_DATASOURCE_PASSWORD={password}
SPRING_DATASOURCE_HIKARI_MAXIMUM_POOL_SIZE=20
SPRING_DATASOURCE_HIKARI_PROFILE_SQL=false
SPRING_JPA_PROPERTIES_HIBERNATE_CONNECTION_PROVIDER_DISABLES_AUTOCOMMIT=true
SPRING_JPA_DATASOURCE_PLATFORM=org.hibernate.dialect.MySQL8Dialect
```

  * 원격지 데이터베이스로 **Amazon Aurora MySQL**을 사용할 경우 예외 없이 무조건 **JDBC Driver**로 **AWS**가 특별히 맞춤형으로 제작한 `software.aws.rds.jdbc.mysql.Driver` 사용을 추천한다. 순정의 **com.mysql.cj.jdbc.Driver**를 사용할 경우 프라이머리 디비 인스턴스의 장애로 레플리카 인스턴스를 프라이머리 디비로 승격시키는 **Failover** 발생시 **DNS** 리졸빙의 시간차 반영으로 인해 필연적으로 쓰기 쿼리 실행시 **java.sql.SQLException: Running in read-only mode** 예외가 발생하지만, **AWS** 전용 드라이버를 사용하면 애플리케이션 입장에서는 장애 인지 없이 평균 2초, 최대 4초 만에 **Failover**가 완료된다. 프로덕션 환경에서는 안정성을 위해 쓰지 않을 이유가 없다. (심지어 순정 **MySQL** 인스턴스에도 문제 없이 사용이 가능하다.)
  * **AWS JDBC Driver for MySQL**의 커넥션 스트링에 `logger=software.aws.rds.jdbc.mysql.shading.com.mysql.cj.log.Slf4JLogger&profileSQL=true`를 추가하면 **JDBC** 레벨에서 실행 쿼리를 콘솔 로그로 확인할 수 있어 개발 환경 또는 디버깅 상황에서 추천한다.

### read-write, read-only 자동 분기 DataSource 빈 제작
  * **read-write**, **read-only** 자동 분기 **DataSource** 빈의 설정은 프로덕션 환경에서 무척 중요하다. **AWS Aurora MySQL**의 경우 최대 15개의 레플리카에 **read-only** 쿼리를 분산하여 프라이머리 디비의 부하를 줄일 수 있다. 작동 원리는 다음과 같다. `@Transactional(readOnly = false)`이 명시된 클래스 또는 메써드는 **read-write** 커넥션 풀을 사용하게 된다. 반대로 `@Transactional(readOnly = true)`이 명시된 클래스 또는 메써드는 **read-only** 커넥션 풀을 사용하게 된다.

```kotlin
import com.zaxxer.hikari.HikariConfig
import com.zaxxer.hikari.HikariDataSource
import org.springframework.beans.factory.annotation.Qualifier
import org.springframework.beans.factory.annotation.Value
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import org.springframework.context.annotation.Primary
import org.springframework.data.jpa.repository.config.EnableJpaRepositories
import org.springframework.transaction.annotation.EnableTransactionManagement
import javax.sql.DataSource

@Configuration
@EnableJpaRepositories(
    // JPA 활성화를 위한 프로젝트의 @Repository 클래스가 위치한 패키지명을 명시
    basePackages = ["com.jsonobject.example.repository"],
    // Infobip Spring Data Querydsl 기반의 @Repository 클래스 사용시 명시, 아닐 경우 생략
    repositoryFactoryBeanClass = ExtendedQuerydslJpaRepositoryFactoryBean::class
)
@EnableTransactionManagement
class DatabaseConfig(
    @Value("\${spring.datasource.driver-class-name}") private val DRIVER_CLASS_NAME: String,
    @Value("\${spring.datasource.url.read-write}") private val READ_WRITE_URL: String,
    @Value("\${spring.datasource.url.read-only}") private val READ_ONLY_URL: String,
    @Value("\${spring.datasource.username}") private val USERNAME: String,
    @Value("\${spring.datasource.password}") private val PASSWORD: String,
    @Value("\${spring.datasource.hikari.maximum-pool-size}") private val MAXIMUM_POOL_SIZE: Int,
    @Value("\${spring.datasource.hikari.profile-sql}") private val PROFILE_SQL: Boolean
) {
    @Bean(name = ["readWriteDataSource"])
    fun readWriteDataSource(): DataSource {

        return buildDataSource(
            DRIVER_CLASS_NAME,
            READ_WRITE_URL,
            USERNAME,
            PASSWORD,
            "read-write",
            MAXIMUM_POOL_SIZE,
            PROFILE_SQL
        )
    }

    @Bean(name = ["readOnlyDataSource"])
    fun readOnlyDataSource(): DataSource {

        return buildDataSource(
            DRIVER_CLASS_NAME,
            READ_ONLY_URL,
            USERNAME,
            PASSWORD,
            "read-only",
            MAXIMUM_POOL_SIZE,
            PROFILE_SQL
        )
    }

    @Primary
    @Bean(name = ["dataSource"])
    fun dataSource(
        @Qualifier("readWriteDataSource") readWriteDataSource: DataSource,
        @Qualifier("readOnlyDataSource") readOnlyDataSource: DataSource
    ): DataSource {

        val routingDataSource = TransactionRoutingDataSource()
        val dataSourceMap: MutableMap<Any, Any> = HashMap()
        dataSourceMap[DataSourceType.READ_WRITE] = readWriteDataSource
        dataSourceMap[DataSourceType.READ_ONLY] = readOnlyDataSource
        routingDataSource.setTargetDataSources(dataSourceMap)

        return routingDataSource
    }

    private fun buildDataSource(
        driverClassName: String,
        jdbcUrl: String,
        username: String,
        password: String,
        poolName: String,
        maximumPoolSize: Int,
        profileSql: Boolean
    ): DataSource {

        val config = HikariConfig()
        config.driverClassName = driverClassName
        config.jdbcUrl = jdbcUrl
        config.username = username
        config.password = password
        config.poolName = poolName
        config.maximumPoolSize = maximumPoolSize
        config.addDataSourceProperty("profileSql", profileSql)
        if (DRIVER_CLASS_NAME in arrayOf("com.mysql.cj.jdbc.Driver", "software.aws.rds.jdbc.mysql.Driver")) {
            config.connectionInitSql = "SET NAMES utf8mb4"
            config.addDataSourceProperty("cachePrepStmts", true)
            config.addDataSourceProperty("prepStmtCacheSize", 250)
            config.addDataSourceProperty("prepStmtCacheSqlLimit", 2048)
            config.addDataSourceProperty("useServerPrepStmts", true)
            config.addDataSourceProperty("useLocalSessionState", true)
            config.addDataSourceProperty("rewriteBatchedStatements", true)
            config.addDataSourceProperty("cacheResultSetMetadata", true)
            config.addDataSourceProperty("cacheServerConfiguration", true)
            config.addDataSourceProperty("elideSetAutoCommits", true)
            config.addDataSourceProperty("maintainTimeStats", false)
            config.addDataSourceProperty("rewriteBatchedStatements", true)
        }

        return HikariDataSource(config)
    }
}

// read-write, read-only 자동 분기 DataSource 빈 제작
class TransactionRoutingDataSource : AbstractRoutingDataSource() {
    
    override fun determineCurrentLookupKey(): DataSourceType {

        return if (TransactionSynchronizationManager.isCurrentTransactionReadOnly()) {
            DataSourceType.READ_ONLY
        } else {
            DataSourceType.READ_WRITE
        }
    }
}

enum class DataSourceType {
    READ_WRITE,
    READ_ONLY
}
```

### JPA 엔티티 설계 예
  * **JPA** 설계의 시작점인 엔티티를 설계한 예이다.

```kotlin
import io.hypersistence.utils.hibernate.id.Tsid
import jakarta.persistence.Column
import jakarta.persistence.Entity
import jakarta.persistence.Id
import jakarta.persistence.Table
import org.hibernate.proxy.HibernateProxy
import java.io.Serializable

@Entity
@Table(name = "foo")
class Foo : Serializable {

    @Id
    @Tsid
    @Column
    var id: Long? = null

    @Column
    var bar: String? = null

    // equals()
    // hashCode()
    // toString()
}
```

  * **PK** 역할을 수행하는 **Long** 타입 필드에 `@Tsid`를 명시하면 **FooRepository.save()** 실행 시점에 애플리케이션 레벨에서 **TSID** 값을 생성하여 저장한다. 기본적으로 생성된 시간 순으로 정렬이 가능하며, **Long** 타입일 경우 **8 bytes** 을 소모하고, **String** 타입일 경우 **13 bytes**를 소모하여 비슷한 류의 **UUID** 중에서 가장 공간 효율적이라 데이터베이스의 기본키에 어울린다. 보편적으로 더 많이 사용하는 `@GeneratedValue(strategy = GenerationType.IDENTITY)` 대비 가지는 장점은 **JPA** 레벨에서의 **Bulk Insert**가 가능하다.
  * 모든 엔티티는 기본 필드 설계 후에 **equals()**, **hashCode()**, **toString()** 메써드를 구현해야 한다. **IntelliJ IDEA** 유료 구독자는 **JPA Buddy** 플러그인의 모든 기능을 사용할 수 있어 쉽게 자동 생성할 수 있다.

### JPA 리파지터리 빈 설계 예
  * 실제 물리적 테이블에 대한 **CRUD**를 실행할 **JPA** 리파지터리 빈을 설계할 차례이다.

```kotlin
import com.infobip.spring.data.jpa.ExtendedQuerydslJpaRepository

interface FooRepository : ExtendedQuerydslJpaRepository<Foo, Long>
```

  * **Infobip Spring Data Querydsl**을 사용하면 `ExtendedQuerydslJpaRepository<T, ID>` 인터페이스 상속 만으로 위와 같이 한 줄로 리파지터리 빈을 쉽게 생성할 수 있다. 자세한 사용법은 [여기](https://github.com/infobip/infobip-spring-data-querydsl#jpa-module)를 참고한다.
  * 또는 **Spring Data JPA**에서 제공하는 `QuerydslRepositorySupport`를 상속하는 방식으로 아래와 같이 작성할 수 있다. 이 경우 커스터마이징의 범위가 넓어진다.

```kotlin
import com.querydsl.jpa.impl.JPAQuery
import com.querydsl.jpa.impl.JPAQueryFactory
import jakarta.annotation.Resource
import jakarta.persistence.EntityManager
import org.springframework.beans.factory.annotation.Autowired
import org.springframework.stereotype.Repository
import org.springframework.transaction.annotation.Isolation
import org.springframework.transaction.annotation.Transactional
import com.jsonobject.example.entity.QFoo.foo

@Repository
@Transactional(readOnly = true, isolation = Isolation.READ_COMMITTED)
class FooRepositorySupport(
    @Autowired
    @Resource(name = "jpaQueryFactory")
    private val query: JPAQueryFactory,
    private val em: EntityManager
) : QuerydslRepositorySupport(Foo::class.java) {

    fun fetchById(id: Long?): Foo? {

        id ?: return null

        return this.fetch(
            foo.id.eq(id)
        )
    }
    
    @Transactional(readOnly = false)
    fun updateBarById(id: Long?, bar: String?): Long {
 
        id ?: return 0
 
        return query
            .update(foo)
            .set(foo.bar, bar)
            .where(foo.id.eq(id))
            .execute()
            // JPA의 1차 캐시에 엔티티의 변경 내용을 반영
            .also { em.refresh(em.getReference(Foo::class.java, id)) }
    }
}
```

### 참고 글
  * [Hibernate with Kotlin - powered by Spring Boot](https://kotlinexpertise.com/hibernate-with-kotlin-spring-boot/)
  * [Infobip Spring Data Querydsl](https://github.com/infobip/infobip-spring-data-querydsl)
  * [Read-write and read-only transaction routing with Spring](https://vladmihalcea.com/read-write-read-only-transaction-routing-spring/)
  * [Hibernate setAutoCommit 최적화를 통한 성능 튜닝](https://pkgonan.github.io/2019/01/hibrnate-autocommit-tuning)
