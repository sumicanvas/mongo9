# MongoDB Atlas Load Shedding 및 Backpressure-aware Driver 가이드

이 문서는 MongoDB Atlas에서 Load Shedding을 적용하기 전 확인할 드라이버 버전, 앱 업데이트 필요 여부, Atlas 설정 및 운영 시 주의사항을 정리합니다.

## 핵심 요약

* Intelligent Workload Management(IWM)의 기본 보호 기능은 적격 Atlas 클러스터에서 동작합니다. Load Shedding은 IWM 전체 활성화와 별개인 선택 기능으로, 과부하가 지속될 때 일부 작업을 거절합니다.
* Load Shedding 적용 전에 앱이 사용하는 MongoDB 드라이버의 버전을 확인하세요. 표의 최소 버전 이상이면 해당 드라이버가 overload 오류를 인식하고, 재시도 가능한 작업을 지수 백오프와 jitter로 처리할 수 있습니다.
* 드라이버를 업데이트해야 한다면 앱에서 실제로 사용하는 드라이버 패키지만 업데이트하면 됩니다. 모든 언어 드라이버나 Atlas 서버를 함께 업데이트하라는 뜻은 아닙니다.
* 드라이버 지원 버전은 Load Shedding을 켜기 위한 기술적 필수 조건과는 별개입니다. 지원되지 않는 드라이버를 사용해도 적격 Atlas 클러스터에서 설정은 가능하지만, 앱에서 overload 오류 감지와 안전한 재시도 처리를 직접 해야 할 수 있습니다.

## Backpressure-aware 드라이버 최소 버전

아래는 MongoDB 문서에 게시된 최소 버전입니다. 앱의 manifest, lockfile 또는 런타임에서 실제 설치된 패키지 버전을 확인해 비교하세요.

| 드라이버/클라이언트 라이브러리 | 최소 backpressure-aware 버전 |
|---|---:|
| C Driver | 2.5 |
| C++ Driver | 4.6 |
| .NET/C# Driver | 3.11 |
| Go Driver | 2.9 |
| Java Sync Driver | 5.12 |
| Java Reactive Streams Driver | 5.12 |
| Kotlin Coroutine Driver | 5.12 |
| Node.js Driver | 7.6 |
| PHP Library | 2.5 (PHP extension/PHPC 2.5.0 이상 필요) |
| PyMongo | 4.18 |
| Scala Driver | 5.12 |
| Ruby | 2.26 |
| Rust | 3.9 |

## 앱 드라이버를 업데이트해야 하나요?

* 현재 사용하는 드라이버가 표의 최소 버전 이상이면, backpressure-aware 동작을 위해 별도 업그레이드가 꼭 필요한 것은 아닙니다. 다만 조직의 보안·지원 정책에 따라 더 최신 버전 사용 여부를 검토하세요.
* 현재 버전이 최소 버전보다 낮고 드라이버의 내장 overload 처리를 사용하려면, 해당 앱의 MongoDB 드라이버 의존성을 최소 버전 이상으로 올린 뒤 호환성 테스트를 진행하세요.
* 여러 서비스가 서로 다른 언어 또는 드라이버를 사용한다면, MongoDB에 연결하는 각 서비스의 실제 드라이버를 개별 확인하세요. 사용하지 않는 언어의 드라이버까지 업데이트할 필요는 없습니다.
* 드라이버를 업데이트하지 않는 경우 앱에서 `SystemOverloadedError` 등 overload 신호를 감지하고, 재시도 가능한 조건을 확인한 뒤 제한된 지수 백오프와 jitter를 구현해야 합니다.

## 재시도 동작과 주의점

Load Shedding으로 거절된 작업은 `SystemOverloadedError` 라벨이 포함된 오류를 반환할 수 있습니다. Backpressure-aware 드라이버는 이 오류를 과부하 신호로 인식하지만, 오류에 `RetryableError`처럼 재시도를 유발하는 라벨이 있는 경우에 안전한 재시도를 수행합니다. `SystemOverloadedError`만 보고 무조건 재시도해서는 안 됩니다.

드라이버가 재시도를 하더라도 애플리케이션은 전체 재시도 시간, 재시도 상한, 계속 실패할 때 작업을 포기하거나 부하를 줄이는 조건을 관리해야 합니다. 무제한 재시도나 즉시 반복 재시도는 과부하를 악화시킬 수 있습니다.

## Atlas 적용 조건

* IWM은 MongoDB 9.0 이상을 실행하는 Atlas Dedicated 클러스터에서 활성화됩니다. IWM의 always-on 보호와 Load Shedding은 같은 설정이 아닙니다.
* Load Shedding은 MongoDB 9.0 이상을 실행하는 M30 이상 replica set에서만 효과가 있습니다.
* M10/M20에서는 설정 항목이 보일 수 있지만 켜도 Effective state가 Disabled로 남습니다. Sharded cluster에는 Load Shedding이 적용되지 않습니다.
* Free 및 Flex 클러스터는 해당 IWM 기능을 사용할 수 없습니다.
* Load Shedding은 기본적으로 `Atlas Managed`입니다. MongoDB 9.0에서는 기본 Effective state가 Disabled이므로, 명시적으로 켜려면 설정을 `Enabled`로 바꾸세요.

## Atlas UI에서 Load Shedding 활성화

1. Atlas에서 대상 Project를 선택하고 **Database → Clusters**로 이동합니다.
2. 대상 클러스터의 설정 편집 화면을 엽니다.
3. **Additional Settings → More Configuration Options**를 펼칩니다.
4. **Intelligent Workload Management → Load Shedding**을 **Enabled**로 설정합니다.
5. 변경 사항을 검토하고 적용합니다.
6. 설정에 표시되는 **Effective state**가 활성 상태인지 확인합니다. M10/M20이나 미지원 topology/version에서는 `Enabled`를 선택해도 실제 상태가 활성화되지 않을 수 있습니다.

Load Shedding을 켜면 클러스터가 지속적으로 과부하 상태일 때 작업 거절과 overload 오류가 발생할 수 있습니다. 따라서 드라이버와 앱의 오류 처리 준비를 먼저 확인하는 것을 권장합니다.

## 검증 및 운영 모니터링

* 운영 적용 전 비운영 환경이나 계획된 검증 시간에 Atlas의 **Test Resilience → Load Shedding** 테스트를 사용해 드라이버, 앱 오류 처리, 알림 및 대응 절차를 점검할 수 있습니다. 테스트는 검증 목적으로 작업을 거절할 수 있으므로 테스트 범위와 시간을 확인하세요.
* Atlas Metrics의 **Operation Rate Limiting**에서 queued operations와 rejected operations를 관찰합니다.
* `Sustained load shedding`, `Sustained ingress queue pressure`, `Sustained execution-control queue pressure` 관련 알림과 애플리케이션 오류율·재시도율·지연시간을 함께 살펴보세요.
* 거절이 지속되면 단순히 재시도 횟수만 늘리기보다 요청량, 재시도 정책, 쿼리/인덱스, 클러스터 용량 및 autoscaling을 함께 검토하세요.

### Atlas 네트워킹·암호화

* **Google Cloud PSC 포트 매핑:** 여러 Atlas 노드를 서로 다른 포트로 구분하면서 고객 측 IP 주소 하나를 공유할 수 있어, 기존 구성 대비 IP 주소 및 네트워크 리소스 사용량을 줄입니다. 보통 리전별 IP 사용량 절감으로 설명하는 것이 정확합니다. ([PSC 포트 매핑 설명](https://www.mongodb.com/company/blog/technical/port-mapping-for-google-private-service-connect-on-mongodb-atlas))
* **AWS PrivateLink 리전 간 연결:** 다른 AWS 리전에 있는 애플리케이션이 Atlas 클러스터에 private endpoint를 통해 사설 연결을 할 수 있습니다. 연결을 허용할 리전 구성을 확인해야 합니다. ([Atlas 변경 기록](https://www.mongodb.com/docs/atlas/release-notes/changelog/))
* **Azure Key Vault 저장 데이터 암호화(EAR) Secretless 인증:** Atlas 관리 Azure Service Principal을 승인해, 장기 보관 client secret 대신 Azure가 발급하는 단기 OAuth 2.0 토큰으로 Key Vault에 인증합니다. 이는 저장 데이터 암호화용 고객 관리 키의 인증 방식이며, Key Vault나 암호화 키 자체를 없애는 기능은 아닙니다. ([Secretless 인증 설정](https://www.mongodb.com/docs/atlas/security/azure-kms-secretless/))
* **발표 문구:** 세 항목은 MongoDB 9.0의 보안 연결 및 암호화 확장이라는 큰 주제에 해당합니다. 다만 Azure Key Vault 항목은 네트워킹 자체라기보다 암호화 키 접근 인증 개선입니다. MongoDB 공식 제품 블로그의 표현은 “expands secure connectivity and encryption capabilities”입니다. ([MongoDB 9.0 발표](https://www.mongodb.com/company/blog/product-release-announcements/mongodb-9-0-this-is-what-a-database-should-be))

### MongoDB Server 9.0

* **서버 측 JavaScript:** 8.0에서 deprecated였던 `$function`, `$accumulator`, `$where`가 9.0에서 WASM 기반 JavaScript 엔진을 사용해 더 이상 deprecated가 아닙니다. 단, `ppc64le`에서는 JavaScript 엔진이 포함되지 않아 사용할 수 없습니다. 또한 JavaScript의 local Date 연산은 호스트 시간대가 아니라 UTC로 처리되므로 업그레이드 전 동작을 점검하세요. ([9.0 릴리스 노트](https://www.mongodb.com/docs/manual/release-notes/9.0/), [호환성 변경사항](https://www.mongodb.com/docs/manual/release-notes/9.0-compatibility/))
* **`useInternalAuthzForX509`:** 9.0에 추가된 파라미터입니다. `true`이면 LDAP authorization이 설정되어 있어도 X.509 인증 클라이언트에 내부 authorization을 사용합니다. X.509 이외의 인증 방식에는 영향을 주지 않습니다. ([9.0 릴리스 노트](https://www.mongodb.com/docs/manual/release-notes/9.0/))
* **`usersInfo.authenticatedMechanism`:** 9.0에서 `usersInfo` 명령에 추가된 optional 문자열 **입력 필드**입니다. 조회 시 어떤 인증 메커니즘 및 authorization backend를 기준으로 사용자 정보를 확인할지 지정합니다. 응답에 추가되는 필드라고 표현하지 않도록 주의하세요. ([9.0 릴리스 노트](https://www.mongodb.com/docs/manual/release-notes/9.0/))
* **감사 로그 연결 필드:** MongoDB 감사 스키마에는 `directRemote`, OCSF 스키마에는 `direct_endpoint`가 추가되어 실제 TCP peer를 기록합니다. PROXY protocol header에 실린 클라이언트 주소 주장과 별도로 peer 주소를 확인할 수 있습니다. ([9.0 릴리스 노트](https://www.mongodb.com/docs/manual/release-notes/9.0/))
* **수신 BSON 검증:** 9.0부터 서버가 수신하는 BSON 객체를 기본 검증하며, `net.wireObjectCheck`와 관련 `--objcheck` / `--noobjcheck` 인수는 deprecated입니다. ([9.0 릴리스 노트](https://www.mongodb.com/docs/manual/release-notes/9.0/))




## 적용 전 체크리스트

* [ ] Atlas 클러스터가 MongoDB 9.0+ M30+ replica set인지 확인
* [ ] 앱이 사용하는 실제 드라이버 패키지 및 버전 확인
* [ ] 최소 버전 미만이면 드라이버 업그레이드 또는 앱 수준 overload 처리 계획 수립
* [ ] `SystemOverloadedError`와 재시도 가능 라벨을 구분해 처리
* [ ] 재시도 횟수, 총 대기 시간, jitter 및 중단 조건 설정
* [ ] Atlas metrics, 알림 및 운영 대응 담당자 확인
* [ ] 가능하면 운영 전 Load Shedding 테스트로 전체 경로 검증



## 공식 문서

* [Handle Overload Errors from Load Shedding](https://www.mongodb.com/docs/atlas/overload-errors/)
* [Intelligent Workload Management](https://www.mongodb.com/docs/atlas/intelligent-workload-management/)
* [Configure Additional Settings](https://www.mongodb.com/docs/atlas/cluster-additional-settings/)
* [Test Cluster Load Shedding](https://www.mongodb.com/docs/atlas/tutorial/test-resilience/test-load-shedding/)
* [Review Alert Conditions](https://www.mongodb.com/docs/atlas/reference/alert-conditions/)
