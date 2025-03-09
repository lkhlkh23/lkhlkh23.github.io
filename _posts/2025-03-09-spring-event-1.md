---
layout: post
title: Spring Event 정리! 
subtitle: 느슨한 결합 Spring Event
excerpt_image: https://raw.githubusercontent.com/lkhlkh23/lkhlkh23.github.io/master/images/2025-03-09/banner.png
categories: spring
tags: [java, Spring]
---
날씨가 따뜻해지면서 따뜻한 봄이라는 이벤트가 오고있다. 그래서 오늘은 Spring Event 에 대해 정리했다.

![0.png](https://raw.githubusercontent.com/lkhlkh23/lkhlkh23.github.io/master/images/2025-03-09/0.png)

하긴 뭘해 … 치과에서 치아빼고, 통장에서 병원비 빼고 … 치아에서 출혈 빼고 …

### What is Spring Event ..?!

`Spring Event` 는 스프링 프레임워크에서 제공하는 **구독-발행 (publish-subscribe)** 모델을 사용하여 애플리케이션 내부에서 컴포넌트 간의 이벤트 중심 아키텍처를 구축할 수 있게 도와준다.

애플리케이션 내부 컴포넌트의 의존성을 분리하여 **느슨한 결합**을 구현할 수 있다. 그리고, 새로운 이벤트 리스너를 추가할 때, 기존 로직에 영향을 주지 않고, 이벤트 리스너를 추가하여 비지니스 로직의 **확장성**을 높일 수 있다. 또한, 각각의 컴포넌트가 독립적이기 때문에 **재사용성도** 높일 수 있으며, **단위 테스트가 편리**하다.

하지만, 메세지의 순서를 고려해야하기 때문에 **코드의 복잡성**과 **코드파악이** **어렵다**는 단점이 있다.

### How to implement Spring Event ..?!

**구현 코드**

```java
@Service
@RequiredArgsConstructor
@Slf4j
public class FindPasswordService {

	private final AccountRepository repository;
	private final ApplicationEventPublisher publisher;

	@Transactional
	public String getPassword(final String name) throws Exception {
		final AccountEntity account = repository.findByName(name)
												.orElseThrow(() -> new Exception("Not Exist Member"));

		log.info("event publish start");
		publisher.publishEvent(new SendNotificationEvent(account.getId(), account.getName(), account.getPassword(), account.getRole()));
		log.info("event publish end");

		return account.getPassword();
	}
}
```

```java
@Getter
@RequiredArgsConstructor
public class SendNotificationEvent {
  // ApplicationEvent 상속받거나, 단순한 POJO 클래스 사용 가능 
	private final Long id;
	private final String name;
	private final String password;
	private final String role;
}
```

```java
@Service
@Slf4j
@EnableAsync
@RequiredArgsConstructor
public class NotificationService {

	private final AccountHistoryRepository accountRepository;

	@EventListener(value = SendNotificationEvent.class)
	public void sendNotification(final SendNotificationEvent event) {
		log.info("sendNotification");

		final AccountHistoryEntity entity = from(event.getName());
		accountRepository.save(entity);
	}

	@TransactionalEventListener(value = SendNotificationEvent.class)
	public void sendNotificationWithTransaction(final SendNotificationEvent event) {
		final String transactionName = TransactionSynchronizationManager.getCurrentTransactionName();
		log.info("sendNotificationWithTransaction. transaction : {}", transactionName);

		final AccountHistoryEntity entity = from(event.getName());
		accountRepository.save(entity);
	}

	@EventListener(value = SendNotificationEvent.class)
	@Async
	public void sendAsyncNotification(final SendNotificationEvent event) throws InterruptedException {
		log.info("sendAsyncNotification");

		Thread.sleep(3000);
		log.info("sleep 3000 end");

		accountRepository.save(from(event.getName()));
	}

	@TransactionalEventListener(value = SendNotificationEvent.class)
	@Async
	public void sendAsyncNotificationWithTransaction(final SendNotificationEvent event) throws InterruptedException {
		log.info("sendAsyncNotificationWithTransaction");

		Thread.sleep(3000);
		log.info("sleep 3000 end");

		accountRepository.save(from(event.getName()));
	}

	private AccountHistoryEntity from(final String name) {
		return AccountHistoryEntity.builder()
							.name(name)
							.loginAt(LocalDateTime.now())
							.build();
	}
}
```

**실행결과-1 (EventListener)**

![1.png](https://raw.githubusercontent.com/lkhlkh23/lkhlkh23.github.io/master/images/2025-03-09/1.png)

이벤트 발행 클래스 (FindPasswordService) 와 이벤트를 구독해서 처리하는 클래스 (NotificationService) 가 동일한 스레드에서 처리되는 것을 로그를 통해 확인할 수 있다.

**실행결과-2 (TransactionalEventListener)**

![2.png](https://raw.githubusercontent.com/lkhlkh23/lkhlkh23.github.io/master/images/2025-03-09/2.png)

이벤트를 구독해서 처리하는 클래스 (NotificationService) 에서 insert 가 발생하지 않았다. TransactionalEventListener 의 실행단계의 디폴트값은 AFTER_COMMIT 이기 때문에 이벤트 발행 클래스 (FindPasswordService) 에서 트랜잭션이 커밋이 완료된 시점에 실행되었다. 그렇기 때문에 insert 쿼리가 데이터베이스에 반영되지 않았다. insert 쿼리를 실행시키기 위해서는 실행단계를 BEFORE_COMMIT 으로 설정하면 된다.

**실행결과-3 (EventListener + Async)**

![3.png](https://raw.githubusercontent.com/lkhlkh23/lkhlkh23.github.io/master/images/2025-03-09/3.png)

이벤트 발행 클래스 (FindPasswordService) 와 이벤트를 구독해서 처리하는 클래스 (NotificationService) 가 서로 다른 스레드에서 수행되었다. 스레드가 다르기 때문에 트랜잭션도 다르다. 스레드 로컬은 멀티스레드 환경에서 각 스레드마다 고유한 데이터를 저장할 수 있어 스레드 간 데이터 충돌을 방지한다. 그리고 트랜잭션은 스레드 로컬에 연결되었다. 그렇기 때문에 서로 다른 스레드는 서로 다른 트랜잭션을 가진다.

**실행결과-4 (TransactionalEventListener + Async)**

![4.png](https://raw.githubusercontent.com/lkhlkh23/lkhlkh23.github.io/master/images/2025-03-09/4.png)

TransactionalEventListener 가 커밋 이후에 실행되었지만, 비동기에 의해서 다른 스레드, 다른 스레드 로컬, 다른 트랜잭션으로 처리가 되기 때문에 실행결과-2 와 다르게 insert 쿼리가 실행된다.

### When should Spring Event be used ..?

Spring Event 를 적용할만한 가장 대표적인 사례는 **회원가입 후 사용자에게 카카오 알림을 전달**하는 기능이다. 회원가입이라는 **트랜잭션을 완료 후, 후속 작업인 알림톨 전달**을 할 수 있게 도와준다. 그리고 알림톡 전달이 독립적인 기능이기 때문에 회원가입 뿐만 아니라, 결제, 주문 등 다양한 기능에서도 **재사용**할 수 있다. 그리고 알림톡이 아닌, 이메일 전달 기능으로 **변경할 때 기존 코드의 영향을 최소화**하고, 다른 기능으로 대체할 수 있다.

### What is different between Spring Event and AOP ..?!

Spring Event 를 봤을 때, 제일 먼저 떠오른 것이 AOP 였다. Spring Event 와 AOP 모두 실행 흐름을 제어해서 특정 기능을 수행하도록 하는 것이 비슷했다. 그리고 독립적인 기능들을 흐름을 이용해서 분리한 방식도 비슷했다. 하지만 명확한 차이가 있다.

| 비교 항목 | AOP (Aspect-Oriented Programming) | Spring 이벤트 (Application Event) |
| --- | --- | --- |
| **목적** | 부가적인 기능(로깅, 트랜잭션 관리 등) 분리 | 비즈니스 로직 간의 독립성 유지 및 비동기 처리 |
| **동작 방식** | 메서드 실행을 가로채서 추가 작업 수행 | 이벤트를 발행(Publish)하고, 리스너가 처리(Consume) |
| **트리거(Trigger)** | 특정 메서드 실행 전/후/예외 발생 시 | 이벤트 객체를 `publishEvent()`로 발행할 때 |
| **결합도** | 특정 메서드나 클래스에 의존 → 상대적으로 강한 결합 가능성 있음 | 이벤트 발행자와 리스너가 서로 몰라도 됨 → 느슨한 결합 |
| **적용 방식** | `@Aspect`, `@Before`, `@After`, `@Around` 등 | `@EventListener`, `@TransactionalEventListener` |
| **비동기 지원** | 기본적으로 동기 실행 (비동기 설정 가능) | `@Async` 사용 시 비동기 지원 가능 |
| **트랜잭션 영향** | 주로 현재 트랜잭션 내에서 실행됨 | `@TransactionalEventListener` 사용 시 트랜잭션 완료 후 실행 가능 (`AFTER_COMMIT`) |
| **주요 사용 사례** | 로깅, 트랜잭션 관리, 보안 검사, 성능 모니터링 등 | 도메인 이벤트 처리 (회원가입 후 이메일 발송, 주문 후 결제 시스템 호출 등) |

### And I am

요즘 분노가 차오른다. 이러지도 저러지도 못하는 상황에서 화가난다. 이러지도 저러지도 못하는 상황을 만든 사람을 볼때마다 화가난다. 그리고 그 상황을 벗어나지 못하는것이 나의 무력함 때문이라는 것에 화가난다.

![5.png](https://raw.githubusercontent.com/lkhlkh23/lkhlkh23.github.io/master/images/2025-03-09/5.jpeg)
