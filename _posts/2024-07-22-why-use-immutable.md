---
layout: post
title: 불변객체를 사용해야 하는 이유
subtitle: 변경 가능성을 최소화 해라
excerpt_image: https://raw.githubusercontent.com/lkhlkh23/lkhlkh23.github.io/master/images/2024-07-22/banner.png
categories: java
tags: [java, immutable]
---
final 을 써야하는 이유에 대해 포스팅을 하다가, 불변객체를 써야하는 이유까지 다시 복습하게 되었다. 
다시 이펙티브 자바를 읽었다. 이 책은 정말 매력적이다. 읽을때마다 항상 새롭다. 
늘 뇌가 리셋되었다고 할까?! 나의 뇌를 순수하게 만든다고 할까?!


### What is Immutable Class

객체 내부에 정의된 필드의 값을 수정할 수 없는 클래스이다. 불변 객체에 정의된 필드는 GC 에 의해 소멸되는 순간까지 절대 변하지 않는다.

### Rules of Immutable Class

Rule.1 setter 와 같은 객체의 상태를 변경하는 메소드 제공 금지

Rule.2 클래스를 상속할 수 없도록 final class 로 선언

Rule.3 클래스 내부의 모든 필드 private final 로 선언

Rule.4 클래스 내부 필드에 가변 객체가 있다면, 가변 객체에 접근하지 못하도록 설정 필요

처음에 Rule.3 과 Rule.4 의 규칙이 충돌되는 것이 아닌가?! 라는 의문을 가졌다. 모든 필드는 private final 로 선언해서 수정할 수 없는 형태로 만드는데, 가변객체가 존재할 수 있다고?! 이상했다. 나와 같이 혼란이 왔다면, 아래 코드를 보자



```java
public final class Period {

	private final Date startAt;
	private final Date endAt;

	public Period(Date startAt, Date endAt) {
		this.startAt = startAt;
		this.endAt = endAt;
	}

	public Date getStartAt() {
		return startAt;
	}

	public Date getEndAt() {
		return endAt;
	}
}
```

여기서 퀴즈! Period 클래스는 불변 클래스의 규칙을 모두 준수하고 있는가?!

![0.png](https://raw.githubusercontent.com/lkhlkh23/lkhlkh23.github.io/master/images/2024-07-22/0.png)

정답은 No! Rule.4 를 준수하고 있지 않다. 만약 Date 가 아닌 LocalDateTime 이었다면 불변 클래스의 규칙을 모두 준수했을 것이다. 이유는 Date 는 setYear() 과 같은 메소드를 통해 내부 필드의 값을 변경할 수 있지만, **LocalDateTime 은 내부 필드의 값을 변경할 수 없는 불변클래스** 이다. Date 가 불변클래스가 아니기 때문에 테스트 코드와 같이 값을 변경해서 문제를 발생시킬 수 있다.

```java
public class ImmutableTest {

  @Test
  void test_copy_protect_1() {
    final Date startAt = new Date();
    final Date endAt = new Date();
    final Period period = new Period(startAt, endAt);

    System.out.println("startAt : " + period.getStartAt());

    period.getStartAt().setMonth(0);
    System.out.println("startAt : " + period.getStartAt());
  }

  @Test
  void test_copy_protect_2() {
    final Date startAt = new Date();
    final Date endAt = new Date();
    final Period period = new Period(startAt, endAt);
    startAt.setMonth(3);

    System.out.println("startAt : " + period.getStartAt());
  }

}
```

위와같은 문제를 해결하기 위해서는 방어적 복사본을 적용해야 한다.

```java
public final class Period {

	private final Date startAt;
	private final Date endAt;

	public Period(Date startAt, Date endAt) {
		this.startAt = new Date(startAt.getTime());
		this.endAt = new Date(endAt.getTime());
	}

	public Date getStartAt() {
		return new Date(startAt.getTime());
	}

	public Date getEndAt() {
		return new Date(endAt.getTime());
	}
}
```

생성자와 getter 에서 신규 Date 객체로 재생성해서 처리하고 있기 때문에 Period 클래스 내부의 필드를 변경할 수 없게 되었다. Date 가 불변클래스가 아니기 때문에 위와 같은 **방어적 복사본**이 적용된 로직이 필요하다.

### Advantages of Immutable Instance

**단순함을 통한 안전성**

불변객체는 생성된 시점에서 파괴될 때까지 상태를 그대로 간직한다. 그렇기 때문에 멀티 스레드 환경에서 동시에 사용해도 상태가 변하지 않는다. 동기화 작업없이 **멀티 스레드 환경에서 안전하게 공유**할 수 있다.

**Cache, Map, Set 등의 자료구조에 최적화**

Map 에서 Key 를 인식하기 위해 equals() 와 hash() 를 이용한다. 그런데 객체의 상태가 변하면 equals() 와 hash() 의 결과가 달라지기 때문에 문제가 발생한다.

- Key 를 이용한 탐색 불가
- 중복 저장

그렇기 때문에 불변 객체가 Key 에 적합하다.

**실패 원자적인 메소드 (Failure Atomicity) 생성 가능**

메소드 실행 도중에 예외가 발생했을 경우, 객체 상태가 변경된 채로 남아 일광성을 잃는 경우가 있다.

> 실패 원자성 **(Failure Atomicity)**
작업이 실패했을 때, 시스템이 일관된 상태로 돌아가야함을 보장하는 성격
>

실패 원자적이지 못한 코드의 예제는 아래와 같다.

```java
public class Account {

	private int balance;

	public Account(final int balance) {
		this.balance = balance;
	}

	public void withdraw(final int amount) {
		if (amount <= 0) {
			throw new IllegalArgumentException("Amount must be greater than zero");
		}
		
		if (amount > balance) {
			throw new RuntimeException("Insufficient balance");
		}
		this.balance -= amount;
	}

}
```
```java
@Service
public class WithdrawService {

	private final UserAccountRepository userAccountRepository;
	private final UserAccountHistoryRepository userAccountHistoryRepository;

	@Transactional
	public void withdraw(final Account account, final int amount) {
		account.withdraw(amount);
		userAccountRepository.withdraw(amount);
		userAccountHistoryRepository.addHistory(account, amount);
	}

}
```

UserAccountHistoryRepository 작업하는 과정에서 예외가 발생했을 때, UserAccountRepository 는 트랜잭션을 통해 원복될 수 있지만, Account 객체의 상태는 원복되지 않는다. 이 코드는 실패 원자적이지 못한다.

실패원자적인 코드로 변경한다면 아래와 같다.

```java
public class Account {

    private final int balance;

    public Account(final int balance) {
        this.balance = balance;
    }

    public int getBalance() {
        return balance;
    }

    public Account withdraw(final int amount) {
        if (amount <= 0) {
            throw new IllegalArgumentException("Amount must be greater than zero");
        }

        if (amount > balance) {
            throw new RuntimeException("Insufficient balance");
        }

        return new Account(this.balance - amount);
    }
}
```

### Disadvantages of Immutable Instance

**성능 오버헤드**

불변객체는 상태가 변경될때마다 새로운 객체를 생성해야하므로 전체 메모리 사용량의 증가와 오버헤드를 발생시킬 수 있다. 또한, 가비지 컬렉션의 부담이 커질 수 있다는 문제가 발생할 수 있다.

그렇기 때문에 자주 사용되는 불변객체에 대한 풀을 생성하는 등의 방식을 고안해서 객체 재사용을 권장해야 한다.

![1.png](https://raw.githubusercontent.com/lkhlkh23/lkhlkh23.github.io/master/images/2024-07-22/1.png)

실제 개발을 하다보면 항상 불변 클래스로 개발하기가 쉽지 않다. **불변할 수 없다면, 변경 가능성을 최소화해서 최대한 불변 클래스에 가깝게 만들자!**
