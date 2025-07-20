### 4-1. 카프카 컨슈머: 개념

Consumer는 Publisher-Subscriber Model 에서 Subscribe 역할을 담당하는 클라이언트.

Consumer는 Broker에 저장된 메시지를 가져오는 역할을 수행.

### 4-2. 컨슈머 그룹

![image.png](attachment:ef0aa86a-20f4-49ba-a3a7-cd4ad2e824f2:image.png)

<aside>
💡

**파티션 할당 원칙**

- 동일한 컨슈머 그룹 내에서는 하나의 파티션이 오직 하나의 컨슈머에게만 할당된다.
    - 즉, 동일한 컨슈머 그룹 내에 컨슈머를 늘리면 하나의 토픽에 대해 여러 파티션을 병렬적으로 수행가능
    - 파티션 개수를 미리 크게 잡아놓고 컨슈머 그룹 내에 컨슈머를 늘리는 방식이 일반적인 규모 확장 방식이다.
- 하나의 파티션은 특정 Consumer Group 안에서 반드시 1개의 Consumer에게 할당되어야 합니다.
- 컨슈머 수 > 파티션 수인 경우, 일부 컨슈머는 유휴 상태
    - TCP Connection을 불필요하게 낭비
- 컨슈머 수 < 파티션 수인 경우, 하나의 컨슈머가 여러 파티션을 담당
</aside>

<aside>
💡

**만약 컨슈머 그룹이 여러개라면?**

Kafka에서는 특정 Partition에 대한 Offset 관리를 Consumer Group 단위로 수행한다.

즉, Consumer Group이 여러 개라면, 서로 독립적인 Offset를 갖는다.

이 말은 동일한 메세지를 똑같이 받는것을 의미한다.

![image.png](attachment:0b7e34bf-3edb-459b-bc7d-0a62878abeaa:image.png)

출처: https://devocean.sk.com/community/detail.do?ID=165478&boardType=DEVOCEAN_STUDY&page=1

**동일한 메세지를 받으면 똑같은 처리하는게 아닌가?**

동일한 메시지를 받지만 각 서비스마다 완전히 다른 일을 하기 때문

예를들어, **"사용자가 상품을 주문했다"** 라는 하나의 이벤트가 발생했을 때

```json
{
"orderId": "ORDER-123",
"userId": "user456",
"productId": "PROD-789",
"quantity": 2,
"price": 50000
}
```

**각 서비스가 이 동일한 메시지로 하는 일**

- **주문 서비스(컨슈머 그룹 A)**
    - 주문 상태를 "결제 대기"로 업데이트
    - 결제 시스템에 결제 요청
- **재고 서비스 (컨슈머 그룹 B)**
    - 해당 상품의 재고 2개 차감
    - 재고 부족시 알림
- **알림 서비스 (컨슈머 그룹 C)**
    - 고객에게 "주문 접수 완료" SMS/이메일 발송
    - 판매자에게 "새 주문" 알림
- 목표
    - **느슨한 결합**: 각 서비스가 독립적으로 동작
    - **장애 격리**: 한 서비스 장애가 다른 서비스에 영향 안 줌
</aside>

---

### **4-3. 리밸런싱 (Rebalancing)**

컨슈머 그룹 내에 컨슈머가 추가/제거되면 파티션 재할당이 일어납니다.

- [ ]  4.0.0 부터 리벨런싱을 컨슈머그룹안이 아닌, 카프카 자체에서 한다.

<aside>
💡

**리벨런싱 발생할 수 있는 상황**

- Consumer Group 내에 컨슈머가 **생성 혹은 삭제**된 경우
- [**`*max.poll.interval.ms*`**](https://www.notion.so/4-22e24902e505802bb832f1b23fc9fb13?pvs=21)로 설정된 시간 내에 poll() 요청을 보내지 못한 경우
- [**`*session.timeout.ms*`**](https://www.notion.so/4-22e24902e505802bb832f1b23fc9fb13?pvs=21)로 설정된 시간 내에 하트비트를 보내지 못한 경우
</aside>

<aside>
💡

**리벨런싱을 해야할 때, 컨슈머는 파티션 할당을 어떻게 결정하는 가?**

- 컨슈머그룹 리더가 파티션 할당 전략 실행
- 파티션 할당 전략
    - **레인지 파티션 할당 전략**
        - 컨슈머가 받아야 할 파티션 수를 결정하는데, 이는 해당 토픽의 전체 파티션 수를 컨슈머 그룹의 총 컨슈머 수로 나눈 값
        - 컨슈머 간 불균형 발생 가능 (첫 번째 컨슈머들이 더 많은 파티션을 받을 수 있음)
    - **라운드 로빈 파티션 할당 전략**

      ![image.png](attachment:061a0323-f3db-4ac0-b3a0-99862ac2ad99:image.png)

        - 모든 파티션을 순환하며 컨슈머에게 하나씩 할당
        - 리밸런싱이 발생할 때 모든 컨슈머가 중단
        - 또한 모든 파티션을 균등하게 분배하려 하기 때문에 하나의 컨슈머만 다운되더라도 모든 컨슈머의 리밸런싱이 발생하는 단점
        - 리밸런싱이 발생할 때 되도록 기존 할당된 파티션이 할당이 안된다는 단점
    - **스티키 파티션 할당 전략**
        - 기존 매핑 정보를 이용하여 원래의 매핑 정보를 최대한 지키려고 노력
        - 즉, 기존의 매핑 정보가 존재하는 경우를 먼저 할당하며, 매핑 정보가 일치하지 않는 케이스에 대해서 균등하게 파티션을 재분배
    - 위 3개 전략은 모두 Rebalancing Protocol이 EAGER
    - EAGER란, 리밸런싱이 발생할 때 모든 컨슈머는 메시지를 구독할 수 없는 Stop the world 현상이 발생
    - **따라서, 필요한 파티션에 대해서만 리밸런싱을 점진적으로 하는 방식인 COOPERATIVE 사용**
</aside>

**협력적 리밸런싱 (Kafka 3.1 이후 기본값)**

일부 파티션만 컨슈머 간에 이동합니다. 리밸런싱에 참여하지 않는 컨슈머들은 메시지 처리를 계속할 수 있습니다. 이전 방식(eager rebalance)과 달리 stop-the-world가 발생되지 않습니다.

**협력적 리밸런싱 과정**

![image.png](attachment:65d84ea3-fd9a-4631-b8d2-f7d5e9191f66:image.png)

출처: https://www.conduktor.io/kafka/consumer-incremental-rebalance-and-static-group-membership/

1. 컨슈머 그룹 리더가 모든 컨슈머에게 일부 파티션 소유권 해제를 알립니다.
2. 해당 컨슈머들이 파티션 소유권을 포기합니다.
3. 코디네이터가 해제된 파티션을 새로운 컨슈머에게 할당합니다.

<aside>
💡

**코디네이터란?**

- 컨슈머 그룹을 관리하는 브로커
</aside>

**실습**

![image.png](attachment:970d19a9-e832-4dbb-80c8-156c2f2788db:image.png)

- 현재, 모든 파티션과 Consumer Group 내의 컨슈머가 1:1으로 매칭된 상태
- `console-consumer-e905d3ed-adac-438c-a3b1-6974e54f04e7` 을 종료

**종료 후, 로그에 찍힌 내용**

```json
[Group my-consumer-group] Member console-consumer-e905d3ed-adac-438c-a3b1-6974e54f04e7 has left group through explicit `LeaveGroup` request; client reason: the consumer is being closed

`my-consumer-group`에서 한 컨슈머가 정상적으로 그룹을 떠났습니다. (이때, 코디네이터가 리더에게 리벨렌싱 시작한다고 알려줌)
```

```json
Preparing to rebalance group my-consumer-group in state PreparingRebalance with old generation 5

리밸런싱이 시작됩니다. 기존 세대(generation) 5에서 새로운 세대로 넘어갑니다.
```

```json
Stabilized group my-consumer-group generation 6 with 2 members.

리밸런싱이 완료되어 새로운 세대 6이 되었고, 이제 2명의 멤버가 남았습니다.
```

- [ ]  세대가 뭘까?

```json
Assignment received from leader console-consumer-531964aa-19a4-4080-9740-b4fef26f88e9 for group my-consumer-group for generation 6. The group has 2 members

리더 컨슈머가 **파티션 할당을 결정**했고, 2명의 멤버에게 파티션이 재분배되었습니다.
```

---

### 4-4. 정적 그룹 멤버십

컨슈머에게 고정된 ID를 부여해서 불필요한 리밸런싱을 방지하는 기능

기본방식

```json
컨슈머가 재시작할 때마다:
1. 기존 member-id 만료
2. 새로운 member-id 생성 
3. 새로운 멤버로 인식
4. 리밸런싱 발생

예시:
재시작 전: consumer-group-1-abc123
재시작 후: consumer-group-1-def456 (완전히 다른 ID)
```

정적 멤버십

```json
컨슈머가 재시작해도:
1. 동일한 group.instance.id 사용
2. 기존 멤버로 인식
3. 기존 파티션 할당 유지
4. 리밸런싱 없음

예시:
재시작 전: group.instance.id = "consumer-node-1"
재시작 후: group.instance.id = "consumer-node-1" (동일)
```

- 장점: 각 컨슈머에 캐싱했던 파티션 내용을 유지할때 편리
- 단점: 만약 매핑됐던 컨슈머가 종료되었을 때, 일정 기간 동안 어떤 컨슈머도 메세지를 읽지 않기 때문에, 매핑된 컨슈머가 다시 생성되면, 그때서야 밀린 메세지들을 처리하게 된다.
    - 만약 진짜 종료된거면? → `session.timeout.ms` 시간이 지나면, 그때서야 파티션 재할당

---

### 4-5. 카프카 컨슈머 생성하기

```java
@Slf4j
@Component
@Profile({"batch", "local"})
@RequiredArgsConstructor
class SlackAlarmEventListener {

    private final SlackMessageService slackMessageService;

		...
    @KafkaListener(topics = SLACK_TOPIC, groupId = "slack-consumer", containerFactory = "messageEventContainerFactory")
    public void onSlackEvent(SlackMessageEvent event) {
        slackMessageService.sendMessageToUser(event);
    }
}
```

- [ ]  concurrency 옵션을 넣으면 파티션하나당 싱글스레드로 동작 +

- groupID: 컨슈머 그룹을 지정
- topics: slack-consumer라는 컨슈머 그룹이 SLACK_TOPIC을 구독한다는 의미
- messageEventContainerFactory: 역직렬화 설정

---

### **4-6. 폴링 루프 poll()**

- 주기적으로, 할당 받은 파티션에서 메시지를 가져오는 역할을 수행하는 메서드

동작원리

![image.png](attachment:4bf94219-5b14-450e-92d6-6ce78f36c0b6:image.png)

출처: https://d2.naver.com/helloworld/0974525

- HeartBeat 스레드는 poll 메서드 호출 시 ConsumerCoordinator에 의해 생성되고
- KafkaConsumer와는 별도의 스레드로 동작한다.

<aside>
💡

**ConsumerNetworkClient**

KafkaConsumer의 모든 네트워크 통신을 담당

모든 요청은 비동기로 동작한다. 따라서 ConsumerNetworkClient의 응답값은 RequestFuture 클래스로 확인

```java
public class RequestFuture<T> {  

...
/* complete 메서드가 호출되었을 때 호출될 listener를 추가한다. */
public void addListener(RequestFutureListener<T> listener);

/* RequestFuture의 응답 타입을 T에서 S로 바꿔준다. */
public <S> RequestFuture<S> compose(final RequestFutureAdapter<T, S> adapter);
```

- compose 인자에 전달되는 핸들러는 ClientResponse에서 ByteBuffer로 바꾸는 Adapter를 등록

  `client.send(coordinator, requestBuilder).compose(**new** JoinGroupResponseHandler());`

</aside>

<aside>
💡

**ConsumerNetworkClinet가 요청을 처리하는 과정**

![image.png](attachment:cb597848-4b51-4538-8b63-6231d54745b7:image.png)

출처: https://d2.naver.com/helloworld/0974525

1. ConsumerNetworkClient는 send 메서드를 통해 전달된 모든 요청을 ClientRequest로 바꾼다
2. ClientRequest에는 요청이 완료되었을 때 호출될 RequestFuture가 설정되어 있다.
3. ConsumerNetworkClient는 ClientRequest를 바로 전송하지 않고 내부 버퍼인 Unsent Map에 먼저 저장
4. Key는 요청을 전송할 브로커의 호스트이고 Value는 브로커로 전송해야 하는 ClientRequest의 리스트
5. ClientRequest의 전송은 ConsumerNetworkClient의 poll 메서드가 호출될 때 이루어진다

**응답을 처리하는 과정**

![image.png](attachment:aaf788d4-d47b-45cd-98cf-0a697d01acef:image.png)

1. `브로커 → 응답 → NetworkClient → pendingCompletion 큐에 저장`

1. `다시 poll() 호출 → ConsumerNetworkClient → pendingCompletion 확인`
</aside>

---

### 4-7. 컨슈머 속성

- **`max.poll.records`**
    - poll() 호출할때마다 리턴되는 최대 레코드 개수
- `fetch.min.bytes & fetch.max.wait.ms`
    - 카프카가 컨슈머에게 레코드를 보내줄 때, 최소 쌓여야할 데이터 양 & 데이터 쌓일 때까지 얼마나 기다릴지 최대 시간
    - 잘 조절하면 부하 조절 가능
- `enable.auto.commit & [auto.commit.interval.ms](http://auto.commit.interval.ms/)`
    - 자동 커밋 사용할거면 `enable.auto.commit`은 true [`auto.commit.interval.ms](http://auto.commit.interval.ms/)은 얼마나 자주 커밋할지 결정`
-
- **`session.timeout.ms`**
    - **"컨슈머가 하트비트를 보내지 않으면 언제 죽었다고 판단할지"** 정하는 시간
    - **기본값: 45초**
    - 죽었다고 판단되면 리벨런싱 시작.
    - 이 설정과 대게 같이 만지는 속성 **`heartbeat.interval.ms`** (기본값: 3초)
        - **"얼마나 자주 하트비트를 보낼지"** 정하는 주기
        - 보통 [**`session.timeout.ms`](http://session.timeout.ms) 의 1/3 수준으로 설정.**
        - 왜 1/3 일까? → 3번의 재시도 기회가 있기 때문?
- **`max.poll.interval.ms`**
    - 컨슈머가 폴링을 하지 않고도 죽은 것으로 판정되지 않을 수 있는 최대 시간
    - 너무 짧게 설정하면, 처리중에 있는데 카프카가 죽은 컨슈머로 판단해서, 해당 파티션을 다른 컨슈머에 할당하는 리벨런싱 작업을 수행할수도 있다.
    - 너무 길게 설정하면, 정지된 컨슈머인지를 판단하는데 너무 늦게 판단한다.
    - **기본값은 5분**

---

### 4-8. 오프셋 과 커밋

<aside>
💡

1. **자동커밋**

토픽: my-topic (파티션 0)
메시지: **[0] [1] [2]** [3] [4] [5] [6] [7] [8] [9]

시간 흐름:
00:00 ┌─────────────────────────────────────────┐
│ poll() → 메시지 0,1,2 가져옴              │
│ 처리: 메시지 0,1,2 처리 완료              │
00:05 │ 🔄 자동 커밋 발생: offset=3 커밋         │
└─────────────────────────────────────────┘

00:05 ┌─────────────────────────────────────────┐
│ poll() → 메시지 3,4,5 가져옴              │
│ 처리: 메시지 3,4,5 처리 완료              │
00:10 │ 🔄 자동 커밋 발생: offset=6 커밋         │
└─────────────────────────────────────────┘

**__consumer_offsets 토픽**

시간 00:05 → [group=my-group, topic=my-topic, partition=0, offset=3]
시간 00:10 → [group=my-group, topic=my-topic, partition=0, offset=6]
시간 00:15 → [group=my-group, topic=my-topic, partition=0, offset=9]
↑
다음에 여기서부터 읽음

**장애 상황**

메시지 처리 상황:
00:00 ┌────────────────────────┐
│ 메시지 0,1,2 처리 완료   │ ✅
00:05 │ offset=3 커밋          │ ✅
└────────────────────────┘

재시작시: offset=3부터 읽기 시작 ✅

00:05 ┌────────────────────────┐
│ 메시지 3,4,5 처리 중...  │ 🔄
00:07 │ 💥 컨슈머 크래시!       │ ❌
│ (아직 커밋 안됨)        │
└────────────────────────┘

- 아직 커밋되기 전 크래시 난다면, 리벨런싱이 끝나고 메세지 3,4,5 다시 처리

→ **중복 발생**

**자동커밋 장단점**

- 별도 커밋 코드 불필요
- 개발자가 신경쓰지 않아도 됨
- **중복 처리 가능성**: 장애시 메시지 재처리

**결론**

자동 커밋은 편리하지만 "적어도 한 번(at-least-once)" 처리 방식이므로 중복 처리에 대한 고려가 필요

</aside>

- [ ]  문제점: at least most 도 가능 ← 커밋 시점 컨트롤할 수 없기 때문에 매우 위험

<aside>
💡

1. **현재 오프셋 커밋하기**
- `poll()`을 호출하면 여러 레코드를 가져옵니다
- 이 레코드들을 처리합니다
- `commitSync()`를 호출하면 해당 poll에서 가져온 레코드들의 마지막 오프셋 + 1을 커밋합니다

단점

- 만약 레코드들을 처리하는 와중에 크래시가 날 경우,
    - 마지막 메세지 배치의 맨 앞 레코드에서부터 리벨런스 시작 지점까지 모든 레코드들을 두 번 처리됨
- 브로커가 커밋요청에 응답할때까지, 블록되어 애플리케이션 처리량을 제한시킴
</aside>

<aside>
💡

1. **비동기적 커밋**
- 브로커가 커밋 요청보내고 응답을 기다리지않고, 처리를 계속 수행.
    - `commitAsync()`
- 커밋이 실패하면 재시도를 수행하지않음
    - 비동기라 결과를 늦게 보기 때문
    - 만약 재시도를 위해, 콜백을 사용하면, 커밋 순서 문제에 주의 필요.
</aside>

<aside>
💡

1. **동기적 커밋 + 비동기적 커밋**
- 비동기 커밋 수행하다가, 컨슈머 닫기 전, 리밸런스 전 마지막 커밋이라면 성공여부를 파악하기 위해 동기적 커밋 사용 → 실패여부에 따라 재시도 수행
</aside>

---

### 4-9. 리밸런스 리스너

- 리벨런싱 전후에 필요한 작업을 수행할 수 있습니다.
- 예를 들어 오프셋 커밋, 캐시 정리, 연결 해제 등의 작업을 할 수 있습니다.

```java
public interface ConsumerRebalanceListener {
    // 리벨런싱 시작 전 호출 (파티션을 잃기 전)
    void onPartitionsRevoked(Collection<TopicPartition> partitions);
    
    // 리벨런싱 완료 후 호출 (새로운 파티션 할당 받은 후)
    void onPartitionsAssigned(Collection<TopicPartition> partitions);
}
```

---

### 4-10. 폴링 루프 벗어나기

- 다른 스레드에서 `consumer.wakeup()` 을 호출하면, WakeupException을 발생시켜 컨슈머를 종료시킬 수 있다

---

### 4-11. 디시리얼라이저

현재 회사는 JsonDeserializer 사용중

```java
spring.kafka:
  bootstrap-servers: localhost:9092
  producer:
    value-serializer: org.springframework.kafka.support.serializer.JsonSerializer
    properties:
      spring.json.add.type.headers: false
  consumer:
    value-deserializer: org.springframework.kafka.support.serializer.JsonDeserializer
```

- 직렬화와 역직렬화때 다른 직렬화 사용할 수 없다

---

### 4-12. 독립실행컨슈머

- 단순한 구조: 컨슈머 그룹없이, 하나의 컨슈머가 모든 파티션을 맡는 경우.
    - 이땐 리벨런싱 기능도 필요 없음

### 참고자료

1. https://devocean.sk.com/community/detail.do?ID=165478&boardType=DEVOCEAN_STUDY&page=1
2. https://kafka.apache.org/documentation/#consumerapi
3. **카프카 핵심 가이드**
4. https://d2.naver.com/helloworld/0974525