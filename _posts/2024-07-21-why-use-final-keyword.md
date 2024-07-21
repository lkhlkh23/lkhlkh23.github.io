---
layout: post
title: 자바에서 final 사용해야 하는 이유
subtitle: final 사용해야 하는 이유
excerpt_image: https://raw.githubusercontent.com/lkhlkh23/lkhlkh23.github.io/master/images/2024-07-21/banner.png
categories: java
tags: [java, final,]
---

팀원의 코드 리뷰를 하는 과정에서 `final 키워드를 습관적으로 작성하면 좋을 것 같다!` 라고 피드백을 주었다. 그리고 그 이유에 대해서는 명확하게 설명하지 못했다. 왜냐하면 내가 가장 좋아했던 개발자분의 코드에는 항상 `final` 있었기 때문에 나는 그 방식이 좋은 방식이라고 맹신했기 때문이다.

![0.png](https://raw.githubusercontent.com/lkhlkh23/lkhlkh23.github.io/master/images/2024-07-21/0.png)

그래서 나름대로 그 이유를 생각해봤다!

### Why should we use final .?!

`final` 키워드를 통해 `안전성`, `명확한 설계의도` 그리고 `성능 최적화` 라는 장점을 얻을 수 있다.

아래 코드를 보자!

```java
public class DriveAnalyzer {

	public void analyze(int distacne, int errorDistance) {
		// 운전 시간 계산은 거리를 통해 계산
		double averageSpeed = 60.0; 
		double drivingTime = distacne / averageSpeed;

		// 연료비 계산은 (거리 - 오차 거리) 를 통해 계산
		double fuelEfficiency = 15.0; 
		double fuelPrice = 1.5; 
		distacne = distacne - errorDistance;
		double fuelUsed = (distacne) / fuelEfficiency;
		double fuelCost = fuelUsed * fuelPrice;

		// 운전 점수는 (거리 - 오차 거리) 통해 계산
		double maxDistance = 100.0;
		double maxScore = 90.0;
		double drivingScore = (distacne <= maxDistance) ? maxScore : maxScore * (maxDistance / distacne);

		// 결과 출력
		System.out.println("Fuel Cost: " + fuelCost);
		System.out.println("Driving Score: " + drivingScore);
		System.out.println("Driving Time: " + drivingTime);
	}
}

```

위의 코드를 간단하게 설명하자면, 거리와 오차거리라는 파라미터를 받아서, 운전시간과 연료비 계산 그리고 운전 점수를 계산하는 3개의 역할을 수행하는 코드이다. `final` 키워드와 관련된 코드의 문제점을 나열해보겠다.

**왜곡된 설계의도**

`averageSpeed` 는 모든 차량의 평균속도로서 상수와 같은 역할을 수행한다. 하지만, `final` 이라는 키워드가 없기 때문에 코드를 유지보수하는 사람은 언제든지 변할 수 있는 변수로 이해할 수 있다. 처음 설계의 의도가 다음 개발자에게 전달되지 않는 문제가 발생한다.

**코드의 불안전성**

위의 코드는 잠재적인 버그를 가진 코드이다. 메소드에서 3개의 역할을 수행하고 있는데, distacne 입장에서 각각의 역할이 순서에 의존적이다. 운전 시간은 `disatnce` 를 통해 계산하지만, 연료비 계산과 운전점수는 `distacne - errorDistance` 를 이용해서 계산한다. 만약 운전시간 계산이 추가 개발을 통해 요청된 사항이고, 메소드 가장 끝에 있었다면 분명 문제가 발생할 것이다. 그리고 추가 개발을 담당한 개발자는 `disatnce` 가 불변이 아닐 수 있기 때문에 코드를 모두 분석해서 `disatnce` 에 대해서 완전하게 이해한 후 작성을 해야할 것이다. 결국은 기능의 순서에 의존적이기에 코드의 안전성이 떨어진다.

**리팩토링의 불편함**

`analyze` 메소드는 3개의 역할을 수행하고 있기 때문에 많은 역할을 수행하고 있다. 그렇기 때문에 테스트하기도 어려운 구조이다. 이런 코드를 보게된다면 대부분의 개발자라면 `option + cmd + m` 단축키를 수행할 것이다.  `extract method` 를 수행해서 주석을 기준으로 세개의 메소드를 생성할 것이다. 하지만, 문제가 발생한다. `distance` 는 불변하지 않기 때문에 운전점수 계산을 위해 생성한 메소드에서는 `distacne - errorDistance` 를 확인하고 반드시 수동으로 넣어야 한다. 항상 앞에 있는 로직을 완전하게 확인하고 작업을 해야한다.

물론 위의 코드는 문제점을 들어내기 위해 작성된 코드이기 때문에 억지스러울 수 있지만, 실제로 위와 비슷한 구조의 더 장문의 코드를 본적이 있다. 그리고 리팩토링을 할 때 많은 어려움을 가졌다.

그렇다면 이제 `성능 최적화` 라는 장점을 간단하게 설명하려고 한다. `final` 키워드는 컴파일러가 최적화할 수 있는 힌트를 제공한다. 예를 들어, `final` 메서드는 오버라이딩될 수 없으므로, 컴파일러는 이를 인라인으로 최적화할 수 있다. 여기서 `인라인으로 최적화` 라는 단어가 조금 이해가 되지 않을 수 있다. 아래와 같은 코드가 있다.

```java
public final class Math {
    public final int square(int x) {
        return x * x;
    }
}

public class Test {
    public static void main(String[] args) {
        Math math = new Math();
        int result = math.square(5); 
        System.out.println(result);
    }
}

```

인라인으로 최적화를 수행하면 다음과 같이 된다.

```java
public class Test {
    public static void main(String[] args) {
        int result = 5 * 5; // 메서드 호출 대신 실제 코드 삽입
        System.out.println(result);
    }
}

```

메소드 호출의 오버헤드를 감소시켜서 성능 최적화에 도움이 된다. 하지만 이 부분은 크게 공감되지는 않는다. `final` 키워드가 있다고해서 모두 인라인 최적화가 진행되지 않고, 조건이 존재하기 때문이다.

`final` 을 써야하는 다른 개발자들의 의견을 봤을 때, 아래와 같은 의문을 제기하시는분도 계셨다.

Question : IDE 에서 자동화로 만드는 코드에 기본적으로 final이 선언되지 않을까?

순간 이 의문에 대해서 답변이 생각나지 않았다. 정말 똑똑한 사람들이 고민하고 개발한 결정체에서 왜 자동으로 제공하지 않지?! 라는 의문에 나도 빠졌다. 그리고 고민한 결과 내가 내린 답변은 아래와 같다.

자동완성에서 `final` 이 필수가 아닌 이유는, 개발자가 상황에 따라 유연하게 코드를 작성할 수 있도록 개발자에게 선택권을 부여한 것이라고 생각한다. `final` 이 필수적이라고 한다면, 이를 제거하는 과정이 필요할 수 있으며 개발자에게는 불편함과 번거로움을 줄 수 있다. IDE 는 개발 생산성 향상에 도움을 주기 위한 목적으로 개발된 도구이기 때문에 이러한 이유에 적합하지 않다. 그렇기 때문에 개발자의 선택권을 위해 제공하지 않는다고 생각한다.

다음에는 불변객체를 사용해야 하는 이유에 대해 정리를 해봐야 겠다! 이전에 이펙티브 자바에서 읽었던 기억이 있는데, 내용은 기억나지 않는다.
