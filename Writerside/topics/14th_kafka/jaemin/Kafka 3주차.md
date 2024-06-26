# Kafka 3주차

# 학습 목표

<aside>
💡 카프카의 내부 동작에 대해 학습해보자.
1. 리플리케이션에 대해 알아보자
2. 리더와 팔로워의 역할에 대해 알아보자
3. 리더에포크와 복구 동작에 대해 알아보자
4. 컨트롤러와 컨트롤러의 동작에 대해 알아보자
5. 로그와 로드 컴팩션에 대해 알아보자

</aside>

# 카프카 리플리케이션

카프카는 초기 설계 단계부터 중앙 데이터 허브로서 안정적인 서비스가 운영될 수 있도록 카프카 내부에서 리프리케이션이라는 동작을 통해 안정성을 갖추도록 설계되었다.

## 리플리케이션 동작 개요

카프카 클러스터를 구성해 `replication-factor` 를 통해 관리자가 지정한 수만큼의 리플리케이션을 가질 수 있다. N개의 리플리케이션이 있는 경우 N-1 까지의 브로커 장애가 발생해도 메시지 손실 없이 안정적으로 메시지를 주고받을 수 있다.

## 리더와 팔로워

리더는 리플리케이션 중 하나가 선정되며, 모든 읽기와 쓰기는 그 리더를 통해서만 가능하다. 즉, 프로듀서는 리더를 통해서만 메시지를 보내고 컨슈머 또한 해당 리더를 통해서만 메시지를 가져올 수 있다.

그렇다면 리더를 통해서만 메시지를 주고 받는 다면 리더를 제외한 다른 리플리케이션들 즉, 팔로워들은 그저 대기만 하는 것일까?

<aside>
⚠️ 아니다! 리더에 문제가 발생하거나 이슈가 있을 경우를 대비해 언제든지 새로운 리더가 될 준비를 해야한다. 따라서, 컨슈머가 토픽의 메시지를 꺼내 가는 것 처럼 지속적으로 파티션의 리더가 새로운 메시지를 받았는지 확인하고, 있다면 해당 메시지를 리더로부터 복제해야한다.

</aside>

### 복제 유지와 커밋

리더와 팔로워는 ISR(InSyncReplica)라는 논리적 그룹으로 묶여있다. 이렇게 별도의 그룹으로 나누는 이유는 기본적으로 해당 그룹 안에 속한 팔로워들만이 새로운 리더의 자격을 가질 수 있기 때문이다. 즉, 같은 ISR이 아닌 팔로워는 새로운 리더의 자격이 없다.

ISR 내의 팔로워들은 리더와의 데이터 싱크를 위해 지속적으로 리더의 데이터를 Pull하고, 리더는 ISR 내 모든 팔로워가 메시지를 받을 때까지 기다린다. 하지만, 팔로워가 여러가지 이유로 리더로부터 리플리케이션을 하지 못하는 경우 해당 팔로워에게 리더를 넘겨준다면 데이터의 정합성이나 메시지 손실 등의 문제가 발생할 수 있기에 파티션의 리더는 팔로워들이 리플리케이션 동작을 잘 하는지 확인하고 그렇지 않다면 해당 ISR에서 추방하는 역할 또한 진행해야 한다.

ISR 내에서 모든 팔로워에 복제가 완료되면, 리더는 내부적으로 커밋되었다는 표시를 하게 되는데 이때, 마지막 커핏 오프셋 위치를 **하이워터마크** 라고 부른다. 이때, 컨슈머는 커밋된 메시지만 읽어갈 수 있는데 그 이유는 메시지의 일관성을 유지하기 위해서다.

### 컨슈머가 커밋되지 않은 메시지를 읽을 수 있는 경우의 시나리오

> peter-test01 토픽이 있고, 파티션의 갯수는 1이고, replication-factor 3 기준이며, 리더 파티션에 이미 test message1, 2가 프로듀서에 의해 전달되었고 test message1은 커밋됐다는 기준
> 
1. 컨슈머 A는 peter-test01 토픽을 컨슘한다
2. 컨슈머 A는 peter-test01 토픽의 파티션 리더로부터 메시지를 읽어간다. 읽어간 메시지는 test message 1,2이다
3. peter-test01 토픽의 파티션 리더가 있는 브로커에 문제가 발생해 팔로워 중 하나가 새로운 리더가 된다
4. 프로듀서가 보낸 test message2 메시지는 아직 팔로워들에게 리플리케이션이 되지 않은 상태에서 새로운 리더로 변경됐으므로, 새로운 리더는 test message1 메시지만 갖고 있다.
5. 새로운 컨슈머 B가 peter-test01 토픽을 컨슘한다
6. 새로운 리더로부터 메시지를 읽어가고, 읽어간 메시지는 오직 test message1이다.

결과적으로 컨슈머 A와 B는 peter-test01 이라는 동일한 토픽의 파티션을 읽었지만, 컨슈머 A는 test message1과 test message2를 가져왔고 컨슈머 B는 test message1만 가져왔다. 이처럼 커밋되지 않은 메시지를 읽을 수 있다면 메시지 불일치 현상이 발생할 수 있다.

그렇다면 우리는 커밋된 위치를 어떻게 알 수 있을까? 모든 브로커는 커밋된 메시지를 유지하기 위해 로컬 디스크의 `replication-offset-checkpoint` 라는 파일에 마지막 커밋 오프셋 위치를 저장한다.

## 리더와 팔로워의 단계별 리플리케이션 동작

카프카로 향하는 수많은 메시지는 리더를 통해 읽고 쓰기 동작이 일어나는데 리더가 리플리케이션을 위해 팔로워들과 많은 통신을 주고받는다면 리더의 성능은 떨어질 것이다. 다른 메시징 큐 시스템인 래빗MQ의 트랜잭션 모드에서는 모든 미러(팔로워)가 메시지를 받았는지에 대한 ACK를 리더에게 리턴하는 방식으로 설계되었는데 카프카는 리더와 팔로워 간의 리플리케이션 동작을 처리할 때 서로의 통신을 최소화하게 설계해 성능을 높일 수 있었다. 이 방법에 대해 알아보자.

> peter-test01 토픽이 있고, 파티션의 갯수는 1이고, replication-factor 3 기준
> 
1. 프로듀서에 의해 파티션 리더가 오프셋이 0인 message1이라는 메시지를 갖고 있다 (리플리케이션 전)
2. 팔로워들은 리더에게 0번 오프셋 메시지 가져오기(fetch) 요청을 보낸 후 새로운 메시지 message1이 있다는걸 인지하고 message1 리플리케이션을 시작한다
3. 리더는 1번 오프셋의 위치에 새로운 메시지인 message2를 프로듀서로부터 받은 후 저장한다
4. 0번 오프셋에 대한 리플리케이션 동작을 마친 팔로워들은 리더에게 1번 오프셋에 대한 리플리케이션을 요청한다.
5. 팔로워들로부터 1번 오프셋에 대한 리플리케이션 요청을 받은 리더는 팔로워들의 0번 오프셋에 대한 리플리케이션이 성공했음을 인지하고, 오프셋 0에 대해 커밋 표시를 한 후 하이워터마크를 증가시킨다
6. 리더의 응답을 받은 모든 팔로워는 0번 오프셋 메시지가 커밋되었다는 사실을 인지하고 리더와 동일하게 커밋을 표시한다.

만약 팔로워들이 0번 오프셋에 대한 리플리케이션 요청을 하는 경우 0번 오프셋에 대한 리플리케이션이 실패했다고 인지할 수 있다. 즉, 리더는 팔로워들이 보내는 리플리케이션 요청의 오프셋을 보고, 팔로워들이 어느 위치의 오프셋까지 리플리케이션을 성공했는지를 인지할 수 있다.

카프카는 이처럼 ACK 동작을 제거하고 리더와 팔로워간의 리플리케이션 방식을 `Push` 가 아닌 `Pull` 을 기용해 리더의 부하를 줄일 수 있었다.

## 리더에포크와 복구

리더에포크는 카프카의 파티션들이 복구 동작을 할 때 메시지의 안정성을 유지하기 위한 용도로 이용된다. 리더에포크는 컨트롤러에 의해 관리되는 32비트의 숫자로 표현되며 해당 정보는 리플리케이션 프로토콜에 의해 전파되고, 새로운 리더가 변경된 후 변경된 리더에 대한 정보는 팔로워에게 전달된다. 리더에포크는 복구 동작 시 하이워터마크를 대체하는 수단으로도 활용된다. 

### 리더에포크를 사용하지 않은 장애 복구 과정

> partition = 1, replication-factor = 2, min.insync.replicas = 1로 리더에포크가 없다는 가정하에 장애 복구 과정
> 
1. 리더는 프로듀서로부터 message1 메시지를 받았고, 0번 오프셋에 저장, 팔로워는 리더에게 0번 오프셋에 대한 가져오기 요청을 한다.
2. 가져오기 요청을 통해 팔로워는 message1 메시지를 리더로부터 리플리케이션한다
3. 리더는 하이워터마크를 1로 올린다
4. 리더는 프로듀서로부터 다음 메시지인 message2를 받은 뒤 1번 오프셋에 저장한다
5. 팔로워는 다음 메시지인 message2에 대해 리더에게 가져오기 요청을 보내고, 응답으로 리더의 하이워터마크 변화를 감지하고 자신의 하이워터마크도 1로 올린다
6. 팔로워는 1번 오프셋의 message2 메시지를 리더로부터 리플리케이션한다
7. 팔로워는 2번 오프셋에 대한 요청을 리더에게 보내고, 요청을 받은 리더는 하이워터마크를 2로 올린다
8. 팔로워는 2번 오프셋인 message2 메시지까지 리플리케이션을 완료했지만 아직 리더로부터 하이워터마크를 2로 올리는 내용은 전달받지 못한 상태이다.
9. 예상하지 못한 장애로 팔로워가 다운된다.

### 장애에서 복구된 팔로워는 카프카 프로세스가 시작되면서 내부적으로 메시지 복구 과정을 하게 된다

1. 팔로워는 자신이 갖고 있는 메시지들 중에서 자신의 워터마크보다 높은 메시지들은 신뢰할 수 없는 메시지로 판단하고 삭제한다. 따라서 1번 오프셋의 message2는 삭제된다
2. 팔로워는 리더에게 1번 오프셋의 새로운 메시지에 대한 가져오기 요청을 보낸다
3. 이 순간 리더였던 브로커가 예상치 못한 장애로 다운되면서, 해당 파티션에 유일하게 남아 있던 팔로워가 새로운 리더로 승격된다

팔로워가 새로운 리더로 승격된 후 기존의 리더는 1번 오프셋의 message2를 갖고 있었지만, 팔로워는 message2 없이 새로운 리더로 승격됐다. 결국 뉴리더는 message2를 갖고 있지 않다. 리더와 팔로워간의 리플리케이션이 있음에도 불구하고 리더가 변경되는 과정을 통해 최종적으로 1번 오프셋의 message2 메시지가 손실된 것이다.

### 팔로워가 장애 막 복구된 이후 리더에포크를 사용해 복구하는 과정

1. 팔로워는 복구 동작을 하면서 리더에게 리더에포크 요청을 보낸다
2. 요청을 받은 리더는 리더에포크의 응답으로 `1번 오프셋의 message2까지` 라고 팔로워에게 보낸다
3. 팔로워는 자신의 하이워터마크보다 높은 1번 오프셋의 message2를 삭제하지 않고, 리더의 응답을 확인한 후 message2까지 자신의 하이워터마크를 상향 조정한다

이런 과정을 통해 리더에포크 없이 장애 복구한 경우 자신의 하이워터마크보다 높은 메시지를 삭제한 것에 반해 리더에포크 요청을 통해 팔로워의 하이워터마크를 올릴 수 있었고, 메시지 손실은 발생하지 않았다.

### 리더에포크를 적용하지 않았을 때 발생할 수 있는 문제2

> 같은 파티션, 리플리케이션 조건에 리더만 오프셋1까지 저장하고 팔로워는 아직 1번 오프셋 메시지에 대해 리플리케이션 동작을 완료하지 못한 상태에서 해당 브로커들의 장애가 발생해 리더와 팔로워 모두 다운됐다고 가정
> 

브로커가 모두 종료된 후 팔로워가 있던 브로커만 장애에서 복구된 상태에서의 복구 과정

1. 팔로워였던 브로커가 장애에서 먼저 복구된다
2. peter-test01 토픽의 0번 파티션에 리더가 없으므로 팔로워는 새로운 리더로 승격된다
3. 새로운 리더는 프로듀서로부터 다음 메시지 message3를 전달받고 1번 오프셋에 저장한 후 자신의 하이워터마크를 상향 조정한다.
4. 구 리더였던 브로커가 장애에서 복구되었다
5. peter-test01 토픽의 0번 파티션에 이미 리더가 있으므로, 복구된 브로커는 팔로워가 된다
6. 리더와 메시지 정합성 확인을 위해 자신의 하이워터마크를 비교해보니 리더의 하이워터마크와 일치하므로, 브로커는 자신이 갖고 있던 메시지를 삭제하지 않았다
7. 리더는 프로듀서로부터 message4 메시지를 받은 후 오프셋2의 위치에 저장한다
8. 팔로워는 오프셋2인 message4를 리플리케이션하기 위해 준비한다

결과적으로 모든 리플리케이션은 동일한 하이워터마크를 나타내고 있지만 서로의 메시지가 다르다. 리더와 팔로워가 메시지의 동일한 오프셋 위치만으로 복구를 진행하면 메시지 불일치가 발생할 수 있다. 

### 리더에포크를 이용한 복구과정

> 팔로워가 먼저 복구되어 뉴리더가 되었고 구 리더였던 브로커가 장애에서 복구된 상태에서 진행
> 
1. 구 리더였던 브로커가 장애에서 복구된다
2. peter-test01 토픽의 0번 파티션에 이미 리더가 있고 자신은 팔로워가 된다
3. 팔로워는 뉴리더에게 리더에포크 요청을 보낸다
4. 뉴리더는 0번 오프셋까지 유효하다고 응답한다
5. 팔로워는 메시지 일관성을 위해 로컬 파일에서 1번 오프셋인 message2를 삭제한다
6. 팔로워는 리더로부터 1번 오프셋인 message3을 리플리케이션 하기 위해 준비한다

즉, 리더에포크가 없는 경우에 메시지 손실 및 메시지 불일치가 일어날 수 있으며 카프카는 이를 해결하기 위해 리더에포크를 도입하였다

# 컨트롤러

카프카 클러스터는 리더가 다운된 경우 같은 ISR에 있는 팔로워 중에 리더를 선출하는데 이 역할을 하는 `컨트롤러` 에 대해 알아보자. 리더를 선출하기 위한 ISR 리스트 정보는 안전한 저장소에 보관되어 있어야 하는데 가용성 보장을 위해 주키퍼에 저장되어 있다. 

### 컨트롤러가 브로커의 실패를 감지하고 리더를 선출하는 과정

1. 컨트롤러는 브로커가 실패하는 것을 예의주시한다
2. 브로커의 실패가 감지되면 즉시 ISR 리스트 중 하나를 새로운 파티션 리더로 선출한다
3. 새로운 리더의 정보를 주키퍼에 기록한다
4. 변경된 정보를 모든 브로커에게 전달한다

리더가 다운됐다는 것은 결국 해당 파티션에 대한 읽기/쓰기 기능이 마비되었음을 의미한다. 이렇게 기능이 마비된 경우 클라이언트는 자체적인 재시도를 진행하는데 이 재시도 시간 내에 빠르게 리더 선출 작업을 마쳐야 한다

즉, 리더 선출 과정은 컨트롤러에 의해 이뤄지며 파티션이 하나인 경우 컨트롤러가 새로운 리더를 선출하고 다른 모든 브로커들에게 업데이트 정보를 전파한다. 파티션이 1개인 경우에는 빠르게 작업이 완료되지만 파티션이 1만 개일 경우 많은 작업 소요시간이 걸릴 수 있다. 주키퍼는 최신 버전에서 이를 해결하기 위해 불필요한 로깅을 없애고 주키퍼 비동기 API를 반영해 리더 선출 소요 시간을 크게 줄였다. 

### 제어된 종료에서 컨트롤러가 리더를 선출하는 과정

1. 관리자가 브로커 종료 명령어를 실행하고, SIG_TERM 신호가 브로커에게 전달된다
2. SIG_TERM 신호를 받은 브로커는 컨트롤러에게 알린다
3. 컨트롤러는 리더 선출 작업을 진행하고 해당 정보를 주키퍼에 기록한다
4. 컨트롤러는 새로운 리더 정보를 다른 브로커들에게 전송한다
5. 컨트롤러는 종료 요청을 보낸 브로커에게 정상 종료한다는 응답을 보낸다
6. 응답을 받은 브로커는 캐시에 있는 내용을 디스크에 저장하고 종료한다.

### 제어된 종료와 급작스러운 종료의 차이

1. 제어된 종료의 경우 다운타임을 최소화 할 수 있다
    1. 브로커가 종료되기 전, 컨트롤러는 해당 브로커가 리더로 할당된 전체 파티션에 대해 리더 선출 작업을 진행하기 때문
    2. 제어된 종료라 할지라도 리더 선출 작업 시간동안에는 일시적인 다운타임이 발생하지만 리더 선출 작업 대상 파티션들의 리더들이 활성화된 상태에서 순차적으로 리더를 선출하기에 이를 최소화할 수 있다
    3. 브로커 장애로 인한 리더 선출 작업의 경우 파티션들의 다운타임은 새로운 리더 선출 작업이 될 때까지 지속되기에 첫 번째 파티션의 다운타임은 길지 않지만 마지막 대상 파티션의 다운타임은 꽤 오랜 시간 걸릴것이다
2. 제어된 종료의 경우 로그 복구 시간이 짧다

즉, 다양한 장점이 있는 제어된 종료를 사용하는걸 권장한다. 브로커의 설정파일인 server.properties에 `controlled.shutdown.enable=true` 를 사용해 제어된 종료를 사용하자

# 로그(로그 세그먼트)

카프카의 토픽으로 들어오는 메시지는 세그먼트라는 파일에 저장되는데 이 메시지는 정해진 형식을 맞추어 순차적으로 로그 세그먼트 파일에 저장된다. 로그 세그먼트에는 메시지의 내용뿐 아니라 메시지의 키, 밸류, 오프셋, 메시지 크기같은 정보가 함께 저장되며 브로커의 로컬 디스크에 보관된다.

하나의 로그 세그먼트 크기가 너무 크면 파일을 관리하기 어렵기에 로그 세그먼트의 최대 크기는 1GB를 기본값으로 설정되어 있다. 로그 세그먼트가 1GB 보다 커지는 경우에는 기본적으로 롤링전략을 적용해 해당 크기에 도달하면 해당 세그먼트 파일을 클로즈하고, 새로운 로그 세그먼트를 생성한다.

롤링 전략뿐 아니라 로그 세그먼트 파일이 무한히 늘어날 경우를 대비해 로그 세그먼트를 관리하는 방법에 대해 알아보자

## 로그 세그먼트 삭제

로그 세그먼트 삭제 옵션은 브로커의 설정 파일인 `server.properties` 에서 `log.cleanup.policy = delete` 로 명시되어야 한다. 해당 값은 기본값으로 관리자가 따로 옵션을 지정하지 않았다면 삭제 정책이 적용된다.

더불어 `retention.ms` 옵션을 조정해 메시지를 삭제하는 주기를 변경할 수 있다.

로그 세그먼트의 삭제 작업은 일정 주기를 가지고 체크하는데, 카프카의 기본값은 5분 주기이므로 5분 간격으로 로그 세그먼트 파일을 체크하면서 삭제 작업을 수행한다. 실제로 실행해보면 이전 로그 세그먼트 파일이 삭제되고 오프셋 시작 번호를 이용한 파일 이름이 생성된 것을 확인할 수 있다.

`retention.ms` 옵션을 사용하지 않는다면 기본적으로 7일을 주기로 세그먼트 파일을 삭제하게 된다.

## 로그 세그먼트 컴팩션

컴팩션은 로그 세그먼트 관리 정책 중 하나로 로그를 삭제하지 않고 컴팩션하여 보관할 수 있다. 로근 컴팩션은 기본적으로 세그먼트를 대상으로 실행되며 현재 활성화된 세그먼트를 제외하고 나머지 세그먼트들을 대상으로 컴팩션이 실행된다.

컴팩션할지라도 메시지를 모두 무기한 보관한다면 용량이 감당할 수 없이 커질 수 있으므로 카프카에서 로그 세그먼트를 컴팩션하면 메시지의 키값을 기준으로 마지막의 데이터만 보관하게 된다.

로그 컴팩션 기능을 이용하는 대표적인 예제는 카프카의 `__consumer_offset` 토픽인데 이 토픽은 카프카의 내부 토픽으로, 컨슈머 그룹의 정보를 저장하는 토픽이다. 각 컨슈머 그룹의 중요한 정보는 컨슈머 그룹이 어디까지 읽었는지를 나타내는 오프셋 커밋정보인데 해당 토픽에 키(컨슈머 그룹명, 토픽명), 밸류(오프셋 커밋 정보) 형태로 메시지가 저장된다. 컨슈머 그룹은 항상 마지막으로 커밋된 오프셋 정보가 중요하므로 과거에 커밋된 정보들은 삭제돼도 무방하다.

이처럼 로그 컴팩션은 과저 정보가 중요하지 않고 가장 마지막 값이 필요한 경우에 사용한다. 또 다른 예로는 현재의 구매 현황 상태를 보여주는 시스템에서도 로그 컴팩션을 이용할 수 있다. 이때 고유한 사용자 아이디를 키값으로, 현재의 구매 상태 정보를 메시지의 밸류값으로 한다면 구매한 사용자 아이디를 기준으로 최종 상태만 사용자에게 노출하면 되므로 로그 컴팩션 기능을 활용할 수 있다.

로그 컴팩션은 메시지 키를 통해 기능을 활용함으로 로그 컴팩션을 사용할 경우 메시지의 키를 필수적으로 보내야 한다

### 로그 컴팩션의 특징

- 빠른 장애 복구
    - 전체 로그 복구가 아닌 메시지의 키를 기준으로 최신의 상태만 복구하기 때문
- 과도한 입출력 부하
    - 로그 컴팩션 작업이 실행되는 경우 과도한 입출력 부하가 발생할 수 있으니 꼭 필요한 경우에만 사용하자!

### 로그 컴팩션 관련 옵션

| 옵션 이름 | 옵션값 | 적용범위 | 설명 |
| --- | --- | --- | --- |
| cleaup.policy | compact | 토픽의 옵션으로 적용 | 토픽 레벨에서 로그 컴팩션을 설정할 때 적용 |
| log.cleanup.policy | compact | 브로커의 설정 파일에 적용 | 브로커 레벨에서 로그 컴팩션을 설정할 때 적용 |
| log.cleaner.min.compaction.lag.ms | 0 | 브로커의 설정 파일에 적용 | 메시지가 기록된 후 컴팩션하기 전 경과되어야 할 최소시간을 지정. 이 옵션을 설정하지 않으면, 마지막 세그먼트를 제외하고 모든 세그먼트를 컴팩션한다 |
| log.cleaner.max.compaction.lag.ms | 9223372036854775807 | 브로커의 설정 파일에 적용 | 메시지가 기록된 후 컴팩션하기 전 경과되어야 할 최대 시간을 지정 |
| log.cleaner.min.cleanable.ratio | 0.5 | 브로커의 설정 파일에 적용 | 로그에서 압축이 되지 않은 부분을 더티라고 표현한다. ‘전체 로그’ 대비 ‘더티’의 비율이 50%가 넘으면 로그 컴팩션이 실행된다 |