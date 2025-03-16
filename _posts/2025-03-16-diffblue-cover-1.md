---
layout: post
title: Diffblue Cover 를 이용한 테스트 코드 자동화
subtitle: AI 를 이용한 코드 커버리지 강제 버프
excerpt_image: https://raw.githubusercontent.com/lkhlkh23/lkhlkh23.github.io/master/images/2025-03-16/banner.png
categories: java
tags: [java, Spring, coverage]
---
최근 KPI 지표로 코드 커버리지가 설정되었다. TDD 까지는 어려워도 테스트 코드를 작성하는 것에 대해서는 찬성하기 때문에 해달 지표에 대한 불만이 없었다. 그러나 60% 는 … …

![0.png](https://raw.githubusercontent.com/lkhlkh23/lkhlkh23.github.io/master/images/2025-03-16/0.png)

하지만 방법이 없는것이 아니다! 이건 모르겠지?! 예끼?!

### What is Diffblue Cover ..?!

Diffblue Cover 는 AI 기반의 자동화된 Java 단위 테스트 생성 도구이다. 코드 분석을 통해 테스트 코드를 자동으로 생성하여, 개발자가 직접 작성해야 하는 테스트 코드의 양을 줄이고 코드 커버리지를 높이는 데 도움을 준다.

Diffblue Cover 는 `레거시 코드 리팩토링의 많은 도움`을 준다. 리팩토링은 기존 코드의 기능을 유지하면서 구조를 개선하는 작업이기 때문에 기능이 정상적으로 동작하는 것을 보장해야한다. 테스트 코드는 기능 유지를 보장할뿐만 아니라, 리팩토링 수행하는 과정에서 발생하는 불안감을 해소하는데 도움을 준다.

또한, `테스트 커버리지 향상`에 많은 도움을 준다. 80% 이상의 커버리지를 유지하는 Google 에서도 AI 를 이용해서 테스트 코드 생성 및 최적화를 보조적인 기능으로 이용하고 있다.

Diffblue Cover 는 비지니스 요구사항을 반영하지 않는다. Diffblue Cover 는 코드의 실행 흐름을 분석해서 테스트 코드를 작성하기 때문에 비지니스 로직을 검증하지 않는다. 코드의 실행 흐름에 따른 메소드의 실행 유무를 파악할뿐이다. 그리고, 테스트의 가독성이 좋지는 않다.

### How to use Diffblue Cover ..?!

[Diffblue Cover Document](https://docs.diffblue.com/get-started/get-started/get-started-cover-plugin) 에서 다양한 사용법에 대해 소개해주고 있다. 나는 가장 간단한 intellij plugin 이용해봤다. Diffblue Cover 는 Community Edition (무료) 과 Enterprise Edition (유료) 중 하나를 선택할 수 있다. Community Edition 은 Maven, Gradle plugin 을 지원하지 않기 때문에 수동 실행이 필수적이다. 또한, Java 와 Kotlin 만을 지원하는 단점이 있다.

지금은 Java 로 작성된 프로젝트에 대해서 테스트 코드의 품질을 파악하기 위한 목적이기 때문에 Community Edition 을 이용해보려고 한다.

### By Developer vs By Diffblue Cover ..?!

**By Diffblue Cover**

```java
  /**
	 * Method under test:
	 * {@link RegisterSurveyResultService#registerSurveyResult(long, List, List)}
	 */
	@Test
	void testRegisterSurveyResult4() throws Exception {
		// Arrange
		when(processS3UseCase.uploadBanners(Mockito.<List<MultipartFile>>any())).thenReturn(new ArrayList<>());

		ArrayList<SurveyResult> surveyResults = new ArrayList<>();
		SurveyResult buildResult = SurveyResult.builder()
                                                       .bannerPath("Banner Path")
                                                       .description("The characteristics of someone or something")
                                                       .subject("Hello from the Dreaming Spires")
                                                       .surveyResultId(1L)
                                                       .type("Type")
                                                       .build();
		surveyResults.add(buildResult);

		// Act and Assert
		assertThrows(Exception.class,
			() -> registerSurveyResultService.registerSurveyResult(1L, surveyResults, new ArrayList<>()));
		verify(processS3UseCase).uploadBanners(isA(List.class));
	}
```

![1.png](https://raw.githubusercontent.com/lkhlkh23/lkhlkh23.github.io/master/images/2025-03-16/1.png)

Diffblue Cover 는 아래와 같이 8개의 테스트 코드를 생성했다. 다양한 상황에 대해서 테스트를 수행했지만, 테스트 코드에서 보여진것과 같이 어떤 테스트를 수행했는지에 대해서는 명확하게 파악하는 것이 어렵다.

- 정상 테스트
- 파일업로드가 실패할 경우 Exception 발생
- 설문 결과가 없을 경우, Exception 발생
- 설문결과 1개 + 업로드된 파일이 없을 경우, Exception 발생
- 설문결과 2개 + 업로드된 파일이 없을 경우, Exception 발생
- 설문 결과가 없을 경우 + 업로드된 파일이 1개 경우, 영속성 추가 메소드 미호출
- 설문 결과가 없을 경우 + 업로드된 파일이 2개 경우, 영속성 추가 메소드 미호출
- 정상 테스트

**By Developer**

개발자 테스트는 아래와 같이 5개의 테스트 코드를 수행했다.

- 정상 테스트
- 이미지 파일이 없을 경우 NotFondException 발생
- 파일업로드가 실패할 경우 Exception 발생
- 업로드된 파일이 없을 경우, Exception 발생
- 설문 결과가 없을 경우, Exception 발생

### My experience using Diffblue Cover ..?!

Diffblue Cover 는 비즈니스 로직을 검증하기보다는 코드 실행 흐름을 검증하는 데 중점을 둔 것 다. 그렇기 때문에 코드 리팩토링과 코드 커버리지 향상에는 큰 도움이 될 것 같다. 하지만 테스트 코드를 생성하는 데 시간이 꽤 소요되었고, 실시간 성능보다는 배치성 작업에 더 적합한 것 같다.

이제는 CLI 를 이용해서 CI/CD 와 통합해서 사용해보려고 한다. 하지만 유료이다! 사용할 수 없다. 테스트 코드 자동화에 비용이 발생한다고 한다면 … 이 조직은 절대 돈을 쓰지 않을 것이다! 확신한다! 다소 불편하지만, Plugin 을 이용해서 사용해야겠다!

### And I am ..

내가 사용하는 키보드가 너무 시끄럽다고 한다. 어느날 퇴근하고 노트북 앞에 앉으니 기계식 키보드가 사라졌다. 그래서 노트북 키보드를 치면서 입으로 소리를 내고 있다. 그리고 다음날 다시 기계식 키보드가 돌아왔다.
