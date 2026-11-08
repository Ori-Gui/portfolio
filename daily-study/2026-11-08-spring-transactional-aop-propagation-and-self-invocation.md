# [2026-11-08] Spring @Transactional 동작 원리(AOP 프록시)와 전파 속성(Propagation), 자가 호출(Self-Invocation) 한계

> **한 줄 핵심 요약**: `@Transactional`은 프레임워크가 프록시 객체를 한 겹 씌워 **바깥에서 들어오는 호출만** 가로채는 AOP 방식이라, 같은 클래스 안에서 `this.method()`로 부르면 트랜잭션이 아예 시작되지 않는다 — 이 구조를 이해하면 (1) 자가 호출이 왜 무력화되는지, (2) `REQUIRES_NEW`가 왜 커넥션을 2개 잡아 풀을 고갈시키는지, (3) `readOnly = true`가 구체적으로 무엇을 아끼는지, (4) 커밋 이후 알림을 어디에 붙여야 하는지가 하나의 원리로 설명된다. 실무에서 `@Transactional` 버그의 대부분은 "애노테이션을 안 붙였다"가 아니라 **"프록시 경계 밖에서 호출했다"** 이다.

---

## 0. 코드 예제 (전체 실행 가능 버전)

### 0-1. 프로젝트 설정 (Spring Boot 3.3.x, Java 21)

```gradle
// 파일 경로: build.gradle
plugins {
    id 'java'
    id 'org.springframework.boot' version '3.3.5'
    id 'io.spring.dependency-management' version '1.1.6'
}

group = 'study'
version = '0.0.1-SNAPSHOT'

java {
    toolchain {
        languageVersion = JavaLanguageVersion.of(21)
    }
}

repositories {
    mavenCentral()
}

dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-data-jpa'
    implementation 'org.springframework.boot:spring-boot-starter-web'
    implementation 'org.springframework.boot:spring-boot-starter-aop'   // @Async, exposeProxy, AspectJ 모드
    runtimeOnly 'com.mysql:mysql-connector-j'
    testImplementation 'org.springframework.boot:spring-boot-starter-test'
    testRuntimeOnly 'org.junit.platform:junit-platform-launcher'
}

tasks.named('test') {
    useJUnitPlatform()
}
```

```yaml
# 파일 경로: src/main/resources/application.yml
spring:
  datasource:
    url: jdbc:mysql://localhost:3306/study?serverTimezone=Asia/Seoul&characterEncoding=UTF-8
    username: study
    password: study
    hikari:
      maximum-pool-size: 10       # REQUIRES_NEW 실험에서는 이 숫자가 핵심 변수
      connection-timeout: 3000    # 풀 고갈을 빨리 관찰하려고 짧게 (운영 기본값은 30초)
  jpa:
    hibernate:
      ddl-auto: create-drop
    open-in-view: false           # 트랜잭션 경계를 서비스 계층에 고정 (OSIV 끄기)
    properties:
      hibernate:
        format_sql: true

logging:
  level:
    org.springframework.transaction.interceptor: TRACE  # 트랜잭션 시작·참여·커밋·롤백·보류 로그
    org.springframework.orm.jpa: DEBUG
    org.hibernate.SQL: DEBUG
```

### 0-2. 도메인과 리포지토리

```java
// 파일 경로: src/main/java/study/order/domain/OrderStatus.java
package study.order.domain;

public enum OrderStatus {
    PENDING, CONFIRMED, CANCELED
}
```

```java
// 파일 경로: src/main/java/study/order/domain/Order.java
package study.order.domain;

import jakarta.persistence.Column;
import jakarta.persistence.Entity;
import jakarta.persistence.EnumType;
import jakarta.persistence.Enumerated;
import jakarta.persistence.GeneratedValue;
import jakarta.persistence.GenerationType;
import jakarta.persistence.Id;
import jakarta.persistence.Table;
import java.time.LocalDateTime;

@Entity
@Table(name = "orders")
public class Order {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false)
    private Long userId;

    @Column(nullable = false)
    private int totalAmount;

    @Enumerated(EnumType.STRING)
    @Column(nullable = false, length = 20)
    private OrderStatus status;

    @Column(nullable = false)
    private LocalDateTime orderedAt;

    protected Order() {
        // JPA 기본 생성자
    }

    public Order(Long userId, int totalAmount) {
        this.userId = userId;
        this.totalAmount = totalAmount;
        this.status = OrderStatus.PENDING;
        this.orderedAt = LocalDateTime.now();
    }

    public void confirm() {
        this.status = OrderStatus.CONFIRMED;
    }

    public Long getId() {
        return id;
    }

    public Long getUserId() {
        return userId;
    }

    public int getTotalAmount() {
        return totalAmount;
    }

    public OrderStatus getStatus() {
        return status;
    }

    public LocalDateTime getOrderedAt() {
        return orderedAt;
    }
}
```

```java
// 파일 경로: src/main/java/study/order/repository/OrderRepository.java
package study.order.repository;

import org.springframework.data.jpa.repository.JpaRepository;
import study.order.domain.Order;

public interface OrderRepository extends JpaRepository<Order, Long> {
}
```

```java
// 파일 경로: src/main/java/study/audit/domain/AuditLog.java
package study.audit.domain;

import jakarta.persistence.Column;
import jakarta.persistence.Entity;
import jakarta.persistence.GeneratedValue;
import jakarta.persistence.GenerationType;
import jakarta.persistence.Id;
import jakarta.persistence.Table;
import java.time.LocalDateTime;

@Entity
@Table(name = "audit_log")
public class AuditLog {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    // FK를 걸지 않는 이유는 3-3에서 설명한다.
    // REQUIRES_NEW 트랜잭션은 외부 트랜잭션이 아직 커밋하지 않은 주문 row를 볼 수 없다.
    @Column(nullable = false)
    private Long orderId;

    @Column(nullable = false, length = 200)
    private String message;

    @Column(nullable = false)
    private LocalDateTime createdAt;

    protected AuditLog() {
    }

    public AuditLog(Long orderId, String message) {
        this.orderId = orderId;
        this.message = message;
        this.createdAt = LocalDateTime.now();
    }

    public Long getId() {
        return id;
    }

    public Long getOrderId() {
        return orderId;
    }

    public String getMessage() {
        return message;
    }
}
```

```java
// 파일 경로: src/main/java/study/audit/repository/AuditLogRepository.java
package study.audit.repository;

import org.springframework.data.jpa.repository.JpaRepository;
import study.audit.domain.AuditLog;

public interface AuditLogRepository extends JpaRepository<AuditLog, Long> {
}
```

### 0-3. 모범 사례: 감사 로그를 별도 빈으로 분리하고 `REQUIRES_NEW`로 남긴다

```java
// 파일 경로: src/main/java/study/audit/service/AuditLogService.java
package study.audit.service;

import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Propagation;
import org.springframework.transaction.annotation.Transactional;
import study.audit.domain.AuditLog;
import study.audit.repository.AuditLogRepository;

@Service
public class AuditLogService {

    private final AuditLogRepository auditLogRepository;

    public AuditLogService(AuditLogRepository auditLogRepository) {
        this.auditLogRepository = auditLogRepository;
    }

    /**
     * REQUIRES_NEW: 외부 트랜잭션이 있으면 잠시 보류(suspend)하고, 완전히 새로운 물리 트랜잭션에서 실행한다.
     * 외부 트랜잭션이 롤백돼도 이 기록은 이미 커밋되어 남는다. (실패 이력·감사 로그의 전형적인 요구사항)
     *
     * 주의: 커넥션을 하나 더 잡으므로 호출 빈도가 높은 경로에서 남용하면 풀이 고갈된다.
     */
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void record(Long orderId, String message) {
        auditLogRepository.save(new AuditLog(orderId, message));
    }
}
```

```java
// 파일 경로: src/main/java/study/order/service/OrderService.java
package study.order.service;

import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
import study.audit.service.AuditLogService;
import study.order.domain.Order;
import study.order.repository.OrderRepository;

@Service
public class OrderService {

    private final OrderRepository orderRepository;
    private final AuditLogService auditLogService;

    public OrderService(OrderRepository orderRepository, AuditLogService auditLogService) {
        this.orderRepository = orderRepository;
        this.auditLogService = auditLogService;
    }

    /**
     * 정상 흐름: 주문 저장(같은 트랜잭션) + 감사 로그(별도 트랜잭션, 외부 빈 호출이므로 프록시를 탄다).
     */
    @Transactional
    public Long placeOrder(Long userId, int amount) {
        Order order = orderRepository.save(new Order(userId, amount));
        auditLogService.record(order.getId(), "ORDER_CREATED");
        return order.getId();
    }

    /**
     * 롤백 동작 확인용: 주문은 롤백되고, REQUIRES_NEW로 남긴 감사 로그만 커밋된다.
     */
    @Transactional
    public void placeOrderThenFail(Long userId, int amount) {
        Order order = orderRepository.save(new Order(userId, amount));
        auditLogService.record(order.getId(), "ORDER_FAILED");
        throw new IllegalStateException("결제 승인 실패를 가정한 예외");
    }
}
```

### 0-4. 자가 호출로 트랜잭션이 무력화되는 예제 (Before — 이렇게 쓰면 안 된다)

```java
// 파일 경로: src/main/java/study/order/service/SelfInvokingOrderService.java
package study.order.service;

import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Propagation;
import org.springframework.transaction.annotation.Transactional;
import study.audit.domain.AuditLog;
import study.audit.repository.AuditLogRepository;
import study.order.domain.Order;
import study.order.repository.OrderRepository;

@Service
public class SelfInvokingOrderService {

    private final OrderRepository orderRepository;
    private final AuditLogRepository auditLogRepository;

    public SelfInvokingOrderService(OrderRepository orderRepository,
                                    AuditLogRepository auditLogRepository) {
        this.orderRepository = orderRepository;
        this.auditLogRepository = auditLogRepository;
    }

    @Transactional
    public void placeOrderThenFail(Long userId, int amount) {
        Order order = orderRepository.save(new Order(userId, amount));

        // ❌ 자가 호출: this는 프록시가 아니라 실제 대상 객체(Target)다.
        //    REQUIRES_NEW가 무시되고 외부 트랜잭션에 그대로 참여한다.
        //    → 아래 예외로 외부 트랜잭션이 롤백되면 감사 로그도 함께 사라진다.
        this.record(order.getId(), "ORDER_FAILED");

        throw new IllegalStateException("결제 승인 실패를 가정한 예외");
    }

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void record(Long orderId, String message) {
        auditLogRepository.save(new AuditLog(orderId, message));
    }
}
```

### 0-5. 대안 1: Self-Injection (`@Lazy` / `ObjectProvider`)

```java
// 파일 경로: src/main/java/study/order/service/SelfInjectedOrderService.java
package study.order.service;

import org.springframework.context.annotation.Lazy;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Propagation;
import org.springframework.transaction.annotation.Transactional;
import study.audit.domain.AuditLog;
import study.audit.repository.AuditLogRepository;
import study.order.domain.Order;
import study.order.repository.OrderRepository;

@Service
public class SelfInjectedOrderService {

    private final OrderRepository orderRepository;
    private final AuditLogRepository auditLogRepository;
    private final SelfInjectedOrderService self;   // 컨테이너가 넣어주는 값은 "프록시"

    /**
     * Spring Boot 2.6+ 부터 순환 참조가 기본 금지이므로 @Lazy 없이 자기 자신을 주입하면
     * BeanCurrentlyInCreationException이 발생한다. @Lazy는 지연 해석 프록시를 넣어 이 문제를 피한다.
     */
    public SelfInjectedOrderService(OrderRepository orderRepository,
                                    AuditLogRepository auditLogRepository,
                                    @Lazy SelfInjectedOrderService self) {
        this.orderRepository = orderRepository;
        this.auditLogRepository = auditLogRepository;
        this.self = self;
    }

    @Transactional
    public void placeOrder(Long userId, int amount) {
        Order order = orderRepository.save(new Order(userId, amount));

        // ✅ self는 프록시이므로 AOP가 적용된다 → REQUIRES_NEW가 실제로 동작한다.
        self.record(order.getId(), "ORDER_CREATED");
    }

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void record(Long orderId, String message) {
        auditLogRepository.save(new AuditLog(orderId, message));
    }
}
```

```java
// 파일 경로: src/main/java/study/order/service/ProviderOrderService.java
package study.order.service;

import org.springframework.beans.factory.ObjectProvider;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Propagation;
import org.springframework.transaction.annotation.Transactional;
import study.audit.domain.AuditLog;
import study.audit.repository.AuditLogRepository;
import study.order.domain.Order;
import study.order.repository.OrderRepository;

@Service
public class ProviderOrderService {

    private final OrderRepository orderRepository;
    private final AuditLogRepository auditLogRepository;
    private final ObjectProvider<ProviderOrderService> selfProvider;

    public ProviderOrderService(OrderRepository orderRepository,
                                AuditLogRepository auditLogRepository,
                                ObjectProvider<ProviderOrderService> selfProvider) {
        this.orderRepository = orderRepository;
        this.auditLogRepository = auditLogRepository;
        this.selfProvider = selfProvider;
    }

    @Transactional
    public void placeOrder(Long userId, int amount) {
        Order order = orderRepository.save(new Order(userId, amount));

        // getObject() 시점에 컨테이너에서 빈을 꺼내므로 주입 시점의 순환 참조 문제가 없다.
        selfProvider.getObject().record(order.getId(), "ORDER_CREATED");
    }

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void record(Long orderId, String message) {
        auditLogRepository.save(new AuditLog(orderId, message));
    }
}
```

### 0-6. 대안 2: `AopContext.currentProxy()` (권장하지 않지만 원리 이해용)

```java
// 파일 경로: src/main/java/study/config/AopConfig.java
package study.config;

import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.EnableAspectJAutoProxy;

@Configuration
// exposeProxy = true 로 설정해야 현재 스레드의 프록시가 AopContext에 노출된다. (기본값 false)
@EnableAspectJAutoProxy(exposeProxy = true)
public class AopConfig {
}
```

```java
// 파일 경로: src/main/java/study/order/service/AopContextOrderService.java
package study.order.service;

import org.springframework.aop.framework.AopContext;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Propagation;
import org.springframework.transaction.annotation.Transactional;
import study.audit.domain.AuditLog;
import study.audit.repository.AuditLogRepository;
import study.order.domain.Order;
import study.order.repository.OrderRepository;

@Service
public class AopContextOrderService {

    private final OrderRepository orderRepository;
    private final AuditLogRepository auditLogRepository;

    public AopContextOrderService(OrderRepository orderRepository,
                                  AuditLogRepository auditLogRepository) {
        this.orderRepository = orderRepository;
        this.auditLogRepository = auditLogRepository;
    }

    @Transactional
    public void placeOrder(Long userId, int amount) {
        Order order = orderRepository.save(new Order(userId, amount));

        // ✅ 프록시를 꺼내 다시 호출하므로 AOP가 적용된다.
        // 단점: 스프링 API에 비즈니스 코드가 결합되고, exposeProxy 설정에 의존하며,
        //       프록시를 거치지 않고 호출된 메서드 안에서는 IllegalStateException이 난다.
        AopContextOrderService proxy = (AopContextOrderService) AopContext.currentProxy();
        proxy.record(order.getId(), "ORDER_CREATED");
    }

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void record(Long orderId, String message) {
        auditLogRepository.save(new AuditLog(orderId, message));
    }
}
```

### 0-7. 대안 3: `TransactionTemplate` (프로그래밍 방식, 프록시 경계와 무관)

```java
// 파일 경로: src/main/java/study/order/service/TemplateOrderService.java
package study.order.service;

import org.springframework.stereotype.Service;
import org.springframework.transaction.PlatformTransactionManager;
import org.springframework.transaction.TransactionDefinition;
import org.springframework.transaction.annotation.Transactional;
import org.springframework.transaction.support.TransactionTemplate;
import study.audit.domain.AuditLog;
import study.audit.repository.AuditLogRepository;
import study.order.domain.Order;
import study.order.repository.OrderRepository;

@Service
public class TemplateOrderService {

    private final OrderRepository orderRepository;
    private final AuditLogRepository auditLogRepository;
    private final TransactionTemplate requiresNewTemplate;

    public TemplateOrderService(OrderRepository orderRepository,
                                AuditLogRepository auditLogRepository,
                                PlatformTransactionManager transactionManager) {
        this.orderRepository = orderRepository;
        this.auditLogRepository = auditLogRepository;

        this.requiresNewTemplate = new TransactionTemplate(transactionManager);
        this.requiresNewTemplate.setPropagationBehavior(TransactionDefinition.PROPAGATION_REQUIRES_NEW);
        this.requiresNewTemplate.setTimeout(3);   // 초 단위. 외부 연동이 끼는 트랜잭션은 짧게 끊는다.
    }

    @Transactional
    public void placeOrder(Long userId, int amount) {
        Order order = orderRepository.save(new Order(userId, amount));

        // 프록시를 타지 않는 명시적 경계. 자가 호출 문제가 원천적으로 없다.
        requiresNewTemplate.executeWithoutResult(status ->
                auditLogRepository.save(new AuditLog(order.getId(), "ORDER_CREATED")));
    }
}
```

### 0-8. `readOnly = true` 조회 전용 서비스 + Master/Replica 라우팅

```java
// 파일 경로: src/main/java/study/order/service/OrderQueryService.java
package study.order.service;

import java.util.List;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
import study.order.domain.Order;
import study.order.repository.OrderRepository;

@Service
public class OrderQueryService {

    private final OrderRepository orderRepository;

    public OrderQueryService(OrderRepository orderRepository) {
        this.orderRepository = orderRepository;
    }

    /**
     * readOnly = true 의 효과 (1-5에서 상세):
     *  1) Hibernate FlushMode가 MANUAL → 자동 flush(더티 체킹) 생략
     *  2) 엔티티 로딩 시 스냅샷(loaded state)을 남기지 않음 → 메모리·CPU 절감
     *  3) Connection.setReadOnly(true) → Replica 라우팅·DB 힌트의 근거
     * 단, "쓰기를 막아주는 안전장치"가 아니라 최적화 힌트다.
     */
    @Transactional(readOnly = true)
    public List<Order> findOrdersOf(Long userId) {
        return orderRepository.findAll().stream()
                .filter(order -> order.getUserId().equals(userId))
                .toList();
    }
}
```

```java
// 파일 경로: src/main/java/study/config/ReplicaRoutingDataSource.java
package study.config;

import org.springframework.jdbc.datasource.lookup.AbstractRoutingDataSource;
import org.springframework.transaction.support.TransactionSynchronizationManager;

public class ReplicaRoutingDataSource extends AbstractRoutingDataSource {

    /**
     * 현재 스레드에 바인딩된 트랜잭션이 read-only인지로 대상 DataSource를 고른다.
     * 이 값은 트랜잭션 동기화가 시작된 뒤에야 true가 되므로, 커넥션 획득 시점을 미루는
     * LazyConnectionDataSourceProxy와 함께 써야 라우팅이 의도대로 동작한다.
     */
    @Override
    protected Object determineCurrentLookupKey() {
        return TransactionSynchronizationManager.isCurrentTransactionReadOnly() ? "replica" : "primary";
    }
}
```

```java
// 파일 경로: src/main/java/study/config/DataSourceConfig.java
package study.config;

import com.zaxxer.hikari.HikariDataSource;
import java.util.Map;
import javax.sql.DataSource;
import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.jdbc.datasource.LazyConnectionDataSourceProxy;

@Configuration
public class DataSourceConfig {

    @Bean
    @ConfigurationProperties("spring.datasource.primary")
    public HikariDataSource primaryDataSource() {
        return new HikariDataSource();
    }

    @Bean
    @ConfigurationProperties("spring.datasource.replica")
    public HikariDataSource replicaDataSource() {
        return new HikariDataSource();
    }

    @Bean
    public DataSource dataSource() {
        ReplicaRoutingDataSource routing = new ReplicaRoutingDataSource();
        routing.setTargetDataSources(Map.of(
                "primary", primaryDataSource(),
                "replica", replicaDataSource()));
        routing.setDefaultTargetDataSource(primaryDataSource());

        // 트랜잭션 시작 시점이 아니라 첫 SQL 실행 시점에 커넥션을 얻는다.
        // → read-only 플래그·isolation level이 확정된 뒤에 커넥션을 고를 수 있다.
        return new LazyConnectionDataSourceProxy(routing);
    }
}
```

### 0-9. 커밋 이후에만 외부 알림 발송 (`@TransactionalEventListener`)

```java
// 파일 경로: src/main/java/study/order/event/OrderConfirmedEvent.java
package study.order.event;

public record OrderConfirmedEvent(Long orderId, Long userId) {
}
```

```java
// 파일 경로: src/main/java/study/order/service/OrderConfirmService.java
package study.order.service;

import org.springframework.context.ApplicationEventPublisher;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
import study.order.domain.Order;
import study.order.repository.OrderRepository;

@Service
public class OrderConfirmService {

    private final OrderRepository orderRepository;
    private final ApplicationEventPublisher eventPublisher;

    public OrderConfirmService(OrderRepository orderRepository,
                               ApplicationEventPublisher eventPublisher) {
        this.orderRepository = orderRepository;
        this.eventPublisher = eventPublisher;
    }

    @Transactional
    public void confirm(Long orderId) {
        Order order = orderRepository.findById(orderId)
                .orElseThrow(() -> new IllegalArgumentException("주문 없음: " + orderId));
        order.confirm();

        // 발행은 지금 하지만, 리스너 실행은 커밋이 성공한 뒤로 미뤄진다.
        // → 이후 로직이 실패해 롤백되면 메일도 나가지 않는다.
        eventPublisher.publishEvent(new OrderConfirmedEvent(order.getId(), order.getUserId()));
    }
}
```

```java
// 파일 경로: src/main/java/study/order/event/OrderNotificationListener.java
package study.order.event;

import org.springframework.scheduling.annotation.Async;
import org.springframework.stereotype.Component;
import org.springframework.transaction.event.TransactionPhase;
import org.springframework.transaction.event.TransactionalEventListener;
import study.notification.MailClient;

@Component
public class OrderNotificationListener {

    private final MailClient mailClient;

    public OrderNotificationListener(MailClient mailClient) {
        this.mailClient = mailClient;
    }

    /**
     * fallbackExecution = false(기본값): 트랜잭션 없이 발행된 이벤트는 무시한다.
     * AFTER_COMMIT: 커밋 성공 후에만 실행. 롤백 시 실행되지 않는다.
     * @Async: 응답 시간에 영향을 주지 않도록 별도 스레드에서 실행.
     */
    @Async
    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT, fallbackExecution = false)
    public void handle(OrderConfirmedEvent event) {
        mailClient.sendOrderConfirmed(event.userId(), event.orderId());
    }
}
```

```java
// 파일 경로: src/main/java/study/notification/MailClient.java
package study.notification;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.stereotype.Component;

@Component
public class MailClient {

    private static final Logger log = LoggerFactory.getLogger(MailClient.class);

    public void sendOrderConfirmed(Long userId, Long orderId) {
        // 실제 구현에서는 SMTP/SES/알림 서비스 호출. 여기서는 로그로 대체한다.
        log.info("[MAIL] userId={}, orderId={} 주문 확정 안내", userId, orderId);
    }
}
```

```java
// 파일 경로: src/main/java/study/StudyApplication.java
package study;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.scheduling.annotation.EnableAsync;

@EnableAsync   // @Async 활성화
@SpringBootApplication
public class StudyApplication {

    public static void main(String[] args) {
        SpringApplication.run(StudyApplication.class, args);
    }
}
```

### 0-10. 검증 테스트 (프록시 여부 + 롤백 독립성 + 자가 호출 무력화)

```java
// 파일 경로: src/test/java/study/order/TransactionBehaviorTest.java
package study.order;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Test;
import org.springframework.aop.support.AopUtils;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.transaction.support.TransactionSynchronizationManager;
import study.audit.repository.AuditLogRepository;
import study.order.repository.OrderRepository;
import study.order.service.OrderService;
import study.order.service.SelfInvokingOrderService;
import study.order.service.SelfInjectedOrderService;

@SpringBootTest
class TransactionBehaviorTest {

    @Autowired
    private OrderService orderService;
    @Autowired
    private SelfInvokingOrderService selfInvokingOrderService;
    @Autowired
    private SelfInjectedOrderService selfInjectedOrderService;
    @Autowired
    private OrderRepository orderRepository;
    @Autowired
    private AuditLogRepository auditLogRepository;

    /**
     * 테스트 간 간섭 제거. 이 클래스는 @Transactional 테스트가 아니므로
     * REQUIRES_NEW로 커밋된 데이터가 다음 테스트에 남는다 — 그 자체가 3-5의 함정이다.
     */
    @BeforeEach
    void cleanUp() {
        auditLogRepository.deleteAll();
        orderRepository.deleteAll();
    }

    @Test
    @DisplayName("주입된 빈은 CGLIB 프록시이고, 트랜잭션 안에서는 동기화가 활성화된다")
    void injectedBeanIsProxy() {
        assertThat(AopUtils.isAopProxy(orderService)).isTrue();
        assertThat(AopUtils.isCglibProxy(orderService)).isTrue();   // Spring Boot 기본값
        assertThat(TransactionSynchronizationManager.isActualTransactionActive()).isFalse();
    }

    @Test
    @DisplayName("외부 트랜잭션이 롤백돼도 REQUIRES_NEW 감사 로그는 커밋되어 남는다")
    void requiresNewSurvivesOuterRollback() {
        long auditBefore = auditLogRepository.count();

        assertThatThrownBy(() -> orderService.placeOrderThenFail(1L, 10_000))
                .isInstanceOf(IllegalStateException.class);

        assertThat(orderRepository.count()).isZero();                    // 주문은 롤백
        assertThat(auditLogRepository.count()).isEqualTo(auditBefore + 1); // 로그는 커밋
    }

    @Test
    @DisplayName("자가 호출은 REQUIRES_NEW를 무력화해 외부 롤백에 함께 휩쓸린다")
    void selfInvocationIsNotTransactional() {
        long auditBefore = auditLogRepository.count();

        assertThatThrownBy(() -> selfInvokingOrderService.placeOrderThenFail(1L, 10_000))
                .isInstanceOf(IllegalStateException.class);

        assertThat(orderRepository.count()).isZero();
        assertThat(auditLogRepository.count()).isEqualTo(auditBefore);   // 로그도 사라짐
    }

    @Test
    @DisplayName("self 주입 방식은 자가 호출 문제를 해결한다")
    void selfInjectionWorks() {
        long auditBefore = auditLogRepository.count();

        selfInjectedOrderService.placeOrder(1L, 10_000);

        assertThat(orderRepository.count()).isEqualTo(1);
        assertThat(auditLogRepository.count()).isEqualTo(auditBefore + 1);
    }
}
```

```text
# 트랜잭션 동작을 눈으로 확인하는 로그 설정 (application.yml)
# org.springframework.transaction.interceptor: TRACE 를 켜면 아래처럼 전파 속성과 참여/신규 여부가 찍힌다.
Getting transaction for [study.order.service.OrderService.placeOrderThenFail]
Creating new transaction with name [study.order.service.OrderService.placeOrderThenFail]:
    PROPAGATION_REQUIRED, ISOLATION_DEFAULT
Suspending current transaction, creating new transaction with name [study.audit.service.AuditLogService.record]:
    PROPAGATION_REQUIRES_NEW, ISOLATION_DEFAULT
Initiating transaction commit ... [AuditLogService.record]
Resuming suspended transaction after completion of inner transaction
Initiating transaction rollback ... [OrderService.placeOrderThenFail]
```

---

## 1. Spring @Transactional의 핵심 개념과 AOP 프록시 동작 원리

### 1-1. 선언적 트랜잭션 = 프록시 패턴 + 인터셉터

`@Transactional`은 "트랜잭션 코드를 비즈니스 로직에서 분리한다"는 목표를 **프록시 객체를 대신 만들어 주는 방식**으로 달성한다. 스프링 컨테이너는 빈을 등록할 때 `@Transactional`이 붙은 클래스(또는 메서드)를 발견하면, 그 빈을 원본 그대로 노출하지 않고 **원본을 감싼 프록시**를 빈으로 등록한다. 다른 빈에 주입되는 것도, `@Autowired`로 가져오는 것도 이 프록시다.

```text
[다른 빈의 호출]                        [같은 클래스 내부의 호출]

OrderController                        OrderService (Target 인스턴스)
      │  (주입받은 참조)                        │
      ▼                                        │ this.record()
OrderService$$SpringCGLIB$$0 (프록시)          ▼
      │ ① TransactionInterceptor.invoke()   record()  ← 인터셉터를 거치지 않는다
      │ ② BEGIN / COMMIT / ROLLBACK           (트랜잭션 없음)
      ▼
OrderService (Target)  ← 진짜 비즈니스 로직
```

이 구조에서 곧바로 따라오는 결론이 두 가지다.

1. **호출이 프록시를 통과해야만** 트랜잭션이 시작된다. 같은 객체의 `this.method()`는 프록시를 우회해 대상 객체를 직접 호출하므로 인터셉터가 끼어들 틈이 없다.
2. 프록시는 **상속(또는 인터페이스 구현)으로 만든 대역**이므로 `final` 클래스·`final` 메서드·`private` 메서드에는 개입할 수 없다.

### 1-2. CGLIB 프록시 vs JDK Dynamic Proxy

| 구분 | JDK Dynamic Proxy | CGLIB (Spring Boot 기본값) |
|---|---|---|
| 생성 방식 | 인터페이스를 구현한 새 클래스 (`java.lang.reflect.Proxy`) | 대상 클래스를 **상속**한 서브클래스 생성 (바이트코드 조작) |
| 필요 조건 | 반드시 **인터페이스**가 있어야 함 | 인터페이스 불필요, 단 **`final` 클래스 불가** |
| 메서드 개입 범위 | 인터페이스에 선언된 메서드만 | `public`/`protected`/패키지 가시성 메서드 (단, `final`·`private`·`static` 불가) |
| 호출 오버헤드 | 인터페이스 경유 리플렉션 | 서브클래스 오버라이드 → 약간 더 빠름 |
| 부작용 | 인터페이스에 없는 메서드는 프록시 타입으로 호출 불가 | 기본 생성자가 없어도 됨(Objenesis), 대신 `final` 클래스면 기동 실패 |

- Spring Boot는 `spring.aop.proxy-target-class`가 **true(기본값)** 라서, 인터페이스가 있어도 CGLIB 프록시를 만든다. 그래서 `AopUtils.isCglibProxy(bean)`가 `true`로 나온다.
- `final` 클래스에 `@Transactional`을 붙이면 `IllegalArgumentException: Cannot subclass final class ...`로 **애플리케이션이 기동되지 않는다.** 반면 `final` 메서드에 붙이면 기동은 되지만 트랜잭션이 **조용히** 적용되지 않는다. 후자가 더 위험하다.
- `private` 메서드에 붙이는 것도 마찬가지로 조용히 무시된다. 스프링 문서가 "프록시 모드에서는 `public` 메서드에만 붙이라"고 하는 이유다.
- 인터페이스에만 `@Transactional`을 붙이는 방식은 프록시 종류·버전에 따라 탐색되지 않을 수 있어 권장되지 않는다. **구현 클래스에 붙인다.**

### 1-3. 호출에서 커밋까지의 실제 흐름

```text
① 프록시 진입
   OrderService$$SpringCGLIB$$0.placeOrder()
② 인터셉터 체인
   TransactionInterceptor.invoke(MethodInvocation)
     └ TransactionAspectSupport.invokeWithinTransaction()
         ├ TransactionAttributeSource.getTransactionAttribute()
         │    : 클래스/메서드에서 @Transactional 속성(전파·격리수준·readOnly·timeout·rollbackFor) 해석
         ├ PlatformTransactionManager.getTransaction(definition)
         │    └ AbstractPlatformTransactionManager.getTransaction()
         │        ├ doGetTransaction()          : 현재 스레드에 바인딩된 커넥션/EntityManager 조회
         │        ├ isExistingTransaction()     : 이미 트랜잭션 중인가?
         │        ├ handleExistingTransaction() : 전파 속성 분기 (참여 / 보류 / 예외 / savepoint)
         │        └ startTransaction() → doBegin()
         │              - DataSource에서 Connection 획득 (또는 EntityManager 생성)
         │              - setAutoCommit(false), 격리 수준·readOnly 적용
         │              - ConnectionHolder를 ThreadLocal에 바인딩
         ├ invocation.proceed()                 : 실제 비즈니스 메서드 실행
         └ completeTransactionAfterThrowing() / commitTransaction()
               - 예외 타입 판정 → rollbackOn() → rollback or commit
               - triggerBeforeCommit → doCommit → triggerAfterCommit → triggerAfterCompletion
               - ThreadLocal 자원 해제(clear)
```

- `PlatformTransactionManager`의 대표 구현은 `JpaTransactionManager`(JPA), `DataSourceTransactionManager`(JDBC/MyBatis), `HibernateTransactionManager`(Hibernate 네이티브)다. **어떤 것을 쓰는지에 따라 "물리 트랜잭션"의 실체가 커넥션인지 영속성 컨텍스트인지가 달라진다.**
- `TransactionInterceptor`는 `TransactionAspectSupport`를 상속하고, 스프링 6에서도 같은 구조다. 즉 `@Transactional`은 "특별한 기능"이 아니라 **어드바이스 하나**이고, 다른 AOP(예: `@Cacheable`, `@Async`)와 같은 프록시 위에서 순서를 다툰다.

### 1-4. `TransactionSynchronizationManager`와 ThreadLocal 기반 커넥션 바인딩

트랜잭션의 실체는 결국 **"이 스레드가 지금 어떤 커넥션(또는 EntityManager)을 쓰고 있는가"** 이다. 스프링은 이를 스레드마다 다른 저장소(ThreadLocal)에 묶어 둔다.

```java
// 스프링 내부에서 벌어지는 일을 개념적으로 옮긴 의사 코드
// (실제로는 AbstractPlatformTransactionManager / DataSourceTransactionManager가 수행)

// 1) 트랜잭션 시작: 커넥션을 ThreadLocal에 묶는다
TransactionSynchronizationManager.bindResource(dataSource, new ConnectionHolder(connection));
TransactionSynchronizationManager.setCurrentTransactionReadOnly(true);
TransactionSynchronizationManager.setActualTransactionActive(true);

// 2) 같은 스레드의 어떤 DAO든 커넥션을 새로 열지 않고 바인딩된 것을 꺼내 쓴다
Connection con = DataSourceUtils.getConnection(dataSource);   // ThreadLocal 조회

// 3) 트랜잭션 종료: 반드시 해제한다 (해제하지 않으면 커넥션이 풀로 돌아가지 않는다)
TransactionSynchronizationManager.unbindResource(dataSource);
```

- JPA에서는 키가 `DataSource`가 아니라 `EntityManagerFactory`이고 값이 `EntityManagerHolder`다. `JpaTransactionManager`가 `EntityManager`를 만들어 바인딩하고, `SimpleJpaRepository` 같은 리포지토리는 `EntityManagerFactoryUtils.getTransactionalEntityManager(...)`로 **같은 영속성 컨텍스트**를 꺼내 쓴다. 그래서 한 트랜잭션 안의 여러 리포지토리 호출이 하나의 1차 캐시·하나의 flush를 공유한다.
- ThreadLocal 기반이라는 사실이 곧 **한계**다. 스레드가 바뀌면(`@Async`, `ExecutorService`, `CompletableFuture`) 바인딩이 따라가지 않으므로 트랜잭션이 전파되지 않는다(3-4).
- 커넥션 누수·풀 고갈의 대표 원인도 여기에 있다. 트랜잭션이 비정상 종료되면 바인딩이 남아 다음 요청이 커넥션을 얻지 못한다. 스프링이 `clear()`를 `finally`에 두는 이유다.

### 1-5. `@Transactional(readOnly = true)`의 내부 최적화

`readOnly = true`는 "읽기만 하겠다"는 **선언**이고, 실제 효과는 계층별로 나뉜다.

| 계층 | 무엇이 달라지는가 | 얻는 것 |
|---|---|---|
| JPA/Hibernate | 새 트랜잭션에서 `EntityManager`의 `FlushMode`를 `MANUAL`로 설정 | 커밋 시점의 자동 flush와, 쿼리 실행 전 auto-flush(더티 체킹)가 사라짐 |
| Hibernate 세션 | 세션을 read-only로 설정 → 로딩한 엔티티의 **스냅샷(loaded state)을 보관하지 않음** | 엔티티 수만큼의 스냅샷 메모리와 비교 연산(CPU) 절감 |
| JDBC 드라이버 | `Connection.setReadOnly(true)` | 드라이버·프록시가 읽기 전용 힌트로 활용 |
| 인프라 | `TransactionSynchronizationManager.isCurrentTransactionReadOnly()`가 `true` | `AbstractRoutingDataSource`로 **Replica(읽기 전용 노드) 라우팅** |

포인트를 정리하면 다음과 같다.

- **절감되는 것은 "쓰기 SQL"이 아니라 "쓰기 가능성을 확인하는 비용"** 이다. 더티 체킹은 모든 엔티티를 순회하며 스냅샷과 비교하는 작업이고, 조회가 많은 서비스에서는 이 CPU·메모리가 실제 병목이 된다. 대량 조회에서 `readOnly`가 체감되는 이유가 여기 있다.
- **Replica 라우팅은 공짜가 아니다.** 라우팅 DataSource의 `getConnection()`은 트랜잭션 시작 시점(`doBegin`)에 호출되는데, read-only 플래그는 그 직후에야 ThreadLocal에 세팅된다. 그래서 `LazyConnectionDataSourceProxy`로 **커넥션 획득을 첫 SQL 실행 시점까지 미뤄야** `determineCurrentLookupKey()`가 올바른 값을 본다. 이 조합을 모르고 `AbstractRoutingDataSource`만 쓰면 항상 primary로 붙는다.
- **`readOnly`는 안전장치가 아니다.** 강제가 아니라 힌트이므로, `readOnly` 트랜잭션에서 엔티티를 수정하면 예외 없이 **조용히 무시**될 수 있다(MANUAL flush → 커밋 시 flush 생략). "읽기 전용이니까 실수해도 롤백되겠지"는 기대가 틀리는 지점이다.
- **전파 속성과 상호작용한다.** 외부 트랜잭션에 참여(REQUIRED)하는 경우, 이미 시작된 트랜잭션의 flush mode·커넥션을 바꾸지 않으므로 내부 메서드의 `readOnly`는 사실상 무시된다. 반대로 `REQUIRES_NEW`는 새 트랜잭션이므로 readOnly가 정상 적용된다.

---

## 2. 트랜잭션 전파 속성 (Propagation) 실무 비교

### 2-1. 7가지 전파 속성 요약

| 전파 속성 | 기존 트랜잭션이 있으면 | 없으면 | 물리 트랜잭션 수 | 실무에서 쓰는 곳 |
|---|---|---|---|---|
| `REQUIRED` (기본값) | **참여**한다 | 새로 만든다 | 1 | 대부분의 서비스 메서드 |
| `REQUIRES_NEW` | **보류(suspend)** 하고 새로 만든다 | 새로 만든다 | 2 | 감사 로그·실패 이력·재시도 기록 — 롤백에 휩쓸리면 안 되는 기록 |
| `SUPPORTS` | 참여한다 | 트랜잭션 없이 실행 | 0 또는 1 | 조회 전용 DAO/리포지토리 확장 |
| `NOT_SUPPORTED` | 보류하고 트랜잭션 없이 실행 | 트랜잭션 없이 실행 | 0 | 대용량 배치·외부 API처럼 트랜잭션을 오래 잡으면 안 되는 작업 |
| `MANDATORY` | 참여한다 | **예외**(`IllegalTransactionStateException`) | 1 | "반드시 상위 트랜잭션 안에서만 호출" 계약을 강제 |
| `NEVER` | **예외** | 트랜잭션 없이 실행 | 0 | 트랜잭션 금지 구간(예: 자체 커밋하는 외부 연동) |
| `NESTED` | savepoint를 만들어 중첩 실행 | 새로 만든다 | 1 (savepoint) | JDBC 기반 부분 롤백. JPA와는 궁합이 나쁘다 |

`NESTED`는 `REQUIRES_NEW`와 자주 혼동된다. `NESTED`는 **같은 물리 트랜잭션·같은 커넥션**을 쓰고 savepoint만 찍는다. 따라서 (1) 내부에서 예외가 나면 savepoint까지만 롤백되고 외부 트랜잭션은 계속 진행할 수 있으며, (2) 외부가 롤백되면 내부 작업도 함께 사라진다. `REQUIRES_NEW`와 정반대다. 또 `NESTED`는 savepoint를 지원하는 트랜잭션 매니저에서 `setNestedTransactionAllowed(true)`를 켜야 동작하고, savepoint 기반 중첩 롤백은 JPA 1차 캐시의 상태와 어긋나므로(롤백된 엔티티가 영속성 컨텍스트에 남는다) `JpaTransactionManager`는 중첩 트랜잭션을 지원하지 않는다(`NestedTransactionNotSupportedException`). 실무에서 `NESTED`는 JDBC/MyBatis 계열에서만 쓰인다고 보면 된다.

### 2-2. `REQUIRED` vs `REQUIRES_NEW` — 물리 트랜잭션과 보류

```text
[REQUIRED — 참여]
Thread ─┬─ BEGIN(T1) ────────────────────────────────────────── COMMIT(T1)
        │      └ inner() : 같은 커넥션 재사용
        │                  내부에서 예외 → rollback-only 표시 → 바깥도 결국 롤백
        └ 커넥션 1개

[REQUIRES_NEW — 보류 후 신규]
Thread ─┬─ BEGIN(T1) ─ suspend ───────────────────── resume ─ COMMIT/ROLLBACK(T1)
        │                 └─ BEGIN(T2) ─ COMMIT(T2) ─┘
        │                    (별도 커넥션, 독립 커밋)
        └ 커넥션 2개 (동시 점유)
```

- `REQUIRES_NEW`의 내부 트랜잭션은 **독립적**이다. 내부가 커밋된 뒤 외부가 롤백되어도 내부 데이터는 남는다. 감사 로그·실패 이력처럼 "실패했다는 사실 자체를 남겨야 하는" 요구에 맞는다.
- 대신 **격리**의 대가가 있다. 외부 트랜잭션이 아직 커밋하지 않은 변경은 내부 트랜잭션에서 보이지 않는다. 방금 INSERT한 주문의 PK를 감사 로그에 FK로 걸면 참조 대상이 없어 **FK 위반**이 날 수 있다(그래서 0-2의 `AuditLog`에는 FK를 걸지 않았다). 같은 row를 외부에서 UPDATE한 뒤 내부에서 다시 UPDATE하면, 내부 트랜잭션이 외부가 쥔 row lock을 기다리다 `innodb_lock_wait_timeout`(기본 50초)까지 멈춘다 — **자기 자신과의 락 대기**다.
- `REQUIRED`에서 내부가 실패하면 `globalRollbackOnParticipationFailure = true`(기본값)에 의해 **외부 트랜잭션도 rollback-only로 표시**된다. 내부 예외를 `try/catch`로 삼켜도 바깥 커밋 시점에 `UnexpectedRollbackException`이 터진다(3-3).

### 2-3. `REQUIRES_NEW` 남용 시 커넥션 풀 고갈과 교착

이 부분이 실무에서 가장 자주 사고가 나는 지점이다. `REQUIRES_NEW`는 **한 스레드가 커넥션을 2개 동시에 점유**한다.

```text
HikariCP maximum-pool-size = 10, 동시 요청 스레드 = 10, 모든 요청이 placeOrder() → record()(REQUIRES_NEW)

Thread-1  [T1 커넥션#1 점유] ── 대기 ──▶ 커넥션 없음
Thread-2  [T1 커넥션#2 점유] ── 대기 ──▶ 커넥션 없음
...
Thread-10 [T1 커넥션#10 점유] ── 대기 ──▶ 커넥션 없음
                                     ↑ 10개 전부 자기 자신의 T1에 잡혀 있다

→ connection-timeout(기본 30초) 경과 후
   SQLTransientConnectionException: HikariPool-1 - Connection is not available, request timed out
```

**교착(deadlock)의 정체는 DB 락이 아니라 풀 고갈로 인한 상호 대기**다. 10개 스레드가 모두 "자기가 잡은 커넥션"을 놓지 못한 채 두 번째 커넥션을 기다리므로 아무도 진행하지 못한다. 부하가 낮을 때는 잘 돌다가, 트래픽이 풀 크기에 도달하는 순간 전체가 멈추는 형태라 원인을 찾기 어렵다.

대응 순서는 다음과 같다.

1. **애초에 `REQUIRES_NEW`가 필요한지 재검토한다.** 감사 로그는 커밋 이후 이벤트(`@TransactionalEventListener(AFTER_COMMIT)`)로 옮기면 커넥션 1개로 끝난다. "실패 시에도 기록"이 요구라면 `fallbackExecution`과 `REQUIRES_NEW` 리스너 조합이 더 안전하다.
2. 별도 DataSource(별도 풀)를 쓰는 로깅 경로로 분리한다. 업무 트랜잭션이 로그용 풀을 잡아먹지 않는다.
3. Pool 크기 산정 시 **최대 필요 커넥션 = 동시 요청 수 × 요청당 최대 커넥션 수**로 잡는다. `REQUIRES_NEW`가 상시 경로에 있으면 사실상 절반의 동시성을 포기하는 것이다.
4. 트랜잭션 안에 외부 API 호출을 넣지 않는다. 응답이 1초 걸리는 외부 호출이 트랜잭션 안에 있으면 그 1초 동안 커넥션을 쥐고 있고, `REQUIRES_NEW`까지 겹치면 대기 시간이 곱해진다.

```java
// 파일 경로: src/test/java/study/order/ConnectionPoolExhaustionTest.java
// application.yml의 hikari.maximum-pool-size를 2로 낮추고 실행한다.
// 의도적으로 시간 초과를 만드는 테스트라 CI에는 넣지 않고 로컬 재현용으로만 쓴다.
package study.order;

import static org.assertj.core.api.Assertions.assertThat;

import java.util.ArrayList;
import java.util.List;
import java.util.concurrent.CountDownLatch;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.Future;
import java.util.concurrent.TimeUnit;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import study.order.service.OrderService;

@SpringBootTest
class ConnectionPoolExhaustionTest {

    @Autowired
    private OrderService orderService;

    @Test
    void requiresNewStarvesPool() throws Exception {
        int threads = 2;   // pool size 2, 각 스레드가 커넥션 2개(외부 + REQUIRES_NEW)를 필요로 한다
        ExecutorService executor = Executors.newFixedThreadPool(threads);
        CountDownLatch ready = new CountDownLatch(threads);
        CountDownLatch start = new CountDownLatch(1);
        List<Future<?>> futures = new ArrayList<>();

        for (int i = 0; i < threads; i++) {
            futures.add(executor.submit(() -> {
                ready.countDown();
                start.await();                                // 두 스레드가 동시에 T1을 잡도록 맞춘다
                orderService.placeOrderThenFail(1L, 1_000);    // 내부에서 REQUIRES_NEW
                return null;
            }));
        }
        ready.await();
        start.countDown();

        // 두 스레드 모두 "자기 T1 커넥션"을 쥔 채 두 번째 커넥션을 기다린다.
        // 먼저 도착한 스레드는 성공(롤백)하고, 나머지는 connection-timeout 후 예외로 끝난다.
        long failed = futures.stream().filter(future -> {
            try {
                future.get(10, TimeUnit.SECONDS);
                return false;
            } catch (Exception e) {
                return true;   // ExecutionException(SQLTransientConnectionException) 또는 TimeoutException
            }
        }).count();

        executor.shutdown();
        assertThat(failed).isGreaterThanOrEqualTo(1L);
    }
}
```

### 2-4. 전파 속성이 무시되는 지점

- **프록시 경계 밖 호출**: 자가 호출(3-1)은 전파 속성만의 문제가 아니라 **트랜잭션 자체가 시작되지 않는** 문제다. `REQUIRES_NEW`를 붙였는데 로그가 롤백되는 현상은 십중팔구 자가 호출이다.
- **`@Async` · 별도 스레드**: ThreadLocal 바인딩이 없으므로 전파되지 않고 새 트랜잭션이 시작된다(전파 속성은 "현재 스레드에 트랜잭션이 있는가"를 기준으로 판단한다).
- **이벤트 리스너**: `@TransactionalEventListener`는 기본이 `AFTER_COMMIT`이라 **트랜잭션 밖**에서 실행된다. 따라서 리스너 안의 `@Transactional`은 새 트랜잭션을 연다(전파할 대상이 없다). 리스너에서 DB 쓰기가 필요하면 그 사실을 명시적으로 설계해야 한다.
- **`@Transactional`이 여러 개 겹칠 때**: 클래스 레벨과 메서드 레벨이 함께 있으면 **메서드 레벨이 우선**한다. 인터페이스/구현 클래스에 다르게 붙이면 프록시 종류에 따라 해석이 달라지므로 한 곳에만 붙인다.

---

## 3. 실무 주의사항 및 대표 안티패턴 / Edge Cases

### 3-1. 자가 호출(Self-Invocation) — 가장 흔한 "트랜잭션이 안 걸린다"의 원인

```java
@Service
public class BadService {

    @Transactional
    public void outer() {
        this.inner();     // ❌ 프록시 우회: 트랜잭션 없이 실행된다
    }

    @Transactional(propagation = Propagation.REQUIRES_NEW, rollbackFor = Exception.class)
    public void inner() {
        // REQUIRES_NEW도, rollbackFor도, timeout도 전부 무시된다
    }
}
```

원리는 단순하다. 스프링이 주입하는 것은 `BadService`가 아니라 `BadService$$SpringCGLIB$$0`(프록시)이고, `outer()` 안에서의 `this`는 **프록시가 아니라 대상 객체**다. 프록시의 `outer()`가 호출되면 인터셉터가 트랜잭션을 연 뒤 대상 객체의 `outer()`를 실행하고, 그 안의 `this.inner()`는 인터셉터를 다시 거치지 않는다.

증상으로 알아채는 방법:

- `@Transactional`을 붙였는데 예외를 던져도 데이터가 남는다 / 롤백되지 않는다.
- `REQUIRES_NEW`를 붙였는데 외부 롤백에 함께 사라진다.
- `logging.level.org.springframework.transaction.interceptor=TRACE` 로그에 해당 메서드의 `Creating new transaction` / `Getting transaction for [...]` 줄이 **아예 없다**. (프록시를 안 탔다는 가장 확실한 증거)

### 3-2. 자가 호출 해결책 비교

| 방법 | 동작 | 장점 | 단점 / 주의 |
|---|---|---|---|
| **별도 빈으로 분리 (권장)** | 트랜잭션 경계가 다른 `@Service`로 추출 | 구조가 명확, 테스트 쉬움, 부수효과 없음 | 클래스 수 증가. "책임 분리"라는 본래 목적에도 맞음 |
| Self-Injection (`@Lazy`) | 자기 자신(프록시)을 필드/생성자로 주입 | 코드 변경 최소 | Spring Boot 2.6+ 순환 참조 금지 → `@Lazy` 필수. 자기 자신에게 의존하는 설계라 응집도가 나빠짐 |
| `ObjectProvider<Self>` | 주입 시점이 아닌 호출 시점에 빈 획득 | 순환 참조 회피, 지연 조회 | 여전히 "자기 자신 호출"이라는 구조적 냄새 |
| `AopContext.currentProxy()` | 현재 프록시를 꺼내 재호출 | 즉시 적용 | `exposeProxy = true` 필요, 스프링 API가 비즈니스 코드에 침투, 캐스팅 필요, 프록시 없는 호출에서는 예외 |
| `TransactionTemplate` | 코드로 트랜잭션 경계 지정 | 프록시와 무관, 경계가 눈에 보임, `REQUIRES_NEW`/timeout 지정 쉬움 | 선언적이지 않아 코드가 길어짐. 중첩 시 가독성 저하 |
| AspectJ 모드 (`mode = ASPECTJ`) | 프록시 없이 바이트코드 자체를 위빙 | 자가 호출도 적용, `private`도 가능 | 빌드에 aspectjweaver/컴파일 위빙 필요, 러닝커브·운영 복잡도 큼. 실무에서는 거의 쓰지 않음 |

**결론**: 새 코드는 "별도 빈 분리"가 기본이고, 불가피할 때 `TransactionTemplate` 또는 `ObjectProvider`를 쓴다. `AopContext`와 AspectJ는 기존 코드를 고칠 수 없을 때의 최후 수단이다.

### 3-3. 롤백 대상 예외 정책 (Checked vs Unchecked)

| 예외 종류 | 예: | Spring 기본 동작 |
|---|---|---|
| `RuntimeException` 및 하위 | `IllegalStateException`, `IllegalArgumentException` | **롤백** |
| `Error` 및 하위 | `OutOfMemoryError` | **롤백** |
| Checked `Exception` | `IOException`, `SQLException`(체크 예외로 선언된 경우) | **커밋** (롤백하지 않음) |

```java
// 파일 경로: src/main/java/study/order/service/CheckedRollbackService.java
package study.order.service;

import java.io.IOException;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
import study.order.domain.Order;
import study.order.repository.OrderRepository;

@Service
public class CheckedRollbackService {

    private final OrderRepository orderRepository;

    public CheckedRollbackService(OrderRepository orderRepository) {
        this.orderRepository = orderRepository;
    }

    /**
     * Checked Exception은 기본적으로 롤백 대상이 아니다.
     * → 예외가 나도 주문이 커밋되어 남는다. (부분 반영 사고의 전형)
     */
    @Transactional
    public void placeOrderWithoutRollbackFor(Long userId) throws IOException {
        orderRepository.save(new Order(userId, 10_000));
        throw new IOException("외부 저장소 전송 실패");   // 커밋된다
    }

    /**
     * 실무 기본형: rollbackFor = Exception.class 로 checked까지 롤백 대상에 넣는다.
     * noRollbackFor 로 "이 예외는 남기고 싶다"를 예외 처리할 수 있다.
     */
    @Transactional(rollbackFor = Exception.class, noRollbackFor = IllegalArgumentException.class)
    public void placeOrderWithRollbackFor(Long userId) throws IOException {
        orderRepository.save(new Order(userId, 10_000));
        throw new IOException("외부 저장소 전송 실패");   // 롤백된다
    }
}
```

실무에서 함께 알아야 할 롤백 관련 함정들:

- **`try/catch`로 내부 예외를 삼켜도 롤백은 이미 예약된다.** 내부 `REQUIRED` 메서드에서 `RuntimeException`이 나면 그 시점에 트랜잭션이 rollback-only로 표시되고, 바깥에서 예외를 잡아 정상 반환해도 커밋 시 `UnexpectedRollbackException: Transaction silently rolled back because it has been marked as rollback-only`가 발생한다. "일부만 실패해도 나머지는 저장하고 싶다"면 (a) 실패 가능한 부분을 `REQUIRES_NEW`/별도 빈으로 분리하거나, (b) 검증을 먼저 끝내 예외를 만들지 않는 구조로 바꾼다.
- **JPA 예외는 대부분 `PersistenceException`(런타임)이라 롤백 대상**이지만, 문제는 그 뒤다. 한 번 예외가 난 영속성 컨텍스트는 신뢰할 수 없으므로 같은 트랜잭션에서 계속 작업하면 안 된다.
- **`rollbackFor`는 "예외를 잡아서 처리"와 다르다.** 롤백 규칙은 프록시가 예외를 위로 던질 때만 평가된다. 메서드 안에서 모든 예외를 catch해 정상 반환하면 롤백 규칙은 아무 일도 하지 않는다.
- **`readOnly = true`에서의 쓰기는 롤백 대상도 아니다.** flush가 생략되어 조용히 무시된다. 검증 로직이 없다면 로그 한 줄 없이 데이터가 사라진 것처럼 보인다.

### 3-4. `@Async` · 멀티스레드 분기에서의 트랜잭션 단절

```java
// 파일 경로: src/main/java/study/order/service/AsyncOrderService.java
package study.order.service;

import java.util.concurrent.CompletableFuture;
import org.springframework.scheduling.annotation.Async;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
import study.order.domain.Order;
import study.order.repository.OrderRepository;

@Service
public class AsyncOrderService {

    private final OrderRepository orderRepository;

    public AsyncOrderService(OrderRepository orderRepository) {
        this.orderRepository = orderRepository;
    }

    @Transactional
    public void placeOrder(Long userId) {
        orderRepository.save(new Order(userId, 10_000));

        // ❌ @Async 메서드를 같은 클래스에서 호출 → @Async도 자가 호출로 무력화된다(동기 실행).
        //    설령 프록시를 통해 비동기로 실행되더라도 ThreadLocal이 넘어가지 않아
        //    별도 스레드는 "트랜잭션 없음" 상태가 되고, 새 커넥션을 잡는다.
        this.sendMailAsync(userId);
    }

    @Async
    public CompletableFuture<Void> sendMailAsync(Long userId) {
        // 이 스레드에는 외부 트랜잭션의 커넥션이 바인딩되어 있지 않다.
        // 여기서 DB를 읽으면 외부 트랜잭션이 아직 커밋하지 않은 데이터는 보이지 않는다.
        return CompletableFuture.completedFuture(null);
    }
}
```

- `@Async`도 **프록시 기반**이라 자가 호출에 똑같이 무력화된다. `@Async`가 안 먹는 것 같으면 먼저 이걸 의심한다.
- 비동기 스레드로 넘어가면 `TransactionSynchronizationManager`의 ThreadLocal, `SecurityContext`, MDC(로그 추적 ID)가 모두 사라진다. 필요하면 `TaskDecorator`로 명시적으로 복사한다. **트랜잭션은 복사하면 안 된다**(다른 스레드가 같은 커넥션을 동시에 쓰게 된다).
- "커밋 이후 비동기 작업"이 목적이라면 `@Async` + `@TransactionalEventListener(AFTER_COMMIT)` 조합이 정석이다. 커밋 전에 비동기로 던지면 **롤백되었는데 메일이 나가는** 사고가 난다.

### 3-5. 그 밖의 엣지 케이스

- **트랜잭션 안의 외부 I/O**: HTTP 호출·파일 업로드·메일 발송을 트랜잭션 안에 두면 그 시간만큼 커넥션과 row lock을 쥔다. 실패 시 재시도까지 겹치면 락 대기 시간이 급격히 늘어난다. 외부 호출은 커밋 이후로 미룬다.
- **OSIV(Open Session In View)**: Spring Boot 기본값은 `true`다. 트랜잭션이 끝난 뒤 뷰 렌더링까지 `EntityManager`를 열어 두므로 (1) 지연 로딩이 "되는 것처럼" 보여 N+1을 숨기고, (2) 커넥션 반환 시점이 늦어져 풀 사용 시간이 길어진다. `spring.jpa.open-in-view=false`로 끄고 필요한 데이터를 트랜잭션 안에서 DTO로 조립하는 편이 낫다.
- **`@Transactional`과 `@Cacheable`의 순서**: 둘 다 프록시 어드바이스다. 캐시가 트랜잭션보다 바깥에 있으면 커밋 전 값이 캐시에 들어갈 수 있다. 캐시는 커밋 이후에 넣거나 `@TransactionalEventListener`로 분리한다.
- **`timeout`은 "트랜잭션 시작부터" 잰다**: `@Transactional(timeout = 3)`은 커넥션 획득 대기 시간도 포함한다. 풀이 고갈된 상황에서는 timeout이 먼저 터져 원인을 가릴 수 있다.
- **배치 작업의 트랜잭션 크기**: 수만 건을 한 트랜잭션에 넣으면 undo log·락·메모리가 함께 커진다. 청크 단위(예: 1,000건)로 커밋하고, 청크 경계를 `TransactionTemplate`으로 명시하면 재시작 지점도 명확해진다.
- **테스트에서의 함정**: 테스트 클래스에 `@Transactional`을 붙이는 관례(테스트 롤백)는 전파 속성 검증과 충돌한다. 테스트 트랜잭션이 외부 트랜잭션이 되어 `REQUIRED`는 그 안에 참여하지만, `REQUIRES_NEW`로 커밋한 데이터는 **테스트가 끝나도 남아** 다음 테스트를 오염시킨다. 전파 속성을 검증할 때는 테스트에 `@Transactional`을 붙이지 않고, `@BeforeEach`에서 데이터를 정리하거나 `@Sql`로 초기화한다.
- **Kotlin/`final` 클래스**: Java에서는 드물지만 Kotlin 클래스는 기본이 `final`이라 `@Transactional`이 붙은 클래스에 `all-open`/`kotlin-spring` 플러그인이 없으면 프록시 생성이 실패한다.

### 3-6. 아그작(AGZAK) 관점 메모

- **Redis와 RDB는 한 트랜잭션에 묶을 수 없다.** Redis ZSET 랭킹 갱신과 RDB 점수 저장을 하나의 원자적 단위로 만들려 하면 `REQUIRES_NEW`를 아무리 써도 해결되지 않는다(다른 리소스 매니저). RDB 커밋 이후 `@TransactionalEventListener(AFTER_COMMIT)`로 Redis를 갱신하고, 실패 시 재처리(보상/재시도 큐)를 두는 구조가 현실적이다. 정합성이 더 중요하면 아웃박스(outbox) 테이블에 이벤트를 적재하고 별도 워커가 발행한다.
- **ShedLock 배치의 트랜잭션 경계**: 락을 잡은 채로 긴 트랜잭션을 열면 다른 인스턴스가 오래 대기한다. 락 획득 → 청크 단위 트랜잭션 → 락 해제 순서로 잡고, 배치 메서드 전체에 `@Transactional`을 걸지 않는다.
- **JWT 인증 후 사용자별 조회 API**: 대부분 `@Transactional(readOnly = true)`가 맞다. 다만 readOnly를 "쓰기 방지"로 믿지 말고, 쓰기 경로와 읽기 경로를 메서드/클래스 단위로 분리해 경계를 명확히 한다.

---

## 4. 기술 면접 단골 꼬리질문 5선 & 모범 답변

**Q1. 동일 클래스 내부에서 `@Transactional` 메서드를 호출하면 왜 트랜잭션이 적용되지 않나요?**
- **A.** `@Transactional`은 스프링 AOP의 프록시 기반 구현입니다. 컨테이너는 대상 빈을 감싼 프록시를 등록하고, `TransactionInterceptor`가 **프록시로 들어오는 호출**을 가로채 `PlatformTransactionManager`로 트랜잭션을 시작·커밋합니다. 그런데 같은 클래스 안에서 `this.inner()`로 호출하면 `this`는 프록시가 아니라 대상 객체이므로 인터셉터를 거치지 않고, 트랜잭션 코드가 실행될 기회 자체가 없습니다. 그래서 `REQUIRES_NEW`, `rollbackFor`, `timeout` 같은 속성도 전부 무시됩니다. 해결은 (1) 트랜잭션 경계를 별도 빈으로 분리하는 것이 가장 권장되고, (2) `@Lazy` 자기 주입이나 `ObjectProvider`로 프록시를 다시 호출하거나, (3) `TransactionTemplate`으로 경계를 코드에 명시하거나, (4) `AopContext.currentProxy()`(+`exposeProxy=true`)를 쓰는 방법이 있습니다. AspectJ 모드로 위빙하면 프록시 자체를 없앨 수 있지만 빌드 복잡도가 커서 실무에서는 거의 쓰지 않습니다. 참고로 `@Async`, `@Cacheable`도 같은 이유로 자가 호출에서 무력화됩니다.

**Q2. `@Transactional(readOnly = true)`를 설정하면 구체적으로 어떤 성능 최적화가 일어나나요?**
- **A.** 세 계층에서 효과가 나옵니다. 첫째, JPA/Hibernate에서 `JpaTransactionManager`가 새 트랜잭션의 `FlushMode`를 `MANUAL`로 바꿉니다. 그래서 커밋 시 자동 flush와, 쿼리 실행 직전의 auto-flush(더티 체킹)가 사라집니다. 둘째, 세션을 read-only로 설정해 로딩한 엔티티의 **스냅샷(loaded state)을 보관하지 않습니다.** 더티 체킹은 스냅샷과 현재 값을 비교하는 작업이므로, 대량 조회에서 엔티티 수만큼의 메모리와 비교 CPU가 절약됩니다. 셋째, JDBC `Connection.setReadOnly(true)`가 적용되어 드라이버·프록시가 읽기 전용 힌트로 활용하고, `TransactionSynchronizationManager.isCurrentTransactionReadOnly()`를 보고 `AbstractRoutingDataSource`로 **Replica 노드로 라우팅**할 수 있습니다. 다만 두 가지 주의가 필요합니다. 라우팅은 트랜잭션 시작 시점에 커넥션을 얻으면 read-only 플래그가 아직 세팅되지 않으므로 `LazyConnectionDataSourceProxy`로 획득을 첫 SQL까지 미뤄야 합니다. 그리고 readOnly는 강제가 아니라 힌트라서, 이 트랜잭션에서 엔티티를 수정하면 예외 없이 조용히 무시될 수 있습니다.

**Q3. `REQUIRED`와 `REQUIRES_NEW`의 차이는 무엇이며, 외부 트랜잭션 롤백 시 내부 트랜잭션의 커밋 여부는 어떻게 결정되나요?**
- **A.** `REQUIRED`는 기존 트랜잭션이 있으면 **같은 물리 트랜잭션에 참여**하고, 없을 때만 새로 만듭니다. 커넥션 1개를 공유하므로 커밋·롤백이 맨 바깥 경계에서 한 번만 일어납니다. 따라서 내부에서 예외가 나면 (기본 설정인 `globalRollbackOnParticipationFailure=true`에 의해) 트랜잭션이 rollback-only로 표시되고, 바깥에서 예외를 잡아도 최종 커밋 시 `UnexpectedRollbackException`이 발생합니다. `REQUIRES_NEW`는 기존 트랜잭션을 **보류(suspend)** 하고 별도 커넥션으로 새 물리 트랜잭션을 엽니다. 그래서 내부 커밋은 독립적이며, 내부가 커밋된 뒤 외부가 롤백되어도 내부 데이터는 남습니다. 반대로 내부가 롤백되어도 외부는 계속 진행할 수 있습니다. 대가는 두 가지입니다. 한 스레드가 커넥션을 2개 동시에 점유하므로 동시 요청이 풀 크기에 가까워지면 **풀 고갈로 사실상 교착**이 발생하고, 보류된 외부 트랜잭션의 미커밋 변경은 내부에서 보이지 않아 **FK 위반이나 자기 자신과의 row lock 대기**가 생길 수 있습니다. 그래서 `REQUIRES_NEW`는 감사 로그·실패 이력처럼 "롤백에 휩쓸리면 안 되는 짧은 기록"에 한정하고, 가능하면 `AFTER_COMMIT` 이벤트로 대체합니다.

**Q4. Checked Exception이 발생했을 때 왜 Spring은 기본적으로 롤백하지 않으며, 실무에서는 어떻게 대처하나요?**
- **A.** 스프링의 기본 롤백 규칙은 `RuntimeException`과 `Error`만 롤백합니다. 이는 EJB의 관례를 따른 것으로, **checked exception은 "호출자가 복구할 수 있다고 선언한 예외"** 로 보기 때문입니다. 즉 "복구 가능한 상황이니 지금까지의 작업을 유지하라"는 해석입니다. 하지만 실무에서는 checked exception도 대부분 부분 실패를 남기면 안 되는 상황이고, `IOException` 하나로 주문이 커밋되는 사고가 납니다. 그래서 (1) `@Transactional(rollbackFor = Exception.class)`로 롤백 대상을 명시하고, (2) 정말로 남겨야 하는 예외만 `noRollbackFor`로 예외 처리하며, (3) 예외를 삼키는 `try/catch`를 트랜잭션 경계 안에 두지 않습니다. 또 JPA에서는 대부분의 예외가 `PersistenceException`(런타임)이라 롤백 대상이지만, 한 번 예외가 난 영속성 컨텍스트는 신뢰할 수 없으므로 같은 트랜잭션에서 계속 작업하지 않고 즉시 종료합니다.

**Q5. 트랜잭션 커밋이 성공한 후에만 외부 알림(SMS/메일) 발송이나 Kafka 메시지를 발행하려면 어떻게 해야 하나요?**
- **A.** `ApplicationEventPublisher`로 이벤트를 발행하고, 리스너에 `@TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)`을 붙입니다. 그러면 커밋이 성공한 뒤에만 리스너가 실행되고, 롤백되면 실행되지 않습니다. 응답 지연을 줄이려면 `@Async`를 함께 붙이되, `@Async`도 프록시 기반이라 자가 호출로 무력화되지 않도록 별도 빈에 두어야 합니다. 세 가지 함정이 있습니다. 첫째, 기본값 `fallbackExecution = false`라서 **트랜잭션 없이 발행된 이벤트는 조용히 무시**됩니다. 둘째, `AFTER_COMMIT`은 이미 커밋이 끝난 시점이므로 리스너에서 예외가 나도 **롤백할 수 없고**, 예외가 호출자에게 전파되어 API가 500으로 보일 수 있습니다. 셋째, 리스너는 트랜잭션 밖이므로 그 안의 DB 쓰기는 새 트랜잭션이 됩니다. 메시지 유실이 허용되지 않는 경우(Kafka 발행 등)에는 `AFTER_COMMIT`만으로는 부족합니다. 커밋과 발행 사이에 프로세스가 죽으면 이벤트가 사라지기 때문입니다. 이때는 **아웃박스 패턴**으로 같은 트랜잭션에 이벤트 레코드를 적재하고, 별도 워커가 폴링·발행하며 멱등 키로 중복을 제거합니다. 반대로 "커밋 전에 알림이 나가는 것"이 더 큰 문제인 경우(결제 실패인데 메일 발송)에는 `AFTER_COMMIT`이 최소 기준입니다.

---

## 5. 오늘의 체크리스트 (내일 이어서)

- [ ] `logging.level.org.springframework.transaction.interceptor=TRACE`로 `REQUIRED`/`REQUIRES_NEW`/`NESTED` 각각의 로그를 캡처해 "Suspend/Resume" 지점 확인
- [ ] `maximum-pool-size=2`에서 0-3의 `REQUIRES_NEW` 경로를 동시 요청 2개로 돌려 `SQLTransientConnectionException` 재현 및 소요 시간 기록
- [ ] `REQUIRES_NEW` 내부에서 외부 트랜잭션이 INSERT한 row를 FK로 참조하는 케이스 재현 → FK 위반 확인
- [ ] `REQUIRED` 내부 예외를 바깥에서 `try/catch`로 삼켰을 때 `UnexpectedRollbackException` 발생 여부 확인
- [ ] `readOnly = true`에서 엔티티를 수정했을 때 flush가 생략되어 조용히 무시되는지 확인 (`SQL` 로그에 UPDATE 부재)
- [ ] `LazyConnectionDataSourceProxy` 유무에 따라 Replica 라우팅이 실제로 달라지는지 확인 (커넥션 URL 로그)
- [ ] `spring.jpa.open-in-view=false`로 전환했을 때 기존 조회 API에서 `LazyInitializationException`이 나는 지점 목록화
- [ ] `@Async` + `AFTER_COMMIT` 조합에서 커밋 지연·롤백 시 알림 미발송을 테스트로 고정
- [ ] 자가 호출 해결 4가지(분리/`@Lazy`/`ObjectProvider`/`TransactionTemplate`)를 같은 시나리오로 비교하고, 어떤 것을 표준으로 삼을지 결정
- [ ] `@Transactional`이 붙은 테스트에서 `REQUIRES_NEW` 커밋 데이터가 다음 테스트를 오염시키는 문제 정리 (`@Sql` 정리 또는 `@DirtiesContext` 비용 비교)
