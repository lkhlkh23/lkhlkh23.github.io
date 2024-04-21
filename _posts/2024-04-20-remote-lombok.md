---
layout: post
title: lombok 제거 By Record + 객체지향
subtitle: lombok 제거하는 과정
excerpt_image: https://raw.githubusercontent.com/lkhlkh23/lkhlkh23.github.io/master/images/2024-04-20/banner.png
categories: spring
tags: [java, record, lombok]
---
java 17 기반으로 작성된 프로젝트를 java 21 로 버전으로 변경하면서 lombok 도 함께 제거하기로 결정을 했다.
대부분 클래스에 lombok 을 사용하고 있기 때문에 공수가 많이 발생할지라도 안전하게 제거하는 것을 목표로 진행하려고 한다.

그래서 안전하게 lombok 을 제거하는 절차에 대해 간단하게 작성하려고 한다. 아래에 있는 방식은 대중적으로 많이 언급되는 방식들을 이곳 저곳에서 참조해서 만들었다.

### Record 의 특징

자바 14부터 도입된 Record 는 불변 데이터를 간단하게 정의하는데 사용되는 특별한 종류의 클래스이다. Record 는 주로 데이터 전달에 사용된다. 데이터만 포함하는 단순한 객체를 표현할 때 유용하다.

**불변성**

Record 는 **불변 객체**로서 생성 후에는 필드를 변경할 수 없다. 불변객체는 **Thread Safe** 하다. 불변 객체는 수정할 수 없기 때문에 여러 스레드에서 동시에 접근해도 안전하다. 따라서 동기화 작업을 할 필요가 없기 때문에 복잡성을 줄일 수 있다. 또한, 불변 객체는 상태가 변하지 않기 때문에 **코드의 예측성과 신뢰성**을 보장한다.

**간결성**

Record 를 정의할 때, **allArgsConstructor, getter equals, hashCode, toString** 메소드를 자동으로 생성해주기 때문에 코드가 간결해진다.

### 프로젝트에서 lombok 제거에 영향을 주는 경우

1. DTO
2. Mapstruct
3. Domain
4. Entity

### Case.1 DTO 에서 lombok 제거 절차

**Step.0 수정 대상**

```java
@Getter
@Setter
@NoArgsConstructor
@AllArgsConstructor
@ToString
@EqualsAndHashCode
public class Person {

	private String name;
	private int age;
	private String location;

}
```

**Step.1 테스트 코드 작성**

```java
class PersonTest {

	@Test
	void test_person() {
		// given
		final Person person = new Person("LEE", 33, "ilsan");

		// when
		person.setLocation("seoul");

		// then
		assertEquals("LEE", person.getName());
		assertNotNull(person.toString());
		assertNotNull(person.hashCode());
		assertTrue(person.equals(person));
	}

}
```

**Step.2 lombok 어노테이션을 제거**

마우스 우클릭 → Delombok → All lombok annotations 를 선택하면 lombok 어노테이션이 제거되고, 아래와 같이 변환된다.
![2.png](https://raw.githubusercontent.com/lkhlkh23/lkhlkh23.github.io/master/images/2024-04-20/2.png)

```java
public class Person {

	private String name;
	private int age;
	private String location;

	public Person(String name, int age, String location) {
		this.name = name;
		this.age = age;
		this.location = location;
	}

	public Person() {
	}

	public String getName() {
		return this.name;
	}

	public int getAge() {
		return this.age;
	}

	public String getLocation() {
		return this.location;
	}

	public void setName(String name) {
		this.name = name;
	}

	public void setAge(int age) {
		this.age = age;
	}

	public void setLocation(String location) {
		this.location = location;
	}

	public boolean equals(final Object o) {
		if (o == this)
			return true;
		if (!(o instanceof Person))
			return false;
		final Person other = (Person)o;
		if (!other.canEqual((Object)this))
			return false;
		final Object this$name = this.getName();
		final Object other$name = other.getName();
		if (this$name == null ? other$name != null : !this$name.equals(other$name))
			return false;
		if (this.getAge() != other.getAge())
			return false;
		final Object this$location = this.getLocation();
		final Object other$location = other.getLocation();
		if (this$location == null ? other$location != null : !this$location.equals(other$location))
			return false;
		return true;
	}

	protected boolean canEqual(final Object other) {
		return other instanceof Person;
	}

	public int hashCode() {
		final int PRIME = 59;
		int result = 1;
		final Object $name = this.getName();
		result = result * PRIME + ($name == null ? 43 : $name.hashCode());
		result = result * PRIME + this.getAge();
		final Object $location = this.getLocation();
		result = result * PRIME + ($location == null ? 43 : $location.hashCode());
		return result;
	}

	public String toString() {
		return "Person(name=" + this.getName() + ", age=" + this.getAge() + ", location=" + this.getLocation() + ")";
	}
}
```

**Step.3 getter 이름 변경**

Record 에서는 lombok 과는 다르게 필드명으로 getter 를 자동으로 생성한다. (getAge() → age())

```java
public String name() {
	return this.name;
}

public int age() {
	return this.age;
}

public String location() {
	return this.location;
}
```

**Step.4 불필요 setter 제거 및 setter 수정**

먼저 사용하지 않는 setter 를 제거한다. Record 는 불변객체이기 때문에 setter 를 이용해서 객체의 필드를 변경해서는 안된다. 그렇기 때문에 신규로 객체를 생성하는 방법이 옳다.

```java
public Person withLocation(final Person person, final String location) {
	return new Person(person.name, person.age, location);
}
```

**Step.5 객체의 필드를 final 로 변경**

**Step6. Record 에서 기본적으로 제공해주는 메소드를 제거하고, Record 로 변경**

```java
public record Person(String name, int age, String location) {
	public Person withLocation(final Person person, final String location) {
		return new Person(person.name, person.age, location);
	}
}
```

테스트 코드를 Step 이 동작할 때마다 실행을 시키면서 안전하게 Record 로 변경해보자.

### Case.2 Mapstruct

Record 는 불변객체이기 때문에 setter 를 지원하지 않는다. 그렇기 때문에 Mapstruct 가 정상적으로 동작하지 않을 것이라고 생각했다. 하지만 Mapstuct 가 아래와 같이 변경되었다. setter 가 아닌 생성자를 이용했다.

```java
@Generated(
    value = "org.mapstruct.ap.MappingProcessor",
    date = "2024-04-20T14:49:50+0900",
    comments = "version: 1.4.1.Final, compiler: IncrementalProcessingEnvironment from gradle-language-java-8.7.jar, environment: Java 17.0.8 (Oracle Corporation)"
)
public class PersonMapperImpl implements PersonMapper {

    @Override
    public Person toDomain(PersonV1 domain) {
        if ( domain == null ) {
            return null;
        }

        String location = null;
        String name = null;
        int age = 0;

        location = domain.address();
        name = domain.name();
        age = domain.age();

        Person person = new Person( name, age, location );

        return person;
    }
}
```

### Case.3 Domain

Calculator 코드를 보면 Price 와 Receipt 는 필드를 가져와서 다시 Receipt 필드에 값을 수정하고 있다. 조금은 억지스러운 코드일 수 있다. 하지만 이렇게 작성하는 사람 주변에 한명쯤 있을걸?!

```java
@AllArgsConstructor
@Getter
public class Price {

	private String subject;
	private int fee;

}
```

```java
@Setter
@Getter
@AllArgsConstructor
public class Receipt {

	private int total;

}
```

```java
public class Calculator {

	public Receipt calculate(final Price price1, final Price price2) {
		final Receipt receipt = new Receipt(1000);
		receipt.setTotal(receipt.getTotal() + price1.getFee() + price2.getFee());

		return receipt;
	}

}
```

Calculator 에서 필드의 값들을 가져와서 변경하는 방식이 아닌, Receipt 에 메세지를 전달해서 처리하는 방식으로 수정해서 lombok 을 제거했다. 책임을 수행하는 객체들이 메세지를 통한 자유로운 협력을 통해 애플리케이션을 구축하는 방법으로 개발하려고 한다. 이러한 방법을 객체지향이라고 생각한다.

```java
public class Price {

	private String subject;
	private int fee;

	public static Price of(final String subject, final int fee) {
		final Price price = new Price();
		price.subject = subject;
		price.fee = fee;

		return price;
	}

	public int getFee() {
		return this.fee;
	}
}
```

```java
public class Receipt {

	private int total;

	public Receipt(final int total) {
		this.total = total;
	}

	public void calculate(final List<Price> prices) {
		for (final Price price : prices) {
			this.total += price.getFee();
		}
	}

}
```

```java
public class Calculator {

	public Receipt calculate(final Price price1, final Price price2) {
		final Receipt receipt = new Receipt(1000);
		receipt.calculate(List.of(price1, price2));

		return receipt;
	}

}
```

위와 같이 작성을 하면 불필요한 getter/setter 메소드도 제거할 수 있고, 테스트 코드를 작성하기도 쉽다는 장점 또한 가지고 있다.

### Case4. Entity

Entity 에서는 Record 가 적합하지는 않다. Record 는 불변하므로 JPA 의 특성과 잘 맞지 않다. JPA 는 Entity 를 관리하고 상태를 변경하는 것을 지원하는데, Record 는 불변이므로 변경이 어렵다. 또한, JPA Entity 는 기본 생성자를 가져야 하며, Record 의 생성자와는 다르기 때문에 문제가 발생한다.

```java
@Entity
@Table(name = "student")
public record Student(@Id @Column(name = "id") String id,
					  @Column(name = "name") String name,
					  @Column(name = "grade") String grade) {
}
```

![0.png](https://raw.githubusercontent.com/lkhlkh23/lkhlkh23.github.io/master/images/2024-04-20/0.png)

요약을 하자면 Record 는 DTO 에는 적합하지만, JPA Entity 에는 적합하지 않는다.

lombok 을 제거하는 방법에 대해 간단하게 정리를 했다. 대부분의 개발자라면 기본적으로 인지하고 있는 내용이기 때문에 가볍게 읽었을 것이다. 요즘 공부를 소홀히 했는데, 다시 자극을 받게되었다. 하늘은 참 부지런도 하는 것이… 정말 내가 마음 편하게 쉬는 것을 보지 못하는것 같다.
![0.png](https://raw.githubusercontent.com/lkhlkh23/lkhlkh23.github.io/master/images/2024-04-20/1.png)

