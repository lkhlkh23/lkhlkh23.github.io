---
layout: post
title: Swagger ApiResponse 자동화
subtitle: Swagger 의 ApiResponse 를 커스텀 어노테이션을 이용한 자동화
excerpt_image: https://raw.githubusercontent.com/lkhlkh23/lkhlkh23.github.io/master/images/2024-03-27/banner.png
categories: spring
tags: [springboot, swagger, apidoc]
---
### Swagger 정의

Swagger 는 REST API 를 설계, 구축, 문서화 및 사용하는 데 도움이 될 수 있는 API 기반으로 구축된 오픈 소스 도구이다.

Swagger 의 가장 큰 장점은 API 문서 자동화이다. Swagger는 API 를 문서화하기 위한 명세를 작성하고 이를 기반으로 문서를 자동으로 생성할 수 있다.

하지만 나는 이 자동화에 한 단계 더 자동화를 할 예정이다. 일단, 기본적인 Swagger 를 구현해보자

### Swagger 구현

build.gradle 에 아래와 같은 의존성을 추가한다.

```java
implementation 'org.springdoc:springdoc-openapi-starter-webmvc-ui:2.0.2'
```

그리고 Swagger 를 위한 Configuration 생성한다.

```java
@RequiredArgsConstructor
@Configuration
public class SwaggerConfig {

	@Bean
	public GroupedOpenApi openApi() {
		final String[] paths = {"/v1/**"};
		return GroupedOpenApi.builder()
							 .group("Practice API v1")
							 .pathsToMatch(paths)
							 .build();

	}
}
```

마지막으로, RestController 에 정의만 한다면, 끝이다!

추가적으로 @ApiResponse 어노테이션을 활용해서, 실패에 대한 응답 설명과 형식을 정의할 수 있다.

```java
@RestController
@RequestMapping("/v1")
public class PersonController {

	@ApiResponses(value = {
		@ApiResponse(responseCode = "200", description = "Successful Join Us!"),
		@ApiResponse(responseCode = "400", description = "Bad Request")
	})
	@Operation(summary = "회원 단일 요청", description = "회원 단일 응답", tags = { "Person Controller" })
	@GetMapping("/person")
	public Person getPerson() {
		return new Person("dob", 35);
	}
	
}
```

그러면 아래와 같은 화면을 확인할 수 있다. 간단한 코딩으로 문서를 자동으로 만들 수 있다.

![0.png](https://raw.githubusercontent.com/lkhlkh23/lkhlkh23.github.io/master/images/2024-03-27/0.png)

그러나 나는 문제에 직면했다. 내가 구현해야할 API 가 100 개가 넘는다. 그리고 각 API 마다 실패 응답에 대해서 코딩하는 것이 매우 반복적이고, 중복적이며, 많은 시간이 소요되었다.

![1.png](https://raw.githubusercontent.com/lkhlkh23/lkhlkh23.github.io/master/images/2024-03-27/1.png)

그래서 반복적이고, 중복적인 @ApiResponse 를 작성하는 것을 자동화 하려고 한다.

### @ApiResponse 자동화

1. Enum 정의

   아래와 같이 모든 오류에 대해서 Enum 으로 정의했다. 샘플 코드에서는 Enum 이 프로젝트 내부에 정의되었지만, 실제 코드에서는 nexus 에 Enum 을 publish 해서 모든 프로젝트들이 공통적으로 사용할 수 있도록 처리했다.

    ```java
    @AllArgsConstructor(access = AccessLevel.PRIVATE)
    @Getter
    public enum ErrorCode {
    	_100(404, -100, "member not found"),
    	_101(400, -101, "invalid token parameter"),
    	_102(400, -102, "invalid password"),
    	_103(500, -102, "internal server error");
    
    	private int statusCode;
    	private int errorCode;
    	private String message;
    
    	public String getExampleValue() {
    		return String.format("{\n"
    			+ "    \"code\": \"%d\",\n"
    			+ "    \"message\": %s\n"
    			+ "    \"data\": null\n"
    			+ "  }", this.errorCode, this.message);
    	}
    
    }
    ```


1. Custom 어노테이션 생성

   SwaggerHelper 라는 Custom 어노테이션을 신규로 생성한다.
   SwaggerHelper 어노테이션은 API 에서 보여줄 에러 응답에 대한 코드값을 가진다.

    ```java
    @Target({ElementType.METHOD})
    @Retention(RetentionPolicy.RUNTIME)
    public @interface SwaggerHelper {
    	ErrorCode[] targets();
    }
    ```


1. Swagger Configuration 수정

   Swagger 에서 기본적으로 제공하는 OperationCustomizer 를 커스텀하기 위한 Bean 을 생성하는 로직이 추가되었다.

   OperationCustomizer 의 customize 메소드는 API 별로 각각 호출이 된다. 그래서 SwaggerHelper 어노테이션에 정의된 에러코드의 값을 가져와 ApiResponse 를 동적으로 생성해서 정의를 하는 방식으로 구현했다.

   개발자라면 구구절절한 설명보다 코드를 보는것이 동작 방식을 파악하기 쉬울 것이다.

    ```java
    @RequiredArgsConstructor
    @Configuration
    public class SwaggerConfig {
    
    	@Bean
    	public GroupedOpenApi openApi() {
    		final String[] paths = {"/v1/**"};
    
    		return GroupedOpenApi.builder()
    							 .group("Practice API v1")
    							 .pathsToMatch(paths)
    							 .addOperationCustomizer(customize())
    							 .build();
    
    	}
    
    	@Bean
    	public OperationCustomizer customize() {
    		return new CustomOperationCustomizer();
    	}
    	private static class CustomOperationCustomizer implements OperationCustomizer {
    
    		@Override
    		public Operation customize(Operation operation, HandlerMethod handlerMethod) {
    			final SwaggerHelper swaggerHelper = handlerMethod.getMethodAnnotation(SwaggerHelper.class);
    			if(swaggerHelper != null) {
    				final ErrorCode[] errorCodes = swaggerHelper.targets();
    				final Map<Integer, List<ErrorCode>> codes = new HashMap<>();
    				for (final ErrorCode errorCode : errorCodes) {
    					if(!codes.containsKey(errorCode.getStatusCode())) {
    						codes.put(errorCode.getStatusCode(), new ArrayList<>());
    					}
    
    					codes.get(errorCode.getStatusCode()).add(errorCode);
    				}
    
    				final ApiResponses apiResponses = operation.getResponses();
    				for (final Integer key : codes.keySet()) {
    					final ApiResponse item = new ApiResponse();
    					item.setDescription(HttpStatusCode.getMessageByStatus(key));
    
    					final MediaType mediaType = new MediaType();
    					for (final ErrorCode value : codes.get(key)) {
    						final Example example = new Example();
    						example.setValue(value.getExampleValue());
    						example.setDescription(value.getMessage());
    						example.setSummary(String.valueOf(value.getErrorCode()));
    						mediaType.addExamples(String.valueOf(value.getErrorCode()), example);
    					}
    
    					final Content content = new Content();
    					content.put("*/*", mediaType);
    
    					item.setContent(content);
    					apiResponses.addApiResponse(String.valueOf(key), item);
    				}
    			}
    
    			return operation;
    		}
    	}
    
    }
    ```

2. RestController 의 API 에 @SwaggerHelper 추가

    ```java
    @SwaggerHelper(targets = {ErrorCode._100, ErrorCode._101, ErrorCode._102})
    @Operation(summary = "회원 목록 요청", description = "회원 목록 응답", tags = { "Person Controller" })
    @GetMapping("/people")
    public List<Person> getPeople() {
    	return List.of(new Person("dob", 35), new Person("lee", 35));
    }
    ```


1. 결과 확인

   아래와 같이 각각 코드에 따른 설명과 응답객체의 형식도 정의된 것을 확인할 수 있다.


![2.png](https://raw.githubusercontent.com/lkhlkh23/lkhlkh23.github.io/master/images/2024-03-27/2.png)

결과적으로 원하는 방향으로 단순 업무를 자동화해서 해결할 수 있었다. 하지만 이게 무엇이 중요한가…

결국은 프로젝트가 무산되었다. 하지만, 그래도 마무리 작업을 진행하고 있다. 마치 시든꽃에 물을 주듯이…
![3.png](https://raw.githubusercontent.com/lkhlkh23/lkhlkh23.github.io/master/images/2024-03-27/3.png)

