---
layout: post
title: Code Coverage - Jacoco + PITest
subtitle: Jacoco + PITest (Mutation Test)
excerpt_image: https://raw.githubusercontent.com/lkhlkh23/lkhlkh23.github.io/master/images/2024-12-04/banner.png
categories: java
tags: [java, quality, jacoco, test, pitest, mutation]
---
2025년 코드의 품질을 향상시키는 작업을 계획하고 있다. 작업을 위한 학습이다. 실은 뭐라도 하지 않으면 안될것 같다. 생각이 복잡하다.

![0.png](https://raw.githubusercontent.com/lkhlkh23/lkhlkh23.github.io/master/images/2024-12-04/0.png)

### What is Difference Between Code coverage and Test coverage..?!

`Code coverage` 는 개발 코드에서 테스트가 적용된 코드의 비율을 측정하는 메커니즘이다. Code coverage 를 측정하기 위해서 일반적으로 [Jacoco](https://www.baeldung.com/jacoco) 또는 [Cobertura](https://www.baeldung.com/cobertura) 와 같은 도구를 활용한다.

Code coverage 는 정량적인 지표를 제공하기 때문에 테스트되지 않는 코드 영역, 미사용 코드 영역을 쉽게 파악할 수 있다.

하지만, Code coverage 는 코드 영역에 대한 테스트 유무를 파악할 수 있지만, 해당 테스트에 대한 유효성과 정확성까지는 보장하지 않는다. 그리고 Code coverage 를 강제로 늘리기 위해 불필요하고 쓸모없는 테스트가 대량 생산될 수 있다.

`Test coverage` 는 테스트가 애플리케이션의 기능을 얼마나 포괄하는지를 측정하는데 사용하는 매커니즘이다. Test coverage 는 엔드 유저 관점에서 QA 에 의해 계산된다. Test coverage 를 측정하기 위해서는 일반적으로 [Selenium](https://www.baeldung.com/selenium-webdriver-page-object), [Playwright](https://playwright.dev/), [Cypress](https://www.cypress.io/) 와 같은 도구를 활용한다.

수동으로 진행하는 테스트 같은 경우, 전문적인 지식이 필요하지 않고, 구현하기가 쉽다는 장점을 가지고 있다. 그리고, Test coverage 는 Code coverage 와 달리 품질적이므로 정량화하기가 어렵다.

### What is Jacoco..?!

Jacoco 는 Java Code coverage 를 측정하기 위해 사용되는 오픈소스 도구이다. 코드 품질 및 완전성을 평가하는데 도움을 준다. Jacoco 는 Java 에이전트로 동작하여, JVM 바이트코드를 조작해 코드 실행 경로를 추적한다. 그리고, Jenkins, Gradle 등 다양한 빌드도구 및 CI/CD 와 쉽게 통합이 가능하다. 그리고 html, xml, csv 와 같은 다양한 형식의 리포트를 제공한다.

### What is Jacoco Metrics and Limitations..?!

Jacoco 는 csv, xml 과 같은 다양한 형식의 보고서를 생성할 수 있는데, 해당 보고서에서는 아래와 같은 지표들을 보여주고 있다. 여기서 `_COVERED` 는 실행된 코드를 의미한다. 실행되었다는 것은 테스트 코드에 의해 호출이 되었거나, 프로덕션 코드에서 호출이 되었다는 것을 의미한다. 반대로 `_MISSED` 는 전혀 실행되지 않는 코드를 의미한다.

- GROUP : 코드 분석 대상의 최상위 그룹
- PACKAGE : Java 패키지 수준에서 커버리지
- CLASS : Java 클래스 수준에서 커버리지
- INSTRUCTION_MISSED : 실행되지 않은 바이트코드 명령어 수
- INSTRUCTION_COVERED : 실행된 바이트코드 명령어 수
- BRANCH_MISSED : 실행되지 않은 분기 수 (조건문에서 발생할 수 있는 모든 경로 중 실행되지 않은 경로)
- BRANCH_COVERED : 실행된 분기 수
- LINE_MISSED : 실행되지 않은 소스 코드 라인의 수
- LINE_COVERED : 실행된 소스 코드 라인의 수
- COMPLEXITY_MISSED : 실행되지 않은 코드의 복잡도
- COMPLEXITY_COVERED : 실행된 코드의 복잡도
- METHOD_MISSED : 실행되지 않은 메서드의 수
- METHOD_COVERED : 실행된. 메서드의. 수

Jacoco 에서 제공하는 _COVERED 지표에서는 테스트 코드가 존재할지라도 해당 테스트 코드가 유효하고 신뢰할만한 테스트라고 보장하기가 어렵다. Jacoco 가 개발 코드에 대한 정량적인 품질 지표라고 한다면, 이제 소개할 테스트 방법과 도구는 테스트 코드에 대한 품질 지표라고 보면 된다.

### What is Mutation and PITest..?!

`Mutating` 은 코드를 약간 변형하여 원래 코드와 다른 결과를 만들어내는 작업을 말한다. 변경된 코드는 "Mutant(돌연변이)"라고 부른다. (예 : if (a > b) → if (a ≥ b) 로 변형)

`Mutation Test` 는 Mutant 코드가 실패되는지를 평가하는 테스트 기법이다. Mutation Test 는 테스트의 강도를 평가하기 위해 사용된다.  Mutation Test 는 테스트의 강도를 평가해서, 약한 테스트를 식별 및 보완할 수 있도록 도움을 준다. 그리고 PITest 와 Stryker 도구를 이용해서 자동화활 수 있다.

그러나, Mutant 를 생성하고 테스트를 수행하는데 많은 시간이 소요된다는 단점이 있다. 그리고 특정 코드에 대해서는 Mutant 를 생성하고 테스트하기 어려울 수 있다. 예를들어, 외부 API 호출, 데이터베이스에 의존적인 코드와 같이 3rd Party 와 연관된 코드에 대해서는 테스트의 한계가 있다.

이러한 단점을 보완하기 위해서는 전체 코드에 대해서 Mutation Test 를 하는것이 아닌, 특정 코드에 대해서만 선별적으로 진행하는 것을 권장한다. 외부 API 호출이나 비동기 작업에 대해 Mock을 사용하여 한계점을 극복하는 것이 좋다.

PITest 를 위해서 build.gradle 의 설정을 아래와 같이 작성한다.

```groovy
plugins {
	id 'java'
	id 'org.springframework.boot' version '3.4.0'
	id 'io.spring.dependency-management' version '1.1.6'
	id 'info.solidsoft.pitest' version '1.15.0'
}

group = 'test'
version = '0.0.1-SNAPSHOT'

pitest {
	junit5PluginVersion = '1.2.1'
	pitestVersion = '1.15.2'
}

java {
	toolchain {
		languageVersion = JavaLanguageVersion.of(21)
	}
}

configurations {
	compileOnly {
		extendsFrom annotationProcessor
	}
}

repositories {
	mavenCentral()
}

dependencies {
	implementation 'org.springframework.boot:spring-boot-starter'
	compileOnly 'org.projectlombok:lombok'
	annotationProcessor 'org.projectlombok:lombok'
	testImplementation 'org.springframework.boot:spring-boot-starter-test'
	testRuntimeOnly 'org.junit.platform:junit-platform-launcher'

	implementation group: 'org.pitest', name: 'pitest-parent', version: '1.15.0', ext: 'pom'
}

tasks.named('test') {
	useJUnitPlatform()
}

```

그리고 개발 코드와 테스트 코드를 아래과 같이 작성했다.


```java
public class Calculator {

	public int calculate(final int num1, final int num2, final char operand) throws Exception {
		if (operand == '+') {
			return sum(num1, num2);
		}

		if (operand == '-') {
			return subtract(num1, num2);
		}

		if (operand == '*') {
			return multiply(num1, num2);
		}

		throw new Exception("존재하지 않는 연산자 입니다.");
	}

	public int sum(final int num1, final int num2) {
		return num1 + num2;
	}

	public int multiply(final int num1, final int num2) {
		return num1 * num2;
	}

	public int subtract(final int num1, final int num2) {
		return num1 - num2;
	}

}
```

```java
class CalculatorTest {

	@Test
	void test_calculate_1() throws Exception {
		// given
		final Calculator calculator = new Calculator();

		// when
		final int result = calculator.calculate(1, 2, '-');

		// then
		assertEquals(-1, result);
	}

	@Test
	void test_calculate_2() throws Exception {
		// given
		final Calculator calculator = new Calculator();

		// when
		final int result = calculator.calculate(1, 2, '*');

		// then
		assertEquals(2, result);
	}

}
```

그리고나서, PITest 를 실행하고 결과를 확인하자! 결과 레포트는 `build/reports/pitest/index.html` 에서 확인할 수 있다.

```bash
./gradlew pitest
```

레포트를 보면, 테스트 클래스의 Line coverage, Mutation coverage, Test strength 를 확인할 수 있다. `Mutation coverage` 가 높다는 것은 테스트 코드의 변형에 잘 반응하고 있다는 것을 의미한다. 결국은, 테스트 코드를 명확하게 잘 작성했다는 것으로 해석하면 편하다. 

`Test strength` 는 테스트 코드 실행되었는지 유무를 넘어서, 다양한 경로 및 예외적인 조건을 얼마나 충족했는지를 의미한다. 

![2.png](https://raw.githubusercontent.com/lkhlkh23/lkhlkh23.github.io/master/images/2024-12-04/2.png)

![3.png](https://raw.githubusercontent.com/lkhlkh23/lkhlkh23.github.io/master/images/2024-12-04/3.png)

![4.png](https://raw.githubusercontent.com/lkhlkh23/lkhlkh23.github.io/master/images/2024-12-04/4.png)

### And I am ..

최근에 ~~조직개판~~이 아닌, 조직개편이 있었다. 어쩌다보니 팀에서 파트가 방출되었다. 개발의 효율성에 맞게 조직이동을 할 수 있다고 생각할 수 있다. 처음에 나도 그렇게 생각했다. 하지만, 최근에 방출이라는 느낌을 받았다. 이런것을 환승이별의 기분이 이런것일까?!

![1.png](https://raw.githubusercontent.com/lkhlkh23/lkhlkh23.github.io/master/images/2024-12-04/1.png)
