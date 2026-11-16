# [2026-11-16] Kafka 메시징과 Consumer Group — 파티셔닝·오프셋 커밋부터 리밸런싱, 멱등 소비와 DLQ까지

> **한 줄 핵심 요약**: Kafka의 처리량은 "디스크에 순차 append하는 로그 + 파티션 단위 병렬 소비"에서 나오고, 그 대가로 **순서는 파티션 안에서만 보장**되며 **전달은 기본적으로 at-least-once**입니다. 그래서 실무 설계의 핵심은 ① 메시지 키로 순서가 필요한 단위를 같은 파티션에 모으고, ② **컨슈머를 멱등하게 만들어 중복을 흡수**하고, ③ 오프셋 커밋 시점과 DB 트랜잭션 커밋 시점이 어긋나도 데이터가 깨지지 않게 만드는 것입니다.

---

## 1. 개념 및 동작 원리

### 1.1 큐가 아니라 "로그"다 — RabbitMQ/Redis Pub-Sub와의 차이

가장 먼저 정리해야 하는 지점입니다. Kafka는 메시지를 소비하면 지우는 **큐(queue)** 가 아니라, 소비해도 남아 있는 **분산 커밋 로그(distributed commit log)** 입니다.

| 항목 | 전통적 메시지 큐 (RabbitMQ, SQS) | Kafka |
| --- | --- | --- |
| 소비 후 메시지 | 삭제(ack 기반) | **남음**(retention 기간까지 보존) |
| 소비 모델 | 큐 하나를 여러 워커가 나눠 가짐(push) | 컨슈머 그룹이 **오프셋을 스스로 관리**(pull) |
| 재처리 | 별도 구현 필요 | **오프셋 리셋으로 replay** |
| 순서 | 큐 단위 | **파티션 단위** |
| 처리량 | 브로커가 ack·라우팅 관리 | 순차 append + zero-copy → 수십만 msg/s |
| 잘 맞는 일 | 작업 지시(정확히 한 워커가 처리) | 이벤트 스트림(여러 소비자가 같은 데이터를 각자 소비) |

- **포인트**: Kafka에서 "메시지를 지운다"는 개념은 없습니다. 소비자가 어디까지 읽었는지(**offset**)만 기록하고, 데이터 정리는 `log.retention.hours`(기본 168시간 = 7일) 같은 **시간/용량 정책**으로 브로커가 일괄 삭제합니다.
- 그래서 **새 컨슈머 그룹을 붙이면 과거 이벤트를 처음부터 다시 읽을 수 있습니다.** 장애 복구 시 "그때 그 이벤트를 다시 흘려보내기"가 가능한 이유입니다.
- 반대로 **로그를 무한히 쌓으면 디스크가 터집니다.** `cleanup.policy=compact`(키별 최신값만 유지)는 "상태 스냅샷" 성격의 토픽(`__consumer_offsets`, CDC 테이블 미러링)에 씁니다.

### 1.2 토픽 · 파티션 · 오프셋 — 순서 보장의 단위

```text
토픽 order.created.v1 (파티션 3개, replication factor 3)

 P0: [0][1][2][3][4][5] ...   ← append만 가능, offset은 단조 증가
 P1: [0][1][2] ...
 P2: [0][1][2][3] ...
      ↑ 각 파티션은 독립된 로그. 파티션 간 순서는 보장되지 않는다.
```

- **토픽**은 논리 이름, **파티션**은 실제 파일(세그먼트) 단위의 물리 로그입니다.
- **오프셋**은 파티션 내 위치를 가리키는 숫자이고, 파티션마다 **0부터 독립적으로** 증가합니다.
- 순서 보장은 **"같은 파티션에 들어간 메시지"** 에서만 성립합니다. 키를 지정하지 않으면 라운드로빈/스티키 분배로 흩어지므로, 전체 순서를 기대하면 안 됩니다.
- **파티션 수는 늘릴 수 있지만 줄일 수 없습니다.** 그리고 늘리는 순간 `hash(key) % numPartitions` 결과가 바뀌어 **같은 키가 다른 파티션으로 갈 수 있습니다**(이미 적재된 과거 메시지와 순서가 어긋남). 파티션 수 변경은 "순서 보장이 깨져도 되는가"를 먼저 판단해야 하는 작업입니다.

### 1.3 복제와 acks — "유실 없음"을 만드는 조건

```text
        ┌──────── Broker 1 (Leader, P0) ────────┐
        │  append → follower에게 fetch 요청 대기 │
        └───────┬───────────────────┬───────────┘
        fetch   │                   │  fetch
   ┌────────────▼──────┐   ┌────────▼──────────┐
   │ Broker 2 (Follower)│   │ Broker 3 (Follower)│   ← ISR(In-Sync Replica)
   └────────────────────┘   └────────────────────┘
```

| `acks` | 의미 | 유실 가능성 |
| --- | --- | --- |
| `0` | 브로커 응답을 기다리지 않음 | 높음(네트워크만 확인). 로그성 지표에만 |
| `1` | **리더** 디스크 기록까지만 확인 | 리더 장애 시 follower 미복제분 유실 |
| `all` (= `-1`) | **ISR 전체** 기록까지 확인 | 가장 안전. 지연은 증가 |

- `acks=all`만으로는 부족합니다. **`min.insync.replicas=2`** 와 함께 써야 "리더 + 최소 1개 follower에 기록"이 보장됩니다. ISR이 `min.insync.replicas`보다 줄면 프로듀서는 `NotEnoughReplicasException`으로 **쓰기를 거부**합니다(가용성 대신 정합성 선택).
- 실무 표준 조합은 **`replication.factor=3`, `min.insync.replicas=2`, `acks=all`** 입니다. 이 구성이면 브로커 1대가 죽어도 유실 없이 계속 쓸 수 있습니다.
- `unclean.leader.election.enable=false`(기본값)를 유지해야 "동기화 안 된 복제본이 리더가 되어 데이터를 되감는" 사고를 막습니다.

### 1.4 프로듀서 — 배치, 파티셔너, 멱등성

**처리량은 배치에서 나옵니다.** 프로듀서는 메시지를 보내는 즉시 요청하지 않고 `batch.size`(기본 16KB) 또는 `linger.ms`(기본 0, 실무 5~20ms)까지 모아 파티션별로 한 번에 전송합니다. `linger.ms`를 조금 늘리면 지연은 수 ms 늘고 처리량은 크게 오릅니다. 압축(`compression.type=lz4` 또는 `zstd`)도 배치 단위로 적용되어 네트워크·디스크 비용을 줄입니다.

**파티션 결정 규칙**(Kafka 3.x `BuiltInPartitioner`, 구 `DefaultPartitioner` 대체):

1. 키가 있으면 `murmur2(key) % numPartitions` (Java `hashCode()`가 아님 → 다른 언어 클라이언트와 동일 분배 보장)
2. 키가 없으면 **스티키 파티셔너**: 한 배치를 한 파티션에 몰아 보내 배치 효율을 올리고, 배치가 차면 다른 파티션으로 이동
3. 파티션을 직접 지정한 `ProducerRecord(topic, partition, key, value)`면 그대로

**멱등 프로듀서(idempotent producer)**: Kafka 3.0부터 `enable.idempotence=true`가 **기본값**입니다. 프로듀서가 `producerId` + 파티션별 시퀀스 번호를 붙여, 재시도로 인한 **중복 기록과 순서 뒤바뀜을 브로커가 걸러냅니다**. 전제 조건은 `acks=all`, `retries>0`, `max.in.flight.requests.per.connection<=5`입니다.

> 정리: "멱등 프로듀서 = 브로커에 **중복이 쌓이지 않게**" 하는 장치이고, 뒤에서 다룰 "멱등 컨슈머 = 같은 메시지를 두 번 처리해도 **비즈니스 결과가 한 번**" 과는 층위가 다릅니다. 면접에서 이 둘을 구분하면 정확한 답이 됩니다.

### 1.5 Consumer Group — 병렬 소비와 리밸런싱

```text
토픽: order.created.v1 (P0, P1, P2)   group: order-service

[정상]  consumer-A → P0, P1     consumer-B → P2
[추가]  consumer-C 합류 → 리밸런싱 → A:P0, B:P1, C:P2
[이탈]  B 장애 → 리밸런싱 → A:P0, P1, C:P2
```

- **그룹 안에서 하나의 파티션은 정확히 한 컨슈머가 소비**합니다. 그래서 **그룹의 최대 병렬성 = 파티션 수**이고, 파티션이 3개인데 컨슈머를 5개 띄우면 2개는 **놀고 있습니다**(`IdleConsumer`).
- 반대로 **그룹이 다르면** 같은 파티션을 각자 읽습니다. "주문 서비스"와 "정산 서비스"가 각자 그룹을 가지면 같은 이벤트를 독립적으로 소비합니다.
- 스케일 아웃 판단 기준은 컨슈머 대수보다 **파티션 수 × 처리 속도 vs 유입량(lag)** 입니다.

**리밸런싱(rebalance)**: 컨슈머 합류·이탈·구독 토픽 변경·파티션 증가 시 파티션 소유권을 재분배하는 과정입니다. 이때 **잠깐 소비가 멈추고, 커밋되지 않은 오프셋은 되감깁니다**(중복 발생 원인 1위).

| 전략 | 동작 | 특징 |
| --- | --- | --- |
| `RangeAssignor` / `RoundRobinAssignor`(eager) | **모든** 파티션을 반납 후 재할당 | 단순하지만 전체 컨슈머가 stop-the-world |
| `CooperativeStickyAssignor`(cooperative) | 옮겨야 할 파티션만 반납(증분) | 중단 최소화. 신규 서비스 기본 권장 |

**세션·폴링 타임아웃 3종 세트**

| 설정 | 기본값 | 의미 |
| --- | --- | --- |
| `heartbeat.interval.ms` | 3000 | 하트비트 주기(session의 1/3 이하) |
| `session.timeout.ms` | 45000 | 하트비트가 이 시간 끊기면 이탈 판정 |
| `max.poll.interval.ms` | 300000 | **`poll()` 사이 간격** 상한. 초과 시 이탈 판정 |

- 하트비트는 **백그라운드 스레드**가 보냅니다. 즉 애플리케이션이 `poll()` 루프에서 오래 걸려도 하트비트는 계속 나가고, **`max.poll.interval.ms`를 넘겨야** 리밸런싱이 트리거됩니다.
- "처리 로직이 느려 컨슈머가 계속 죽는다"의 90%는 이 값 초과입니다. 해결은 ① `max.poll.records` 줄이기, ② 처리 로직을 별도 스레드로 넘기고 `pause/resume`로 흐름 제어, ③ 파티션·컨슈머 증설입니다.

### 1.6 오프셋 커밋과 전달 보장 3단계

| 보장 | 커밋 시점 | 결과 |
| --- | --- | --- |
| at-most-once | **처리 전** 커밋 | 유실 가능, 중복 없음 |
| at-least-once | **처리 후** 커밋(기본) | **중복 가능**, 유실 없음 |
| exactly-once | 트랜잭션으로 처리+오프셋 원자 커밋 | Kafka 내부 한정으로 성립 |

- 컨슈머 오프셋은 `__consumer_offsets` 토픽(기본 50파티션, compact)에 **그룹·토픽·파티션 단위**로 저장됩니다.
- **"오프셋 N 커밋"은 "N 미만은 다 처리했다"는 뜻이지, "N번 메시지까지 처리했다"가 아닙니다.** 그래서 순서가 뒤섞인 배치 커밋은 앞의 미완료를 삼켜버릴 수 있습니다(자세한 함정은 3장).
- 실무 기본값은 **at-least-once**이고, 그래서 **컨슈머는 반드시 멱등**해야 합니다. "정확히 한 번"을 애플리케이션 레벨에서 보장하는 방법은 3.1절에서 정리합니다.
- `auto.offset.reset`(`earliest`/`latest`/`none`)은 **커밋된 오프셋이 아예 없을 때만** 적용됩니다. "중복이 싫어서 `latest`로 바꾼다"는 발상은 장애 복구 시 이벤트를 통째로 건너뛰는 결과를 낳습니다.

### 1.7 Spring Kafka 매핑

| Kafka 개념 | Spring Kafka |
| --- | --- |
| 컨슈머 폴링 루프 | `KafkaMessageListenerContainer`(내부에서 `poll()` 반복) |
| 리스너 | `@KafkaListener` (`concurrency` = 컨테이너 스레드 수 ≤ 파티션 수) |
| 오프셋 커밋 | `AckMode.BATCH`(기본) / `RECORD` / `MANUAL` / `MANUAL_IMMEDIATE` |
| 오류 처리 | `DefaultErrorHandler`(블로킹 재시도) / `@RetryableTopic`(논블로킹 재시도 + DLT) |
| 발행 | `KafkaTemplate<K, V>` (`send()`는 `CompletableFuture` 반환) |

- Spring Kafka는 `enable.auto.commit`을 **`false`** 로 강제하고 컨테이너가 커밋 시점을 관리합니다(리스너가 정상 종료된 뒤 커밋 → at-least-once).
- 기본 `AckMode.BATCH`는 `poll()` 한 번에 받은 레코드를 **모두 처리한 뒤 마지막 오프셋+1을 커밋**합니다. 중간 실패 시 배치 전체가 재처리되므로 중복이 늘어납니다. 정밀 제어가 필요하면 `MANUAL_IMMEDIATE`를 씁니다.

---

## 2. 실무 적용 — 주문 이벤트 발행/소비 코드

### 2.1 환경과 의존성

```text
# 환경
Java 21 (LTS) / Spring Boot 3.2.5 / spring-kafka 3.1.4
Apache Kafka 3.6.x (KRaft 모드, ZooKeeper 미사용) / Docker Compose 로컬 3브로커
MySQL 8.0 / Gradle 8.7
```

```groovy
// build.gradle
plugins {
    id 'java'
    id 'org.springframework.boot' version '3.2.5'
    id 'io.spring.dependency-management' version '1.1.4'
}

java { toolchain { languageVersion = JavaLanguageVersion.of(21) } }

dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-web'
    implementation 'org.springframework.boot:spring-boot-starter-data-jpa'
    implementation 'org.springframework.kafka:spring-kafka'          // KafkaTemplate, @KafkaListener
    runtimeOnly 'com.mysql:mysql-connector-j'
    compileOnly 'org.projectlombok:lombok'
    annotationProcessor 'org.projectlombok:lombok'
    testImplementation 'org.springframework.kafka:spring-kafka-test' // EmbeddedKafka
}
```

```yaml
# src/main/resources/application.yml
spring:
  kafka:
    bootstrap-servers: localhost:9092,localhost:9093,localhost:9094
    producer:
      key-serializer: org.apache.kafka.common.serialization.StringSerializer
      value-serializer: org.springframework.kafka.support.serializer.JsonSerializer
      acks: all                       # 유실 방지 (기본값이지만 명시)
      properties:
        enable.idempotence: true      # 재시도 중복/순서 뒤바뀜 방지
        max.in.flight.requests.per.connection: 5
        compression.type: lz4
        linger.ms: 10                 # 배치 대기 → 처리량 확보
    consumer:
      group-id: order-service
      key-deserializer: org.apache.kafka.common.serialization.StringDeserializer
      # 역직렬화 실패가 컨슈머를 죽이지 않도록 ErrorHandlingDeserializer로 감싼다
      value-deserializer: org.springframework.kafka.support.serializer.ErrorHandlingDeserializer
      auto-offset-reset: earliest
      enable-auto-commit: false       # Spring Kafka가 강제하지만 의도를 남긴다
      max-poll-records: 100           # 배치 크기 ↓ → max.poll.interval 초과 위험 ↓
      properties:
        spring.deserializer.value.delegate.class: org.springframework.kafka.support.serializer.JsonDeserializer
        spring.json.trusted.packages: "com.example.order.event"
        partition.assignment.strategy: org.apache.kafka.clients.consumer.CooperativeStickyAssignor
        max.poll.interval.ms: 300000
    listener:
      ack-mode: manual_immediate      # DB 커밋 이후에만 오프셋 커밋
      concurrency: 3                  # 파티션 수 이하
```

### 2.2 프로듀서 — 순서가 필요한 단위를 키로 묶기

```java
// src/main/java/com/example/order/event/OrderCreatedEvent.java
package com.example.order.event;

import java.time.LocalDateTime;

public record OrderCreatedEvent(
        String messageId,      // 멱등 소비용 고유 ID (UUID)
        Long orderId,
        Long memberId,
        long amount,
        LocalDateTime createdAt
) {}
```

```java
// src/main/java/com/example/order/producer/OrderEventProducer.java
package com.example.order.producer;

import com.example.order.event.OrderCreatedEvent;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.kafka.core.KafkaTemplate;
import org.springframework.kafka.support.SendResult;
import org.springframework.stereotype.Component;

import java.util.concurrent.CompletableFuture;

@Slf4j
@Component
@RequiredArgsConstructor
public class OrderEventProducer {

    private static final String TOPIC = "order.created.v1";

    private final KafkaTemplate<String, OrderCreatedEvent> kafkaTemplate;

    /**
     * key = orderId → 같은 주문의 이벤트는 항상 같은 파티션 → 파티션 내 순서 보장.
     * (memberId를 키로 쓰면 "회원 단위 순서"가 보장되지만 특정 회원에게 부하가 몰리는 핫 파티션 위험이 있다)
     */
    public CompletableFuture<SendResult<String, OrderCreatedEvent>> publish(OrderCreatedEvent event) {
        return kafkaTemplate.send(TOPIC, String.valueOf(event.orderId()), event)
                .whenComplete((result, ex) -> {
                    if (ex != null) {
                        // 여기서 삼키면 이벤트가 조용히 유실된다. 재발행/알림 정책을 반드시 둔다.
                        log.error("order event publish failed. orderId={}", event.orderId(), ex);
                        return;
                    }
                    var meta = result.getRecordMetadata();
                    log.info("published orderId={} partition={} offset={}",
                            event.orderId(), meta.partition(), meta.offset());
                });
    }
}
```

> `send()`는 비동기입니다. 반환값을 버리면 **발행 실패를 알 수 없습니다.** 성공/실패를 비즈니스적으로 확인해야 하는 경우(정산·알림)는 `CompletableFuture`를 체이닝하거나, 아래 3.2절의 Outbox 패턴이 정답입니다.

### 2.3 컨슈머 — 멱등 소비 + DB 커밋 후 오프셋 커밋

```sql
-- src/main/resources/db/migration/V2__consumed_message.sql
CREATE TABLE consumed_message (
    message_id  VARCHAR(36) NOT NULL,
    topic       VARCHAR(100) NOT NULL,
    consumed_at DATETIME(6) NOT NULL,
    PRIMARY KEY (message_id)          -- 중복 소비 차단의 핵심
) ENGINE = InnoDB;
```

```java
// src/main/java/com/example/order/consumer/OrderEventConsumer.java
package com.example.order.consumer;

import com.example.order.event.OrderCreatedEvent;
import com.example.order.service.OrderPointService;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.kafka.annotation.KafkaListener;
import org.springframework.kafka.support.Acknowledgment;
import org.springframework.stereotype.Component;

@Slf4j
@Component
@RequiredArgsConstructor
public class OrderEventConsumer {

    private final OrderPointService orderPointService;

    @KafkaListener(topics = "order.created.v1", groupId = "order-service")
    public void consume(OrderCreatedEvent event, Acknowledgment ack) {
        // 처리(비즈니스 로직 + consumed_message INSERT)를 한 트랜잭션으로 묶는다.
        // 중복 메시지면 서비스 계층에서 조용히 skip(true)을 반환한다.
        boolean processed = orderPointService.applyPointOnce(event);

        if (!processed) {
            log.info("duplicate message skipped. messageId={}", event.messageId());
        }
        // DB 트랜잭션이 커밋된 뒤에만 오프셋을 커밋한다.
        // (여기서 예외가 나면 ack를 호출하지 않아 오프셋이 남고, 컨테이너가 재처리한다)
        ack.acknowledge();
    }
}
```

```java
// src/main/java/com/example/order/service/OrderPointService.java
package com.example.order.service;

import com.example.order.domain.ConsumedMessage;
import com.example.order.event.OrderCreatedEvent;
import com.example.order.repository.ConsumedMessageRepository;
import com.example.order.repository.MemberRepository;
import lombok.RequiredArgsConstructor;
import org.springframework.dao.DataIntegrityViolationException;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.time.LocalDateTime;

@Service
@RequiredArgsConstructor
public class OrderPointService {

    private final ConsumedMessageRepository consumedMessageRepository;
    private final MemberRepository memberRepository;

    /**
     * 같은 messageId로 두 번 호출되어도 포인트는 한 번만 적립된다.
     * PK 충돌을 "정상 경로"로 사용한다 — 조회 후 INSERT(check-then-act)는 동시성에서 뚫린다.
     */
    @Transactional
    public boolean applyPointOnce(OrderCreatedEvent event) {
        try {
            consumedMessageRepository.saveAndFlush(new ConsumedMessage(
                    event.messageId(), "order.created.v1", LocalDateTime.now()));
        } catch (DataIntegrityViolationException e) {
            // 이미 처리한 메시지 → 포인트 적립을 건너뛰고 트랜잭션은 정상 종료
            return false;
        }
        memberRepository.increasePoint(event.memberId(), event.amount() / 100);
        return true;
    }
}
```

> `saveAndFlush`로 **즉시 INSERT를 날려** PK 충돌을 그 자리에서 잡는 것이 요점입니다. `save()`만 호출하면 flush가 커밋 시점까지 미뤄져 `catch` 블록이 실행되지 않습니다.

### 2.4 재시도와 DLQ — poison pill이 파티션을 멈추지 않게

```java
// src/main/java/com/example/order/config/KafkaConsumerConfig.java
package com.example.order.config;

import org.apache.kafka.common.TopicPartition;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.kafka.config.ConcurrentKafkaListenerContainerFactory;
import org.springframework.kafka.core.ConsumerFactory;
import org.springframework.kafka.core.KafkaTemplate;
import org.springframework.kafka.listener.DeadLetterPublishingRecoverer;
import org.springframework.kafka.listener.DefaultErrorHandler;
import org.springframework.util.backoff.ExponentialBackOff;

@Configuration
public class KafkaConsumerConfig {

    /**
     * 3회 재시도(0.5s → 1s → 2s) 후 order.created.v1.DLT 로 보낸다.
     * DLT 파티션은 원본과 동일하게 유지해 순서 추적이 가능하다.
     */
    @Bean
    public DefaultErrorHandler errorHandler(KafkaTemplate<String, Object> kafkaTemplate) {
        var recoverer = new DeadLetterPublishingRecoverer(kafkaTemplate,
                (record, ex) -> new TopicPartition(record.topic() + ".DLT", record.partition()));

        var backOff = new ExponentialBackOff(500L, 2.0);
        backOff.setMaxInterval(2000L);
        backOff.setMaxElapsedTime(4000L); // 0.5 + 1 + 2 = 3.5초 후 포기

        var handler = new DefaultErrorHandler(recoverer, backOff);
        // 역직렬화 실패는 재시도해도 절대 성공하지 않는다 → 즉시 DLT
        handler.addNotRetryableExceptions(
                org.springframework.kafka.support.serializer.DeserializationException.class,
                org.springframework.messaging.converter.MessageConversionException.class);
        return handler;
    }
}
```

- **재시도는 파티션을 블로킹합니다.** 같은 파티션의 뒤 메시지는 앞 메시지가 성공할 때까지 기다립니다(순서 보장의 대가). 지연에 민감하면 `@RetryableTopic`으로 **재시도 전용 토픽**(`-retry-0`, `-retry-1`, `-dlt`)에 넣어 논블로킹으로 미루고, 원 파티션은 계속 흐르게 합니다.
- **DLT는 "끝"이 아니라 "격리"입니다.** DLT lag를 모니터링하고, 원인 수정 후 원 토픽으로 재발행하는 절차(운영 런북)를 함께 만들어야 합니다.

---

## 3. 실무 주의사항 (Edge Cases)

### 3.1 중복은 "예외"가 아니라 "정상"이다 — 멱등 소비 3가지 구현

at-least-once 환경에서 아래 상황은 **반드시** 중복을 만듭니다: 리밸런싱 직전 커밋 유실, 처리 후 ack 전 프로세스 종료, 브로커 응답 타임아웃 후 프로듀서 재시도(멱등 프로듀서 미사용 시), DLT 재발행.

| 방법 | 구조 | 주의 |
| --- | --- | --- |
| **고유 키 + 유니크 제약** | `consumed_message(message_id)` PK 충돌 무시 | 가장 단순·견고. 메시지에 **발행 시점 UUID**를 넣는 게 핵심 |
| **상태 전이 조건부 UPDATE** | `UPDATE orders SET status='PAID' WHERE id=? AND status='PENDING'`, 영향 행 수 0이면 skip | "이미 처리됨"을 자연스럽게 흡수. 상태 머신이 명확할 때 |
| **업서트(UPSERT)** | `INSERT ... ON DUPLICATE KEY UPDATE` / `MERGE` | 결과가 같아지는 연산(절대값 설정)에만. 누적 합산에는 부적합 |

- **안 좋은 예**: `if (!repository.existsByOrderId(id)) { save(...) }` — 동시에 두 스레드가 들어오면 둘 다 통과합니다(check-then-act 경쟁 조건). 유니크 제약이 최종 방어선입니다.
- **누적 연산은 특히 위험**합니다. 포인트·재고·잔액은 "두 번 더하기"가 그대로 사고가 되므로 PK/유니크 기반 차단이 필수입니다.

### 3.2 이중 쓰기(dual write) — DB 커밋과 Kafka 발행은 원자적이지 않다

```java
// 흔한 안티패턴: DB 커밋은 성공, 발행은 실패 → 정산 누락 / 반대면 유령 이벤트
@Transactional
public void createOrder(...) {
    orderRepository.save(order);
    kafkaTemplate.send("order.created.v1", order.getId(), event); // 실패해도 롤백되지 않음
}
```

- Kafka는 DB 트랜잭션에 참여하지 않습니다(`KafkaTransactionManager`는 Kafka만 제어). `send()` 실패가 DB 롤백을 일으키지 않고, DB 롤백 후에도 이미 나간 이벤트는 되돌릴 수 없습니다.
- **해결 1 — Transactional Outbox (권장)**: 같은 DB 트랜잭션에 `outbox` 테이블 INSERT → 별도 릴레이(폴링 스케줄러 또는 Debezium CDC)가 미발행 건을 읽어 Kafka로 보내고 `published=true`로 표시. 최소 1회 발행이므로 컨슈머 멱등은 여전히 필요합니다.

```java
@Transactional
public void createOrder(OrderRequest request) {
    Order order = orderRepository.save(Order.from(request));
    outboxRepository.save(OutboxEvent.of("order.created.v1", String.valueOf(order.getId()), payload(order)));
    // 발행은 커밋 이후 별도 릴레이. 여기서 Kafka를 직접 호출하지 않는다.
}
```
- **해결 2 — 발행 후 커밋 (덜 안전)**: `send().get()`으로 브로커 ack를 받은 뒤 DB 커밋. 순서는 맞지만 **발행 성공 + DB 커밋 실패** 시 유령 이벤트가 남습니다. 컨슈머 멱등 + "존재하지 않는 주문" 방어 로직이 필요합니다.

### 3.3 `@Transactional`과 `@KafkaListener`를 같은 메서드에 붙이지 말 것

```java
// 위험: 리스너가 리턴하면 컨테이너가 오프셋을 커밋하는데,
// 트랜잭션 프록시의 커밋은 그 뒤에 일어날 수 있다 → "오프셋은 갔는데 DB는 롤백"
@KafkaListener(topics = "order.created.v1")
@Transactional
public void consume(OrderCreatedEvent event) { ... }
```

- DB 트랜잭션은 **서비스 계층**에 두고, 리스너는 `서비스 호출 → 성공 시 ack`만 담당합니다(2.3절 코드). 트랜잭션 경계와 오프셋 커밋 경계를 **같은 순서로 명시**하는 것이 안전합니다.
- 반대로 "**DB 커밋 전에 ack**"하면, ack 직후 프로세스가 죽어 DB는 롤백되었는데 오프셋은 전진한 상태가 됩니다 → **유실**. ack는 반드시 마지막입니다.
- 참고: `MANUAL`(다음 poll까지 지연 커밋)과 `MANUAL_IMMEDIATE`(즉시 커밋)의 차이는 재기동 시 재처리 범위에 영향을 줍니다. 정합성이 중요한 곳은 `MANUAL_IMMEDIATE` + DB 커밋 후 ack 조합이 해석하기 쉽습니다.

### 3.4 순서 보장이 깨지는 4가지 경로

1. **키 없는 발행** — 스티키 파티셔너가 배치 단위로 파티션을 옮깁니다. 순서가 필요하면 **키를 반드시 준다**.
2. **파티션 수 증가** — `hash(key) % N`이 바뀌어 같은 키가 다른 파티션으로. 파티션 증설은 순서 요구가 없는 토픽에서만.
3. **`max.in.flight > 1` + 재시도 + 멱등성 미사용** — 재시도분이 뒤에 도착해 순서가 뒤집힙니다. `enable.idempotence=true`로 해결.
4. **컨슈머 내부 병렬 처리** — 리스너에서 `CompletableFuture`/스레드 풀로 넘기면 완료 순서가 뒤섞입니다. 병렬 처리가 필요하면 **키별로 같은 워커**에 배정(consistent hashing)하거나 파티션 내에서는 단일 스레드를 유지합니다.

> 같은 파티션을 재시도하는 동안 뒤 메시지가 대기하는 것은 **의도된 동작**입니다. "순서 보장 + 빠른 실패"는 함께 얻기 어렵고, 이 트레이드오프를 설명할 수 있으면 면접에서 강한 답이 됩니다.

### 3.5 운영에서 자주 터지는 것들

- **컨슈머 lag 폭증**: `kafka-consumer-groups.sh --describe`로 `LAG` 확인 → 파티션 증설(단, 순서 영향 검토) 또는 컨슈머 `concurrency` 증설. lag는 **처리 시간 × 유입량**이 병렬성보다 클 때 계속 커집니다.
- **리밸런싱 폭풍**: 배포로 컨슈머가 동시에 내려가면 반복 리밸런싱. **static membership**(`group.instance.id`)과 cooperative 전략, 롤링 배포로 완화합니다.
- **`max.poll.interval.ms` 초과**: `max.poll.records`(기본 500)를 줄이고 배치 처리 시간을 측정합니다. 외부 API를 리스너 안에서 호출하는 구조라면 즉시 분리 대상입니다.
- **역직렬화 실패 무한 루프**: `ErrorHandlingDeserializer` 없이 스키마가 바뀌면 컨슈머가 같은 레코드에서 계속 예외 → 파티션 정지. 반드시 감싸고, 실패는 DLT로 격리합니다.
- **스키마 진화**: 이벤트 필드를 지우거나 타입을 바꾸면 과거 메시지 역직렬화가 깨집니다. **필드 추가는 하되 삭제·타입 변경은 하지 않는** 하위 호환 규칙(or Avro/Protobuf + Schema Registry)을 정합니다.
- **`@KafkaListener`의 `concurrency` > 파티션 수**: 남는 스레드는 유휴입니다. 파티션 3개에 `concurrency=10`은 낭비이며, 오히려 리밸런싱 시 소유권만 흔듭니다.
- **트랜잭셔널 프로듀서의 `transactional.id` 재사용**: 같은 `transactional.id`를 두 인스턴스가 쓰면 서로를 펜싱(fencing)해 한쪽이 죽습니다. 인스턴스별 고유값을 씁니다.
- **보존 기간과 replay**: `retention`이 지난 오프셋으로 리셋하면 `OffsetOutOfRangeException` → `auto.offset.reset` 정책으로 넘어갑니다. 장기 재처리 계획이 있다면 보존 기간을 먼저 확인합니다.
- **메시지 크기**: 기본 상한 `max.request.size` 1MB. 대용량 페이로드는 **Claim Check 패턴**(본문은 S3/오브젝트 스토리지, 메시지에는 참조 키만)으로 분리합니다.

### 3.6 언제 Kafka를 쓰지 말아야 하는가

- **요청-응답이 필요할 때**: 응답 토픽 + correlationId로 흉내는 낼 수 있지만, 동기 API가 단순합니다.
- **작업 큐(정확히 한 워커가 처리)만 필요할 때**: SQS/RabbitMQ가 운영 부담이 적습니다. Kafka는 파티션·오프셋·리밸런싱을 직접 이해해야 하는 비용이 있습니다.
- **초당 수 건 수준**: 운영 복잡도가 이득을 넘습니다. 스케줄러 + DB 폴링으로 충분한 경우가 많습니다.
- **트랜잭션 정합성이 최우선인 소규모 서비스**: Outbox + 폴링만으로도 요구를 만족하는 경우가 많습니다.

---

## 4. 기술 면접 단골 꼬리 질문 & 답변 팁

- **Q. Kafka가 빠른 이유를 설명해 보세요.**
  - **A.** "① append-only 로그라 디스크 **순차 쓰기**가 가능하고, ② 페이지 캐시와 **zero-copy**(`sendfile`)로 커널↔유저 복사를 줄이며, ③ 프로듀서·컨슈머가 **배치**로 묶어 요청 수를 줄이고, ④ 파티션 단위로 **소비를 병렬화**할 수 있기 때문입니다. 브로커가 소비 상태를 관리하지 않고 컨슈머가 오프셋을 직접 관리하는 것도 부하를 줄입니다."

- **Q. 메시지 순서는 어떻게 보장하나요?**
  - **A.** "**파티션 내에서만** 보장됩니다. 그래서 순서가 필요한 단위(주문 ID, 사용자 ID)를 **메시지 키**로 주면 murmur2 해시로 항상 같은 파티션에 들어갑니다. 토픽 전체 순서는 보장되지 않고, 파티션을 늘리면 키-파티션 매핑이 바뀌어 순서가 깨질 수 있으므로 증설은 신중해야 합니다."

- **Q. 메시지가 유실되지 않으려면 어떤 설정이 필요하나요?**
  - **A.** "프로듀서는 `acks=all`과 `enable.idempotence=true`, 브로커는 `replication.factor=3` + `min.insync.replicas=2`, 그리고 `unclean.leader.election.enable=false`입니다. 컨슈머 쪽은 `enable.auto.commit=false`로 **처리 완료 후 커밋**해야 합니다. 이 조합이 '브로커 1대 장애 + 재시도 중복'까지 견디는 최소 구성입니다."

- **Q. exactly-once는 가능한가요?**
  - **A.** "**Kafka 안에서 Kafka로** 가는 구간(consume-transform-produce)은 `transactional.id`와 `isolation.level=read_committed`로 원자 커밋이 가능합니다. 하지만 **DB와 Kafka를 함께** 정확히 한 번은 불가능해서, 실무에서는 **at-least-once + 멱등 컨슈머**(유니크 키, 조건부 UPDATE) 또는 **Outbox 패턴**으로 '결과적으로 한 번'을 만듭니다. 이 구분을 명확히 말하는 것이 중요합니다."

- **Q. 리밸런싱이 일어나면 무슨 일이 생기나요?**
  - **A.** "파티션 소유권이 재분배되고, 커밋되지 않은 오프셋은 되감겨 **중복 처리**가 발생할 수 있습니다. eager 전략은 전체 파티션을 반납해 일시 정지가 길고, `CooperativeStickyAssignor`는 옮길 파티션만 반납해 중단을 줄입니다. 완화책으로 static membership, 짧은 처리 시간, `max.poll.interval.ms` 조정을 씁니다."

- **Q. 컨슈머를 늘리면 항상 빨라지나요?**
  - **A.** "아닙니다. **파티션 수가 병렬성의 상한**이라 파티션 3개에 컨슈머 5개를 띄우면 2개는 유휴입니다. 병목이 DB 커넥션 풀이나 외부 API라면 컨슈머를 늘려도 lag는 줄지 않고 하류만 터집니다. 먼저 lag와 처리 시간을 측정해 병목을 특정해야 합니다."

- **Q. 같은 메시지가 두 번 들어오면 어떻게 처리하나요?**
  - **A.** "발행 시점에 넣은 `messageId`를 **유니크 키로 저장**하고, PK 충돌(`DataIntegrityViolationException`)이나 조건부 UPDATE의 영향 행 수 0을 '이미 처리됨'으로 해석해 비즈니스 로직을 건너뜁니다. 조회 후 INSERT하는 check-then-act는 동시성에서 뚫리므로, **DB 제약을 최종 방어선**으로 씁니다."

- **Q. DB 저장과 Kafka 발행을 한 트랜잭션으로 묶을 수 있나요? (이중 쓰기)**
  - **A.** "묶을 수 없습니다. Kafka는 DB 트랜잭션에 참여하지 않습니다. 실무에서는 **Outbox 테이블에 같은 트랜잭션으로 이벤트를 적재**하고 별도 릴레이(폴링 스케줄러, Debezium CDC)가 발행하는 방식으로 원자성을 확보합니다. 이 경우에도 최소 1회 발행이므로 컨슈머 멱등은 필수입니다."

- **Q. 처리 실패 메시지는 어떻게 다루나요? (poison pill)**
  - **A.** "`DefaultErrorHandler`로 제한 횟수만큼 지수 백오프 재시도하고, 소진되면 `DeadLetterPublishingRecoverer`로 **DLT에 격리**합니다. 재시도는 원 파티션을 블로킹하므로 지연에 민감하면 `@RetryableTopic`의 **재시도 전용 토픽**으로 논블로킹 처리합니다. 역직렬화 실패처럼 재시도가 무의미한 예외는 즉시 DLT로 보냅니다."

- **Q. RabbitMQ 대신 Kafka를 선택하는 기준은?**
  - **A.** "**이벤트 스트림**이면 Kafka입니다. 같은 데이터를 여러 서비스가 각자 소비하고, 과거 이벤트 replay가 필요하고, 처리량이 크고 순서를 파티션 단위로 관리할 수 있을 때 적합합니다. 반대로 **작업 큐**(한 작업은 정확히 한 워커가 처리, 우선순위·지연 큐·복잡한 라우팅)나 소규모 트래픽이면 RabbitMQ/SQS가 운영 부담이 적습니다."
