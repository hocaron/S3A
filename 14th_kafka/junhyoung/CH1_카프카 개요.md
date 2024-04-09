# 1장 카프카 개요

### 1.1  잘란도와 트위터의 카프카 도입 사례

잘란도는  🛍️온라인 쇼핑몰이며 무신사와 유사. 국내에서 무신사가 있다면 유럽에는 잘란도

신뢰성이 있는 데이터가 되기 위해선 인바운드 데이터와 아웃바운드 데이터가 ✔️일치해야한다.

### 과거에는..
초기에는 API와 DB로 연결하는 CRUD로 구성하여 DB 업데이트가 된 후 아웃바운드 이벤트가 생성 🔄

### 동기화문제
1. 여러 네트워크를 이용하는 환경에서 모든 데이터 변경에 대한 올바른 전달 보장 문제
2. 동일한 DB를 동시에 수정하면서 순서를 보장해야하는 문제, 수정된 이벤트들을 순서대로 아웃바운드 전송하는 문제
3. Client의 다양한 요구사항을 효율적으로 지원하기 어려운 문제
4. 빠른 전송을 위한 Client 또는 대량의 배치 전송을 위한 클라이언트를 지원하기 어려운 문제

- 동기 방식의 한계 -> 비동기 방식인 카프카 도입

### 잘란도가 카프카를 도입한 이유
1. 높은 처리량 📈
2. 순서 보장 🔢
3. 적어도 한 번 전송 방식 (at least once)
4. 강력한 파티셔닝 🧱
5. 자연스러운 백프레셔 핸들링
6. 로그 컴팬션 📚
7. ETC..

### 적어도 한 번(at least once) 전송 방식 
분산된 네트워크 환경에서 멱등성은 매우 중요함 🔍
~~~
멱등성(dempotent)이란?
동일한 작업을 여러 번 수행해도 결과가 달라지지 않는 것을 의미 (5.3 절에서 자세히 다룸)
~~~
적어도 한 번 전송방식만 사용하더라도 간혹 이벤트들이 중복 발생할 수는 있으나 누락 없는 재전송이 가능해 메시지 손실에 대한 걱정이 사라짐

### Kafka에서 파티셔닝이란?
~~~
큰 데이터 스트림을 작은 단위로 나누는 과정
~~~
각 파티션들을 다른 파티션들과 관계 없이 처리할 수 있어 수평 확장이 가능해짐
### 자연스러운 백프레셔 핸들링 (Natural Backpressure Handling)이란?
~~~
백프레셔란?
시스템의 처리 용량을 초과하는 데이터가 유입될 때 발생하는 현상
~~~
카프카의 Client는 pull 방식을 사용. 자기 자신의 속도로 데이터를 처리할 수 있어 성능과 편리함에 집중할 수 있음 🏃‍💨

### Kafka에서 **로그컴팩션**(log compaction)이란?
특정 토픽의 로그 파일 크기를 관리하기 위한 메커니즘
로그 컴팩션 기능을 통해 스냅샷 역할이 가능해졌음. 모니터링을 통해 지연에 대한 문제를 빠르게 해결할 수 있게됨

### 카프카는 BPS와 상관없이 지연이 거의 발생하지 않음
-> 이벤트버스는 서빙 레이어와 스토리지 레이어가 분리되어 있어 추가적인 홉이 필요하지만 카프카는 하나의 프로세스에서 스토리지와 요청을 모두처리하기 때문
![bps](https://raw.githubusercontent.com/mash-up-kr/S3A/master/14th_kafka/junhyoung/image/ch1/bps.png)


### 1.3 카프카의 주요 특징

![throughput](https://raw.githubusercontent.com/mash-up-kr/S3A/master/14th_kafka/junhyoung/image/ch1/throughput.png)

높은 처리량과 낮은 지연 시간
높은 확장성
고가용성
내구성
개발 편의성
운영 및 관리 편의성

### 1.4 카프카의 성장
![growthOfKafka.png](https://raw.githubusercontent.com/mash-up-kr/S3A/master/14th_kafka/junhyoung/image/ch1/growthOfKafka.png)

- 리플리케이션 기능 추가(v0.8)
- 스키마 레지스트리 공개(v0.8.2)
- 카프카 커넥트 공개(v0.9)
- 카프카 스트림즈 공개(v0.10)
- KSQL 공개
- 주키퍼 의존성에서 해방(v3.0)

### 1.5 다양한 카프카의 사용 사례
#### 데이터 파이프라인: 넷플릭스 사례
![Netflix Architecture](https://raw.githubusercontent.com/mash-up-kr/S3A/master/14th_kafka/junhyoung/image/ch1/netflix.png)

#### 데이터 통합: 우버 사례
![Uber Architecture](https://raw.githubusercontent.com/mash-up-kr/S3A/master/14th_kafka/junhyoung/image/ch1/uber.png)











