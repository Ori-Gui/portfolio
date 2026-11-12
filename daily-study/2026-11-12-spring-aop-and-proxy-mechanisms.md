# [2026-11-12] Spring AOP 동작 원리와 프록시 패턴 (CGLIB vs JDK Dynamic Proxy)

> **한 줄 핵심 요약**: Spring AOP는 **빈을 프록시로 갈아끼우는 방식(런타임 위빙)** 으로 동작하고, 그 프록시를 만드는 두 가지 방법이 **JDK Dynamic Proxy(인터페이스 구현)** 와 **CGLIB(클래스 상속)** 다. Spring Boot는 2.x부터 `spring.aop.proxy-target-class=true`(CGLIB)가 기본값이라 `final` 클래스·`final` 메서드·`private` 메서드에 붙은 어노테이션이 조용히 무시되고, **같은 빈 내부의 `this.inner()` 호출**은 프록시를 우회해 `@Transactional`·`@Async`·`@Cacheable`이 모두 무력화된다. 실무 대응은 ① 트랜잭션/캐시 경계를 **별도 빈으로 분리**가 기본이고, ② `ObjectProvider<Self>` 지연 주입, ③ `AopContext.currentProxy()`(`exposeProxy=true`)는 차선책이다.

> 이 노트는 **프록시가 어떻게 만들어지고 호출이 어떻게 가로채지는지(생성 메커니즘)** 에 집중한다. `@Transactional`의 전파 속성·readOnly·롤백 정책은 [2026-11-08 노트](./2026-11-08-spring-transactional-aop-propagation-and-self-invocation.md)에서 다뤘으므로 겹치는 부분은 링크로 대신한다.

---

## 1. Spring AOP 개념과 프록시 메커니즘

### 1-1. AOP가 해결하는 문제

로깅, 트랜잭션, 캐시, 인증, 재시도, 성능 측정은 **여러 비즈니스 메서드에 똑같이 반복되는 관심사(횡단 관심사, cross-cutting concern)** 다. 이 코드를 서비스 메서드마다 직접 넣으면 두 가지가 무너진다.

- 비즈니스 로직과 부가 기능이 한 메서드에 섞여 **읽히지 않는 코드**가 된다.
- 정책이 바뀌면(로그 포맷 변경, 캐시 TTL 변경) **수십 개 메서드를 동시에 고쳐야** 한다.

AOP는 이 부가 기능을 **별도의 모듈(Aspect)로 분리하고, 호출 시점에 끼워 넣는** 방식으로 해결한다. 스프링에서는 "끼워 넣기"를 소스 수정이 아니라 **프록시 객체로 감싸기**로 구현한다.

### 1-2. 용어 정리

| 용어 | 의미 | 스프링에서의 실체 |
|---|---|---|
| **Aspect(관점)** | 횡단 관심사를 모아 놓은 모듈 | `@Aspect` 클래스 |
| **JoinPoint(조인 포인트)** | 부가 기능이 끼어들 수 있는 지점 | **메서드 실행 시점만** 지원 |
| **Pointcut(포인트컷)** | 수많은 JoinPoint 중 실제로 적용할 대상 선택식 | `execution(* com.example..*Service.*(..))` |
| **Advice(어드바이스)** | 그 지점에서 실행할 동작 | `@Before`, `@AfterReturning`, `@AfterThrowing`, `@After`, `@Around` |
| **Advisor(어드바이저)** | Pointcut + Advice의 묶음 | 프록시가 실제로 들고 다니는 단위 |
| **Weaving(위빙)** | Aspect를 대상 코드에 엮는 작업 | **런타임에 프록시 생성**으로 수행 |
| **Target(대상 객체)** | 실제 비즈니스 로직을 가진 원본 인스턴스 | `this`가 가리키는 객체 |
| **Proxy** | Target을 감싸 호출을 가로채는 대역 | 컨테이너에 등록되는 최종 빈 |

여기서 중요한 제약이 하나 나온다. **Spring AOP는 메서드 실행 조인 포인트만 지원한다.** AspectJ는 필드 접근, 생성자 호출, `static` 초기화 시점까지 조인 포인트로 삼지만, 스프링 AOP는 "프록시가 메서드 호출을 가로챈다"는 구현 방식 때문에 그 밖의 지점에는 개입할 수 없다. 이 차이가 뒤에서 다룰 AspectJ 모드(3-6)와 `final`/`private` 제약의 근원이다.

### 1-3. 런타임 위빙: 빈 후처리기가 프록시를 끼워 넣는다

스프링 AOP의 핵심은 **"빈을 만든 다음, 그 빈을 프록시로 교체해서 컨테이너에 등록한다"** 이다. 교체를 담당하는 것이 `BeanPostProcessor`의 구현체인 `AbstractAutoProxyCreator`(정확히는 `AnnotationAwareAspectJAutoProxyCreator`)다.

```text
[빈 생성 파이프라인]

BeanDefinition 읽기
   │
   ▼
① 인스턴스화 (생성자 호출)
   │
   ▼
② 의존성 주입 (populateBean)
   │
   ▼
③ 초기화 (@PostConstruct → InitializingBean.afterPropertiesSet → init-method)
   │
   ▼
④ BeanPostProcessor.postProcessAfterInitialization(bean, beanName)
   │
   └─ AbstractAutoProxyCreator.wrapIfNecessary(bean, beanName, cacheKey)
        ├ (a) getAdvicesAndAdvisorsForBean(beanClass, beanName, ...)
        │      └ 모든 Advisor를 수집해 Pointcut으로 이 빈의 메서드가 매칭되는지 검사
        │         · AspectJExpressionPointcut이 클래스 로딩 없이 메서드 시그니처로 판정
        │         · 매칭 결과는 캐시(BeanFactory 기반)에 보관
        ├ (b) 매칭된 Advisor가 없으면 → 원본 빈 그대로 반환 (프록시 없음)
        └ (c) 매칭되면 → ProxyFactory 생성
               ├ proxyFactory.addAdvisors(matchingAdvisors)
               ├ proxyFactory.setProxyTargetClass(...)   ← CGLIB / JDK 선택 기준
               └ proxyFactory.getProxy(classLoader)
                    └ AopProxyFactory(DefaultAopProxyFactory).createAopProxy()
                       ├ proxyTargetClass = true  또는 인터페이스 없음 → ObjenesisCglibAopProxy
                       └ proxyTargetClass = false + 인터페이스 있음   → JdkDynamicAopProxy
   │
   ▼
⑤ 컨테이너 빈 맵에는 "프록시"가 등록된다
   (다른 빈에 @Autowired/@Inject 되는 것도, getBean()이 돌려주는 것도 프록시)
```

이 구조에서 실무적으로 중요한 점은 다음과 같다.

- **프록시는 빈 하나당 한 번 만들어진다.** 매 요청마다 새로 만들지 않는다. 그래서 "생성 비용"은 기동 시간에만 영향을 주고, "호출 비용"이 런타임 성능을 좌우한다(2-4).
- **자기 호출은 이 파이프라인을 다시 타지 않는다.** 프록시는 메서드 호출 시점에만 개입하고, Target 내부의 `this.inner()`는 프록시를 거치지 않는다(3-1).
- **매칭이 없으면 프록시가 아예 안 만들어진다.** `@Transactional`을 붙였는데 `AopUtils.isAopProxy(bean)`가 `false`라면, 애노테이션 문제가 아니라 Pointcut/프록시 설정 문제다.
- `@Aspect` 클래스 자체도 빈이다. Aspect가 `@Component`로 등록되지 않으면(또는 `@Bean`으로 등록되지 않으면) 어드바이저 수집 대상에서 빠져 아무 일도 일어나지 않는다.

### 1-4. 호출 시점: 인터셉터 체인

프록시에 도착한 호출은 **하나의 인터셉터 체인**을 통과한다. `@Around`가 `proceed()`로 다음 인터셉터(또는 Target)를 호출하고, `@Before`/`@After` 계열은 내부적으로 각각 `MethodBeforeAdviceInterceptor`, `AspectJAfterAdvice` 같은 인터셉터로 변환되어 같은 체인에 들어간다.

```text
Client
  │  inventoryService.decrease(id, qty)      ← 클라이언트가 들고 있는 참조 = 프록시
  ▼
Proxy(OrderService$$SpringCGLIB$$0)
  │  DynamicAdvisedInterceptor.intercept(proxy, method, args, methodProxy)
  ▼
ReflectiveMethodInvocation.proceed()
  ├─ [1] TraceAspect.around        (@Around: 시작 시각 기록 → proceed)
  ├─ [2] TransactionInterceptor    (@Transactional: BEGIN → proceed)
  ├─ [3] CacheInterceptor          (@Cacheable: 캐시 조회 → hit면 즉시 반환)
  └─ [4] Target.decrease(id, qty)  ← 실제 비즈니스 로직
  ▲        각 인터셉터의 뒷단 코드가 역순으로 실행 (커밋 → 로그 마감)
  │  Advisor 순서는 @Order / Ordered로 결정된다
```

- 체인 순서는 **Advisor의 order**로 정해진다. 실행 순서가 의미를 가지면(예: 캐시를 트랜잭션보다 바깥에 두면 커밋 전 값이 캐시에 들어갈 수 있다) 순서를 명시해야 한다. 기본 order에 의존하지 않는 편이 안전하다.
- 프록시는 **빈당 하나**이고 그 안에 여러 Advisor가 **하나의 체인**으로 들어간다. "프록시가 여러 겹" 생기는 것이 아니라, 필요하면 명시적으로 `ProxyFactory`를 중첩 생성할 때만 겹친다.
- `@Async`는 이 체인에 특성이 하나 붙는다. `AsyncExecutionInterceptor`가 Target 호출을 **다른 스레드에 위임하고 즉시 반환**하므로, 그 뒤의 인터셉터와 Target은 다른 스레드에서 실행된다. ThreadLocal 기반 자원(트랜잭션·SecurityContext·MDC)은 따라가지 않는다.

### 1-5. Advice 5종과 실무 선택

| Advice | 실행 시점 | 주 용도 | 주의 |
|---|---|---|---|
| `@Around` | 전후 모두, 반환값/예외 제어 가능 | 시간 측정, 캐시, 재시도, 트랜잭션 | `proceed()`를 호출하지 않으면 Target이 실행되지 않는다. `try/finally`로 뒷정리 |
| `@Before` | Target 실행 직전 | 파라미터 검증, 로그 | Target 반환값에 접근 불가 |
| `@AfterReturning` | 정상 반환 후 | 결과 로그, 반환값 가공(`returning`) | 예외 시 실행되지 않음 |
| `@AfterThrowing` | 예외 발생 시 | 에러 로그, 알림, 변환 | 예외를 삼키려면 `@Around`를 쓴다 |
| `@After` | 정상/예외 모두 | 자원 해제(=`finally`) | 반환값 접근 불가 |

실무에서는 `@Around` 하나로 통일하는 경우가 많다. 다만 `@Around`는 Target을 호출하지 않는 실수를 컴파일러가 잡아주지 못하므로, "왜 이 API가 아무것도 안 하지?"를 만나면 Aspect의 `proceed()` 누락을 먼저 확인한다.

### 1-6. 프록시가 적용되지 않는 조건 (5가지)

| 조건 | 결과 | 확인 방법 |
|---|---|---|
| 같은 빈 내부에서 `this.method()` 호출 | 프록시 우회 → 어드바이스 전부 미적용 | `AopUtils.isAopProxy(this)`가 `false` |
| `private` 메서드에 애노테이션 | 조용히 무시 (오류 없음) | 애노테이션을 `public`으로 옮기고 동작 확인 |
| `final` 메서드 | 조용히 무시 (CGLIB 오버라이드 불가) | 메서드에서 `final` 제거 |
| `final` 클래스 | CGLIB이면 **기동 실패**(`Cannot subclass final class`) | 스택트레이스에 `Enhancer` |
| `static` 메서드 | 적용 대상 아님 (인스턴스 디스패치가 아님) | 설계상 분리 |

`private`/`final` 메서드의 실패가 위험한 이유는 **오류가 나지 않는다는 점**이다. 기동도 되고 로그도 없고, 기능만 조용히 빠진다. 코드 리뷰에서 `final` 키워드와 `@Transactional`/`@Cacheable`의 조합을 눈으로 확인하는 습관이 필요하다.

---

## 2. JDK Dynamic Proxy vs CGLIB 상세 비교

### 2-1. JDK Dynamic Proxy: 인터페이스를 구현한 대역

JDK가 기본 제공하는 방식이다. `java.lang.reflect.Proxy`가 **런타임에 인터페이스를 구현한 클래스(`$Proxy0`)를 생성**하고, 그 클래스의 모든 메서드는 `InvocationHandler.invoke()`로 위임한다.

```java
// JDK Dynamic Proxy의 본질 (개념 재현 코드, 파일 경로: src/test/java/study/aop/JdkProxyConceptTest.java)
package study.aop;

import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Method;
import java.lang.reflect.Proxy;

public class JdkProxyConceptTest {

    interface Calculator {
        int add(int a, int b);
    }

    static class CalculatorImpl implements Calculator {
        @Override
        public int add(int a, int b) {
            return a + b;
        }
    }

    public static void main(String[] args) {
        Calculator target = new CalculatorImpl();

        // ① 인터페이스 목록으로 프록시 클래스를 생성한다 (캐시되지 않으면 클래스 생성 비용 발생)
        Calculator proxy = (Calculator) Proxy.newProxyInstance(
                Calculator.class.getClassLoader(),          // 프록시 클래스를 정의할 클래스로더
                new Class<?>[]{Calculator.class},           // 구현할 인터페이스 (필수)
                new InvocationHandler() {                   // 모든 메서드 호출이 여기로 들어온다
                    @Override
                    public Object invoke(Object proxyObj, Method method, Object[] args) throws Throwable {
                        System.out.println("[before] " + method.getName());
                        Object result = method.invoke(target, args);   // ← 리플렉션 호출
                        System.out.println("[after]  " + result);
                        return result;
                    }
                });

        System.out.println(proxy.add(1, 2));                    // 3
        System.out.println(proxy.getClass().getName());         // jdk.proxy1.$Proxy0 (JDK 21)
        System.out.println(proxy.getClass().getSuperclass());   // class java.lang.reflect.Proxy
        // (CalculatorImpl) proxy  ← ClassCastException. 인터페이스 타입으로만 쓸 수 있다.
    }
}
```

구조적 특징은 다음과 같다.

- **인터페이스가 없으면 프록시를 만들 수 없다.** 인터페이스가 없으면 `IllegalArgumentException`이 난다.
- **인터페이스에 선언된 메서드만** 가로챌 수 있다. Target 클래스에만 있는 `public` 메서드는 프록시 타입으로 호출할 수 없다(인터페이스로 캐스팅돼 있으므로 컴파일 단계에서 막힌다).
- 반환 타입이 인터페이스이므로, **구현 클래스 타입으로 주입받으려 하면 실패**한다. `@Autowired OrderServiceImpl` 같은 코드가 깨지는 이유다.
- 프록시 클래스 생성은 인터페이스 개수·메서드 수에 비례하지만, CGLIB처럼 상속 계층 전체를 복제하지는 않는다. **생성 비용이 상대적으로 낮다.**

### 2-2. CGLIB: 대상 클래스를 상속한 서브클래스

CGLIB(Code Generation Library, 스프링 6부터는 `spring-core`에 리패키징되어 내장)는 **ASM으로 바이트코드를 직접 생성해 대상 클래스를 상속한 서브클래스**를 만든다. 오버라이드된 메서드가 `MethodInterceptor`로 위임하고, 원본은 `MethodProxy.invokeSuper()`로 호출한다.

```java
// CGLIB의 본질 (개념 재현 코드, 파일 경로: src/test/java/study/aop/CglibProxyConceptTest.java)
package study.aop;

import java.lang.reflect.Method;

import org.springframework.cglib.proxy.Enhancer;          // 스프링 6: spring-core에 내장된 CGLIB
import org.springframework.cglib.proxy.MethodInterceptor;
import org.springframework.cglib.proxy.MethodProxy;

public class CglibProxyConceptTest {

    static class Calculator {                            // 인터페이스가 없다
        public int add(int a, int b) {
            return a + b;
        }
    }

    public static void main(String[] args) {
        Calculator target = new Calculator();

        Enhancer enhancer = new Enhancer();
        enhancer.setSuperclass(Calculator.class);         // ← 상속으로 만든다
        enhancer.setCallback(new MethodInterceptor() {
            @Override
            public Object intercept(Object proxyObj, Method method, Object[] args, MethodProxy methodProxy)
                    throws Throwable {
                System.out.println("[before] " + method.getName());
                // invokeSuper: 리플렉션 없이, 생성된 FastClass의 인덱스로 super 메서드를 직접 호출
                Object result = methodProxy.invokeSuper(proxyObj, args);
                System.out.println("[after]  " + result);
                return result;
            }
        });

        Calculator proxy = (Calculator) enhancer.create();  // 기본 생성자가 없어도 생성 가능(Objenesis)
        System.out.println(proxy.add(1, 2));                              // 3
        System.out.println(proxy.getClass().getName());                   // study.aop.CglibProxyConceptTest$Calculator$$EnhancerByCGLIB$$...
        System.out.println(proxy.getClass().getSuperclass().getName());   // study.aop.CglibProxyConceptTest$Calculator
        System.out.println(proxy instanceof Calculator);                  // true (상속이므로)
    }
}
```

구조적 특징은 다음과 같다.

- **인터페이스가 필요 없다.** 일반 클래스에도 적용된다. 실무 코드가 "인터페이스 없이 클래스로만" 작성되어 있어도 AOP가 걸리는 이유다.
- **상속이므로 `final` 클래스는 만들 수 없고, `final` 메서드는 오버라이드할 수 없다.** `private` 메서드도 오버라이드 대상이 아니다.
- `instanceof` 검사가 자연스럽다. `(구현클래스) proxy` 캐스팅도 가능해 **구체 타입 주입이 깨지지 않는다.**
- 인스턴스 생성은 스프링이 `ObjenesisCglibAopProxy`에서 **Objenesis**로 처리한다. 대상 클래스에 기본 생성자가 없어도(생성자 주입만 있는 클래스여도) 프록시를 만들 수 있다.
- `MethodProxy.invokeSuper()`는 **FastClass**라는 보조 클래스를 통해 메서드를 인덱스로 호출한다. 가상 디스패치 테이블 조회와 리플렉션 `Method.invoke`를 모두 피하므로 호출 경로가 짧다.
- 단점은 **생성 비용과 메타스페이스 사용량**이다. 상속 클래스 + FastClass를 생성하므로 첫 프록시 생성이 JDK 방식보다 무겁다. CGLIB은 생성된 클래스를 `Enhancer`의 캐시에 보관하므로 같은 설정의 두 번째 프록시부터는 비용이 줄어든다.

### 2-3. 한눈 비교

| 항목 | JDK Dynamic Proxy | CGLIB (Spring Boot 기본값) |
|---|---|---|
| 생성 방식 | `Proxy.newProxyInstance` → 인터페이스 구현 클래스 생성 | `Enhancer` + ASM → 대상 클래스 **상속** 서브클래스 생성 |
| 필수 조건 | **인터페이스 1개 이상** | 인터페이스 불필요. **`final` 클래스 불가** |
| 개입 가능 메서드 | 인터페이스에 선언된 메서드 | `public`/`protected`/패키지 가시성 (단, `final`·`private`·`static` 불가) |
| Target 호출 방식 | `Method.invoke` (리플렉션) | `MethodProxy.invokeSuper` (FastClass 인덱스 호출) |
| 생성 비용 | 낮음 (바이트코드 생성 최소) | 높음 (상속 클래스 + FastClass 생성), 캐시로 완화 |
| 호출 비용 | 리플렉션 기반. Java 8 인플레이션, JDK 18 JEP 416 이후 격차 축소 | 일반적으로 약간 빠름 |
| 타입 캐스팅 | 인터페이스 타입으로만 사용 가능 | 구현 클래스 타입으로도 사용 가능 |
| 주입 문제 | `@Autowired`를 구현 클래스 타입으로 받으면 실패 | 없음 |
| 디버깅 | 스택트레이스에 `$Proxy` + InvocationHandler | 스택트레이스에 `$$SpringCGLIB$$` |
| 기동 실패 위험 | 인터페이스 누락 시 런타임 오류 | `final` 클래스에 AOP 지정 시 기동 실패 |
| 대표 사용처 | JDK 기반 표준, Spring Data JPA 리포지토리(인터페이스) | `@Configuration`, `@Transactional` 기본, `@Async`, `@Cacheable` |

### 2-4. 성능: "CGLIB가 더 빠르다"는 절반만 맞는 말

실무에서 자주 인용되는 두 문장을 분리해서 봐야 한다.

**(1) 생성 속도는 JDK Proxy가 빠르다.**
JDK Proxy는 인터페이스 정보로 프록시 클래스를 만들고 그 결과를 JDK 내부적으로 재사용한다. 반면 CGLIB은 ASM으로 상속 클래스 바이트코드와 FastClass를 생성한다. 빈이 수천 개인 애플리케이션에서 프록시를 대량 생성하는 기동 구간에서는 JDK 쪽이 유리하다. 다만 컨테이너가 빈당 1회만 생성하고 CGLIB도 캐시를 쓰므로, **기동 시간에 미치는 영향은 보통 무시할 수준**이다. 문제가 되는 경우는 요청마다 `ProxyFactory`로 프록시를 새로 만드는 잘못된 코드다.

**(2) 호출 속도는 전통적으로 CGLIB이 약간 우세하지만, 이제 선택 이유가 되지 못한다.**
- JDK Proxy는 `Method.invoke`를 거친다. 초기 JVM은 리플렉션 호출을 네이티브로 처리했지만, Java 8 이후 일정 횟수를 넘기면 **바이트코드 액세서를 생성(inflation)** 해 호출 비용을 줄였다.
- **JDK 18의 JEP 416**은 핵심 리플렉션을 Method Handle 기반으로 재구현해 `Method.invoke` 오버헤드를 더 줄였다. Java 21에서도 이 구현이 유지된다.
- 따라서 최신 JVM에서 두 방식의 차이는 **호출당 수 나노초 수준**이고, 실제 서비스의 병목(DB I/O, 외부 API, 직렬화)에 비하면 측정 노이즈에 가깝다.

결론적으로 **프록시 종류는 성능이 아니라 제약과 설계로 고른다.** 정확한 수치가 필요하면 `System.nanoTime()` 루프가 아니라 JMH(`@Benchmark`, 워밍업, `@Fork`)로 재야 한다. 루프 측정은 JIT 컴파일·탈최적화·죽은 코드 제거(dead code elimination) 때문에 결과가 뒤집히기 쉽다.

### 2-5. 왜 Spring Boot 2.x부터 CGLIB가 기본값인가

Spring Boot의 `AopAutoConfiguration`은 `spring.aop.proxy-target-class`가 **없으면 `true`로 간주**해 CGLIB 설정을 활성화한다. (Spring Framework 단독, 즉 `@EnableAspectJAutoProxy`만 쓰는 경우의 기본값은 `false` = 인터페이스가 있으면 JDK Proxy다. **Boot와 순수 Spring의 기본값이 다르다**는 점이 면접에서 자주 나온다.)

CGLIB를 기본으로 바꾼 실무적 이유는 다음과 같다.

1. **인터페이스 없이도 동작한다.** 신입·소규모 프로젝트에서 `@Service` 클래스에 인터페이스 없이 `@Transactional`을 붙이는 코드가 대부분인데, JDK Proxy 기본값이면 트랜잭션이 조용히 걸리지 않는다. 이 혼란을 없앴다.
2. **구체 타입 주입이 깨지지 않는다.** `@Autowired OrderService`(클래스 타입)로 받는 코드가 JDK Proxy에서는 실패한다. CGLIB는 상속이므로 문제가 없다.
3. **`@Configuration` 클래스 프록시와 일관된다.** `@Configuration(proxyBeanMethods = true)`는 이미 CGLIB로 설정 클래스를 프록시해 `@Bean` 메서드가 싱글톤을 반환하도록 만든다. AOP 프록시도 같은 방식으로 통일하는 편이 예측 가능하다.
4. **과거 호출 성능 우위.** 리플렉션 기반 호출이 느리던 시절의 벤치마크가 기본값 결정에 영향을 줬다(2-4에서 본 것처럼 지금은 근거가 약해졌다).
5. **`final` 클래스 실패를 기동 시점에 알 수 있다.** JDK Proxy는 "인터페이스가 없어서 프록시를 못 만든다"를 런타임/주입 시점에 알지만, CGLIB는 기동 중 `Cannot subclass final class`로 즉시 드러난다. 실패를 빨리 발견하는 쪽이 낫다는 판단이다.

설정 방법은 세 가지 레벨이 있다.

```yaml
# 파일 경로: src/main/resources/application.yml
spring:
  aop:
    proxy-target-class: true    # 기본값. false로 두면 인터페이스가 있을 때 JDK Dynamic Proxy 사용
```

```java
// 파일 경로: src/main/java/study/aop/config/AspectProxyModeConfig.java
package study.aop.config;

import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.EnableAspectJAutoProxy;

@Configuration
@EnableAspectJAutoProxy(proxyTargetClass = false)   // 순수 Spring 기본값과 동일하게 되돌린다
public class AspectProxyModeConfig {
}
```

```java
// 파일 경로: src/main/java/study/aop/config/TxConfig.java
package study.aop.config;

import org.springframework.context.annotation.Configuration;
import org.springframework.transaction.annotation.EnableTransactionManagement;

@Configuration
@EnableTransactionManagement(proxyTargetClass = false)   // 트랜잭션 프록시만 JDK 방식으로
public class TxConfig {
}
```

`spring.aop.proxy-target-class`는 **AOP 전체(트랜잭션·캐시·비동기 포함)** 에 적용되고, `@EnableTransactionManagement(proxyTargetClass = ...)`는 트랜잭션 프록시에만 적용된다. 두 설정이 다르면 "트랜잭션은 JDK, 커스텀 Aspect는 CGLIB"처럼 섞일 수 있으므로 한쪽으로 통일한다.

### 2-6. 제약과 실패 모드

```java
// 파일 경로: src/main/java/study/aop/bad/FinalClassService.java
package study.aop.bad;

import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

@Service
@Transactional
public final class FinalClassService {   // ❌ CGLIB: Cannot subclass final class → 기동 실패
    public void doWork() {
    }
}
```

```java
// 파일 경로: src/main/java/study/aop/bad/FinalMethodService.java
package study.aop.bad;

import org.springframework.cache.annotation.Cacheable;
import org.springframework.stereotype.Service;

@Service
public class FinalMethodService {

    @Cacheable("prices")
    public final int priceOf(Long itemId) {   // ❌ 조용히 무시: 매 호출마다 실행된다
        return 10_000;
    }

    @Cacheable("prices")
    private int privatePriceOf(Long itemId) {   // ❌ 조용히 무시: 프록시 대상이 아니다
        return 10_000;
    }

    @Cacheable("prices")
    public static int staticPriceOf(Long itemId) {   // ❌ 정적 메서드는 애초에 대상이 아니다
        return 10_000;
    }
}
```

정리하면 실패 모드는 세 가지다.

| 원인 | 증상 | 조치 |
|---|---|---|
| `final` 클래스 + AOP | 기동 실패 (`Cannot subclass final class ...`) | `final` 제거, 또는 인터페이스 추출 후 JDK Proxy |
| `final`/`private` 메서드 | **오류 없음**, 기능만 빠짐 | 애노테이션을 `public` 비-`final` 메서드로 이동 |
| Kotlin 클래스 | Kotlin 기본이 `final` → 기동 실패 | `kotlin-spring`(`all-open`) 플러그인 적용 |
| `static` 메서드 | 적용 안 됨 | 유틸은 AOP 대상으로 두지 않는다 |
| 인터페이스만 `@Transactional` | 프록시 종류·설정에 따라 탐색 실패 | **구현 클래스에 애노테이션을 붙인다** |

Kotlin 이슈는 Java 프로젝트에서도 참고할 만하다. Kotlin은 클래스·메서드가 기본 `final`이라 `kotlin-spring` 플러그인이 없으면 `@Transactional`이 붙은 클래스의 프록시 생성에서 실패한다.

### 2-7. 프록시가 무엇으로 만들어졌는지 확인하는 법

가장 빠른 확인은 빈의 클래스 이름이다.

```text
CGLIB (Spring Boot 3.x):  study.aop.target.InventoryService$$SpringCGLIB$$0
JDK Dynamic Proxy:        jdk.proxy2.$Proxy52
프록시 없음(원본):          study.aop.target.InventoryService
```

- `$$SpringCGLIB$$`는 스프링 6의 네이밍 정책이다. 5.x에서는 `$$EnhancerBySpringCGLIB$$`로 보인다.
- `$Proxy` + 숫자는 JDK Proxy다. 숫자는 생성 순서에 따라 달라진다.
- 코드에서는 `AopUtils.isAopProxy()`, `AopUtils.isCglibProxy()`, `AopUtils.isJdkProxy()`, `AopUtils.getTargetClass()`로 판정한다(4-2).

---

## 3. 실무 트러블슈팅: Self-Invocation (내부 메서드 호출) 이슈와 3대 해결책

### 3-1. 원인: `this`는 프록시가 아니라 Target이다

```text
[정상: 외부 호출]                    [문제: 자기 호출]

Controller                          OrderService (Target)
   │ @Autowired                          │
   ▼                                     │ placeOrder() 안에서
OrderService$$SpringCGLIB$$0 (프록시)    │ this.record()  ← 여기의 this는 프록시가 아니다
   │ ① Advisor 체인 진입                 ▼
   │ ② intercept → proceed            record()  ← 체인을 타지 않음
   ▼                                        → @Transactional/@Async/@Cacheable 전부 미적용
OrderService (Target)
   │ ③ placeOrder() 내부
   └── this.record()  ← 체인 종료 후 Target 내부이므로 다시 프록시로 돌아가지 않는다
```

핵심은 **"프록시는 컨테이너가 주입해 준 참조에만 존재한다"** 는 점이다. Target 객체 내부에서 `this`를 쓰면 그 객체는 자기 자신일 뿐이고, 그 객체에는 어드바이스가 없다. 자바의 메서드 호출은 정적 디스패치(또는 가상 디스패치)로 대상 객체의 메서드를 직접 실행하며, "프록시를 한 번 더 거치는" 경로가 없다.

증상은 애노테이션별로 다르게 나타난다.

| 애노테이션 | 자기 호출 시 증상 |
|---|---|
| `@Transactional` | 트랜잭션이 시작되지 않음. 예외를 던져도 롤백 안 됨, `REQUIRES_NEW` 무시 |
| `@Async` | **동기로 실행**된다. 응답 시간이 그대로 늘어남 |
| `@Cacheable` / `@CacheEvict` | 캐시를 조회/갱신하지 않음. 항상 DB를 때려 부하가 늘고, 캐시는 영원히 갱신 안 됨 |
| `@Retryable` | 재시도가 일어나지 않음. 첫 실패로 끝남 |
| `@PreAuthorize` | **권한 검사가 생략**된다. 보안 사고로 이어짐 |
| `@Validated`(메서드 검증) | 파라미터 검증이 생략됨 |
| 커스텀 `@AuditLog` 등 | 로그가 남지 않음. "감사 로그가 비어 있다"는 문의로 돌아옴 |

`@PreAuthorize`와 `@Validated`는 앞의 셋보다 위험하다. 트랜잭션·캐시는 성능·정합성 문제지만, 권한 검사 누락은 **인가 우회**다. 서비스 내부에서 "관리자 전용 메서드"를 `this.`로 호출하는 코드가 있으면 즉시 분리해야 한다.

탐지 순서는 다음과 같다.

1. `AopUtils.isAopProxy(this)`가 `false`인지 확인한다(디버거에서 `this`를 직접 검사).
2. `logging.level.org.springframework.transaction.interceptor=TRACE` / `...cache=TRACE` / `...aop=TRACE`로 해당 메서드의 인터셉터 로그가 **아예 없는지** 본다. 로그가 없다는 것이 프록시를 타지 않았다는 가장 확실한 증거다.
3. 운영 환경이면 Arthas의 `watch`/`trace`로 메서드 호출 경로를 확인한다. `this`와 `AopContext.currentProxy()`를 함께 출력하면 즉시 판별된다.

### 3-2. 해결 ① 별도 빈 분리 (권장)

가장 먼저 선택해야 하는 방법이다. "프록시를 우회하지 않게 호출한다"가 아니라 **"경계가 다른 일은 다른 빈에 둔다"** 는 설계 수정이다.

```java
// ❌ Before — 파일 경로: src/main/java/study/aop/order/BadOrderService.java
package study.aop.order;

import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Propagation;
import org.springframework.transaction.annotation.Transactional;

@Service
public class BadOrderService {

    private final OrderRepository orderRepository;
    private final CouponClient couponClient;

    public BadOrderService(OrderRepository orderRepository, CouponClient couponClient) {
        this.orderRepository = orderRepository;
        this.couponClient = couponClient;
    }

    @Transactional
    public Long placeOrder(OrderCommand command) {
        Order order = orderRepository.save(Order.from(command));

        // ❌ 자기 호출: 아래 메서드에 붙은 @Transactional(REQUIRES_NEW)이 무시된다.
        //    쿠폰 발급이 실패해 예외가 나도 "발급 이력"은 주문과 함께 롤백된다.
        this.issueCoupon(order);

        return order.getId();
    }

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void issueCoupon(Order order) {
        couponClient.issue(order.getUserId(), order.getId());
        orderRepository.saveCouponHistory(order.getId());   // 실패 이력은 남겨야 한다
    }
}
```

```java
// ✅ After — 파일 경로: src/main/java/study/aop/order/CouponIssueService.java
package study.aop.order;

import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Propagation;
import org.springframework.transaction.annotation.Transactional;

@Service
public class CouponIssueService {   // 경계가 다른 일은 별도 빈으로 분리 (실패 이력은 독립 커밋)

    private final OrderRepository orderRepository;
    private final CouponClient couponClient;

    public CouponIssueService(OrderRepository orderRepository, CouponClient couponClient) {
        this.orderRepository = orderRepository;
        this.couponClient = couponClient;
    }

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void issue(Order order) {
        couponClient.issue(order.getUserId(), order.getId());
        orderRepository.saveCouponHistory(order.getId());
    }
}
```

```java
// ✅ After — 파일 경로: src/main/java/study/aop/order/OrderService.java
package study.aop.order;

import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

@Service
public class OrderService {

    private final OrderRepository orderRepository;
    private final CouponIssueService couponIssueService;   // 주입된 참조 = 프록시

    public OrderService(OrderRepository orderRepository, CouponIssueService couponIssueService) {
        this.orderRepository = orderRepository;
        this.couponIssueService = couponIssueService;
    }

    @Transactional
    public Long placeOrder(OrderCommand command) {
        Order order = orderRepository.save(Order.from(command));

        // ✅ 다른 빈의 메서드 호출 → 프록시를 타므로 @Transactional(REQUIRES_NEW)이 동작한다.
        couponIssueService.issue(order);

        return order.getId();
    }
}
```

장점은 명확하다. 프록시를 의식한 코드가 아니라 **책임 분리**이므로 테스트가 쉽고, 순환 참조가 생기지 않으며, 다른 곳에서도 재사용할 수 있다. "이 메서드는 왜 별도 클래스인가?"라는 질문에 "커밋 경계가 다르기 때문"이라고 답할 수 있는 구조가 좋은 구조다.

### 3-3. 해결 ② `ObjectProvider<Self>` 지연 주입

기존 클래스를 쪼개기 어려울 때 쓴다. 자기 자신의 **프록시**를 컨테이너에서 나중에 꺼내 호출한다.

```java
// 파일 경로: src/main/java/study/aop/order/ProviderOrderService.java
package study.aop.order;

import org.springframework.beans.factory.ObjectProvider;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Propagation;
import org.springframework.transaction.annotation.Transactional;

@Service
public class ProviderOrderService {

    private final OrderRepository orderRepository;
    private final ObjectProvider<ProviderOrderService> selfProvider;   // 프록시를 지연 조회

    public ProviderOrderService(OrderRepository orderRepository,
                                ObjectProvider<ProviderOrderService> selfProvider) {
        this.orderRepository = orderRepository;
        this.selfProvider = selfProvider;
    }

    @Transactional
    public Long placeOrder(OrderCommand command) {
        Order order = orderRepository.save(Order.from(command));

        // getObject() 시점에 컨테이너가 프록시를 돌려준다 → AOP 적용
        selfProvider.getObject().recordHistory(order);

        return order.getId();
    }

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void recordHistory(Order order) {
        orderRepository.saveCouponHistory(order.getId());
    }
}
```

`@Lazy` 생성자 주입도 같은 효과다. 다만 **Spring Boot 2.6부터 순환 참조가 기본 금지**이므로 `@Lazy` 없이 자기 자신을 주입하면 `BeanCurrentlyInCreationException`이 난다.

```java
// 파일 경로: src/main/java/study/aop/order/LazySelfOrderService.java
package study.aop.order;

import org.springframework.context.annotation.Lazy;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Propagation;
import org.springframework.transaction.annotation.Transactional;

@Service
public class LazySelfOrderService {

    private final OrderRepository orderRepository;
    private final LazySelfOrderService self;   // @Lazy: 지연 해석 프록시가 주입된다

    public LazySelfOrderService(OrderRepository orderRepository,
                                @Lazy LazySelfOrderService self) {
        this.orderRepository = orderRepository;
        this.self = self;
    }

    @Transactional
    public Long placeOrder(OrderCommand command) {
        Order order = orderRepository.save(Order.from(command));
        self.recordHistory(order);   // 프록시 경유
        return order.getId();
    }

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void recordHistory(Order order) {
        orderRepository.saveCouponHistory(order.getId());
    }
}
```

주의할 점은 이 방식이 **"자기 자신에게 의존하는 설계"** 라는 냄새를 남긴다는 것이다. 코드 리뷰에서 "왜 이 빈이 자기 자신을 주입받나?"라는 질문이 나오면, 대부분 ①로 리팩터링하는 것이 답이다.

### 3-4. 해결 ③ `AopContext.currentProxy()` — 한계가 많은 최후 수단

`@EnableAspectJAutoProxy(exposeProxy = true)`로 현재 스레드의 프록시를 `AopContext`에 노출시켜 꺼내 쓰는 방법이다.

```java
// 파일 경로: src/main/java/study/aop/config/ExposeProxyConfig.java
package study.aop.config;

import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.EnableAspectJAutoProxy;

@Configuration
@EnableAspectJAutoProxy(exposeProxy = true)   // 기본값 false. 켜야 AopContext.currentProxy()가 동작한다
public class ExposeProxyConfig {
}
```

```java
// 파일 경로: src/main/java/study/aop/order/AopContextOrderService.java
package study.aop.order;

import org.springframework.aop.framework.AopContext;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Propagation;
import org.springframework.transaction.annotation.Transactional;

@Service
public class AopContextOrderService {

    private final OrderRepository orderRepository;

    public AopContextOrderService(OrderRepository orderRepository) {
        this.orderRepository = orderRepository;
    }

    @Transactional
    public Long placeOrder(OrderCommand command) {
        Order order = orderRepository.save(Order.from(command));

        // 현재 스레드에 바인딩된 프록시를 꺼내 다시 호출한다.
        // JDK Dynamic Proxy 설정이면 인터페이스 타입으로 캐스팅해야 한다.
        AopContextOrderService proxy = (AopContextOrderService) AopContext.currentProxy();
        proxy.recordHistory(order);

        return order.getId();
    }

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void recordHistory(Order order) {
        orderRepository.saveCouponHistory(order.getId());
    }
}
```

한계를 정리하면 다음과 같다. 이 목록이 길어서 "권장하지 않는 방법"이 된다.

| 한계 | 내용 |
|---|---|
| 전역 설정 의존 | `exposeProxy = true`는 애플리케이션 전체 프록시에 적용된다. 라이브러리 코드라면 켤 수 없다 |
| 스프링 API 침투 | 비즈니스 코드에 `AopContext`라는 인프라 타입이 들어온다. 순수 단위 테스트에서 프록시가 없으면 `IllegalStateException` |
| 캐스팅 취약 | JDK Proxy면 인터페이스로 캐스팅해야 하고, CGLIB면 구현 클래스로 캐스팅한다. 설정이 바뀌면 런타임 오류 |
| 중첩 호출 시 혼동 | 프록시를 거치지 않고 진입한 경로(배치, 테스트, 이벤트 리스너)에서는 `currentProxy()`가 예외를 던진다 |
| ThreadLocal 비용 | 호출마다 ThreadLocal 조회가 붙는다. 무시할 수준이지만 공짜는 아니다 |
| 예외 가시성 | 프록시가 없으면 `IllegalStateException: Cannot find current proxy` — 원인과 거리가 먼 곳에서 터진다 |

"당장 한 줄로 고쳐야 하는 핫픽스"에는 쓸 수 있지만, 표준으로 삼으면 안 된다. 다음 스프린트에 ①로 되돌리는 것을 전제로 임시 적용하는 편이 맞다.

### 3-5. 그 밖의 선택지

| 방법 | 원리 | 적합한 상황 |
|---|---|---|
| `TransactionTemplate` | 트랜잭션 경계를 코드로 직접 연다. 프록시와 무관 | 자기 호출 구간에만 별도 트랜잭션이 필요하고 클래스 분리가 과할 때 |
| `@TransactionalEventListener(AFTER_COMMIT)` | 커밋 후 실행 → 경계가 이미 닫혀 있다 | "커밋 후 알림·캐시 갱신"이 목적일 때 (자기 호출 문제 자체를 없앤다) |
| AspectJ 모드 (`AdviceMode.ASPECTJ`) | 프록시 없이 바이트코드에 직접 위빙 | `private` 메서드·자기 호출까지 적용해야 할 때. 빌드에 `aspectjweaver` 필요, 운영 복잡도 큼 |
| `@Cacheable` 대신 `CacheManager` 직접 사용 | 코드로 캐시 조회/저장 | 캐시 대상이 동적으로 결정되거나 자기 호출 구조를 유지해야 할 때 |

`AspectJ` 모드는 스프링 AOP의 근본 한계를 없애지만, 컴파일/로드타임 위빙 설정과 `-javaagent` 운영 이슈가 붙는다. 신입·중소 규모 서비스에서 선택할 이유는 거의 없다.

### 3-6. 판단 기준

```text
자기 호출을 발견했다
   │
   ├─ 커밋/캐시/인가 경계가 실제로 다른가?
   │     ├─ YES → ① 별도 빈 분리 (항상 이 선택이 먼저다)
   │     └─ NO  → 애노테이션을 붙인 게 잘못이다. 호출 구조를 정리한다
   │
   ├─ 클래스 분리가 구조적으로 불가능한가(상속·프레임워크 콜백 등)?
   │     └─ YES → ② ObjectProvider / @Lazy 자기 주입
   │
   ├─ 한 줄 핫픽스가 필요한가?
   │     └─ YES → ③ AopContext (+ exposeProxy), 다음 릴리스에서 ①로 되돌린다
   │
   └─ "커밋 후 작업"이 목적인가?
         └─ YES → @TransactionalEventListener(AFTER_COMMIT) / 이벤트로 분리
```

### 3-7. 함께 알아야 할 함정

- **`@Async` + 자기 호출**: `@Async`가 무시되면 **조용히 동기 실행**된다. 응답 시간이 늘어난 것을 "트래픽 증가"로 오해하기 쉽다. 비동기 작업은 별도 빈(`NotificationService`)에 두는 것이 정석이다.
- **`@Cacheable` + 자기 호출**: 캐시가 아예 동작하지 않으므로 DB 부하가 cache hit 기대치만큼 늘어난다. 반대로 `@CacheEvict`가 무시되면 **만료되지 않은 오래된 값**이 계속 서빙된다. 정합성 사고는 이쪽이 더 위험하다.
- **`@PreAuthorize` + 자기 호출**: 인가가 생략된다. 서비스 내부 편의 메서드에 `@PreAuthorize`를 붙이고 `this.`로 부르는 패턴은 보안 결함으로 취급해야 한다.
- **`@Configuration(proxyBeanMethods = false)`**: `@Bean` 메서드를 직접 호출해도 프록시가 개입하지 않는다. 그래서 `@Bean` 메서드 안에서 다른 `@Bean` 메서드를 호출하면 **새 인스턴스**가 만들어진다. 스프링 6/Boot 3은 `proxyBeanMethods = false`가 기본 권장이라, `@Bean` 간 의존은 메서드 호출이 아니라 **파라미터 주입**으로 받아야 한다.
- **자기 호출을 테스트로 고정하기**: 해결 후에는 "이 메서드는 실제로 별도 트랜잭션에서 커밋되는가"를 테스트로 못 박아야 한다. 트랜잭션 격리·캐시 hit 여부는 리팩터링에서 조용히 깨지는 대표 지점이다.

---

## 4. 실전 코드 예시

### 4-1. 실습 프로젝트 설정

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
    implementation 'org.springframework.boot:spring-boot-starter'            // 코어 컨테이너 + spring-core(CGLIB 내장)
    implementation 'org.springframework.boot:spring-boot-starter-aop'        // aspectjweaver, AOP 인프라
    implementation 'org.springframework.boot:spring-boot-starter-cache'      // @Cacheable (ConcurrentMapCacheManager)
    testImplementation 'org.springframework.boot:spring-boot-starter-test'
    testRuntimeOnly 'org.junit.platform:junit-platform-launcher'
}

tasks.named('test') {
    useJUnitPlatform()
}
```

```java
// 파일 경로: src/main/java/study/aop/StudyAopApplication.java
package study.aop;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cache.annotation.EnableCaching;

@EnableCaching   // @Cacheable 활성화 (자기 호출 실험용)
@SpringBootApplication
public class StudyAopApplication {

    public static void main(String[] args) {
        SpringApplication.run(StudyAopApplication.class, args);
    }
}
```

### 4-2. 프록시 검증 테스트 (CGLIB vs JDK)

DB 없이 컨텍스트만으로 검증한다. 같은 Target 클래스를 **CGLIB 컨텍스트**와 **JDK 컨텍스트**에 각각 등록해 프록시 타입과 호출 가로채기 여부를 비교한다.

```java
// 파일 경로: src/main/java/study/aop/target/InventoryUseCase.java
package study.aop.target;

public interface InventoryUseCase {

    int decrease(Long itemId, int quantity);
}
```

```java
// 파일 경로: src/main/java/study/aop/target/InventoryService.java
package study.aop.target;

import org.springframework.stereotype.Service;

@Service
public class InventoryService implements InventoryUseCase {

    @Override
    public int decrease(Long itemId, int quantity) {
        return quantity * 100;
    }

    // 인터페이스에 없는 메서드: JDK Proxy로는 호출할 수 없다(2-1)
    public String onlyInClass() {
        return "class-only";
    }
}
```

```java
// 파일 경로: src/main/java/study/aop/aspect/CallCountAspect.java
package study.aop.aspect;

import java.util.concurrent.atomic.AtomicInteger;

import org.aspectj.lang.ProceedingJoinPoint;
import org.aspectj.lang.annotation.Around;
import org.aspectj.lang.annotation.Aspect;
import org.springframework.stereotype.Component;

@Aspect
@Component
public class CallCountAspect {

    private final AtomicInteger callCount = new AtomicInteger();

    /** InventoryService의 모든 메서드 실행을 가로채 호출 횟수를 센다. */
    @Around("execution(* study.aop.target.InventoryService.*(..))")
    public Object count(ProceedingJoinPoint joinPoint) throws Throwable {
        callCount.incrementAndGet();
        return joinPoint.proceed();   // 다음 인터셉터 또는 Target 호출 (proceed 누락 시 Target이 실행되지 않는다)
    }

    public int callCount() {
        return callCount.get();
    }

    public void reset() {
        callCount.set(0);
    }
}
```

```java
// 파일 경로: src/test/java/study/aop/lab/CglibProxyConfig.java
// 실습용 설정은 테스트 소스에 둔다. 메인 코드에 두면 애플리케이션 컨텍스트가 두 설정을 함께
// 스캔해 proxyTargetClass 설정이 충돌한다.
package study.aop.lab;

import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.EnableAspectJAutoProxy;

@Configuration
@EnableAspectJAutoProxy(proxyTargetClass = true)   // CGLIB 강제 (Boot 기본값)
public class CglibProxyConfig {
}
```

```java
// 파일 경로: src/test/java/study/aop/lab/JdkProxyConfig.java
package study.aop.lab;

import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.EnableAspectJAutoProxy;

@Configuration
@EnableAspectJAutoProxy(proxyTargetClass = false)   // 인터페이스가 있으면 JDK Dynamic Proxy
public class JdkProxyConfig {
}
```

```java
// 파일 경로: src/test/java/study/aop/ProxyCreationTest.java
package study.aop;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Test;
import org.springframework.aop.framework.Advised;
import org.springframework.aop.support.AopUtils;
import org.springframework.context.annotation.AnnotationConfigApplicationContext;

import study.aop.aspect.CallCountAspect;
import study.aop.lab.CglibProxyConfig;
import study.aop.lab.JdkProxyConfig;
import study.aop.target.InventoryService;
import study.aop.target.InventoryUseCase;

class ProxyCreationTest {

    /** 컴포넌트 스캔 대신 명시 등록: 두 프록시 설정이 한 컨텍스트에 섞이지 않게 한다. */
    private static AnnotationConfigApplicationContext context(Class<?> config) {
        AnnotationConfigApplicationContext ctx = new AnnotationConfigApplicationContext();
        ctx.register(config, InventoryService.class, CallCountAspect.class);
        ctx.refresh();
        return ctx;
    }

    @Test
    @DisplayName("CGLIB: 대상 클래스를 상속한 프록시가 만들어지고 호출이 가로채진다")
    void cglibProxy() {
        try (AnnotationConfigApplicationContext ctx = context(CglibProxyConfig.class)) {

            InventoryUseCase bean = ctx.getBean(InventoryUseCase.class);
            CallCountAspect aspect = ctx.getBean(CallCountAspect.class);

            // 프록시 여부·종류 판정
            assertThat(AopUtils.isAopProxy(bean)).isTrue();
            assertThat(AopUtils.isCglibProxy(bean)).isTrue();
            assertThat(AopUtils.isJdkProxy(bean)).isFalse();
            assertThat(AopUtils.getTargetClass(bean)).isEqualTo(InventoryService.class);

            // 클래스 이름과 상속 관계로 확인
            System.out.println("proxy class = " + bean.getClass().getName());               // ...InventoryService$$SpringCGLIB$$0
            assertThat(bean.getClass().getName()).contains("$$SpringCGLIB$$");
            assertThat(bean.getClass().getSuperclass()).isEqualTo(InventoryService.class);

            // 구현 클래스 타입 캐스팅이 가능하다 (상속이므로)
            assertThat(((InventoryService) bean).onlyInClass()).isEqualTo("class-only");

            // 어드바이저 체인: 우리 Aspect가 1개 실려 있다
            assertThat(((Advised) bean).getAdvisors()).hasSize(1);

            // 실제 호출이 Aspect를 통과하는지 횟수로 확인
            aspect.reset();
            assertThat(bean.decrease(1L, 2)).isEqualTo(200);
            assertThat(aspect.callCount()).isEqualTo(1);
        }
    }

    @Test
    @DisplayName("JDK Proxy: 인터페이스를 구현한 프록시가 만들어지고, 구현 클래스로 캐스팅할 수 없다")
    void jdkProxy() {
        try (AnnotationConfigApplicationContext ctx = context(JdkProxyConfig.class)) {

            InventoryUseCase bean = ctx.getBean(InventoryUseCase.class);
            CallCountAspect aspect = ctx.getBean(CallCountAspect.class);

            assertThat(AopUtils.isAopProxy(bean)).isTrue();
            assertThat(AopUtils.isJdkProxy(bean)).isTrue();
            assertThat(AopUtils.isCglibProxy(bean)).isFalse();

            System.out.println("proxy class = " + bean.getClass().getName());               // jdk.proxy2.$ProxyNN
            assertThat(bean.getClass().getName()).contains("$Proxy");
            assertThat(bean.getClass().getSuperclass()).isEqualTo(java.lang.reflect.Proxy.class);

            // 상속이 아니므로 구현 클래스 타입으로는 쓸 수 없다
            assertThatThrownBy(() -> {
                InventoryService casted = (InventoryService) bean;
                casted.onlyInClass();
            }).isInstanceOf(ClassCastException.class);

            aspect.reset();
            assertThat(bean.decrease(1L, 2)).isEqualTo(200);
            assertThat(aspect.callCount()).isEqualTo(1);
        }
    }
}
```

이 테스트가 확인해 주는 것은 세 가지다. (1) 두 방식 모두 AOP가 동작한다, (2) CGLIB만 구현 클래스 타입으로 쓸 수 있다, (3) 프록시가 없으면 `Advised` 캐스팅과 `callCount` 증가가 모두 실패한다. 마지막 항목은 1-6의 "프록시 미적용"을 테스트로 번역한 것이다.

### 4-3. 자기 호출 전/후 코드 (@Cacheable)

`@Cacheable`은 자기 호출 문제를 **숫자로** 보여 주기 좋다. 캐시 키가 같으면 Target 메서드 실행 횟수가 늘지 않아야 정상이다.

```java
// ❌ Before — 파일 경로: src/main/java/study/aop/price/BadPriceService.java
package study.aop.price;

import java.util.concurrent.atomic.AtomicInteger;

import org.springframework.cache.annotation.Cacheable;
import org.springframework.stereotype.Service;

@Service
public class BadPriceService {

    private final AtomicInteger dbCalls = new AtomicInteger();

    /** 외부에서 호출하면 캐시가 동작한다. */
    public int getPrice(Long itemId) {
        return this.loadFromDb(itemId);   // ❌ 자기 호출 → @Cacheable 무시
    }

    @Cacheable(cacheNames = "prices", key = "#itemId")
    public int loadFromDb(Long itemId) {
        dbCalls.incrementAndGet();
        return 10_000;                    // 실제로는 DB 조회
    }

    public int dbCalls() {
        return dbCalls.get();
    }
}
```

```java
// ✅ After (① 별도 빈 분리 — 권장) — 파일 경로: src/main/java/study/aop/price/PriceCache.java
package study.aop.price;

import java.util.concurrent.atomic.AtomicInteger;

import org.springframework.cache.annotation.Cacheable;
import org.springframework.stereotype.Component;

@Component
public class PriceCache {

    private final AtomicInteger dbCalls = new AtomicInteger();

    @Cacheable(cacheNames = "prices", key = "#itemId")
    public int load(Long itemId) {
        dbCalls.incrementAndGet();
        return 10_000;
    }

    public int dbCalls() {
        return dbCalls.get();
    }
}
```

```java
// ✅ After — 파일 경로: src/main/java/study/aop/price/PriceService.java
package study.aop.price;

import org.springframework.stereotype.Service;

@Service
public class PriceService {

    private final PriceCache priceCache;   // 주입된 참조 = 프록시

    public PriceService(PriceCache priceCache) {
        this.priceCache = priceCache;
    }

    public int getPrice(Long itemId) {
        return priceCache.load(itemId);    // ✅ 프록시 경유 → 두 번째 호출부터 캐시 hit
    }
}
```

```java
// 검증 테스트 — 파일 경로: src/test/java/study/aop/SelfInvocationCacheTest.java
package study.aop;

import static org.assertj.core.api.Assertions.assertThat;

import java.util.Objects;

import org.junit.jupiter.api.AfterAll;
import org.junit.jupiter.api.BeforeAll;
import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Test;
import org.springframework.cache.CacheManager;
import org.springframework.cache.annotation.EnableCaching;
import org.springframework.context.annotation.AnnotationConfigApplicationContext;
import org.springframework.context.annotation.Configuration;

import study.aop.price.BadPriceService;
import study.aop.price.PriceCache;
import study.aop.price.PriceService;

class SelfInvocationCacheTest {

    /** 부트 컨텍스트 없이 캐시 실험에 필요한 빈만 등록한다 (프록시 모드 실습 설정과 섞이지 않게). */
    @Configuration
    @EnableCaching
    static class CacheLabConfig {
    }

    private static AnnotationConfigApplicationContext ctx;
    private static BadPriceService badPriceService;
    private static PriceService priceService;
    private static PriceCache priceCache;
    private static CacheManager cacheManager;

    @BeforeAll
    static void setUp() {
        ctx = new AnnotationConfigApplicationContext();
        ctx.register(CacheLabConfig.class, BadPriceService.class, PriceService.class, PriceCache.class);
        ctx.refresh();
        badPriceService = ctx.getBean(BadPriceService.class);
        priceService = ctx.getBean(PriceService.class);
        priceCache = ctx.getBean(PriceCache.class);
        cacheManager = ctx.getBean(CacheManager.class);
    }

    @AfterAll
    static void tearDown() {
        ctx.close();
    }

    @Test
    @DisplayName("자기 호출: @Cacheable이 무시되어 호출마다 DB를 조회한다")
    void selfInvocationBypassesCache() {
        badPriceService.getPrice(1L);
        badPriceService.getPrice(1L);

        assertThat(badPriceService.dbCalls()).isEqualTo(2);   // 캐시가 동작했다면 1
    }

    @Test
    @DisplayName("별도 빈 분리: 두 번째 호출은 캐시에서 반환된다")
    void separatedBeanHitsCache() {
        Objects.requireNonNull(cacheManager.getCache("prices")).clear();
        priceService.getPrice(2L);
        priceService.getPrice(2L);

        assertThat(priceCache.dbCalls()).isEqualTo(1);        // 캐시 hit
    }
}
```

주의: 캐시는 컨텍스트에 하나만 있으므로 테스트 간 오염을 막으려면 키를 분리하거나 `@BeforeEach`에서 `cacheManager.getCache("prices").clear()`로 비운다. 실제 프로젝트에서는 `ConcurrentMapCacheManager`가 아닌 Redis 캐시를 쓰므로 캐시 키 네이밍과 TTL까지 함께 검증해야 한다.

### 4-4. 프록시 생성·호출 비용 측정 (참고용, JMH 권장)

간단한 측정 코드는 **경향**만 볼 수 있다. 정확한 비교는 JMH로 해야 한다.

```java
// 파일 경로: src/test/java/study/aop/ProxyCostMeasureTest.java
// 주의: JIT 워밍업·dead code elimination을 통제하지 못한 측정이다. 절대값이 아니라 경향만 본다.
package study.aop;

import org.junit.jupiter.api.Test;
import org.springframework.cglib.proxy.Enhancer;
import org.springframework.cglib.proxy.MethodInterceptor;

class ProxyCostMeasureTest {

    static class Target {
        int add(int a, int b) {
            return a + b;
        }
    }

    interface Op {
        int add(int a, int b);
    }

    static class OpImpl implements Op {
        @Override
        public int add(int a, int b) {
            return a + b;
        }
    }

    @Test
    void measure() {
        int creationLoops = 10_000;
        int callLoops = 1_000_000;

        // 1) 생성 비용: CGLIB은 Enhancer.create()마다 클래스 생성/캐시 조회가 붙는다
        long cglibStart = System.nanoTime();
        for (int i = 0; i < creationLoops; i++) {
            Enhancer enhancer = new Enhancer();
            enhancer.setSuperclass(Target.class);
            enhancer.setCallback((MethodInterceptor) (obj, method, args, proxy) -> proxy.invokeSuper(obj, args));
            enhancer.create();
        }
        long cglibCreation = System.nanoTime() - cglibStart;

        // 2) 직접 호출 vs 프록시 호출 (웜업 포함)
        Op direct = new OpImpl();
        Op proxy = (Op) java.lang.reflect.Proxy.newProxyInstance(
                Op.class.getClassLoader(),
                new Class<?>[]{Op.class},
                (p, method, args) -> method.invoke(direct, args));

        long directStart = System.nanoTime();
        int sum = 0;
        for (int i = 0; i < callLoops; i++) {
            sum += direct.add(i, 1);
        }
        long directCall = System.nanoTime() - directStart;

        long proxyStart = System.nanoTime();
        for (int i = 0; i < callLoops; i++) {
            sum += proxy.add(i, 1);
        }
        long proxyCall = System.nanoTime() - proxyStart;

        System.out.printf("CGLIB 생성: %.0f ns/건%n", (double) cglibCreation / creationLoops);
        System.out.printf("직접 호출 : %.2f ns/건%n", (double) directCall / callLoops);
        System.out.printf("JDK 프록시: %.2f ns/건%n", (double) proxyCall / callLoops);
        System.out.println("(sum=" + sum + " — dead code elimination 방지용)");
    }
}
```

측정 시 지켜야 할 최소한의 규칙:

1. **워밍업을 먼저 돌린다.** JIT가 컴파일하기 전 코드는 인터프리터로 실행되어 수십 배 느리다.
2. **결과를 소비한다.** 합계를 출력하지 않으면 JIT가 루프 전체를 제거할 수 있다.
3. **한 JVM에서 여러 번 반복**하고 중앙값을 본다. 단일 측정은 노이즈가 크다.
4. **운영 JDK·GC·CPU에서 잰다.** 노트북 측정값을 운영 결정의 근거로 쓰지 않는다.

---

## 5. 기술 면접 단골 꼬리 질문 & 모범 답변 5선

**Q1. Spring AOP와 AspectJ의 차이는 무엇이고, 실무에서 AspectJ를 선택할 상황은 언제인가요?**
- **A.** 둘의 차이는 **위빙 시점과 조인 포인트 범위**입니다. Spring AOP는 런타임에 빈을 프록시로 감싸 메서드 호출을 가로채는 방식이라 조인 포인트가 **메서드 실행**으로 한정되고, 같은 빈 내부 호출·`private` 메서드·`final` 메서드에는 적용되지 않습니다. AspectJ는 컴파일 타임이나 클래스 로드 타임에 바이트코드 자체를 위빙하므로 필드 접근·생성자 호출·`static` 초기화까지 조인 포인트로 삼고, 프록시가 없으니 자기 호출 문제도 없습니다. 대신 `aspectjweaver`와 컴파일러/`-javaagent` 설정이 필요하고, 빌드·배포 파이프라인이 복잡해지며 디버깅이 어려워집니다. 그래서 실무에서는 대부분 Spring AOP로 충분하고, AspectJ는 (1) `private` 메서드나 자기 호출까지 반드시 적용해야 하는 레거시 코드, (2) 성능 측정처럼 프록시 오버헤드조차 피해야 하는 경우에만 검토합니다. 스프링도 `@Transactional(mode = ASPECTJ)` 같은 선택지를 제공하지만 기본은 프록시 모드입니다.

**Q2. JDK Dynamic Proxy와 CGLIB의 차이를 설명하고, Spring Boot가 CGLIB를 기본값으로 둔 이유는 무엇인가요?**
- **A.** JDK Dynamic Proxy는 `java.lang.reflect.Proxy`로 **인터페이스를 구현한 클래스**를 런타임에 생성하고 모든 호출을 `InvocationHandler`로 위임합니다. 인터페이스가 필수이고, 인터페이스에 선언되지 않은 메서드는 가로챌 수 없으며, 구현 클래스 타입으로 캐스팅할 수 없습니다. CGLIB는 ASM으로 **대상 클래스를 상속한 서브클래스**를 생성해 메서드를 오버라이드하고 `MethodProxy.invokeSuper()`로 원본을 호출합니다. 인터페이스가 필요 없고 구현 클래스 타입으로도 쓸 수 있지만, `final` 클래스는 만들 수 없고 `final`·`private`·`static` 메서드는 오버라이드할 수 없습니다. 성능은 생성 속도가 JDK 방식이 빠르고 호출 속도는 CGLIB가 약간 빠른데, Java 8의 리플렉션 인플레이션과 JDK 18의 JEP 416 이후 격차가 줄어 성능으로 선택할 이유는 거의 없습니다. Spring Boot가 CGLIB를 기본으로 둔 이유는 **인터페이스 없이 클래스에 애노테이션만 붙여도 동작하고, `@Autowired`를 구체 클래스 타입으로 받는 코드가 깨지지 않기 때문**입니다. `spring.aop.proxy-target-class`가 없으면 `true`로 간주하고, 순수 Spring의 `@EnableAspectJAutoProxy` 기본값은 반대로 `false`(인터페이스가 있으면 JDK)라는 차이도 함께 기억해야 합니다.

**Q3. 같은 클래스 안에서 `@Transactional` 메서드를 호출하면 왜 동작하지 않으며, 어떻게 해결하나요?**
- **A.** `@Transactional`은 프록시 기반 AOP라서 **프록시로 들어오는 호출**만 `TransactionInterceptor`가 가로챕니다. 그런데 같은 클래스 안에서 `this.inner()`로 호출하면 `this`는 프록시가 아니라 Target 객체이므로 인터셉터 체인을 타지 않고, 트랜잭션 코드가 실행될 기회가 없습니다. 그래서 `REQUIRES_NEW`, `readOnly`, `rollbackFor` 같은 속성이 전부 무시되고, 예외를 던져도 롤백되지 않습니다. 해결은 세 가지입니다. 첫째, **트랜잭션 경계가 다른 일을 별도 빈으로 분리**하는 것이 가장 권장됩니다. 구조가 명확하고 테스트하기 쉽습니다. 둘째, `ObjectProvider<Self>`나 `@Lazy`로 자기 자신의 프록시를 주입받아 호출하는 방법인데, Boot 2.6부터 순환 참조가 금지되어 `@Lazy`가 필수이고 "자기 자신에게 의존하는 설계"라는 냄새가 남습니다. 셋째, `@EnableAspectJAutoProxy(exposeProxy = true)`와 `AopContext.currentProxy()`로 현재 프록시를 꺼내는 방법인데, 전역 설정에 의존하고 스프링 API가 비즈니스 코드에 침투하며 캐스팅이 취약해 핫픽스용입니다. `@Async`, `@Cacheable`, `@PreAuthorize`도 같은 원리로 무력화되며, 특히 `@PreAuthorize`는 권한 검사가 생략되는 보안 문제가 됩니다.

**Q4. `final` 클래스나 `final` 메서드에 AOP 애노테이션을 붙이면 어떻게 되나요? 그리고 프록시가 적용됐는지 어떻게 확인하나요?**
- **A.** CGLIB는 대상 클래스를 **상속**해 프록시를 만들기 때문에 `final` 클래스에는 프록시를 만들 수 없고, `final` 메서드는 오버라이드할 수 없어 가로채지 못합니다. 차이는 실패 방식입니다. `final` 클래스에 `@Transactional`을 붙이면 기동 중 `Cannot subclass final class` 예외로 **애플리케이션이 뜨지 않아** 바로 알 수 있습니다. 반면 `final` 메서드나 `private` 메서드에 붙이면 기동도 정상이고 예외도 없이 **조용히 무시**되어 가장 위험합니다. 확인 방법은 세 가지입니다. (1) `AopUtils.isAopProxy(bean)`, `isCglibProxy`, `isJdkProxy`로 프록시 여부와 종류를 판정하고 `AopUtils.getTargetClass()`로 원본 클래스를 확인합니다. (2) 빈 클래스 이름을 출력해 `$$SpringCGLIB$$`나 `$Proxy`가 붙는지 봅니다. (3) `logging.level.org.springframework.transaction.interceptor=TRACE` 같은 인터셉터 로그에서 해당 메서드의 트랜잭션 시작 로그가 아예 없는지 확인합니다. 로그가 없다는 것이 프록시를 타지 않았다는 가장 확실한 증거입니다. Kotlin은 클래스 기본이 `final`이라 `kotlin-spring`(all-open) 플러그인이 없으면 같은 문제가 납니다.

**Q5. 하나의 빈에 `@Transactional`, `@Cacheable`, `@Async`가 함께 붙으면 어떻게 동작하나요? 프록시가 여러 개 생기나요?**
- **A.** 프록시는 **빈당 하나**만 만들어지고, 매칭된 Advisor들이 그 프록시 안에 **하나의 인터셉터 체인**으로 순서대로 들어갑니다. 즉 `@Transactional`용 프록시, `@Cacheable`용 프록시가 따로 겹치는 것이 아니라, 하나의 프록시가 체인을 순서대로 실행합니다. 그래서 **체인 순서가 의미를 가집니다.** 예를 들어 캐시 인터셉터가 트랜잭션 인터셉터보다 바깥에 있으면 캐시 조회가 트랜잭션 시작 전에 일어나고, 반대면 트랜잭션을 연 뒤 캐시를 봅니다. 순서는 Advisor의 `order`(`@Order`, `Ordered`)로 정해지는데, 기본값을 계약처럼 믿지 말고 순서가 중요한 경우 명시하거나 아예 역할을 분리하는 편이 안전합니다. 또 `@Async`는 체인 중간에서 Target 호출을 다른 스레드로 넘기고 즉시 반환하므로, 그 뒤의 인터셉터와 Target은 **다른 스레드**에서 실행됩니다. ThreadLocal에 묶인 트랜잭션·SecurityContext·MDC가 따라가지 않으므로, "비동기 + 트랜잭션"이 필요하면 커밋 이후 이벤트(`@TransactionalEventListener(AFTER_COMMIT)`)로 분리하는 것이 정석입니다. 그리고 이 모든 체인이 자기 호출에서는 통째로 생략된다는 점이 실무에서 가장 자주 사고를 냅니다.

---

## 6. 오늘의 체크리스트 (내일 이어서)

- [ ] `AnnotationConfigApplicationContext` 두 개(CGLIB/JDK)로 같은 Target을 등록해 `AopUtils.isCglibProxy`/`isJdkProxy` 결과와 프록시 클래스 이름 차이 캡처
- [ ] `spring.aop.proxy-target-class=false`로 부팅했을 때 기존 `@Transactional` 서비스 중 **구체 클래스 타입 주입**으로 깨지는 지점 목록화
- [ ] `final` 클래스·`final` 메서드·`private` 메서드 각각에 `@Cacheable`을 붙여 (기동 실패 / 조용히 무시) 차이 재현
- [ ] 자기 호출 3가지 해결책(별도 빈 / `ObjectProvider` / `AopContext`)을 같은 시나리오로 구현하고 테스트 코드 비교
- [ ] `@PreAuthorize` 자기 호출로 인가가 생략되는 케이스를 재현하고, 분리 후 403이 나오는지 테스트로 고정
- [ ] `@Async` 자기 호출(동기 실행)과 분리 후(비동기 실행) 응답 시간 차이 측정
- [ ] 커스텀 Aspect의 `@Around`에서 `proceed()`를 호출하지 않았을 때의 증상과 로그 확인
- [ ] `@Configuration(proxyBeanMethods = false)`에서 `@Bean` 메서드 직접 호출 시 인스턴스가 중복 생성되는지 확인
- [ ] JMH로 JDK Proxy vs CGLIB 호출 비용 측정(웜업 포함) 후 결과를 노트에 기록

---

## 참고

- Spring Framework Reference — Proxying Mechanisms: https://docs.spring.io/spring-framework/reference/core/aop/proxying.html
- Spring Framework Reference — Aspect Oriented Programming: https://docs.spring.io/spring-framework/reference/core/aop.html
- Spring Boot Reference — AOP: https://docs.spring.io/spring-boot/reference/features/aop.html
- JEP 416: Reimplement Core Reflection with Method Handles (JDK 18): https://openjdk.org/jeps/416
- 이전 노트 — `@Transactional` 전파·readOnly·롤백: [2026-11-08](./2026-11-08-spring-transactional-aop-propagation-and-self-invocation.md)
