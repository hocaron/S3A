# 1장 카프카 개요
## 1.1 잘란도(Zalando)와 트위터(Twitter)의 카프카 도입 사례
: 기업의 카프카 도입의 당위성과 카프카 도입으로 얻을 수 있는 장점을 알아본다.

<br/>

### 1.1.1 유럽 최대 온라인 패션몰 잘란도의 도전 사례
- 2020년 기준 실사용자 수 3,100만 명, 연간 주문 수 1억 4,500만 건, 상품 판매 건수 50만 건 기록
- 점차 회사의 규모가 커지고 사업도 다각화되면서 내부적으로도 데이터에 대한 요구사항 증가
- 다양한 데이터 요구사항을 근본적으로 해결하기 위한 대책 강구
- 데이터의 변화가 스트림으로 컨슈머 측에 전달되는 이벤트 드리븐 시스템으로 전환 결정
- 데이터를 소비하는 컨슈머들은 자신의 요구사항에 따라 데이터를 처리하거나 구독 가능

<br/>

> **위기에 봉착한 잘란도**

: API와 PostgreSQL로 연결하는 CRUD 타입으로 구성, DB 업데이트 후 아웃바인드 이벤트 생성으로 구성

<br/>

**👍 데이터 오차 축소 성공**

**👎 동기화 방식에서 한계**

- 여러 네트워크를 이용하는 환경에서 모든 데이터 변경에 대한 올바른 전달 보장 문제
- 동일한 데이터를 동시에 수정하면서 정확하게 순서를 보장해야 하는 문제
- 수정된 이벤트들을 정확한 순서대로 아웃바인드 전송하는 문제
- 다양한 클라이언트들의 요구사항을 효율적으로 지원하기 어려운 문제
- 빠른 전송을 위한 클라이언트 또는 대량의 배치 전송을 위한 클라이언트를 지원하기 어려운 문제

<br/>

> **비동기 방식의 대표 스트리밍 플랫폼, 카프카 도입**

: 잘란도가 매력을 느낀 각 카프카 기능의 장점

<br/>

**👍 빠른 데이터 수집이 가능한 높은 처리량**

- HTTP 기반의 이벤트도 카프카로 처리되는 응답시간은 불과 한자릿수의 밀리초(ms) 단위
- 빠르고 안전한 데이터 전송이 가능
- 전보다 더 광범위한 데이터 흐름이 가능

**👍 순서 보장**

- 이벤트 처리 순서 보장
- 무수한 복잡성들이 제거되면서 구조가 간결

**👍 적어도 한 번 전송 방식**

- 멱등성 보장
- 복잡한 트랜잭션 처리 없이 재전송 처리가 가능하므로 아키텍처도 더욱 단순해지고 처리량 향샹

**👍 자연스러운 백프레셔 핸들링**

- 카프카 클라이언트는 풀(pull) 방식으로 동작
    - 자기 자신의 속도로 데이터 처리 가능
    - 간단하고 편리하게 클라이언트 구현 가능

**👍 강력한 파티셔닝**

- 논리적으로 토픽을 여러 개로 나눌 수 있다.
- 효과적인 수평 확장이 가능하다. (각 파티션들은 다른 파티션들과 관계없이 처리 가능)

**👍 그 외 여러 가지 기능**

- 스냅샷 역할 가능 (로그 컴팩션 기능)
- 새로운 애플리케이션이 나중에 메시지를 읽어가는 방식도 ok
- 애플리케이션의 병목 현상을 정확하게 파악 가능 (비동기 방식)
- 모니터링을 통해 지연에 대한 문제를 빠르게 해결 가능

<br/>

> **카프카로 도약의 기회를 얻은 잘란도**

- 내부의 데이터 처리 간소화
- 높은 처리량을 바탕으로 스트림 데이터 처리의 확장성 향상
- 주요 비즈니스 영역의 중요한 문제들을 해결하는 목적으로도 활용

<br/>

### 1.1.2 SNS 절대 강자 트위터의 카프카 활용 사례
: 당시 카프카의 상황

<br/>

> **카프카로 유턴한 트위터**

- 카프카의 불안정성으로 인해 인하우스 메시지 시스템(이벤트 버스) 구축
- 카프카가 큰 진전을 이룸 → 다시 카프카로 전환

<br/>

**👍 비용 절감 효과**

<img width="400" alt="image" src="https://github.com/mash-up-kr/S3A/assets/55437339/75269aae-31e0-475e-9ec3-aa6d76a4104d" />

🔼 트위터에서 측정한 이벤트 버스와 카프카의 성능 비교

- 성능적인 측면에서 메시지 처리량과 관계없이 이벤트 버스보다 카프카 응답 속도가 더 빠르다.
- 하나의 프로세스에서 스토리지와 요청을 모두 처리한다.
- 이벤트 버스에 비해 많은 하드웨어를 필요로 하지 않는다. → 리소스 60~70% 절감

**👍 강력한 커뮤니티**

- 카프카를 향상하고 버그를 수정하는 엔지니어들이 전 세계 수백 명에 이른다.
- 오픈소스 프로젝트로서 유명세를 탔기 때문에, 다양한 사용자들의 경험이 웹에 공유되어 있다.
- 대중화된 카프카를 채택하는 것이 카프카를 사용하는 새로운 데이터 엔지니어를 고용하기 쉽다.

<br/>

> **세계 유수 기업이 선택하는 카프카**

- 동기/비동기 데이터 전송에 대한 고민이 있는가?
- 실시간 데이터 처리에 대한 고민이 있는가?
- 현재의 데이터 처리에 한계를 느끼는가?
- 새로운 데이터 파이프라인이 복잡하다고 느끼는가?
- 데이터 처리의 비용 절감을 고려하고 있는가?

<br/>

---

## 1.2 국내외 카프카 이용 현황
: 얼마나 많은 기업에서 카프카를 사용하고 있을까?

<br/>

### 해외 카프카 이용 현황

<img width="400" alt="image" src="https://github.com/mash-up-kr/S3A/assets/55437339/18ff58b4-84b1-430f-a0cd-2e993fbdaabd" />

<br/><br/>

### 국내 카프카 이용 현황
IT 회사뿐만 아니라 쇼핑, 배송, 통신 등 많은 대기업에서 카프카를 검토하거나 이미 도입하여 운영

<br/>

---

## 1.3 카프카의 주요 특징
: 카프카의 기능 알아보기

<br/>

### 높은 처리량과 낮은 지연 시간

<img width="400" alt="img" src="https://github.com/mash-up-kr/S3A/assets/55437339/e20a94c2-b828-47db-bbc7-91b3a48cbb1f" />

🔼 메시징 시스템 간의 성능 비교
- 아파치 카프카, 펄사 메시징 시스템, 래빗 MQ 총 3가지 메시징 시스템의 비교
- 처리량과 응답 속도를 같이 비교했을 때 카프카가 압도적

<br/>

### 높은 확장성
- 카프카는 손쉽게 확장 가능하도록 설계되어 있다.

<br/>

### 고가용성
- 카프카는 고가용성을 갖추면서도 지연 없는 빠른 메시지 처리 기능을 유지한다. (꽤나 어려운 일!)

<br/>

### 내구성
- 프로듀서의 acks라는 옵션을 조정하여 메시지의 내구성 강화
- 카프카로 전송되는 모든 메시지는 카프카의 로컬 디스크에 안전하게 저장
- 메시지들은 2~3대의 브로커에 저장

<br/>

### 개발 편의성
- 프로듀서(producer)와 컨슈머(consumer)가 완벽하게 분리되어 독립적으로 동작한다. (편해진 개발 👍)
- Kafka Connect와 Schema Registry 제공
  - Schema Registry: 데이터를 파싱하는 데 많은 시간을 소모하는 비효율적인 현실을 보완하고자 스키마를 정의해서 사용할 수 있도록 개발된 애플리케이션
  - Kafka Connect: 카프카와 연동해 손쉽게 소스와 싱크로 데이터를 보내고 받을 수 있는 별도의 애플리케이션 (producer, consumer 개발X)
- 효율적인 장애 대응과 서비스 품질 개선이 가능해질 것

<br/>

### 운영 및 관리 편의성
- 카프카의 운영 및 관리 편의성, 안정적인 운영을 위한 모니터링 등
- 그라파나(Grafana) 등을 이용

<br/>

---

## 1.4 카프카의 성장

<img width="400" alt="image" src="https://github.com/mash-up-kr/S3A/assets/55437339/ecdc3457-779e-471e-aa57-237ba7fb0638" />

🔼 2011년 카프카의 탄생 ~ 2021.09까지의 주요 서비스 출시일

<br/>

### 리플리케이션 기능 추가(v0.8)
- 2013년 12월 버전 0.8 공개
- 내부 카프카 클러스터에서 브로커 장애가 발생해도 리플리케이션 기능으로 데이터 유실 없이 안정적으로 사용 가능

<br/>

### 스키마 레지스트리 공개(v0.8.2)
- producer와 consumer 간에 서로 데이터 구조를 설명할 수 있는 스키마를 등록 지정하여 사용 가능
- 스키마에 정의된 데이터만 주고받을 수 있다. (비정형 데이터를 파싱하는 데 소모하던 리소스 감소)

<br/>

### 카프카 커넥트 공개(v0.9)
- 2015년 11월 0.9 버전 발표
- 카프카 커넥트를 이용해 별도의 코드 작성 없이 다양한 프로토콜과 카프카 연동 가능

<br/>

### 카프카 스트림즈 공개(v0.10)
- 2016년 5월 버전 0.10 발표
- 실시간 처리에 대한 니즈를 충족시키는 목적
- 많은 개발자/기업에서의 실시간 분석, 처리 등이 가능

<br/>

### KSQL 공개
- 2017년 8월 단독 공개
- 스트림, 배치 처리 가능, 익숙한 SQL 언어로 처리 가능

<br/>

### 주키퍼 의존성에서 해방(v3.0)
- 주키퍼 없이 동작 가능한 카프카 공개
- 아직은 여전히 실제 운영 환경에서 사용하는 것을 추천하지 않는다고 함

<br/>

---

## 1.5 다양한 카프카의 사용 사례
: 카프카가 어떤 용도로 이용되는지 살펴보자.

<br/>

### 데이터 파이프라인: 넷플릭스 사례

<img width="400" alt="img" src="https://github.com/mash-up-kr/S3A/assets/55437339/4d580199-088d-4a86-887a-d4313dac1276" />

🔼 넷플릭스의 카프카 사용 사례
- 파이프라인을 연결해주는 역할로 카프카를 사용한다.
- 데이터 파이프라인을 통해 사용자의 경험을 예측해 능동적으로 대응할 수 있고, 오류 발생 시에도 실시간으로 대응이 가능하다.

<br/>

### 데이터 통합: 우버 사례

<img width="400" alt="img" src="https://github.com/mash-up-kr/S3A/assets/55437339/3b147a64-4d3b-4e7a-a5f0-6e3124c8db9a" />

🔼 우버의 카프카 사용 사례
- 카프카를 중심으로 데이터 통합이 일어나면서 카프카와 연결되는 시스템이 점점 다양해진다. (모든 데이터가 카프카를 오가는 구조)
- 카프카가 애플리케이션 분석, 디버깅, 알람 등으로 이용되고 있음을 알 수 있다.

<br/>

### 머신러닝 분야 활용 사례

<img width="400" alt="img" src="https://github.com/mash-up-kr/S3A/assets/55437339/435c53ca-da95-4456-8337-aa87db849a39" />

🔼 머신러닝 분야에서의 사용 사례
- 컨플루언트에서 공개한 머신러닝 분야에서의 카프카 사용 사례
- 데이터 전송과 파이프라인을 위해 카프카 커넥트 사용
- 모델 생성 또는 상용 머신러닝 앱의 실시간 처리를 위해서 카프카 스트림즈 사용
- 스키마 정의를 위한 스키마 레지스트리 사용
- SQL 기반 쿼리를 위한 KSQL 사용

<br/>

### 스마트 시티 분야 활용 사례

<img width="400" alt="img" src="https://github.com/mash-up-kr/S3A/assets/55437339/9b5d3130-c631-45b0-b41e-96530550453f" />

🔼 스마트 시티에서의 카프카 사용 사례
- 스마트 시티는 다양한 형태의 IoT 데이터를 수집해 정보를 얻고 이를 바탕으로 자산, 리소스, 서비스의 운용 효율을 높인다.
- 데이터를 수집, 분석, 활용하는 예
- 여러 도시에서 수집된 데이터를 분석하려면 중앙에 위치한 카프카로 모든 데이터를 전송
- 카프카와 카프카 사이의 안정적인 데이터 전송을 위해 카프카 커넥트 활용
- 데이터 변화 등에 유연하게 대응하기 위해서 스키마 레지스트리 등을 활용
