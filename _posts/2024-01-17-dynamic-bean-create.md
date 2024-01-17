---
layout: post
title: 프로퍼티 파일을 이용해서 Bean 동적 생성
subtitle: BeanPostProcessor 이용해서 Bean 동적 생성
excerpt_image: https://raw.githubusercontent.com/lkhlkh23/lkhlkh23.github.io/master/images/2024-01-17/banner.png
categories: spring
tags: [java, spring, bean]
---

![banner](https://raw.githubusercontent.com/lkhlkh23/lkhlkh23.github.io/master/images/2024-01-17/banner.png)

최근에 속성 값만 다를 뿐, 동일한 클래스의 Bean 을 생성하는 코드를 작성했다. 다시 코드를 봤을 때 중복 코드가 신경이 많이 쓰였다. 또한, 속성 값이 더 추가되었을 때 더 많은 중복 코드가 발생할 것 같은 불길한 예감이 들었다.

그래서, Bean 을 동적으로 생성할 수 있는 방법을 고민하게 되었다.

Spring 에서는 개발자가 정의한 Bean 을 생성하기 전에 개입할 수 있는 인터페이스를 제공하고 있다. BeanPostProcessor 와 BeanFactoryPostProcessor 를 이용하면 된다.

### BeanFactoryPostProcessor

Bean 생성되기 전에 개입할 수 있는 postProcessBeanFactory() 메소드를 제공한다. 앞으로 Bean 이 될 예비 Bean 들이 저장되있는 BeanFactory 를 제공한다. Bean 생성 전에 저장된 예비 Bean 에 대해 재정의할 수 있다. 참고로 BeanFactory 에서 제공되는 것은 생성 완료된 **Bean 이 아닌, 초기화되지 않는 Bean** 예비 Bean 이다.

### BeanPostProcessor

Bean 이 생성되고, init() 메소드 호출 전후에 개입할 수 있다.

- postProcessBeforeInitialization() : init() 메소드 호출 전에 실행
- postProcessAfterInitialization() : init() 메소드 호출 후에 실행

그렇다면, 나는 어떤 방식을 이용해서 동적으로 Bean 을 생성해야할까?!

내가 선택한 것은 **BeanPostProcessor** 이다.

나는 아래와 같은 yaml 파일에 있는 값을 이용해서 Bean 을 생성해야 한다.

```yaml
containers:
  - name: consumer-1
    id: id-1
    password: password-1
    regions:
      - east
      - west
  - name: consumer-2
    id: id-2
    password: password-2
    regions:
      - south
      - north
```

그리고, 이러한 yaml 파일에 설정한 속성값에 대해서 클래스로 정의했다.

```java
@Component
@Getter
@Setter
@ConfigurationProperties
public class ContainerProps {

	private List<Container> containers;

	@Getter
	@Setter
	public static class Container {

		private String name;
		private String id;
		private String password;
		private List<String> regions;

	}
}
```

이제 왜 내가 **BeanPostProcessor** 를 선택했는지 감이 왔는가?!

BeanFactoryPostProcessor 를 이용했다면, ContainerProps 라는 Bean 이 생성되지 않았기 때문에 개입할 수 없다. 그러므로 **BeanPostProcessor** 이용해서 ContainerProps 이 생성이 완료된 시점에 개입해서  동적으로 Bean 을 생성해야 한다.

**BeanPostProcessor** 는 아래와 같이 정의했다.

```java
@Component
@RequiredArgsConstructor
public class DynamicBeanInitializer implements BeanPostProcessor {

	private final ApplicationContext context;

	@Override
	public Object postProcessBeforeInitialization(final Object bean, final String beanName) throws BeansException {
		if(!(bean instanceof ContainerProps)) {
			return bean;
		}

		final ContainerProps props = ((ContainerProps) bean);
		for (final ContainerProps.Container container : props.getContainers()) {
			for (final String region : container.getRegions()) {
				final ContainerBean containerBean = new ContainerBean(container.getName(), container.getId(), container.getPassword(), region);
				((ConfigurableApplicationContext) context).getBeanFactory().registerSingleton(container.getName() + region, containerBean);
			}
		}

		return bean;
	}
}
```

생성된 Bean 중에서 application.yaml 속성값을 정의한 Bean 일 때, 속성 값을 읽어서 BeanFactory 에 Bean 을 등록하는 방식으로 구현이 되있다. 생각보다 매우 심플하다.

그러면 검증은 어떻게 할 수 있을까?!

```java
@SpringBootTest
class BeanApplicationTests {

	@Autowired
	private ApplicationContext context;

	@Autowired
	@Qualifier("consumer-1east")
	private ContainerBean bean;

	@Test
	void test_create_dynamic_bean() {
		// given

		// when
		final ContainerBean containerBean = (ContainerBean) context.getBean("consumer-1east");

		// then
		assertEquals("consumer-1", containerBean.getName());
	}

}
```

Bean 들이 모두 정의된 ApplicationContext 에 동적으로 생성한 Bean 이름이 있는지 확인하는 방식으로 검증했다. 코드는 아래 git repository 를 참고해라!

- [dynamic-bean-creator-git](https://github.com/lkhlkh23/dynamic-bean-creator)

연말정산을 해야한다. 작년에는 할머니, 할아버지를 나의 부양가족으로 넣어서 만족스러운 환급을 받았다. 그러나 오늘 집에 가는 길에 엄마와 통화하면서 알게되었다. 막내삼촌이 할머니를 뺴앗았다.

한번 맛본 자본의 맛을 뺏긴 것에 대한 분노로 하마터면 부천으로 달려갈뻔 했다.

![banner](https://raw.githubusercontent.com/lkhlkh23/lkhlkh23.github.io/master/images/2024-01-17/1.png)

참고로 나는 물욕이 없다.


