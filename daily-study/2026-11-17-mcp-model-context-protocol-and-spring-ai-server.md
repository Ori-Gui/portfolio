# [2026-11-17] MCP(Model Context Protocol) — 호스트·클라이언트·서버 구조부터 2026-07-28 무상태 개정, Spring AI 2.0 도구 서버까지

> **한 줄 핵심 요약**: MCP는 LLM 애플리케이션과 외부 도구·데이터의 연결을 JSON-RPC 2.0 표준 하나로 고정한 프로토콜입니다. 실무에서 중요한 건 세 가지입니다. ① 도구는 **모델이 스스로 고르는(model-controlled)** 기능이라 스키마와 설명이 곧 프롬프트 비용이자 정확도이고, ② 2026-07-28 개정으로 **핸드셰이크와 세션이 사라진 무상태 프로토콜**이 되어 로드밸런서 뒤 수평 확장이 기본 전제가 되었으며, ③ 쓰기 도구는 **언제든 재시도되는 API**로 취급해 멱등성·권한·사람 확인을 서버 쪽에서 강제해야 합니다.

---

## 1. 개념 및 동작 원리

### 1.1 function calling을 "표준 부품"으로 분리한 것

LLM의 function calling 자체는 MCP 이전에도 있었습니다. 문제는 **연결 방식이 벤더·앱마다 달라진다**는 점이었습니다. 도구 N개를 앱 M개에서 쓰려면 N×M개의 어댑터를 각자 구현해야 했고, 도구 스키마는 앱 코드 안에 파묻혀 다른 앱에서 재사용할 수 없었습니다.

MCP는 이 어댑터를 **프로세스/서비스 경계 밖으로** 끌어냅니다. 도구 제공자는 "MCP 서버"를 한 번 만들고, 사용하는 쪽(Claude Desktop, IDE, 사내 agent)은 "MCP 클라이언트"를 붙이기만 합니다.

| 항목 | 벤더 function calling | MCP |
| --- | --- | --- |
| 통합 단위 | 앱 코드 안의 함수 | 독립 프로세스 또는 원격 서비스 |
| 재사용 | 다른 앱에서 쓰려면 다시 구현 | 같은 서버에 어떤 호스트든 접속 |
| 스키마 | 벤더 SDK별 JSON 스키마 | `tools/list`가 주는 JSON Schema 2020-12 |
| 실행 위치 | 호스트 프로세스 내부 | stdio 자식 프로세스 또는 원격 HTTP |
| 권한 | 앱 코드에 하드코딩 | OAuth 스코프 + 호스트의 동의 UI |
| 변경 반영 | 재배포 | `notifications/tools/list_changed` |

- **MCP는 모델을 호출하는 프로토콜이 아닙니다.** 스펙의 정의도 "컨텍스트 교환(context exchange)에 집중한 프로토콜"입니다. 어떤 모델을 쓸지, 프롬프트를 어떻게 조립할지, 도구 호출을 승인할지는 전부 **호스트**의 책임입니다. 이 경계를 먼저 잡아야 "MCP 서버에 비즈니스 판단을 넣지 않는다"는 원칙이 자연스럽게 따라옵니다.

### 1.2 호스트–클라이언트–서버: 1:1 연결과 경계

```text
┌──────────────────── Host (Claude Desktop, IDE, 사내 agent) ────────────────────┐
│  LLM + 대화 상태 + 동의 UI + 정책                                              │
│    │                                                                           │
│    ├─ Client A ──(1:1)──▶ Server A : filesystem   (stdio, 로컬 자식 프로세스)   │
│    ├─ Client B ──(1:1)──▶ Server B : order-service (Streamable HTTP, 사내 k8s) │
│    └─ Client C ──(1:1)──▶ Server C : github        (원격, OAuth 2.1)           │
└────────────────────────────────────────────────────────────────────────────────┘
```

- **호스트**: 클라이언트 인스턴스를 만들고 수명·권한·동의를 관리합니다. "이 도구를 실행해도 되는가"의 최종 판단 주체입니다.
- **클라이언트**: 서버 **정확히 1개**와만 통신합니다. 요청마다 프로토콜 버전과 자신의 capability를 실어 보냅니다.
- **서버**: 좁고 잘 정의된 기능만 노출합니다. **대화 전체를 볼 수 없고, 다른 서버를 들여다볼 수도 없습니다.**

이 격리는 보안 설계의 근간입니다. 예를 들어 사내 DB MCP 서버는 사용자의 전체 대화를 볼 수 없고, "이 주문을 취소해줘"라는 도구 인자와 자기 스코프 안의 데이터만 봅니다. 사고가 나도 **블라스트 반경(blast radius)이 그 서버의 권한 안으로 갇힙니다.**

### 1.3 프리미티브 — 누가 제어하는지가 설계 기준

| 프리미티브 | 제어 주체 | 형태 | 예시 |
| --- | --- | --- | --- |
| **tools** | 모델(model-controlled) | `name` + `inputSchema`(+`outputSchema`, `annotations`) | `get_order`, `create_issue` |
| **resources** | 애플리케이션(호스트) | URI + MIME 타입 | `file:///project/src/main.rs`, `config://db` |
| **prompts** | 사용자(user-controlled) | 이름 있는 템플릿 | `/summarize-pr`, `/review-checklist` |
| **elicitation** | 서버 → 사용자 | `InputRequiredResult` 안의 입력 요청 | "주문 ID가 필요합니다" |
| **utilities** | 양방향 | progress, completion, (확장) tasks | 긴 작업 진행률, 인자 자동완성 |

- **선택 기준**: 부작용이 있고 인자가 필요한 동작이면 tool, 읽기 전용 컨텍스트를 그대로 주입하는 것이면 resource, 사람이 명시적으로 골라 실행하는 절차면 prompt입니다.
- 실무에서 대부분의 MCP 서버는 **tool만** 노출합니다. resource는 "호스트가 알아서 읽어가는 파일·문서"에, prompt는 "슬래시 커맨드"가 필요한 제품에서 씁니다.
- 2026-07-28 개정에서 **roots·sampling·logging은 Deprecated**가 되었습니다. 신규 구현은 각각 도구 인자/서버 설정, LLM API 직접 호출, stderr·OpenTelemetry로 대체합니다. 기존 자료를 보고 따라 만들면 이미 Deprecated된 기능을 배우기 쉬운 지점입니다.

### 1.4 전송: stdio와 Streamable HTTP

| 항목 | stdio | Streamable HTTP |
| --- | --- | --- |
| 실행 형태 | 호스트가 자식 프로세스를 실행 | 독립 서비스(다중 클라이언트) |
| 프레이밍 | 줄바꿈으로 구분된 JSON-RPC | 메시지 1건 = HTTP POST 1건, 응답은 JSON 1개 또는 요청 스코프 SSE |
| 인증 | 프로세스 환경변수 | OAuth 2.1 Bearer 토큰 |
| 적합한 곳 | 로컬 도구(파일·git·로컬 DB 터널) | 다중 사용자·원격 서비스 |
| 주의 | **stdout은 프로토콜 프레임 전용** | `Origin` 검증, `MCP-Protocol-Version` 헤더, GET/DELETE는 405 |

2026-07-28의 Streamable HTTP는 이전 리비전과 모양이 다릅니다.

- **POST 단일 엔드포인트**만 둡니다. 클라이언트는 `Accept: application/json, text/event-stream`을 모두 보내야 하고, 서버는 둘 중 아무 형식으로 응답해도 됩니다(클라이언트가 둘 다 지원해야 함).
- **GET 스트림이 제거**되어 서버가 먼저 말을 거는 통로가 없습니다. 변경 알림은 `subscriptions/listen` 요청의 응답 스트림 하나로만 흐릅니다.
- **세션 ID가 없습니다.** `Mcp-Session-Id`는 무시되고, GET/DELETE는 `405 Method Not Allowed`가 정상 응답입니다.
- **스트림 재개가 없습니다.** `Last-Event-ID`를 지원하지 않으므로 응답 스트림이 끊기면 그 요청은 유실이고, 클라이언트는 **새 요청 ID로 다시 보내야 합니다.** → 서버 도구의 멱등성이 선택이 아니라 필수가 되는 이유입니다.
- 프록시 뒤에 둘 때는 `X-Accel-Buffering: no`(nginx 버퍼링 해제)와 주기적 SSE 주석(`:`) keep-alive가 필요합니다. 안 그러면 이벤트가 뭉쳐서 도착합니다.

### 1.5 2026-07-28의 핵심: 무상태화

| 항목 | 2025-11-25까지 | 2026-07-28 |
| --- | --- | --- |
| 핸드셰이크 | `initialize` / `notifications/initialized` | 없음. 요청마다 `_meta`에 `protocolVersion`·`clientCapabilities` |
| 세션 | `Mcp-Session-Id` 헤더 | 없음. 상태는 **서버가 발급한 핸들**을 도구 인자로 전달 |
| 버전·기능 발견 | `initialize` 응답 | `server/discover` (필수 구현 RPC) |
| 목록 API | 연결별로 달라질 수 있음 | 연결과 무관, **권한(스코프)에 따라서만** 달라짐 |
| 목록 캐시 | 없음 | `ttlMs` + `cacheScope` 필수, 결정적 정렬 권장 |
| 결과 타입 | 없음 | `resultType: "complete" \| "input_required"` |
| 스트림 재개 | `Last-Event-ID` | 없음(재발행) |

- **이득**: sticky session 없이 로드밸런서·오토스케일·서버리스에 올릴 수 있고, 요청 하나만 보고 처리하므로 재시도가 단순해집니다. 게이트웨이가 목록 응답을 캐싱할 여지도 생깁니다.
- **비용**: 매 요청에 메타데이터가 중복되고, 처리 중 스트림이 끊기면 같은 요청을 다시 보내게 됩니다. 그래서 "상태는 서버 메모리에" 두는 설계가 금지되고, **핸들(예: `checkout_session`)을 도구 인자로 명시**해야 합니다.
- Java 관점에서는 오히려 잘 맞습니다. 상태가 없으니 요청당 스레드 하나로 처리하는 구조(Virtual Threads)와 조합이 쉽고, 세션 친화성 때문에 특정 파드로 몰리는 문제를 걱정하지 않아도 됩니다.

### 1.6 MRTR(Multi Round-Trip Requests): 서버가 사용자에게 묻는 방법

무상태가 되면서 "서버가 클라이언트로 별도 요청을 보내는" 방식이 사라졌습니다. 대신 **응답 안에 질문을 담아 보내고, 클라이언트가 원래 요청을 다시 보내면서 답을 함께 실어 주는** 패턴을 씁니다.

```json
// 1) 최초 요청에 정보가 부족 → 서버가 input_required로 응답
{
  "jsonrpc": "2.0", "id": 2, "method": "tools/call",
  "params": { "name": "cancel_order", "arguments": { "reasonCode": "CUSTOMER_REQUEST" } }
}

{
  "jsonrpc": "2.0", "id": 2,
  "result": {
    "resultType": "input_required",
    "inputRequests": {
      "order_id": {
        "method": "elicitation/create",
        "params": {
          "mode": "form",
          "message": "취소할 주문 ID를 입력해 주세요",
          "requestedSchema": {
            "type": "object",
            "properties": { "value": { "type": "string" } },
            "required": ["value"]
          }
        }
      }
    },
    "requestState": "eyJyZWFzb24iOiJDVVNUT01FUl9SRVFVRVNUIn0..."
  }
}

// 2) 같은 요청을 '다른 id'로 재전송하며 inputResponses와 requestState를 함께 전달
{
  "jsonrpc": "2.0", "id": 3, "method": "tools/call",
  "params": {
    "name": "cancel_order",
    "arguments": { "reasonCode": "CUSTOMER_REQUEST" },
    "inputResponses": { "order_id": { "action": "accept", "content": { "value": "6f1c..." } } },
    "requestState": "eyJyZWFzb24iOiJDVVNUT01FUl9SRVFVRVNUIn0..."
  }
}
```

- **재시도는 반드시 새 JSON-RPC `id`로** 보내야 합니다. 같은 id를 재사용하면 중복 요청으로 취급될 수 있습니다.
- `requestState`는 서버가 1차 요청의 문맥을 스스로 복원하기 위한 값입니다. 무상태 서버에서 "아까 그 요청"을 이어 붙이는 유일한 합법적 통로이므로, **서명 또는 암호화해서 변조를 막아야** 합니다(사용자가 `order_id`를 바꿔 넣는 것은 정상이지만, `reason` 같은 서버 결정 값을 바꾸게 두면 안 됩니다).

### 1.7 인가: MCP 서버는 OAuth 2.1 리소스 서버

- 전송 계층 인가는 HTTP 전송에서만 의미가 있습니다(**stdio는 대상 아님**, 자격증명은 환경변수).
- MCP 서버는 **RFC 9728(보호 리소스 메타데이터)** 를 구현하고, 미인증 요청에는 `401` + `WWW-Authenticate: Bearer resource_metadata="...", scope="..."`로 응답합니다. 클라이언트는 그 메타데이터로 인가 서버를 찾아갑니다.
- 토큰 요청에는 **RFC 8707 `resource` 지시자**를 넣어 "이 토큰은 이 MCP 서버 전용"임을 묶습니다. 이게 없으면 다른 서비스용 토큰이 재사용될 수 있습니다.
- 인가 응답의 `iss`(**RFC 9207**)를 기록된 발급자와 대조해야 합니다(코드 탈취 방지).
- **동적 클라이언트 등록(DCR)은 2026-07-28에서 deprecated**입니다. Client ID Metadata Documents가 우선이고, DCR은 호환용으로만 남습니다.
- **토큰 그대로 넘기기(token passthrough) 금지**: MCP 서버가 받은 토큰을 하위 API에 그대로 전달하면, 하위 API 입장에서 감사 로그의 주체가 뒤섞이고 스코프가 무의미해집니다. 서버가 자기 자격으로 교환해 호출해야 합니다.

### 1.8 생태계와 Java 진영 현황

- 2024-11 Anthropic이 공개 → 2025-03 OpenAI·이후 Google·Microsoft·Cursor가 채택 → **2025-12-09 Linux Foundation 산하 Agentic AI Foundation(AAIF)** 으로 이관(Anthropic·Block·OpenAI 공동 창립, AWS·Google·Microsoft·Cloudflare·Bloomberg 후원). 이관 시점 기준 공개 MCP 서버 1만 개 이상, ChatGPT·Cursor·Gemini·Microsoft Copilot·VS Code가 채택했습니다.
- 스펙 리비전: `2024-11-05`(HTTP+SSE) → `2025-03-26`(Streamable HTTP) → `2025-06-18` → `2025-11-25` → **`2026-07-28`(무상태, 현재 안정판)**.
- Java: **MCP Java SDK 2.2.0**(2.x 라인이 2026-07-28 개정 대응), **Spring AI 2.0.2**.
- **버전 매트릭스가 실무 함정입니다.**

| 스택 | Spring Boot | MCP 서버 구현 방식 |
| --- | --- | --- |
| Spring AI `2.0.x` | Boot `4.0.x` / `4.1.x` (Framework 7) | `@McpTool` + annotation scanner, starter가 자동 등록 |
| Spring AI `1.0.x` | Boot `3.4.x` / `3.5.x` | `@Tool` + `MethodToolCallbackProvider` 수동 등록 |

기존 Boot 3 프로젝트에 MCP를 붙이려면 1.0.x 라인을 쓰거나 Boot 4로 올려야 합니다. 남의 블로그 코드를 복사할 때 import 패키지(`org.springframework.ai.mcp.annotation.*`는 2.0 계열)부터 확인해야 하는 이유입니다.

---

## 2. 실무 적용 및 주의사항 (Edge Cases)

### 2.1 원칙: tool은 어댑터, 트랜잭션은 서비스

MCP 도구 메서드에 도메인 로직을 넣으면 세 가지가 동시에 무너집니다. 재사용(다른 진입점에서 못 씀), 트랜잭션 경계(프록시를 타지 않는 내부 호출), 테스트 가능성(모델이 읽을 문구가 로직에 섞임).

```text
Controller / MCP Tool  ← 입력 파싱·검증·결과 크기 제한·오류 문구 변환
        ↓
     Service           ← 도메인 규칙 + 트랜잭션 경계
        ↓
    Repository
```

### 2.2 의존성과 설정 (Spring Boot 4.0 + Spring AI 2.0.2 + Java 21)

```xml
<!-- pom.xml -->
<dependencyManagement>
  <dependencies>
    <dependency>
      <groupId>org.springframework.ai</groupId>
      <artifactId>spring-ai-bom</artifactId>
      <version>2.0.2</version>
      <type>pom</type>
      <scope>import</scope>
    </dependency>
  </dependencies>
</dependencyManagement>

<dependencies>
  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
  </dependency>
  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-jpa</artifactId>
  </dependency>
  <dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-starter-mcp-server-webmvc</artifactId>
  </dependency>
</dependencies>
```

```yaml
# src/main/resources/application.yml
spring:
  application:
    name: order-mcp-server
  main:
    banner-mode: "off"          # stdout 오염 방지(stdio 전송을 함께 쓸 때 특히 중요)
  ai:
    mcp:
      server:
        name: order-mcp-server
        version: 1.0.0
        type: SYNC              # SYNC는 동기 메서드만, ASYNC는 비동기 메서드만 등록된다
        protocol: STREAMABLE    # STDIO / STREAMABLE / STATELESS / (deprecated) SSE
        annotation-scanner:
          enabled: true
server:
  port: 8081
```

- `spring.ai.mcp.server.stdio=true`로 바꾸면 stdio 서버가 됩니다. 이때 **stdout은 JSON-RPC 프레임 전용**이므로 `System.out.println`이나 배너, 로그 프레임워크의 콘솔 어펜더가 섞이면 즉시 프로토콜 파싱 오류가 납니다. 로그는 stderr로 보냅니다.
- HTTP 서버 스타터는 **기본적으로 인증을 붙이지 않습니다.** `POST /mcp`에 닿을 수 있는 클라이언트는 등록된 모든 도구를 나열·호출할 수 있으므로, localhost 밖에 노출하기 전에 Spring Security 등으로 경계를 세워야 합니다.

### 2.3 도구 구현 예시

```java
// src/main/java/com/example/orders/mcp/dto/OrderSummary.java
package com.example.orders.mcp.dto;

import java.util.List;

/** 도구 응답은 항상 DTO(record)로. JPA 엔티티를 그대로 반환하면 지연 로딩·순환 참조·내부 컬럼 노출이 따라온다. */
public record OrderSummary(
        String orderId,
        String status,
        int totalAmount,
        List<LineItem> items          // 개수 상한을 서비스에서 강제한다(예: 20건)
) {
    public record LineItem(String productName, int quantity, int amount) {
    }
}
```

```java
// src/main/java/com/example/orders/mcp/OrderMcpTools.java
package com.example.orders.mcp;

import com.example.orders.mcp.dto.CancelResult;
import com.example.orders.mcp.dto.OrderSummary;
import com.example.orders.order.CancelReason;
import com.example.orders.order.OrderCommandService;
import com.example.orders.order.OrderQueryService;
import org.springframework.ai.mcp.annotation.McpTool;
import org.springframework.ai.mcp.annotation.McpToolParam;
import org.springframework.stereotype.Component;

import java.util.UUID;

/**
 * MCP 도구 계층(어댑터).
 * 역할: (1) 입력 파싱·검증, (2) 결과 크기 제한, (3) 모델이 읽을 오류 문구 변환.
 * 도메인 규칙과 트랜잭션은 Service에 둔다.
 */
@Component
public class OrderMcpTools {

    private final OrderQueryService orderQueryService;
    private final OrderCommandService orderCommandService;

    public OrderMcpTools(OrderQueryService orderQueryService, OrderCommandService orderCommandService) {
        this.orderQueryService = orderQueryService;
        this.orderCommandService = orderCommandService;
    }

    @McpTool(
            name = "get_order",
            title = "주문 단건 조회",
            description = "주문 UUID로 주문 상태와 라인 항목 요약(최대 20건)을 반환합니다. 결제·배송 상세는 포함하지 않습니다.",
            generateOutputSchema = true,
            annotations = @McpTool.McpAnnotations(
                    readOnlyHint = true,
                    destructiveHint = false,
                    idempotentHint = true,
                    openWorldHint = false))
    public OrderSummary getOrder(
            @McpToolParam(description = "주문 UUID (예: 6f1c2b8e-...)", required = true) String orderId) {
        try {
            return orderQueryService.findSummary(UUID.fromString(orderId));
        } catch (IllegalArgumentException e) {
            // RuntimeException은 모델에게 "오류 결과"로 전달되어 인자를 고쳐 재시도할 수 있다.
            // 내부 예외 메시지·스택은 노출하지 않고 고칠 방법만 알려준다.
            throw new InvalidToolArgumentException("orderId는 UUID 형식이어야 합니다.");
        }
    }

    @McpTool(
            name = "cancel_order",
            title = "주문 취소",
            description = "주문을 취소합니다. 결제 취소는 비동기로 처리되며 결과 상태값을 반환합니다. "
                        + "같은 idempotencyKey로 다시 호출해도 중복 처리되지 않습니다.",
            generateOutputSchema = true,
            annotations = @McpTool.McpAnnotations(
                    readOnlyHint = false,
                    destructiveHint = true,       // 클라이언트가 사람 확인 UI를 띄울 수 있는 힌트
                    idempotentHint = true,
                    openWorldHint = false))
    public CancelResult cancelOrder(
            @McpToolParam(description = "주문 UUID", required = true) String orderId,
            @McpToolParam(description = "취소 사유 코드: CUSTOMER_REQUEST | OUT_OF_STOCK | PAYMENT_FAILED",
                          required = true) String reasonCode,
            @McpToolParam(description = "호출 단위 멱등 키(UUID). 재시도 시 같은 값을 그대로 사용합니다.",
                          required = true) String idempotencyKey) {
        return orderCommandService.cancel(
                UUID.fromString(orderId),
                CancelReason.fromCode(reasonCode),
                idempotencyKey);
    }
}
```

```java
// src/main/java/com/example/orders/order/OrderCommandService.java
package com.example.orders.order;

import com.example.orders.mcp.dto.CancelResult;
import com.example.orders.order.event.OrderCanceledEvent;
import org.springframework.context.ApplicationEventPublisher;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.time.Clock;
import java.time.Instant;
import java.util.UUID;

@Service
public class OrderCommandService {

    private final OrderRepository orderRepository;
    private final OrderCancelLogRepository cancelLogRepository;
    private final ApplicationEventPublisher events;
    private final Clock clock;

    public OrderCommandService(OrderRepository orderRepository,
                               OrderCancelLogRepository cancelLogRepository,
                               ApplicationEventPublisher events,
                               Clock clock) {
        this.orderRepository = orderRepository;
        this.cancelLogRepository = cancelLogRepository;
        this.events = events;
        this.clock = clock;
    }

    /**
     * MCP 도구에서 호출되므로 "재시도 가능한 API"로 설계한다.
     * 같은 idempotencyKey는 최초 1회만 처리되고, 이후 호출은 같은 결과를 돌려준다.
     */
    @Transactional
    public CancelResult cancel(UUID orderId, CancelReason reason, String idempotencyKey) {
        return cancelLogRepository.findByIdempotencyKey(idempotencyKey)
                .map(OrderCancelLog::toResult)
                .orElseGet(() -> doCancel(orderId, reason, idempotencyKey));
    }

    private CancelResult doCancel(UUID orderId, CancelReason reason, String idempotencyKey) {
        // 1) 같은 주문에 대한 동시 취소를 직렬화한다(SELECT ... FOR UPDATE)
        Order order = orderRepository.findByIdForUpdate(orderId)
                .orElseThrow(() -> new OrderNotFoundException(orderId));

        // 2) 상태 검증은 도메인 객체 안에서. 이미 취소된 주문이면 예외 대신 같은 결과로 흡수한다.
        order.cancel(reason, Instant.now(clock));

        // 3) 멱등 로그를 같은 트랜잭션에 남긴다 → 커밋 성공 = 처리 완료.
        OrderCancelLog log = cancelLogRepository.save(OrderCancelLog.of(idempotencyKey, order));

        // 4) 외부 알림·결제 취소는 커밋 이후에만 나가야 한다(@TransactionalEventListener(AFTER_COMMIT)).
        events.publishEvent(new OrderCanceledEvent(order.getId(), reason));
        return log.toResult();
    }
}
```

```java
// src/main/java/com/example/orders/config/McpClientConfig.java
package com.example.orders.config;

import org.springframework.ai.chat.client.ChatClient;
import org.springframework.ai.tool.ToolCallbackProvider;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

/** 호스트(agent) 쪽: MCP 클라이언트 스타터가 만든 ToolCallbackProvider를 채팅 모델에 연결한다. */
@Configuration
public class McpClientConfig {

    @Bean
    ChatClient chatClient(ChatClient.Builder builder, ToolCallbackProvider mcpToolCallbacks) {
        // Spring AI 2.0부터 defaultTools(Object...)로 통합되었다(defaultToolCallbacks는 deprecated).
        return builder
                .defaultSystem("주문 관련 요청은 반드시 제공된 도구로 사실을 확인한 뒤 답한다.")
                .defaultTools(mcpToolCallbacks)
                .build();
    }
}
```

### 2.4 주의사항

**① 도구 스키마는 곧 프롬프트 비용이다 (컨텍스트 예산)**

- 도구가 늘어나면 `tools/list` 결과 전체가 매 요청 프롬프트에 들어갑니다. 도구 20~30개를 넘기면 **선택 정확도가 떨어지고 비용이 선형으로 증가**합니다.
- 대응: 도구 이름·설명을 짧고 판별 가능하게(무엇을/언제/무엇을 반환하는지) 쓰고, 목록 조회는 페이지네이션(`nextCursor`)과 캐시 힌트(`ttlMs`, `cacheScope`)를 활용합니다. 서버가 목록을 **결정적 순서**로 주면 프롬프트 캐시 적중률이 올라간다는 점이 2026-07-28에서 권고로 명시되었습니다.
- 스코프 기반으로 **볼 수 있는 도구를 다르게** 주는 것도 합법입니다(목록은 연결이 아니라 권한에 따라 달라질 수 있음). 관리자용 도구를 일반 사용자 스코프에 노출하지 않는 기본 방어선이 됩니다.

**② 응답 크기 폭발**

- "주문 목록 조회" 도구가 1만 건을 반환하면 모델은 그걸 다 읽고도 요약하지 못합니다. **도구 인자에 `limit`을 두고 서버에서 상한을 강제**하고, 응답은 `structuredContent` + `outputSchema`로 구조를 명시한 뒤 필요한 필드만 담습니다.
- JPA에서 흔한 실수: 엔티티를 그대로 반환 → `LazyInitializationException`(직렬화 시점에 세션 없음) 또는 N+1. 도구 응답은 **항상 DTO record**로 변환하고, 연관은 `fetch join`/`@EntityGraph` 또는 별도 조회로 필요한 만큼만 가져옵니다.

**③ 쓰기 도구 = 재시도되는 API**

- 무상태 개정으로 스트림 재개가 사라져, 클라이언트는 실패 시 같은 요청을 **새 id로 다시** 보냅니다. 즉 "취소"가 두 번 도착할 수 있습니다.
- 대응은 REST 때와 같습니다: 멱등 키 인자 + **DB 유니크 제약**을 최종 방어선으로. 조회 후 INSERT하는 check-then-act는 동시 요청에서 뚫립니다.

```java
// 경쟁 상황: 동시에 같은 idempotencyKey가 들어오면 유니크 제약이 잡아준다.
try {
    return cancelLogRepository.saveAndFlush(OrderCancelLog.of(idempotencyKey, order)).toResult();
} catch (DataIntegrityViolationException e) {
    // 이미 다른 트랜잭션이 처리했다 → 기존 결과를 재조회해 같은 응답을 돌려준다.
    return cancelLogRepository.findByIdempotencyKey(idempotencyKey)
            .orElseThrow(() -> e)
            .toResult();
}
```

- `readOnlyHint`·`destructiveHint`·`idempotentHint`·`openWorldHint`는 **클라이언트용 힌트일 뿐 강제력이 없습니다.** 스펙도 "신뢰된 서버가 준 것이 아니면 힌트를 신뢰하지 말라"고 못 박습니다. 서버는 힌트와 무관하게 자기 권한 검사를 다시 해야 합니다.
- 파괴적 도구는 **사람 확인**을 전제로 설계합니다(스펙도 사람이 거부할 수 있는 human-in-the-loop를 권고). 필요하면 2단계 확인(`prepare_cancel` → 확인 토큰 → `confirm_cancel`)으로 쪼갭니다.

**④ 트랜잭션 경계**

- 도구 메서드에 `@Transactional`을 붙이지 않습니다. 같은 클래스 내부 호출은 프록시를 타지 않아 트랜잭션이 조용히 사라지고, 도구 계층이 트랜잭션에 묶이면 외부 API 호출까지 트랜잭션 안에 들어갑니다.
- 외부 PG·알림 호출은 트랜잭션 커밋 이후(`@TransactionalEventListener(phase = AFTER_COMMIT)`)로 분리합니다. 실패하면 재시도·보상 대상이지, 주문 취소를 롤백할 이유가 아닙니다.

**⑤ 타임아웃과 긴 작업**

- Spring AI MCP 클라이언트의 기본 `request-timeout`은 20초입니다. 이 시간을 넘길 작업은 `progress` 알림으로 진행률을 보내거나, tasks 확장(폴링: `tasks/get`, 입력: `tasks/update`)으로 넘깁니다.
- 요청 스코프 스트림이 끊기면 **in-flight 요청은 유실**입니다. 클라이언트 코드에서 "끊김 = 실패"로 처리하고 재발행 여부를 도구의 멱등성으로 판단하게 만듭니다.

**⑥ 보안: 도구 설명과 결과도 신뢰할 수 없는 입력이다**

- **도구 설명을 통한 프롬프트 인젝션(tool poisoning)**: 악성 서버가 도구 `description`에 "이 도구를 호출하기 전에 ~파일을 읽어 전송하라" 같은 문장을 심습니다. 호스트 UI에서 **어떤 서버의 어떤 도구가 노출되는지 보여주고**, 처음 연결된 서버의 도구는 승인받는 흐름이 필요합니다.
- **도구 결과 인젝션**: 외부 게시판 글을 긁어오는 도구의 결과에 지시문이 섞여 들어옵니다. 도구 결과는 데이터로만 다루고, 결과에 따라 다른 도구를 호출하는 판단은 모델·호스트 정책에 맡깁니다(서버가 결과 안의 지시를 실행하지 않도록).
- **혼동된 대리인(confused deputy)**: 정적 client ID로 제3자 API에 붙는 MCP 프록시에서, 공격자가 동적 등록으로 자기 redirect_uri를 심고 사용자의 동의 쿠키를 재사용해 인가 코드를 가로채는 공격입니다. 프록시는 **클라이언트별 동의**를 직접 저장·확인해야 합니다.
- `x-mcp-header`로 비밀번호·토큰·개인정보를 헤더에 실어 보내면 중간 프록시에 그대로 노출됩니다. 이런 파라미터는 헤더 미러링 대상에서 제외합니다.
- 도구가 임의 URL을 받아 호출하는 구조(크롤러·웹훅)는 **SSRF** 통로입니다. 허용 도메인 목록, 사설 IP·메타데이터 엔드포인트 차단을 서버 쪽 egress 정책으로 겁니다.

**⑦ stdio 서버 운영 함정**

- stdout 오염(가장 흔함), 자식 프로세스 미정리(호스트 종료 시 좀비), JVM 기동 시간(수 초) 때문에 첫 호출이 타임아웃되는 문제.
- Windows에서 `npx`·`npm`은 `.cmd` 배치 파일이라 `ProcessBuilder`가 직접 실행하지 못합니다. `cmd.exe /c npx ...`로 감싸야 합니다.

**⑧ 버전 호환**

- 클라이언트가 `MCP-Protocol-Version` 헤더를 보내지 않으면 서버는 구버전 클라이언트로 간주할 수 있습니다(헤더 도입 전 리비전은 `2025-03-26` 취급). 사내 배포에서는 **서버가 지원 버전을 `server/discover`로 광고하고, 미지원 버전은 명시적 오류로 거절**하게 만드는 편이 디버깅이 빠릅니다.

**⑨ 관측성**

- `_meta`의 `traceparent`·`tracestate`로 분산 트레이스를 이어 붙이는 규약이 문서화되어 있습니다. 도구별 호출 수·p95 지연·오류율·결과 크기를 지표로 두지 않으면 "모델이 도구를 이상하게 고른다"는 문제를 추적할 수 없습니다.
- 도구 호출 로그에는 개인정보·토큰이 그대로 남기 쉽습니다. 인자 마스킹 규칙을 로그 수집 단계에서 적용합니다.

### 2.5 배포 전 체크리스트

- [ ] `tools/list`를 실제 클라이언트로 호출해 스키마·설명·정렬이 의도대로인가
- [ ] 잘못된 인자(형식 오류·범위 초과)에 대해 모델이 고칠 수 있는 오류 문구를 주는가
- [ ] 미인증·타 스코프 요청이 401/403으로 막히는가(HTTP 서버는 기본이 무인증)
- [ ] 같은 요청을 두 번 보내도 결과가 한 번만 반영되는가(멱등 키)
- [ ] 응답 크기에 상한이 있는가(목록 도구 `limit`, 필드 수)
- [ ] 타임아웃·취소 시 자원이 정리되는가(커넥션, 락, 임시 파일)
- [ ] stdio 모드에서 stdout이 프레임 외 텍스트로 오염되지 않는가
- [ ] 종료 시 클라이언트·자식 프로세스가 정리되는가

---

## 3. 기술 면접 단골 꼬리 질문 & 답변 팁

- **Q. MCP가 무엇이고, 기존 function calling과 무엇이 다른가요?**
  - **A.** "function calling은 모델이 '함수를 부르는' 기능이고, MCP는 그 함수를 **프로세스 밖의 서버로 분리해 표준 프로토콜로 노출**하는 규격입니다. 벤더별 스키마·실행 방식 차이를 흡수해서, 도구를 한 번 구현하면 어떤 호스트든 붙일 수 있습니다. JSON-RPC 2.0 위에서 `tools/list`로 발견하고 `tools/call`로 실행합니다. 모델 호출 자체는 여전히 호스트가 합니다."

- **Q. 호스트·클라이언트·서버 역할을 나눠 설명해 보세요.**
  - **A.** "호스트는 LLM과 대화 상태·동의 UI·정책을 가진 애플리케이션이고, 클라이언트는 **서버 1개와만** 통신하는 커넥터입니다. 서버는 좁은 기능만 노출합니다. 이 분리의 핵심은 **서버가 대화 전체를 볼 수 없다**는 점입니다. 권한이 필요한 만큼만 주고, 사고가 나도 그 서버 스코프 안에서만 영향이 나도록 경계를 긋습니다."

- **Q. tools와 resources, prompts는 어떻게 다른가요?**
  - **A.** "제어 주체가 다릅니다. tools는 **모델**이 스스로 고르는 실행 단위(부작용 가능), resources는 **애플리케이션**이 URI로 읽어가는 읽기 전용 컨텍스트, prompts는 **사용자**가 명시적으로 고르는 템플릿입니다. 부작용과 인자가 있으면 tool, 그냥 읽을 자료면 resource로 두는 게 기준입니다."

- **Q. 원격 MCP 서버는 어떻게 인증하나요?**
  - **A.** "MCP 서버는 **OAuth 2.1 리소스 서버**로 동작합니다. 미인증 요청에 401과 `WWW-Authenticate`로 보호 리소스 메타데이터(RFC 9728)를 알려주고, 클라이언트가 인가 서버를 찾아 스코프를 받아 옵니다. 토큰 요청에는 `resource` 지시자(RFC 8707)를 넣어 대상 서버에 묶고, 응답의 `iss`(RFC 9207)를 검증합니다. stdio는 네트워크가 아니므로 OAuth 대상이 아니라 환경변수로 자격증명을 받습니다."

- **Q. MCP 2026-07-28 개정에서 가장 큰 변화는?**
  - **A.** "**무상태화**입니다. `initialize` 핸드셰이크와 `Mcp-Session-Id` 세션이 사라지고, 요청마다 `_meta`에 프로토콜 버전과 클라이언트 capability를 싣습니다. 서버가 상태를 가져야 하면 **서버가 발급한 핸들을 도구 인자로** 넘깁니다. 목록 응답에는 `ttlMs`·`cacheScope`가 붙어 캐싱이 가능해졌고, GET 스트림과 `Last-Event-ID` 재개는 제거되어 변경 알림은 `subscriptions/listen` 하나로 모입니다. 이유는 로드밸런서·오토스케일 환경에서 sticky session 없이 확장하기 위해서입니다."

- **Q. 무상태가 되면 생기는 단점은?**
  - **A.** "세 가지입니다. 매 요청에 메타데이터가 중복되고, 응답 스트림이 끊기면 in-flight 요청이 유실되어 **재발행**해야 하며(그래서 쓰기 도구는 멱등해야 합니다), 다단계 흐름은 핸들·`requestState`로 명시해야 합니다. 대신 재시도·확장·프록시 캐싱이 단순해집니다."

- **Q. 서버가 사용자에게 추가 정보를 물어야 할 때는 어떻게 하나요?**
  - **A.** "서버가 별도 요청을 보내는 대신 `resultType: input_required`인 `InputRequiredResult`에 `inputRequests`를 담아 응답하고, 클라이언트가 **새 요청 id로** 원래 요청을 다시 보내면서 `inputResponses`와 `requestState`를 함께 전달합니다. 이게 MRTR 패턴입니다. `requestState`는 서버가 서명해 변조를 막아야 합니다."

- **Q. MCP 도구를 LLM에 노출할 때 가장 위험한 지점은?**
  - **A.** "도구 설명·결과가 **신뢰할 수 없는 입력**이라는 점입니다. 악성 서버가 설명에 지시문을 심는 tool poisoning, 도구 결과에 섞인 프롬프트 인젝션, 프록시의 혼동된 대리인 공격이 대표적입니다. 대응은 ① 노출되는 도구를 사용자에게 보여주고 승인받기, ② 파괴적 도구는 사람 확인을 거치게 하기, ③ 서버에서 스코프 검사를 다시 하기, ④ 결과를 지시가 아닌 데이터로만 취급하기입니다. 힌트(`destructiveHint`)는 참고일 뿐 강제가 아니라는 점도 같이 말해야 합니다."

- **Q. 쓰기 도구는 어떻게 안전하게 만드나요?**
  - **A.** "클라이언트가 실패 시 같은 요청을 다시 보낼 수 있다고 가정합니다. 멱등 키 인자를 받아 **유니크 제약**으로 중복을 걸러내고, 재호출에는 최초 결과를 그대로 돌려줍니다. check-then-act는 동시성에서 뚫리므로 DB 제약을 최종 방어선으로 둡니다. 권한 검사는 모델이 준 인자를 믿지 않고 서버 세션·토큰 기준으로 다시 합니다."

- **Q. 트랜잭션은 어디에 두나요?**
  - **A.** "도구 계층이 아니라 서비스 계층입니다. 도구는 입력 검증과 변환만 합니다. 같은 클래스 내부 호출은 프록시를 타지 않아 `@Transactional`이 무시될 수 있고, 도구 메서드에 트랜잭션을 걸면 외부 API 호출까지 트랜잭션에 끌려 들어갑니다. 외부 호출은 커밋 이후 이벤트로 분리합니다."

- **Q. MCP가 있으면 OpenAPI/REST는 필요 없나요?**
  - **A.** "아닙니다. REST는 서버 내부·외부 API의 구현 방식으로 남고, MCP는 그 위에 얹는 **에이전트용 접근 계층**입니다. MCP는 실행 시점의 발견(`tools/list`), 권한 협상, 도구 의미 힌트, 사람 승인 흐름까지 프로토콜에 포함한다는 점이 다릅니다. 사내 주문 API를 MCP 도구로 감싸되, 내부 구현은 그대로 REST + Service 계층을 씁니다."

- **Q. Java/Spring에서는 어떻게 구현하나요?**
  - **A.** "Spring AI 2.0.x(Boot 4.0/4.1)에서 `spring-ai-starter-mcp-server-webmvc`를 넣고 `@McpTool`·`@McpToolParam`으로 도구를 선언하면 스키마가 자동 생성됩니다. 전송은 `spring.ai.mcp.server.protocol`로 STDIO/STREAMABLE/STATELESS를 고릅니다. Boot 3 프로젝트라면 Spring AI 1.0.x 라인에서 `@Tool` + `MethodToolCallbackProvider`로 등록합니다. 어떤 경우든 도구는 서비스 계층을 감싸는 얇은 어댑터로 두고, 응답은 엔티티가 아닌 record DTO로 반환합니다."

---

## 참고

- MCP 스펙(2026-07-28): https://modelcontextprotocol.io/specification/2026-07-28/architecture
- 변경 이력: https://modelcontextprotocol.io/specification/2026-07-28/changelog
- Tools 정의: https://modelcontextprotocol.io/specification/2026-07-28/server/tools
- 인가: https://modelcontextprotocol.io/specification/2026-07-28/basic/authorization
- Spring AI MCP 서버 스타터: https://docs.spring.io/spring-ai/reference/api/mcp/mcp-server-boot-starter-docs.html
- Spring AI 2.0 GA 안내: https://spring.io/blog/2026/06/12/spring-ai-2-0-0-GA-available-now
- MCP 재단 이관: https://www.anthropic.com/news/donating-the-model-context-protocol-and-establishing-of-the-agentic-ai-foundation
