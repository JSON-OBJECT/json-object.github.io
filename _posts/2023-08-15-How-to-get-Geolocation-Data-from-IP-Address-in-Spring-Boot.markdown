---
layout: post
title: "Kotlin + Spring Boot, IP 주소로부터 Geolocation 정보 조회하기"
author: "Taehyeong Lee"
tags: [Kotlin, Geolocation]
---
### 개요
  * 프로덕션 레벨의 백엔드 서비스를 운영하다보면 비지니스 차원이든, 관제 목적이든 요청 **IP** 주소를 기반으로 지역 정보를 추출해야 하는 요건이 발생한다. 해결책은 여러가지 방법이 있지만 이번 글에서는 로컬 데이터베이스 파일을 이용하여 제한없이 안전하고 빠르게 **Geolocation** 정보를 획득하는 방법을 정리했다.

### GeoLite2 로컬 데이터베이스 다운로드
  * `MaxMind`는 2002년 창업하여 **IP** 인텔리전스 분야만 20년 넘게 영위한 전문 기업이다. 이 회사가 무료로 제공하는 `GeoLite2` **IP Geolocation** 로컬 데이터베이스 파일을 이용하여 예제를 작성할 것이다. 제작사 홈페이지에 회원 가입 후 로그인하면 무료로 다운로드할 수 있다. [[다운로드 링크]](https://www.maxmind.com/en/accounts/current/geoip/downloads)
  * 다운로드 방법의 다른 대안으로 일반 유저인 **P3TERX**가 자신의 **GitHub** 저장소에 수년간 최신 버전을 제공하고 있어 **MaxMind** 홈페이지 회원 가입 및 로그인 없이도 다운로드가 가능하다. [[GitHub 저장소 링크]](https://github.com/P3TERX/GeoLite.mmdb)

```bash
$ wget -nv -O GeoLite2-ASN.mmdb https://git.io/GeoLite2-ASN.mmdb
$ wget -nv -O GeoLite2-City.mmdb https://git.io/GeoLite2-City.mmdb
$ wget -nv -O GeoLite2-Country.mmdb https://git.io/GeoLite2-Country.mmdb
```

  * 데이터베이스는 아래 3개 파일로 구성되며, 2주 간격으로 홈페이지에 새로운 버전이 업로드된다. 무료 버전은 최대 조회 횟수가 무제한이라는 장점이 있지만, 유료 버전보다 상대적으로 부정확하고, 새로 업데이트된 파일을 수작업으로 로그인해서 직접 갱신해야 하는 번거로움이 있다.

```kotlin
GeoLite2-ASN.mmdb / 7.83 MB
GeoLite2-City.mmdb / 68.4 MB
GeoLite2-Country.mmdb / 5.91 MB
```

### build.gradle.kts
  * 프로젝트 루트의 **build.gradle.kts**에 아래 내용을 추가한다.

```kotlin
dependencies {
    implementation("com.maxmind.geoip2:geoip2:4.1.0")
}
```

### GeoLocationConfig.kt
  * 가장 먼저 `DatabaseReader` 클래스를 스프링 싱글턴 빈으로 등록한다.

```kotlin
import com.maxmind.db.CHMCache
import com.maxmind.geoip2.DatabaseReader
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import java.io.File
 
@Configuration
class GeoLocationConfig {
 
    @Bean("databaseReader")
    fun databaseReader(): DatabaseReader {
 
        return DatabaseReader
            .Builder(File("/GeoLite2-City.mmdb"))
            .withCache(CHMCache())
            .build()
    }
}
```

  * 앞서 다운로드한 로컬 데이터베이스 파일의 위치가 **/**라고 가정했다. 프로젝트 상황에 맞게 자유롭게 지정하면 된다.
  * 굳이 싱글턴 빈으로 등록하는 이유는 로컬 캐시 역할의 `CHMCache` 인스턴스를 애플리케이션 기동 중에 재사용하기 위함이다. 한번 요청된 **IP** 주소는 최대 2,000개까지 저장하여 로컬 데이터베이스 파일에서 조회하지 않고 로컬 캐시에서 응답한다. 2,000개가 초과하면 오래된 순서로 제거한다.)

### GeoLocationService.kt
  * **GeoLocationService** 클래스를 아래와 같이 제작한다. **IP** 주소를 기반으로 **Geolocation** 정보를 조회하여 반환하는 역할이다.

```kotlin
import com.maxmind.geoip2.DatabaseReader
import com.maxmind.geoip2.model.CityResponse
import com.maxmind.geoip2.record.Country
import org.springframework.stereotype.Service
import java.io.Serializable
import java.net.InetAddress
 
@Service
class GeoLocationService(
    private val databaseReader: DatabaseReader
) {
    fun getGeoLocation(ipAddress: String?): GeoLocationDTO? {
 
        if (ipAddress.isNullOrBlank()) return null
 
        return try {
 
            val response: CityResponse = databaseReader.city(InetAddress.getByName(ipAddress))
            val country: Country = response.country
            val subdivision = response.getMostSpecificSubdivision()
 
            GeoLocationDTO(
                ipAddress = ipAddress,
                country = country.name,
                countryCode = country.isoCode,
                subdivision = subdivision.name,
                subdivisionCode = subdivision.isoCode
            )
 
        } catch (ex: Exception) {
            null
        }
    }
}
 
data class GeoLocationDTO(
 
    var ipAddress: String? = null,
    var country: String? = null,
    var countryCode: String? = null,
    var subdivision: String? = null,
    var subdivisionCode: String? = null
 
) : Serializable
```

### 사용 예
  * 앞서 제작한 서비스 빈을 아래와 같이 사용할 수 있다.

```kotlin
// country: South Korea, country_code: KR, subdivision: Seoul, subdivision_code: 11
val geolocation = geoLocationService.getGeoLocation({ip-address})
```

### 프로덕션 도입 경험
  * 도입에 있어 가장 중요한 요소는 **IP Geolocation** 정보의 정확도이다. 제작사인 **MaxMind** 측에서도 **IP** 주소에 기반한 지역 정보 추정은 본질적으로 부정확할 수 있으니 비지니스 모델에 진지하게 사용하지는 말라고 권장하고 있다. 실제 고객 유입에 대응하여 프로덕션 레벨로 사용해보니 **Country(ex: Korea)**는 100% 정확했고 **Subdivision(ex: Seoul)**도 아직은 빈번한 오차를 경험하지 않았다. 그런데 **City(ex: Gangnam-gu)**는 당장 내가 위치한 사무실 위치도 틀리게 나올 정도로 너무 들쑥날쑥해서 신뢰할 수 없었다. 그래서 본 글의 예제에서도 **Country**, **Subdivision** 획득 부분까지만 작성했다.
  * 정확도 다음으로 중요한 요소는 조회 속도와 부하이다. 나는 백엔드의 모든 **API** 요청의 전처리 구간에 사용하기 때문에 상당히 중요했다. 일단 **SQLite** 기반의 로컬 데이터베이스이므로 네트워크 부하는 전혀 없다. 항상 최신 버전을 백엔드의 **Docker** 이미지에 포함하여 빌드하면 된다. 본 예제에서의 같이 로컬 캐시를 활성화하고 프로덕션 레벨에서 실사용 결과 조회 속도는 **99.9%**가 **0ms**를 응답하여 매우 빠르다. 아주 간헐적으로 10ms 미만으로 응답할 뿐이다. 결국 문제 없이 프로덕션에 정착했다.

### 참고 글
  * [MaxMind - GeoLite2 Free Geolocation Data](https://dev.maxmind.com/geoip/geolite2-free-geolocation-data)
