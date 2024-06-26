# 15.이벤트 주도 아키텍처 (EDA)

특징

- loose coupling
- 확장 가능성
- 내결함성

주의 : 부주의 적용 → 모듈식 모놀리스를 분산된 커다란 진흙덩어리로 만듦

정의 : 시스템 컴포넌트가 이벤트 메시지를 교환, 비동기적으로 서로 커뮤니케이션하는 아키텍처 스타일 

내부 동작 순서 : 

1. 이벤트 발행
2. 발생한 이벤트 다른 컴포넌트에서 구독
3. 이벤트에 따른 반응

ex_ SAGA 패턴

### 이벤트 주도 아키텍처와 이벤트 소싱

이벤트 소싱 - 상태 변화를 일련의 이벤트로 캡쳐하는 방법 (서비스 내부) 

둘 간의 차이점 : 

- EDA : 서비스 간 통신
- 이벤트 소싱 : 서비스 내부 발생, 이벤트는 서비스에서 구현된 상태 전환을 의미,
 비즈니스 도메인의 복잡성을 줄이기 위해

## 이벤트

### 이벤트, 커맨드, 메시지

메시지에는 두가지 유형 존재 

- 이벤트 : 이미 발생한 변화를 설명
- 커맨드 : 수행돼야 할 작업 설명

둘 다 비동기 통신 가능. but 커맨드는 거부당할 수 있음 

이벤트는 취소할 수 없음 (이미 과거) 

### 구조

이벤트 : 선택한 메시징 플랫폼을 사용하여 직렬화 및 전송할 수 있는 데이터 레코드 

구조 : 메타데이터 + 페이로드( 이벤트의 유형도 포함) 

### 이벤트 유형

- 이벤트 알림
- 이벤트를 통한 상태 전송
- 도메인 이벤트의 세가지 유형

1. 이벤트 알림 

다른 컴포넌트가 반응할 비즈니스 도메인의 변경에 관한 메시지 

오로지 알림이기 때문에 반응에 대한 필요한 모든 정보는 필요하지 않음 

따라서 추가적인 정보를 필요하다면, 수신 측에서 재질의 

이벤트 알림과 보안, 동시성 

보안적 측면 - 민감한 정보에 대해 공유되는 것을 차단, 구독자가 데이터 접근에 대한 추가 권한 확인 가능

동시성 - 비동기적인 특정으로 인해 이미 만료된 상태로 구독자에게는 렌더링될 수 있음 
     만약 race condition에 민감한 경우 명시적으로 질의 → 최신성 보장 

1. 이벤트를 통한 상태 전송

구독자에게 제공자의 내부 상태에 대한 변경사항 알림 

- 이벤트 알림과의 차이점  : 상태 변화에 대해 반영된 모든 데이터 포함
- 만약 큰 자료구조를 사용하여 기존 처리라면, 실제 수정된 필드만 ECST 메시지에 포함
- 만약 ECST 메시지에 완전한 스냅샷 포함 / 업데이트된 필드만 포함되는 경우, 
이벤트 스트림을 이용하여 엔티티 상태의 로컬 캐시 보유하고 이용 가능

1. 도메인 이벤트 
- 이벤트 알림과 ECST 메시지 사이 어딘가 쯤의 특성

이벤트 알림과의 관계 

- 도메인 이벤트는 이벤트를 설명하는 모든 정보를 포함
- 모델링의 의도는 비즈니스 도메인을 모델링하고 설명하기 위한 것 (이벤트 알림 - 컴포넌트간 연동 수월)

ECST과의 관계 

- ECST : 제공자 데이터를 로컬 캐시로 보유하기에 충분한 정보 제공
- 모든 도메인 이벤트는 풍부한 모델을 노출해서는 안된다.
    - 사용자가 구독하지 않은 다른 도메인 이벤트가 동일한 필드에 영향을 줄 수 있으므로, 
    특정 도메인 이벤트에 포함된 데이터 조차 애그리게이트의 상태를 캐싱하기에 충분하지 않음
- 모델링의 의도가 다름
    - 도메인 이벤트 : 수명 주기 동안 비즈니스 이벤트만 설명하고자 함 (애그리게이트의 상태를 설명하는 것이 아님)

## 이벤트 주도 연동 설계

EDA 시스템에서 이벤트 : 컴포넌트가 연동하는 방식과 컴포넌트의 경계 자체에 모두 영향을 주는 가장 중요한 설계 요소 

강하게 결합된 분산 시스템이 있다고 가정하자 

### 시간 결합

- 엄격한 실행 순서에 따라 달라지는 컨텍스트에서는 각각의 컴포넌트 내에 기반한 비즈니스 규칙에 의해 일관성 없는 결과를 불러 일으킬 수 있음

### 기능 결합

만약 하나의 바운디드 컨텍스트에서 사용하는 도메인 이벤트를 다른 곳에서도 복제하여 사용한다면, 
한 쪽의 변경으로 인해 다른 바운디드 컨텍스트 또한 변경사항을 복제해야함 

### 구현 결합

만약 하나의 바운디드 컨텍스트가 이벤트 소싱 모델로 모든 도메인 이벤트를 생성하고, 다른 바운디드 컨텍스트에서 해당 이벤트를 구독한다고 가정하자 
이 때 도메인 이벤트의 종류가 추가되거나 스키마가 변경된다면, 구독중인 다른 바운디드 컨텍스트에도 모두 반영되어야 한다. - 일관성 없는 데이터 문제로 이어질 수 있음 

### 이벤트 주도 연동의 리펙토링

구현 결합과 기능 결합 해결하기 
- 기존과 같은 모든 도메인 이벤트를 노출하는 것이 아닌, 더 제한된 이벤트 집합 / 다른 유형의 이벤트를 노출하도록 하여 구현 상세를 루즈 커플링하게 한다.
- 제공자의 프로젝션 로직을 캡슐화
    - 구현 상세를 노출하는 것이 아닌, 사용자 주도 컨트랙트 패턴을 사용
    - OHS를 통해 공표된 언어의 일부로 만들고, 사용자가 필요로 하는 모델을 프로젝션(EC
        - 사용자는 제공자의 구현 모델을 알지 못함

시간 결합 해결하기 
- 제공자 측에서는 이벤트 알림 메시지 발행 - 사용자 측에서 필요한 데이터를 가져오도록 트리거
- 이벤트 알림을 받으면 사용자 측에서는 상태를 가져온다

## 이벤트 주도 설계 휴리스틱

### 최악의 상황을 가정하라
늘 가정할 것
- 네트워크는 느려진다
- 가장 어렵거나 중요한 순간 서버 장애는 발생할 것
- 이벤트는 순서대로 도착하지 않음
- 이벤트는 중복된다.

> 만사 오케이란 생각은 버려라

해결법

- 메시지를 안정적으로 발송 : 아웃박스 패턴 사용
- 메시지를 발송할 때 구독자가 메시지 중복을 제거하고 순서가 잘못된 메시지를 식별하고 재정렬할 수 있도록 할 것
- 보상 조치를 실행해야 하는 교차 바운디드 컨텍스트 프로세스를 조율할 때는 사가 패턴과 프로세스 관리자 패턴을 이용할 것

### 퍼블릭 이벤트와 프라이빗 이벤트를 사용

- 이벤트 소싱 애그리게이트에서 도메인 이벤트 발송시, 세부 정보를 노출하지 않도록 주의
    - 이벤트를 바운디드 컨텍스트의 퍼블릭 인터페이스의 내재된 부분으로 취급할 것 (OHS - 공표된 언어)
- 바운디드 컨텍스트의 퍼블릭 인터페이스 설계시 다른 유형의 이벤트를 활용할 것
    - ECST는 구현 모델을 사용자가 필요로 하는 정보만 전달하는 더욱 간결한 모델로 압축할 것
    - 이벤트 알림 메시지를 사용하여 퍼블릭 인터페이스를 최소화 시키자
- 외부 바운디드 컨텍스트과의 연동 :  도메인 이벤트를 최소화하여 사용
    - 전용 퍼블릭 도메인 이벤트를 설계할 것

### 일관성 요구사항 평가

- 컴포넌트가 궁극적으로 일관된 데이터를 처리할 수 있는 경우 :
    - 이벤트를 통한 상태 전송 메시지 사용
- 사용자가 제공자의 마지막으로 변경된 상태를 읽어야하는 경우 :
    - 제공자의 최신 상태를 가져오는 후속 질의와 함꼐 이벤트 알림 메시지 발행


---
# 16. 

- OLTP(Online Transactional Processing) : 실시간 데이터 처리 시스템 구축에 사용, 내부적으로 시스템 데이터를 조작 및 작업에 대한 실시간 트랜잭션 구현
    - 비즈니스 도메인에 속한 다양한 엔티티를 중심으로 구축 및 모델들에 대한 수명주기를 구현하여 상호작용
    - 실시간 비즈니스 트랜잭션을 지원하도록 최적화
    - 가장 정밀한 데이터가 필요함
- OLAP(Online Analytical Processing) : 데이터 메시
    - 분석 데이터를 통한 통찰 획득 및 고객 요구사항 깊이감있게 파악, 기계학습 모델 훈련에 사용
    - 다양한 통찰 제공
    - 개별 비즈니스 엔티티를 무시, 팩트 테이블과 디멘전 테이블을 모델링하여 비즈니스 활동에 집중
    - 가장 큰 차이점 : 데이터 세분화 정도
        - OLAP - 분석 모델에서는 데이터를 취합하는 것이 다양한 유스케이스에 더욱 효과적

### 팩트 테이블

팩트 : 이미 발생한 비즈니스 활동 

- 도메인 이벤트와 유사하나, 과거 시제의 동사로 팩트를 명명하도록 요구 x
- 비즈니스 프로세스 활동을 나타냄
- 동작을 표현(동사)

팩트 테이블 : 

- 팩트 레코드는 삭제 및 수정 불가, 오로지 추가만 가능

### 디멘전 테이블

- 디멘전은 팩트를 묘사(형용사)
- 디멘전은 팩트의 속성을 설명하므로 팩트 테이블에 있는 키를 외부 키로 디멘전 테이블에 참조
- 고도로 정규화되어있음
    - 분석 시스템에서 유연한 질의를 지원해야 하므로 (질의 패턴 예측 불가)
    - 정규화를 통해 동적인 질의 및 필터링을 지원, 다양한 디멘전에 걸친 팩트 데이터에 대한 그룹화 지원

## 분석 모델

### 스타 스키마

- 팩트와 디멘젼 관계가 다대일 관계
- 각 디멘전 레코드는 여러 팩트에서 사용
- 단일 팩트의 FK는 각기 하나의 디멘전 레코드를 가리킴

![image](https://user-images.githubusercontent.com/27190617/233236997-6a8f8e8a-e4ff-44b8-9884-ee862e3995b2.png)
### 스노플레이크 스키마

- 디멘전은 여러 수준으로 구성
- 추가적인 정규화로 인해 스노플레키으 스키마는 더 작은 공간에 디멘전 데이터 저장 및 유지보수 용이
- 그러나 팩트 데이터 조회시 더 많은 디멘전 테이블을 조회해야 하므로 리소스 과필요

![image](https://user-images.githubusercontent.com/27190617/233237023-b52ad81c-931e-4bf0-9d8b-251bb24effd9.png)

두 스키마를 통해 모두 데이터 분석가가 비즈니스 성과 분석 / 통찰 / BI 리포트 생성

## 분석 데이터 관리 플랫폼

### 데이터 웨어하우스

DWH : 기업의 모든 실시간 데이터 추출(E) - 분석 모델로 변환(T) - 분석 지향 DB 적재(L) 

- 데이터는 실시간 데이터 처리 DB, 스트림 이벤트, 로그 등 다양한 원천에서 수집
- 변환 중 필터링 및 순서 조정, 병합 등 가능
- 변환 시 데이터 저장을 위한 임시 저장소가 필요할 수 있음(스테이징 영역)
- 데이터 웨어하우스 아키텍처 중심에는 엔터프라이즈 전반의 모델을 구축하는 목표가 있음
    - 모든 시스템에서 생성되는 데이터 묘사 및 분석할 수 있어야 함
    - 하지만 이는 작은 규모의 조직에서는 실용적이지 못함
    - 각각의 ML, 리포트 구축 등에 대한 과제에 대한 솔루션을 각각 처리하는 것이 더 효율적
    - 전사적인 모델 구축은 데이터 마트를 사용하여 해결 가능

![image](https://user-images.githubusercontent.com/27190617/233236811-94dce492-841d-45c9-a0c9-a8097c925b8d.png)
### 데이터 마트

데이터 마트 : 단일 비즈니스에 대해 별도로 정의된 분석 요구사항에 관련된 데이터를 저장하는 일종의 DB 


![image](https://user-images.githubusercontent.com/27190617/233236762-9e8b8308-1823-476f-9f30-df0cbab5cf70.png)

- ETL 마트가 직접 ETL 프로세스로 추출 / 데이터 웨어하우스에서 데이터를 추출하는 마트 방식 존재
- DWH에서 마트로 데이터 유입시, 여전히 데이터 웨어하우스에 대한 전사적 모델 정의가 필요한데,
- 이 때 대안으로 데이터 마트가 직접 가져오는 전담 ETL 프로세스를 구현할 수도 있음

데이터 웨어하우스 아키텍처의 또다른 어려운 점 : OLAP 시스템과 OLTP 시스템 간의 강결합을 만듦

- 직접 ETL 스크립트가 DB에 접근하여 데이터를 가져온다면, DB의 스키마 변경시 스크립트가 먹통이 된다.
- 이러한 문제는 데이터 레이크 아키텍처를 사용하여 해결한다.

### 데이터 레이크 아키텍처
![image](https://user-images.githubusercontent.com/27190617/233236894-88039075-a157-4f52-bf50-7d5db394e6a3.png)

- 마찬가지로 데이터 처리 시스템 혹은 스트림, 배치성으로 데이터를 유입
- 원본 형태 그대로 저장 (데이터 레이크는 무스키마)
- 분석 모델에 대한 변환은 추후에
- DE 와 BI 엔지니어는 데이터 레이크의 데이터를 이해하여 DWH 에 변환하여 저장할 ETL스크립트를 구현
- 사실상 ELTL 이다.
- 이를 통해 미래에 새로운 모델 추가 및 기존 원본 데이터 초기화 가능
- 하지만, 전체적인 시스템의 복잡성은 올라감, 여러 버전의 ETL 스크립트도 생김

### 데이터 웨어하우스와 데이터 레이크 아키텍처

- 두 접근 법 모두 규모가 커지면 너무나 많은 ETL 스크립트가 생긴다.
- 또한 OLTP 의 경계를 침범하게 되어 구현 상세에 의존성 생김
- OLAP 는 OLTP 의 도메인 전문가가 아니다. 따라서 나중에 몰라
- 이러한 한계에 대해 새롭게 나온 것이 데이터 메시


## 데이터 메시

데이터 메시 아키텍처 - 분석 데이터에 대한 모델과 소유 경계 정의 및 프로젝션 

네가지 핵심 원칙 

- 도메인 기준의 데이터 분리
- 제품 관점에서의 데이터 다루기
- 자율성 활성화
- 에코시스템 구축

### 도메인 기준의 데이터 분리

- 모놀리식 분석 모델 구축이 아닌, 원천 데이터에 분석 모델을 일치 시켜 데이터를 사용하므로 여러 분석 모델을 사용 (각각의 바운디드 컨텍스트 마다 분석 모델을 분해)
- 결과적으로 해당 모델을 소유한 팀이 분석 모델로 변환하는 책임을 가짐

### 제품 관점에서 데이터 다루기

- 바운디드 컨텍스트가 잘 정의된 출력 포트를 통해서만 분석 데이터를 제공
- 이 때 분석 데이터는 통상적인 퍼블릭 AP와 동일하게 취급해야한다.
    - 필요한 엔드포인트인 데이터 출력 포트를 쉽게 찾을 수 있어야 함
    - 분석 엔드포인트는 제공하는 데이터와 형식을 설명하는 잘 정의된 스키마를 가져야 함
    - 분석 데이터는 신뢰할 수 있어야 하고 다른 API와 마찬가지로 서비스 수준 계약을 정의하고 모니터링 실시
    - 분석 모델을 일반적인 API 처럼 버전 관리

데이터 메시의 가장 큰 관심사 : 데이터 품질에 대한 책임 

- 분산 데이터 관리 아키텍처의 목표 : 조직의 데이터 분석 요건을 충족할 수 있도록 작은 크기의 분석 모델이 엮일 수 있게 하는 것
- 사용자마다 여러 형태의 분석 데이터가 필요할 수 있음
    - 데이터 제품은 다양한 사용자의 요구사항을 충족하는 여러 형태의 데이터를 제공하도록 다양한 언어도 제공할 수 있어야 함

### 자율성 활성화

- 데이터 제품은 상호 운용이 가능해야함
- 플랫폼이 상호운용이 가능한 데이터 제품 구축, 실행, 유지보수에서 복잡성을 추상화할 필요가 있음
- 따라서 데이터 인프라스트럭처 플랫폼 팀이 있으면 좋다

### 에코시스템 구축

- 분석 데이터 도메인 관점에서 상호운용과 에코시스템을 가능하게 할 연합 거버넌스 기구 임명
- 해당 그룹은 정상적인 에코시스템을 보장하는 규칙을 정의할 책임을 가짐
    - 모든 데이터 제품과 그 인터페이스에 규칙 적용

### 데이터 메시와 도메인 주도 설계 엮기

데이터 메시 아키텍처의 네가지 원칙 

- 유비쿼터스 언어와 결과 도메인 지식은 분석 모델 설계를 위한 필수 요소
- 자신의 실시간 데이터 모델과 다른 모델로 바운디드 컨텍스트 데이터를 노출하는 것은 오픈 호스트 패턴
- CQRS 패턴은 동일한 데이터에 대한 여러 모델을 쉽게 생성해주므로, 실시간 데이터 모델을 통해 분석 모델로 변환한다.
- 분석 유스케이스를 구현하기 위해 다양한 바운디드 컨텍스트 모델을 묶는데, 실시간 데이터 모델을 위한 바운디드 컨텍스트 연동 패턴은 분석 모델에도 적용할 수 있다 (파트너십 패턴, 분리형 노선 …)
