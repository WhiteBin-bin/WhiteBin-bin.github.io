---
title: "Spring & Design Pattern"
date: 2026-02-23 23:00:00 +0900
categories: [BackEnd]
subcategory: Spring, DesignPattern
description: "Spring & Design Pattern에 대해서 적어봤습니다."
og_image: /assets/img/posts/Spring&DesignPattern.png
---

![](/assets/img/posts/Spring&DesignPattern.png)

## 주제 선정 이유  

프로젝트를 진행하면서 `Spring`을 사용하다 보면 `@Transactional`, `JdbcTemplate`, `AOP`와 같은 기능들을 자연스럽게 적용하게 되고, 복잡한 로직도 비교적 쉽게 구현할 수 있어 생산성 측면에서 큰 이점을 느끼게 되지만 이러한 기능들이 
내부적으로 어떤 방식으로 동작하는지에 대해서는 깊게 고민하지 않은 채 사용하는 경우가 많습니다.  

실제로 기능은 문제없이 동작하지만, 트랜잭션이 언제 시작되고 어떻게 커밋되는지, 그리고 스프링이 왜 인터페이스 
기반 구조를 강조하는지에 대해서는 명확히 설명하기 어렵다는 점을 느끼게 되고 스프링이 프록시 기반으로 
동작한다는 점, `Template`이라는 이름의 클래스들이 반복적으로 등장한다는 점, 인터페이스와 구현체가 분리된 구조가 일관되게 적용된다는 점을 보면서 이러한 구조가 단순한 관습이 아니라 의도된 설계라는 생각이 들게 됩니다.  

그래서 이번 글에서는 <span style="background-color:#6DB33F">**스프링 내부에 적용된 핵심 디자인 패턴들이 어떤 역할을 하며, 이러한 구조가 왜 필요한지**</span>를 
중심으로 살펴보고, 이를 통해 스프링의 설계 원리를 정리해보고자 합니다.

## 디자인 패턴이 뭘까?

![](/assets/img/posts/DesignPattern.png)

소프트웨어 설계 과정에서 반복적으로 등장하는 문제를 해결하기 위한 검증된 설계 방식이고 쉽게 설명하면 코드를 
어떻게 구조화할 것인지에 대한 하나의 설계 템플릿이라고 볼 수 있는데 스프링은 이러한 디자인 패턴을 적극적으로 
활용해 확장성과 유연성, 유지보수성을 동시에 확보한 프레임워크입니다.

### 스프링에서 패턴이 중요한 이유

엔터프라이즈 애플리케이션에서는 비즈니스 로직 외에도 트랜잭션, 로깅, 보안, 예외 처리 같은 인프라 스트럭처 
요구사항이 필연적으로 따라오며 문제는 이런 코드가 서비스/컨트롤러 곳곳에 섞이기 시작하면, 핵심 로직이 흐려지고 유지보수 비용이 폭발한다는 점이며 스프링은 이 문제를 패턴을 프레임워크 내부에 내재화하는 방식으로 풀어냈습니다.
- `IoC` / `DI`로 객체 생성과 조립 책임을 컨테이너로 넘기고  
- `AOP`로 횡단 관심사를 비즈니스 코드 밖으로 분리하며  
- `Template` 계열로 반복되는 보일러플레이트를 템플릿+콜백으로 압축합니다.

즉, 스프링을 편하게 쓰는 것을 넘어 제대로 쓰는 것은 내가 작성한 코드가 어떤 패턴 위에서 동작하는지 이해하는 것과 
연결됩니다.

### 템플릿 메서드 패턴

![](/assets/img/posts/TemplateMethod.png)

상위 클래스에서 전체 실행 흐름을 정의하고, 하위 클래스에서 일부 단계만 재정의하도록 만드는 구조이며 `JdbcTemplate`, `TransactionTemplate` 등이 예시입니다.

```java
// 템플릿 (상위 클래스)
public abstract class AbstractProcess {

    // 템플릿 메서드 (전체 흐름 고정)
    public void execute() {
        step1();
        step2();
        hook();   // 하위 클래스에서 재정의
        step3();
    }

    private void step1() {
        System.out.println("공통 로직 1");
    }

    private void step2() {
        System.out.println("공통 로직 2");
    }

    protected abstract void hook(); // 변경 지점

    private void step3() {
        System.out.println("공통 로직 3");
    }
}

// 하위 클래스
public class ConcreteProcess extends AbstractProcess {
    @Override
    protected void hook() {
        System.out.println("변경되는 로직");
    }
}
```

### 전략 패턴

![](/assets/img/posts/StrategyPattern.png)

행동을 객체로 분리하고, 실행 시점에 교체 가능하도록 만드는 구조이며 인터페이스를 중심으로 구현체를 주입받는 
방식이 이에 해당되고 `PlatformTransactionManager`, `ViewResolver` 등이 예시입니다.

```java
// 전략 인터페이스
public interface PaymentStrategy {
    void pay(int amount);
}

// 전략 구현체 1
public class CardPayment implements PaymentStrategy {
    @Override
    public void pay(int amount) {
        System.out.println("카드 결제: " + amount);
    }
}

// 전략 구현체 2
public class KakaoPayPayment implements PaymentStrategy {
    @Override
    public void pay(int amount) {
        System.out.println("카카오페이 결제: " + amount);
    }
}

// Context
public class PaymentService {

    private final PaymentStrategy paymentStrategy;

    public PaymentService(PaymentStrategy paymentStrategy) {
        this.paymentStrategy = paymentStrategy;
    }

    public void process(int amount) {
        paymentStrategy.pay(amount);
    }
}
```

### 템플릿 콜백 패턴

![](/assets/img/posts/TemplateCallback.png)

템플릿 메서드 패턴을 더 유연하게 만든 구조로, 상속 대신 콜백 객체를 통해 변경되는 로직을 전달하고 `JdbcTemplate.query()`, `RowMapper`, `TransactionTemplate.execute()`등이 예시입니다.

```java
public class SimpleTemplate {

    public void execute(Callback callback) {
        System.out.println("공통 전처리");
        callback.call();  // 변경 지점
        System.out.println("공통 후처리");
    }
}

@FunctionalInterface
interface Callback {
    void call();
}

// 사용
SimpleTemplate template = new SimpleTemplate();

template.execute(() -> {
    System.out.println("비즈니스 로직 실행");
});
```

### 프록시 패턴

![](/assets/img/posts/ProxyPattern.png)

실제 객체 대신 대리 객체가 먼저 호출을 받고, 부가 기능을 추가한 뒤 실제 객체를 호출하는 구조이며 
`@Transactional`, `AOP` 기능이 프록시 기반으로 동작합니다.

```java
// 실제 객체 인터페이스
public interface Service {
    void execute();
}

// 실제 객체
public class RealService implements Service {
    @Override
    public void execute() {
        System.out.println("비즈니스 로직 실행");
    }
}

// 프록시
public class ServiceProxy implements Service {

    private final Service target;

    public ServiceProxy(Service target) {
        this.target = target;
    }

    @Override
    public void execute() {
        System.out.println("트랜잭션 시작");
        target.execute();
        System.out.println("트랜잭션 커밋");
    }
}
```

### 데코레이터 패턴

![](/assets/img/posts/DecoratePattern.png)

기존 객체를 감싸 기능을 확장하는 구조이며 기존 코드를 수정하지 않고 기능을 추가할 수 있다는 점이 특징이고 `HttpServletRequestWrapper`, `Filter Chain` 구조 등이 예시입니다.

```java
// 공통 인터페이스
public interface Notifier {
    void send(String message);
}

// 기본 구현체 (핵심 기능)
public class BasicNotifier implements Notifier {

    @Override
    public void send(String message) {
        System.out.println("기본 알림 전송: " + message);
    }
}

// 데코레이터 추상 클래스
public abstract class NotifierDecorator implements Notifier {

    protected Notifier notifier;

    public NotifierDecorator(Notifier notifier) {
        this.notifier = notifier;
    }

    @Override
    public void send(String message) {
        notifier.send(message);
    }
}

// 기능 확장 1 (SMS 추가)
public class SmsNotifier extends NotifierDecorator {

    public SmsNotifier(Notifier notifier) {
        super(notifier);
    }

    @Override
    public void send(String message) {
        super.send(message);
        sendSms(message);
    }

    private void sendSms(String message) {
        System.out.println("SMS 알림 전송: " + message);
    }
}

// 기능 확장 2 (Email 추가)
public class EmailNotifier extends NotifierDecorator {

    public EmailNotifier(Notifier notifier) {
        super(notifier);
    }

    @Override
    public void send(String message) {
        super.send(message);
        sendEmail(message);
    }

    private void sendEmail(String message) {
        System.out.println("이메일 알림 전송: " + message);
    }
}
```

### 싱글톤 패턴

![](/assets/img/posts/SingleTon.png)

애플리케이션 내에서 특정 객체를 하나만 생성하고 공유하는 구조이며 스프링에서는 빈의 기본 스코프가 
`singleton`이고 `GoF` 방식처럼 클래스 자체가 싱글톤이 되는 것이 아니라, 스프링 컨테이너가 객체를 하나만 생성하고 재사용하도록 관리한다는 점이 핵심입니다.
- 객체 생성 비용 절감
- 공용 서비스/리포지토리 객체 공유
- 단, 상태를 가지면 동시성 문제 발생 가능

```java
// 전통적인 싱글톤
public class Singleton {

    private static final Singleton instance = new Singleton();

    private Singleton() {}

    public static Singleton getInstance() {
        return instance;
    }
}
```

### 팩토리 패턴

![](/assets/img/posts/FactoryPattern.png)

객체 생성 책임을 별도의 팩토리로 위임하는 구조이며 클라이언트는 직접 `new` 하지 않고 팩토리를 통해 객체를 받고 
스프링에서는 `BeanFactory`, `ApplicationContext`가 거대한 팩토리 역할을 하며 `getBean()`이 팩토리 메서드처럼 
동작합니다.
- 객체 생성과 사용의 분리
- 결합도 분리
- `IOC` / `DI`의 기반 구조

```java
// 공통 인터페이스
public interface Product {
    void use();
}

// 구현체
public class ConcreteProduct implements Product {
    @Override
    public void use() {
        System.out.println("제품 사용");
    }
}

// 팩토리
public class ProductFactory {
    public static Product createProduct() {
        return new ConcreteProduct();
    }
}

// 사용
Product product = ProductFactory.createProduct();
product.use();
```

### 어댑터 패턴

![](/assets/img/posts/AdapterPattern.png)

서로 다른 인터페이스를 가진 객체들을 연결해주는 구조이며 기존 코드를 수정하지 않고도 새로운 구조에 맞게 
변환할 수 있으며 `Spring MVC`에서는 `HandlerAdapter`가 대표적인 예시입니다.

```java
// 기존 클래스
public class LegacyService {
    public void specificRequest() {
        System.out.println("레거시 실행");
    }
}

// 타겟 인터페이스
public interface Target {
    void request();
}

// 어댑터
public class Adapter implements Target {

    private final LegacyService legacyService;

    public Adapter(LegacyService legacyService) {
        this.legacyService = legacyService;
    }

    @Override
    public void request() {
        legacyService.specificRequest();
    }
}

// 사용
Target target = new Adapter(new LegacyService());
target.request();
```

### 옵저버 패턴

![](/assets/img/posts/ObserverPattern.png)

어떤 이벤트가 발생했을 때 이를 구독하고 있는 여러 객체에게 알림을 전달하는 구조이며 스프링에서는 `ApplicationEventPublisher`와 `@EventListener`를 통해 구현되고 컴포넌트 간 결합도를 낮추는데 사용됩니다.
- 이벤트 기반 구조
- 핵심 로직과 부가 기능 분리
- 비동기 처리 확장 가능

```java
// 옵저버 인터페이스
public interface Observer {
    void update(String message);
}

// 구체 옵저버
public class ConcreteObserver implements Observer {
    @Override
    public void update(String message) {
        System.out.println("알림 수신: " + message);
    }
}

// 주체(Subject)
public class Subject {

    private final List<Observer> observers = new ArrayList<>();

    public void addObserver(Observer observer) {
        observers.add(observer);
    }

    public void notifyObservers(String message) {
        for (Observer observer : observers) {
            observer.update(message);
        }
    }
}

// 사용
Subject subject = new Subject();
subject.addObserver(new ConcreteObserver());
subject.notifyObservers("이벤트 발생");
```

## 그래서 어떻게 쓰라고?

![](/assets/img/posts/jake.gif)

### 싱글톤은 무상태로 설계

스프링 빈은 기본적으로 싱글톤이므로 서비스, 리포지토리 같은 빈에 상태를 저장하면 동시성 문제가 발생할 수 
있습니다.
- 인스턴스 변수에 요청 데이터를 저장하지 말 것
- 상태가 필요한 경우 지역 변수 사용
- 꼭 필요한 경우 `request` / `session` 스코프 활용

### new 대신 컨테이너를 사용

객체를 직접 생성하지 말고 스프링 컨테이너에 맡겨야 합니다.
- `new`를 줄이고 의존성 주입을 사용
- 인터페이스 기반으로 설계
- 구현체 교체 가능하도록 구조 유지

### 횡단 관심사는 AOP

트랜잭션, 로깅, 보안 같은 코드를 비즈니스 로직 안에 직접 작성하지 말고 `AOP`로 분리합니다.
- `@Transactional`
- `@Aspect`
- `@Around`, `@Before`

### 확장은 상속보다 조합을 사용

기능을 추가할 때 기존 클래스를 수정하기보다는 감싸서 확장하는 방식으로 고민해보는것도 좋습니다.
- 필터 체인
- 래퍼 클래스
- 프록시 구조

### 이벤트로 결합도를 낮춘다

하나의 로직에 여러 부가 기능이 붙는다면 직접 호출 대신 이벤트 발행 구조를 고려해볼 수 있습니다.
- `ApplicationEventPublisher`
- `@EventListener`
- 필요 시 `@Async`로 비동기 처리

## 마무리
스프링의 디자인 패턴을 정리하면서 느낀 핵심을 다시 정리해보면 다음과 같습니다.

1. 스프링은 단순히 기능을 제공하는 프레임워크가 아니라, 싱글톤·팩토리·전략·프록시·옵저버 등 다양한 
디자인 패턴을 유기적으로 결합해 설계된 구조라는 점을 이해하게 되었습니다.
2. `@Transactional`, `JdbcTemplate`, `AOP` 같은 기능들은 단순한 편의 기능이 아니라, 템플릿·프록시·전략 패턴 위에서 동작하는 결과물이며, 내부 구조를 이해할수록 왜 이렇게 설계되었는지 자연스럽게 보이기 시작했습니다.
3. 결국 스프링을 잘 사용한다는 것은 애노테이션을 많이 아는 것이 아니라, 객체 생성은 컨테이너에 맡기고, 
횡단 관심사는 분리하며, 확장은 조합으로 해결하는 구조적 사고를 갖는 것이라는 점을 체감하게 되었습니다.