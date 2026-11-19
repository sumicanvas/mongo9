# MongoDB 9.0: Hibernate ORM GA, 타임시리즈 및 서버 측 JavaScript

이 문서는 MongoDB Extension for Hibernate ORM의 GA(정식 출시)와 MongoDB 9.0의 타임시리즈·서버 측 JavaScript 변경을 설명합니다. 세 항목은 Java 개발 경험, 타임시리즈 컬렉션의 내부 카탈로그 표현, JavaScript 실행 보안이라는 서로 다른 영역의 변경입니다.

## 요약

* **Hibernate ORM Extension GA**: Java 개발자는 Hibernate의 `@Entity`, HQL/JPQL, Criteria Queries, Session API를 사용해 MongoDB 데이터를 다룰 수 있습니다. GA는 INNER/LEFT OUTER JOIN, 낙관적 잠금, Spring Boot 자동 설정 등을 포함합니다. Hibernate ORM 7.4 LTS가 필요합니다.
* **타임시리즈**: MongoDB 9.0은 타임시리즈 컬렉션의 카탈로그 표현을 단일 사용자 대상 네임스페이스로 바꿉니다. 버킷 기반 압축 저장은 유지되며, 컬렉션 rename이 지원됩니다.
* **서버 측 JavaScript**: MongoDB 9.0은 WASM 기반 JavaScript 엔진을 사용해 더 강한 격리·샌드박싱을 제공합니다. `$where`, `$function`, `$accumulator`는 8.0에서 deprecated 됐지만 9.0에서는 더 이상 deprecated 상태가 아닙니다.

## 1. Hibernate ORM Extension GA

MongoDB Extension for Hibernate ORM은 Hibernate를 사용 중인 Java 애플리케이션이 MongoDB를 백엔드 데이터베이스로 사용할 수 있게 해주는 공식 오픈소스 확장입니다. Hibernate ORM의 익숙한 API를 유지하면서 MongoDB 명령으로 질의를 변환합니다.

### GA에서 지원하는 주요 기능

* **INNER 및 LEFT OUTER JOIN**: HQL/JPQL의 표준 JOIN 문법으로 관련 엔티티를 조회합니다.
* **낙관적 잠금**: `@Version` 필드로 동시 수정 충돌을 탐지합니다.
* **Spring Boot 자동 설정**: MongoDB 연결 정보에 따라 dialect와 connection provider를 자동 설정합니다.
* **임베디드 문서**: 연관 객체를 문서 안에 중첩해 MongoDB 문서 모델을 활용할 수 있습니다.
* **MongoDB Query API**: HQL로 다루기 어려운 고급 기능은 MongoDB 고유 질의 기능을 활용할 수 있습니다. 발표에서 vector search, 지리공간 질의, MongoDB Search가 예로 제시됐습니다.
* **진단 개선**: 예외 메시지와 로그가 더 명확해졌습니다.

### 예제: 엔티티 관계, JOIN, 버전 기반 충돌 감지

```java
@Entity
class Customer {
    @Id
    String id;

    String country;
}

@Entity
class PurchaseOrder {
    @Id
    String id;

    @ManyToOne
    Customer customer;

    @Version
    long version;

    BigDecimal total;
}
```

캐나다 고객의 주문을 찾는 HQL 예제입니다.

```java
var orders = session.createQuery("""
    select o
    from PurchaseOrder o
    join o.customer c
    where c.country = :country
    """, PurchaseOrder.class)
    .setParameter("country", "CANADA")
    .getResultList();
```

애플리케이션은 HQL을 작성하고, 확장은 이를 MongoDB 질의로 변환합니다. `@Version`은 같은 엔티티를 여러 작업자가 동시에 수정할 때 버전 차이로 충돌을 감지하는 데 사용됩니다.

### 적용 시 유의점

GA는 모든 Hibernate 또는 SQL 기능이 MongoDB에서 동일하게 동작한다는 의미는 아닙니다. MongoDB 문서 모델에 자연스럽게 대응하지 않는 일부 SQL/HQL 구성은 지원되지 않을 수 있습니다. 기존 애플리케이션을 이전할 때 다음을 확인하세요.

* 사용하는 HQL/JPQL 기능과 엔티티 매핑의 지원 여부
* JOIN, 트랜잭션, 잠금 및 ID 생성 동작
* 관계형 테이블로 분리했던 데이터를 임베디드 문서로 모델링할 수 있는지
* 확장이 지원하지 않는 MongoDB 고유 기능에 네이티브 Query API가 필요한지

## 2. MongoDB 9.0 타임시리즈 변경

### 카탈로그 표현의 변화

이전 버전에서는 사용자가 보는 타임시리즈 컬렉션이 내부 `system.buckets` 컬렉션 위의 비물질화 뷰처럼 표현됐습니다. MongoDB 9.0에서는 타임시리즈 컬렉션이 카탈로그에서 하나의 사용자 대상 네임스페이스로 표현됩니다. 측정값을 버킷으로 묶어 압축 저장하는 방식은 유지됩니다.

즉, 이 변경은 측정값의 논리적 문서 구조를 새 모델로 바꾸는 것이 아니라, 컬렉션의 내부 카탈로그·네임스페이스 표현을 바꾸는 것입니다. 애플리케이션은 계속 타임시리즈 컬렉션 이름으로 읽고 씁니다. 내부 `system.buckets` 네임스페이스를 안정적인 애플리케이션 API로 취급하지 마세요.

### 사용자에게 보이는 변화

* MongoDB 9.0부터 타임시리즈 컬렉션에 `renameCollection`을 사용할 수 있습니다.
* 타임시리즈 컬렉션에 `find`, `aggregate`, `listIndexes`를 실행했을 때 커서 응답의 `ns`는 사용자가 요청한 네임스페이스를 반환합니다. 이전 버전에서는 내부 네임스페이스가 반환될 수 있었습니다.
* 데이터 모델과 버킷 압축 방식은 유지됩니다.

### 예제: 컬렉션 생성, 조회, 이름 변경

```javascript
db.createCollection("sensorReadings", {
  timeseries: {
    timeField: "ts",
    metaField: "device",
    granularity: "seconds"
  }
});

db.sensorReadings.insertMany([
  { ts: ISODate("2026-11-19T00:00:00Z"), device: "sensor-17", tempC: 24.1 },
  { ts: ISODate("2026-11-19T00:00:10Z"), device: "sensor-17", tempC: 24.4 }
]);

db.sensorReadings.aggregate([
  {
    $match: {
      device: "sensor-17",
      ts: { $gte: ISODate("2026-11-19T00:00:00Z") }
    }
  },
  { $sort: { ts: 1 } }
]);
```

MongoDB 9.0부터 타임시리즈 컬렉션도 이름을 변경할 수 있습니다.

```javascript
db.adminCommand({
  renameCollection: "iot.sensorReadings",
  to: "iot.sensorReadingsArchive"
});
```

### 업그레이드 주의사항

한 데이터베이스에 타임시리즈 컬렉션이 5,000개를 초과하는 클러스터는 MongoDB 9.0 업그레이드에 더 오랜 시간이 걸리고 일시적인 쓰기 지연이 발생할 수 있습니다. 사용하지 않는 타임시리즈 컬렉션을 정리하고, 테스트 환경에서 업그레이드 시간과 쓰기 지연을 먼저 확인하세요.

## 3. MongoDB 9.0 서버 측 JavaScript

MongoDB 9.0은 서버 측 JavaScript를 WASM(WebAssembly) 기반 엔진에서 실행합니다. 목표는 JavaScript 실행을 이전 구현보다 강하게 격리하고 샌드박스화하는 것입니다.

`$where`, `$function`, `$accumulator`는 MongoDB 8.0에서 deprecated 됐지만, 9.0에서는 deprecated 상태가 아닙니다. 이는 JavaScript를 일반적인 질의 방식으로 권장한다는 의미는 아닙니다. 단순 조건은 Query API나 기본 aggregation 연산자로 표현하는 것이 보통 더 명확합니다.

### 예제: `$where` 대신 Query API 사용

아래 두 질의는 `tempC`가 80보다 크고 상태가 `ALARM`인 문서를 찾습니다.

```javascript
// 서버 측 JavaScript를 사용하는 방식
db.readings.find({
  $where: function () {
    return this.tempC > 80 && this.status === "ALARM";
  }
});
```

```javascript
// 일반 조건을 Query API로 표현
db.readings.find({
  tempC: { $gt: 80 },
  status: "ALARM"
});
```

가능한 경우 두 번째처럼 기본 Query API를 우선 사용하세요. 사용자 정의 집계 표현식 등 기본 연산자로 표현하기 어려운 경우에만 서버 측 JavaScript 사용을 검토하고, 업그레이드 전에 결과와 성능을 테스트하세요.

### 날짜 동작의 호환성 변화

MongoDB 9.0에서 서버 측 JavaScript의 로컬 날짜 연산은 `mongod` 호스트의 시간대와 관계없이 UTC 기준으로 계산됩니다. 예를 들어 `Date.prototype.getHours()`는 서버의 현지 시간대가 아니라 UTC 시각을 반환합니다. 저장된 BSON 날짜 값, `getTime()`, `toISOString()`, `getUTC*()` 메서드는 영향을 받지 않습니다.

시간대별 계산은 JavaScript의 로컬 날짜 메서드 대신 aggregation 날짜 연산자에 시간대를 명시하세요.

```javascript
db.readings.aggregate([
  {
    $project: {
      ts: 1,
      seoulHour: {
        $hour: { date: "$ts", timezone: "Asia/Seoul" }
      }
    }
  }
]);
```

### 플랫폼 및 map-reduce

* `ppc64le` 아키텍처에서는 서버 측 JavaScript가 지원되지 않습니다. `$where`, `$function`, `$accumulator`, map-reduce에 의존하는 애플리케이션은 배포 아키텍처를 확인해야 합니다.
* `$where`·`$function`·`$accumulator`의 deprecated 상태가 바뀐 것과 map-reduce의 상태는 별개입니다. map-reduce가 deprecated 해제됐다고 가정하지 말고, 사용 중이라면 aggregation pipeline으로의 전환 여부를 따로 검토하세요.

## 업그레이드 전 체크리스트

* Hibernate를 사용하는 앱은 Hibernate ORM 7.4 LTS 및 확장의 호환 버전을 확인합니다.
* Hibernate의 HQL/매핑, JOIN, 트랜잭션, 낙관적 잠금을 실제 앱 질의로 검증합니다.
* 타임시리즈 컬렉션 수를 데이터베이스별로 확인하고, 5,000개 초과 여부와 업그레이드 영향을 검토합니다.
* 카탈로그 정보를 읽거나 `system.buckets`에 직접 접근하는 운영 스크립트가 있는지 확인합니다.
* 서버 측 JavaScript 연산자와 날짜 로컬 메서드를 코드·질의에서 찾아 테스트합니다.
* `ppc64le` 사용 여부와 map-reduce 의존성을 확인합니다.

## 참고 자료

* [MongoDB Extension for Hibernate ORM is Now GA](https://www.mongodb.com/company/blog/product-release-announcements/extension-for-hibernate-orm-now-ga)
* [Hibernate ORM Extension 문서](https://www.mongodb.com/docs/languages/java/mongodb-hibernate/current/)
* [MongoDB 9.0 Release Notes](https://www.mongodb.com/docs/manual/release-notes/9.0/)
* [Time Series Collections 문서](https://www.mongodb.com/docs/manual/core/timeseries-collections/)
* [Server-side JavaScript 문서](https://www.mongodb.com/docs/manual/core/server-side-javascript/)

