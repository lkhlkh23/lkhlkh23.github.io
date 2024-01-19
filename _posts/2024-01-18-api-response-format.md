---
layout: post
title: API 응답값 형에 대한 고민-1
subtitle: HttpStatus + JSON Body vs Only JSON Body (JSend)
excerpt_image: https://raw.githubusercontent.com/lkhlkh23/lkhlkh23.github.io/master/images/2024-01-18/banner.png
categories: spring
tags: [java, spring, rest]
---

![banner](https://raw.githubusercontent.com/lkhlkh23/lkhlkh23.github.io/master/images/2024-01-18/banner.png)

# Response

신규로 진행하는 프로젝트에서 API 응답에 대한 표준을 정해야 하는 협의가 내일 예정되있다.

API 응답에 대한 표준은 F/E 개발자와 B/E 개발자 간의 공통적인 약속을 정의함으로써 상호작용에 많은 도움을 줄 수 있다. 또한, API 호출은 F/E 뿐만 아니라, B/E 서버 간에서도 많은 호출이 있기 때문에 이 부분도 같이 고려를 해야한다.

개인적으로 두가지 방식을 고민했고, 하나를 선택하려고 한다.

### HttpStatus + JSON Body

API 호출 성공과 실패 관계없이, HttpStatus 와 JSON Body 를 함께 제공하는 방식이다.

사용자의 잘못된 입력에 대해 400 Bad Request 코드와 함께 JSON message 에서 예외에 대한 메세지를 함께 제공하고 있다.

![0.png](https://raw.githubusercontent.com/lkhlkh23/lkhlkh23.github.io/master/images/2024-01-18/0.png)

![0.png](https://raw.githubusercontent.com/lkhlkh23/lkhlkh23.github.io/master/images/2024-01-18/1.png)

ExceptionHandler 를 이용해서 위와 같은 결과를 받을 수 있도록 구현했다.

@ResponseStatus 이용해서 HttpStatus 코드를 정의했고, @ResponseBody 이용해서 JSON Body 를 정의했다.

```java
@RestControllerAdvice
@Slf4j
public class RestControllerExceptionHandler {

	@ExceptionHandler(WordMatchException.class)
	@ResponseStatus(code = HttpStatus.BAD_REQUEST)
	@ResponseBody
	public Response wordMatchException(final WordMatchException e) {
		log.warn(e.getMessage());
		return new Response(null, e.getMessage(), ResponseStatusCode.FAIL);
	}

}
```

이 방식의 장점은 F/E 에서 예외처리가 굉장히 편하다는 장점이 있다.

**2xx 성공에 대해서는 success 에서 처리하고 4xx 와 5xx 오류에 대해서는 error 에서 처리할 수 있기 때문에 성공과 실패에 대한 응답을 아주 편리하게 구분**할 수 있다.

```jsx
$(document).ready(function () {
    $('#btn').on('click', function() {
        $.ajax({
            type: 'GET',
            url: '/keyword?word=ex',
            contentType: "application/json; charset=utf-8",
            success: function (response) {
                console.log(response.data);
                console.log(response.message);
                console.log(response.status);
            },
            error: function (response) {
                console.log(response.responseJSON.data);
                console.log(response.responseJSON.message);
                console.log(response.responseJSON.status);
            }
        });
    });
});
```

하지만, API 의 호출은 F/E 뿐만 아니라, B/E 서버에서도 빈번하게 발생한다. B/E 서버에서는 Feign, Retrofit 과 같은 라이브러리를 이용해서 다른 API 를 많이 호출한다. 그런데 **예외를 응답하게 되면 호출한 서버는 try-catch 를 이용해서 예외처리가 필요**하다.

### Only JSON Body 를 제공하는 방식

이 방식의 가장 많이 등장하는 방식은 JSend 이다. 이 방식은 성공과 실패 모두 HttpStatus.OK 로 응답한다.
대신 성공과 실패는 JSON Body 에서 구분을 해야한다.

![3.png](https://raw.githubusercontent.com/lkhlkh23/lkhlkh23.github.io/master/images/2024-01-18/3.png)

![4.png](https://raw.githubusercontent.com/lkhlkh23/lkhlkh23.github.io/master/images/2024-01-18/4.png)

JSend 는 세가지 유형의 응답이 존재한다.

`success` 는 단어 그대로 작업이 성공적으로 종료된 유형을 의미한다. `fail` 은 사용자가 유효하지 않는 입력값을 전달했거나, 입력값에 따른 데이터가 없을 때와 같은 상황에 쓰이는 유형을 의미한다. `error` 는 비지니스 로직을 처리하는 과정에서 서버에서 발생한 오류를 의미한다.

JSend 에서 가장 좋은점은 사용자 이슈와 비지니스 로직의 이슈를 명확하게 구분한다는 점이다.

**success response**

```json
## GET /posts

{
    status : "success",
    data : {
        "posts" : [
            { "id" : 1, "title" : "A blog post", "body" : "Some useful content" },
            { "id" : 2, "title" : "Another blog post", "body" : "More content" },
        ]
     }
}
```

**fail response**

```json
## POST /posts.json (with data body: 

{
    "status" : "fail",
    "message" : "A title is required"
}
```

**error response**

```json
## GET /posts

{
    "status" : "error",
    "message" : "Unable to communicate with database"
}
```

이 방식의 장점과 단점은 앞서 소개한 방식의 반대인것 같다.

이 방식은 **succes, fail, error 모두 2xx 상태를 응답하기 때문에 JSON Body 의 status 를 이용해서 성공과 실패를 구분해야 하는 단점**이 있다.

하지만, B/E 서버 사이의 호출에서는 fail, error 에서 빈 데이터를 전달할 뿐, 2xx 로 응답하기 때문에 호출에 대한 try-catch 예외처리는 하지 않아도 된다.

### 결론

`HttpStatus + JSON Body` 와 `Only JSON Body` 중에서 어떤 것을 선택해야 할까?!

내가 선택한 방식은 error 와 fail 에 대해서 다르게 처리하는 방식이다.

```java
@RestControllerAdvice
@Slf4j
public class RestControllerExceptionHandler {

	@ExceptionHandler(WordMatchException.class)
	public Response wordMatchException(final WordMatchException e) {
		log.warn(e.getMessage());
		return new Response(null, e.getMessage(), ResponseStatusCode.FAIL);
	}

	@ExceptionHandler(Exception.class)
	@ResponseStatus(code = HttpStatus.INTERNAL_SERVER_ERROR)
	@ResponseBody
	public Response exception(final Exception e) {
		log.error(e.getMessage());
		return new Response(null, e.getMessage(), ResponseStatusCode.ERROR);
	}

}
```

`error` 에 대해서는 5xx HttpStatus 와 오류 메세지에 대한 JSON Body 를 함께 제공한다. 그리고 log.error 를 통해 예외 이유를 남긴다. (`HttpStatus + JSON Body`)

`fail` 에 대해서는 오류 메세지에 대한 JSON Body 만을 제공하고, log.warn 를 통해 예외 이유를 남긴다. (`Only JSON Body`)

이렇게 선택한 이유는 발생한 예외가 비지니스 로직 문제가 발생했고, 이 문제로 인해 수정이 필요한지 필요하지 않는지를 구분하기 위해서다. 결국은 `운영의 편의성` 때문이다.
내가 B/E 개발자라서, B/E 에서 예외처리가 싫어서가 아니다! 그러하다!

![5.png](https://raw.githubusercontent.com/lkhlkh23/lkhlkh23.github.io/master/images/2024-01-18/5.png)
