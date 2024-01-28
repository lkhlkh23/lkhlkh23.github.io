---
layout: post
title: OAuth 2.0 상세 동작순서와 OAuth 1.0 과 비교
subtitle: OAuth 2.0 상세 동작순서, OAuth 상세 설명, OAuth 1.0 비교
excerpt_image: https://raw.githubusercontent.com/lkhlkh23/lkhlkh23.github.io/master/images/2024-01-28/banner.png
categories: spring
tags: [spring, oauth, oauth1.0, oauth2.0]
---

![banner](https://raw.githubusercontent.com/lkhlkh23/lkhlkh23.github.io/master/images/2024-01-28/banner.png)

이번에 진행하는 프로젝트에서 Social Login 개발이 필요하다. 지금까지 전시 서비스를 주로 개발을 진행하다보니, Sociail Login 을 직접 구현할 기회가 없었다. 
이번에 개발을 진행하기 전에 Social Login 에 대해서 간단하게 정리를 하면서 진행하려고 한다.

우선 이론적인 부분을 먼저 확인하고, 가장 대표적인 Google 이용해서 개발을 진행하려고 한다.

### OAuth 정의

OAuth (Open Authorization) 는 인터넷 사용자들이 비밀번호를 제공하지 않고 다른 웹사이트 상의 자신들의 정보에 대해 웹사이트나 애플리케이션의 접근 권한을 부여할 수 있는 공통적인 수단으로서 사용되는 접근 위임을 위한 개방형 표준 프로토콜이다.

> 개방형 표준 : 기술 표준 문서가 공개되어 있으며 사용이 자유로운 경우를 가리키는 용어
프로토콜 : 통신 규약
>

OAuth에서 'Auth'는 Authentication (인증) 과 Authorization (인가) 를 모두 포함한다. 
그렇기 때문에 OAuth 진행할 때 리소스 제공자는 제3자가 사용자의 권한으로 사용자의 정보를 접근하려는 것을 허용하는지에 대해 동의를 받는 것이다.

### Authentication 인증 vs Authorization 인가

`Authentication 인증` 은 사용자가 누구인지 확인하는 프로세스이고, `Authorization 인가` 는 사용자가 접근할 수 있는 항목을 확인하는 프로세스이다.

결국, OAuth 는 `제3자의 검증`하고, 제3자에게 사용자의 자원을 접근할 수 있도록 `권한을 부여`하는 것이다.

### OAuth 2.0 동작 순서

**역할 설명**

| 용어                   | 설명                                                       |
|----------------------|----------------------------------------------------------|
| Resource Owner       | Resource Server 에 자원을 가지고 있고, Client 를 이용할려는 End User    |
| Client               | Resource Owner 의 리소스에 접근 요청을 하는 서비스                      |
| Authorization Server | 인증/인가를 수행하는 권한 서버                                        |
| Resource Server      | Client 의 접근 자격을 확인하고, Access Token 을 발급하여 권한을 부여하는 역할 수행 |
| Resource Server      | Resource Owner 의 자원을 보유하는 서버                             |

**동작순서**

![0](https://raw.githubusercontent.com/lkhlkh23/lkhlkh23.github.io/master/images/2024-01-28/0.png)

OAuth 2.0 의 과정을 5Step 으로 정리를 했다.

**Step1. 서비스 등록**

Client 는 Authorization Server 에 `Authorized Redirect Url` 과 함께 서비스 등록을 신청한다. `Authorized Redirect Url` 은 Authorization Server 가 인증 완료 후 응답을 보낼 Client 의 URL 에 해당된다.

서비스를 등록하면, Client 의 `Client-Id` 와 `Client-Secret` 이 전달된다.

`Client-Id` 는 Client 를 식별할 수 있는 Key 이고, `Client-Secret` 는 Key 의 패스워드와 같은 개념이다.

`Client-Id` 와 `Client-Secret` 는 인증을 하는데 필요하다.

**Step2. Resource Owner 승인**

Client 는 Resource Owner 에게 로그인 화면을 전달한다. 로그인 버튼의 URL 에는 `Client-Id` , `Authorized Redirect Url` 그리고 `Scope` 를 파라미터로 가지고 있다. 여기서 `Scope` 는 Resource Owner 의 자원에 대해 접근을 할 수 있는 범위를 의미한다. 예를들어, 사진등록, 사진조회, 사진삭제와 같은 기능적인 범위를 의미한다.

Resource Owner 가 로그인하면, Authorization Server 는 파라미터로 있는 정보를 이용해서 검증한다. 그리고 Resource Owner 에게 자원 접근에 대한 Scope 에 대해서 동의를 하는지 물어본다.

Resource Owner 가 동의를 하면 Step2 가 완료된다.

**Step3. Authorization Server 승인**

Authorization Server 는 `Authorization Code` 를 생성하고, Resource Owner 에게 Redirection URL 과 함께 `Authorization Code` 를 전달한다. Resource Owner 에게 전달되면 Client 로 리디렉션되면서  `Authorization Code` 가 전달된다.

`Authorization Code` 는 `Access Token` 발급에 필요한 재료에 불과하다. Client 는 Client 가 보유하고 있는 정보들을  모두 Authorization Server 에게 전달함으로써 `Access Token` 발급 요청을 한다.

Authorization Server 가 `Authorized Redirect Url` , `Client-Id` , `Client-Secret` , `Authorization Code` 를 검증을 하면서 Step3 가 완료된다.

**Step4. Token 발급**

Authorization Server 가 검증을 완료하면, Client 에게 `Access Token` 과 `Refresh Token` 이 발급된다.

`Access Token` 는 Client 가 Resource Server 에 접급할 때, 필요한 Token 이다. `Refresh Token` 는 Access Token 이 만료될 때, 재생성하는 Token 이다.

**Step5. API 호출**

Client 는 Resource Server 에서 Resource Owner 의 정보를 접근할 때, `Access Token` 이용한다.

### OAuth 1.0 vs OAuth 2.0

지금까지 정리한 내용은 OAuth 2.0 이다. 그렇다는 것은 OAuth 1.0 도 존재한다는 뜻이다.

OAuth 1.0 에 대해서도 간단하게 정리를 해보려고 한다.

OAuth 1.0 은 2006년에 트위터와 Gnolia 의 개발자가 API 접근 위임에 대해서 논의한 이후, 2007년에 공유되었다. 그리고, 2010년에 표준 프로토콜로서 발표가 되었다. 내가 개발자를 꿈꾸기도 이전에 이러한 과정들이 고민되어 논의되었다.

OAuth 1.0 은 앞에 소개한 OAuth 2.0 과 유사한 방식으로 동작한다.

하지만 OAuth 1.0 과 OAuth 2.0 은 명확한 차이가 있다.

**Authorization Server 와 Resource Server 의 역할 분리**

OAuth 1.0 에서는 Authorization Server 와 Resource Server 가 각각으로 분리되지 않고 하나의 Server 에서 수행했다.

|  | OAuth 1.0 | OAuth 2.0 |
| --- | --- | --- |
| Resource Owner 인증 | OAuth 1.0 Server | Authorization Server |
| Access Token 발급 | OAuth 1.0 Server | Authorization Server  |
| Resource 관리 | OAuth 1.0 Server | Resource Server |

하지만, OAuth 2.0 에서는 역할에 따라 각각의 서버를 분할했다.

**Scope 기능 추가**

OAuth 1.0 에서는 Access Token 이 발급되면, Client 는 Resource Server 에 있는 Resource Owner 의 모든 자원에 대해 접근을 할 수 있다.

하지만, OAuth 2.0 에서는 Resource Owner 에게 Scope 의 동의를 확인하는 단계가 추가되면서 Scope 기능이 추가되었다.

**토큰 탈취 문제 개선**

OAuth 1.0 에서는 Access Token 의 유효기간이 6개월, 1년과 같이 길었다. Access Token 이 짧다면, Resource Owner 는 항상 인증/인가의 단계를 수행해야하기 때문에 사용자 경험이 좋지 않아서 그럴것 같다.
하지만 Access Token 이 탈취되었을 때, 유효기간이 긴 Access Token 은 보안적으로 굉장히 위험하다.

OAuth 2.0 에서는 Refresh Token 이 추가되었다. Refresh Token 이용해서 Access Token 을 새로 생성할 수 있기 때문에 Access Token 의 유효기간을 짧게 관리할 수 있게되었다.

**제한적인 사용환경**

OAuth 1.0 이 처음 등장했던 2007년에는 지금과 같이 스마트폰이 대중적으로 보급되지 않았고, 웹 브라우저를 많이 사용하던 시절이었다. 그렇기 때문에 OAuth 1.0 은 웹 브라우저에 최적화되있다.

OAuth 2.0 에서는 Grant 를 통해 다양한 환경에서의 인증/인가를 지원할 수 있게되었다.

Grant 에는 `Authorization Code Grant`, `Implicit Grant`, `Resource Owner Password Credentials Grant` 그리고 `Client Credentails Grant` 가 있다.

아래에서 좀더 상세하게 알아보자

### Grant

**Authorization Code Grant**

‘OAuth 2.0 동작순서’ 에서 소개했던 방식이 바로 `Authorization Code Grant` 방식이다.

Server to Server 통신을 통해 Token 을 발급하는 방식으로서 가장 보편적인 방식이며, Resource Owner 와의 상호작용이 필요한 방식이다.

**Implicit Grant**

`Implicit Grant` 는 웹 브라우저에 있는 자바스크립트 기반 클라이언트와 같이 제한된 환경에서 Server to Server 통신을 하지 않는 환경에 적합하다. `Authorization Code Grant` 의 간략한 버전으로 보면된다. `Authorization Code Grant` 는 Accss Token 을 발급받기 위해 Authorization Code 를 통한 검증단계를 복잡하게 수행했는데, `Implicit Grant` 방식에서는 Access Token 을 직접받음으로써 과정이 간단하다.

**Resource Owner Password Credentials Grant**

직접적으로 Resource Owner 의 Id 와 Password 를 받아서 Client 가 요청을 하는 방식이다. 직접적으로 Resource Owner 의 정보가 전달되는 방식이기 때문에 믿을 수 있는 환경에서 사용하는 것을 권장한다.

**Client Credentails Grant**

Resource Owner 와 Authorization Server 가 동일할 때, 사용이 권장되는 방식이다.

동일하기 때문에 복잡한 인증/인가 과정을 간소화한 방식이다.

위에 소개된 4가지의 Grant 는 OAuth 2.1 에서 보안에 취약한 Grant 방식에 대해서 폐지가 되었다.

`Implicit Grant` 와 `Resource Owner Password Credentials Grant` 방식은 사요나라했다.

이제 본격적으로 구현을 해볼려고 한다. 오늘 개발은 내일의 내가 할테니.. 다음에 하겠다.

월요일부터 버스 노선 시간이 변경이 되면서 평소보다 10분 일찍 일어나게됬다. 10분 일찍 출발해서, 10분일찍 회사에 도착하게 된다면 문제가 되지 않지만… 10분 일찍 출발해서 평소와 똑같이 도착하게 된다면…
![0](https://raw.githubusercontent.com/lkhlkh23/lkhlkh23.github.io/master/images/2024-01-28/1.png)

여기까지 하겠다.
