---
layout: post
title: checkstyle 필요성 및 간단한 커스텀
subtitle: chekstyle 따르겠사옵니다!
excerpt_image: https://raw.githubusercontent.com/lkhlkh23/lkhlkh23.github.io/master/images/2023-12-17/banner.png
categories: java
tags: [checkstyle] [intellij]
top: 2
---

# Checkstyle

개발자들은 코드의 가독성을 높이기 위해 많은 시간을 투자하여 노력을 한다. 명확한 의미전달을 할 수 있도록 네이밍을 고려하기도 하고, 역할과 책임에 맞게 코드를 설계하기도 한다.

그럼에도 불구하고 가장 간단하게 코드의 가독성을 향상시킬 수 있는 방법을 하나 정리하려고 한다.

### checkstyle 정의

사전 정의된 규칙 및 표준을 기반으로 Java 코드의 품질을 분석하고 확인하는 데 사용되 **정적 코드 분석** 도구이다.

코드의 가독성과 안정성을 확인하는데 도움이 된다.

> 정적 코드 분석 : 프로그램을 실행하지 않고, 수행되는 분석
>

### checkstyle 장점 그리고 사소한 단점

팀 전체에 일관된 코딩 스타일을 적용하여 **일관성**을 보장한다.

checkstyle 을 적용하지 않으면 동일한 프로젝트를 다수의 개발자가 코딩을 할 때, 서로 다른 스타일로 인해서 코드의 일관성이 없어보이며 굉장히 혼란스럽다.

```jsx
// 선호하지 않는 스타일
private void handleNonPreferred() {
		IntStream.range(0, 1000)
			.parallel()
			.forEach(no -> inner());
}

// 선호하는 스타일
private void handlePreferred() {
		IntStream.range(0, 1000)
			       .parallel()
			       .forEach(no -> inner());
}
```

**자동 코드 검토**가 가능하다.

빌드를 통해 자동으로 코드 스타일을 검토하기 때문에 개발자가 수동으로 작업을 하지 않아 편리하다.
코드를 분석하고 규칙을 검사하는 과정에서 추가적인 리소스가 발생하기 때문에 성능이 저하될 수 있다는 우려를 할 수 있다. 하지만, 나의 리소스가 발생하는 것보다 IDE 의 리소스가 발생하는 것이 더 합리적이지 않을까?!

개발부서 내부의 **업무 협업 능력이 증가**한다.

checkstyle 통해서 개발 부서의 코딩 표준을 정의할 수 있고, 개발자들의 성향과 프로젝트의 특성에 맞게 커스텀할 수 있다.

하지만 작은 단점도 존재한다. 정적 분석을 기반으로 하기 때문에 모든 상황을 고려하지 못할 수 있다. 때로는 실제로는 유효한 코드에 대해 잘못된 경고를 제공할 수 있다.

### checkstyle 적용

개인적으로 [naver checkstyle](https://github.com/naver/hackday-conventions-java/blob/master/rule-config/naver-intellij-formatter.xml) 적용 후, 간단한 커스텀을 진행하여 사용하고 있다.

아래 코드는 checkstyle 이 적용되지 않는 상태이다. 내가 개인적으로 정말 선호하지 않는 스타일의 코드이다.
![banner](https://raw.githubusercontent.com/lkhlkh23/lkhlkh23.github.io/master/images/2023-12-17/0.png)

plugin 설치 및 save action 설정

![banner](https://raw.githubusercontent.com/lkhlkh23/lkhlkh23.github.io/master/images/2023-12-17/1.png)
![banner](https://raw.githubusercontent.com/lkhlkh23/lkhlkh23.github.io/master/images/2023-12-17/2.png)

checkstyle 적용
![banner](https://raw.githubusercontent.com/lkhlkh23/lkhlkh23.github.io/master/images/2023-12-17/3.png)

method argument 수정 및 적용 결과
![banner](https://raw.githubusercontent.com/lkhlkh23/lkhlkh23.github.io/master/images/2023-12-17/4.png)
![banner](https://raw.githubusercontent.com/lkhlkh23/lkhlkh23.github.io/master/images/2023-12-17/5.png)

chain method calls 수정 및 적용 결과
![banner](https://raw.githubusercontent.com/lkhlkh23/lkhlkh23.github.io/master/images/2023-12-17/6.png)
![banner](https://raw.githubusercontent.com/lkhlkh23/lkhlkh23.github.io/master/images/2023-12-17/7.png)

### checkstyle 결론
checkstyle 은 개발자가 초기 설정만 투자한다면 코드 가독성을 향상시키는데 많은 도움을 준다고 생각한다.
개발자가 비지니스 로직에만 집중할 수 있도록 지원해주기 때문에 특별한 사유가 없다면 적용하는 것이 좋다고 생각한다.
만약 checkstyle 이 없다면, 코드 입력한 숫자보다 스페이스키와 엔터키를 더 많이 누르게 될 것이다.
