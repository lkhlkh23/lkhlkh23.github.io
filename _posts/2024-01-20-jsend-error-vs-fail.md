---
layout: post
title: API 응답값 형에 대한 고민-2
subtitle: JSend fail vs error
excerpt_image: https://raw.githubusercontent.com/lkhlkh23/lkhlkh23.github.io/master/images/2024-01-20/banner.png
categories: spring
tags: [java, spring, rest, jsend]
---

![banner](https://raw.githubusercontent.com/lkhlkh23/lkhlkh23.github.io/master/images/2024-01-20/banner.png)

오늘은 API Response Format 에 대한 간단한 협의를 했다. 아래 정리한 내용을 공유했고, 긍정과 부정의 피드백을 하나씩 받았다.

- 참고 : [API 응답값 형에 대한 고민](https://lkhlkh23.github.io/spring/2024/01/18/api-response-format.html)

우선 사용자 이슈와 서버의 비지니스 로직 이슈를 구분하는 `fail` 과 `error` 에 대해서는 긍정적이었다.

하지만, `fail` 과 `error` 경계가 모호했다. 협의하는 과정에서 다양한 상황에 대해서 `fail` 인지 `error` 명확하게 답변하지 못했다. 그래서, 우선 JSend 문서에서 정의한 `fail` 과 `error` 에 대해 다시 이해하려고 한다.

### fail vs error

Jsend 에서 `fail` 과 `error` 를 아래와 같이 정의했다.

<aside>
💡 fail : There was a problem with the data submitted, or some pre-condition of the API call wasn't satisfied

error : An error occurred in processing the request

</aside>

`fail` 은 요청에 문제가 있거나 API 요청의 조건 중 일부가 충족되지 않는 경우를 의미한다. 결국 이것을 나의 방식으로 해석하자면, ‘요청에 대해서 서버가 정상적으로 처리했지만 조건이 맞지 않음’ 이다.

결국은 서버의 문제가 아닌, 사용자의 문제이다.

`error` 는 요청을 처리하는 과정에서 문제가 발생하는 경우를 의미한다. 결국 이것을 나의 방식으로 다시 해석하자면, ‘요청을 서버가 정상적으로 처리하지 못함’ 이다.

결국은 사용자의 문제가 아닌 서버의 문제이다.

이번에는 HttpStatus 를 이용해서 다양한 상황속에서 `fail` 과 `error` 를 구분해보려고 한다.

### **4XX : Client Error Responses**

한국어로 직역하면, `4xx 는 사용자 실수 응답` 이다.

![0](https://raw.githubusercontent.com/lkhlkh23/lkhlkh23.github.io/master/images/2024-01-20/0.png)
뭔가 벌써 4xx 오류에 대해 내가 무엇을 말하고 싶은지 감이 올 것이다. 대중적인 4xx 오류를 살펴보자!

- **400 Bad Request**
  - 잘못된 요청으로 인해 서버가 요청을 처리할 수 없음을 의미
    → 서버의 문제가 아닌, 잘못된 요청으로 인해서 발생 → `fail`
- **401 Unauthorized, 403 Forbidden**
  - 요청에 대해 응답을 받기 위해서는 인증이 필수
    → 서버의 문제가 아닌, 인증되지 않는 사용자의 요청으로 인해서 발생 → `fail`
- **404 Not Found**
  - 요청에 대한 리소스가 존재하지 않음
    → 서버의 문제가 아닌, 단순히 데이터가 존재하지 않아서 발생 → `fail`
- **415 Unsupported Media Type**
  - 서버에서 지원하지 않는 미디어 포맷으로 요청
    → 서버의 문제가 아닌, 서버에서 지원하지 않는 방식으로 사용자의 요청으로 인해서 발생 → `fail`

### **5XX : Server error responses**

한국어로 직역하면, `서버 에러 응답` 이다.

- **500 Internal Server Error**
  - 서버에 문제가 있음을 의미하지만 서버는 정확한 문제에 대해 더 구체적으로 설명할 수 없음을 의미
    → 알 수 없는 서버의 문제 → `error`
- **501 Not Implemented**
  - 서버가 요청을 처리하는데 필요한 기능을 지원하지 않음을 의미
    → 서버가 정상적으로 처리하지 못한 서버의 문제 → `error`
- **503 Service Unavailable**
  - 서버가 요청을 처리할 준비가 되지 않았거나, 서버의 과부하를 의미
    → 서버가 요청에 대해 처리할 능력이 되지 않는 서버의 문제 → `error`

### 결론

인민재판의 소질이 있거나 남탓을 잘한다면 `fail` 과 `error` 를 구분하는 것은 생각보다 쉽다.

사용자로 인한 `4xx` 코드는 `fail` 이고 서버로 인한 `5xx` 코드는 `error` 이다.

너무 복잡하게 생각할 필요가 없다. 우리에게 친숙한 `4xx` 와 `5xx` 로 구분하면 될 것 같다.
![1](https://raw.githubusercontent.com/lkhlkh23/lkhlkh23.github.io/master/images/2024-01-20/1.png)

오늘 피티 가격을 확인해봤다. 예상했지만 엄청난 가격이 당황스러웠다. 나에 대한 적절한 투자가 맞는지 한번 의심했다.
PI, PS 도 얼마 되지 않는데, 이것이 맞은 것일까?!

아.. 참고로 나는 물욕이 없다!

