---
layout: post
title: OpenID Connect 와 Google login 구현
subtitle: OpenID Connect vs OAuth 2.0 그리고 Google login 구현
excerpt_image: https://raw.githubusercontent.com/lkhlkh23/lkhlkh23.github.io/master/images/2024-02-03/banner.png
categories: spring
tags: [spring, openid-connect, google-login, oauth2.0]
---

![banner](https://raw.githubusercontent.com/lkhlkh23/lkhlkh23.github.io/master/images/2024-02-03/banner.png)

이전 글에서는 OAuth 에 대해서 알아봤고, 이제는 Google Login 직접 구현해보려고 한다.

참고로 아래 코드가 완성된 코드이다.

- [OAuth Github Repository](https://github.com/lkhlkh23/practice-oauth)

![0](https://raw.githubusercontent.com/lkhlkh23/lkhlkh23.github.io/master/images/2024-02-03/0.png)

그전에! `OpenID Connect` 에 대해 간단하게 알아보려고 한다. OpenID Connect 가 Google Social Login 의 핵심이기 때문이다.

### OpenID Connect

OpenID Connect 는 줄여서 `OIDC` 라고도 부른다. ~~풀네임을 타이핑하는 것이 귀찮기 때문에가 아닌,~~ 쉽게 부를 수 있도록 OIDC 라고 부르겠다.

`OIDC` 는 OAuth 2.0 의 인증을 담당하는 프로토콜이다. `OIDC` 는 Client 가 Resource Owner 의 신원 검증 및 간단한 정보를 제공해준다. OAuth 2.0 에서는 Resource Server 에서 Resource Owner 의 자원에 접근하기 위해서 인증 작업이 필수적으로 진행되기 때문에 `OIDC` 가 OAuth 2.0 의 상위 계층이 되는 것이다.

`OIDC` 는 Resource Owner 의 정보를 제공해준다고 했는데, JWT (Json Web Token) 형식의 인증정보를 제공하고 있다.  아래 정보는 `OIDC` 를 구현해서 디버깅한 결과이다.

![1](https://raw.githubusercontent.com/lkhlkh23/lkhlkh23.github.io/master/images/2024-02-03/1.png)

`OIDC` 와 `OAuth 2.0` 의 경계가 모호할 것이다. `OIDC` 와 `OAuth 2.0` 는 목적이 다르다.

`OIDC` 의 목적은 `인증`이다. Resource Owner 의 신원을 검증한다.

`OAuth 2.0` 의 목적은 `인가` 이다. OAuth 2.0 는 Resource Owner 의 자원에 접근할 수 있는 권한을 부여받는 것에 목적을 가지고있다. 그리고 권한을 부여받는 과정에서 인증을 진행한다. 그렇기 때문에 `OIDC` 와 `OAuth 2.0` 의 경계를 혼동하는 것 같다.

`OIDC` 는 아래와 같은 절차로 진행된다.

![1.5](https://raw.githubusercontent.com/lkhlkh23/lkhlkh23.github.io/master/images/2024-02-03/1.5.png)

아래 개발 내용은 `REQUESTS ACESS AND ID TOKENS` 절차부터라고 생각하면 된다.

그래서 나는 `OIDC` 를 이용해서 Google Login 을 개발하려고 한다.

### 개발 내용

설명이 필요한 핵심적인 내용만 설명하겠다.

**SecurityConfig**

```java
@RequiredArgsConstructor
@EnableWebSecurity
public class SecurityConfig extends WebSecurityConfigurerAdapter {

	private final CustomOAuth2UserService oauthService;

	@Override
	protected void configure(final HttpSecurity http) throws Exception {
		http
			.csrf().disable()
			.headers().frameOptions().disable()
			.and()
				.authorizeRequests()
				.antMatchers("/", "/css/**", "/js/**", "/img/**", "/view/**").permitAll()
				.antMatchers("/api/v1/**").hasRole(UserRoleType.NORMAL.getRole())
				.anyRequest().authenticated()
			.and()
				.logout()
					.logoutSuccessUrl("/view/home")
			.and()
				.oauth2Login()
					.defaultSuccessUrl("/view/home")
					.userInfoEndpoint()
				.userService(oauthService);
	}

}
```

- csrf().disble()
  - spring security 에서 csrf 토큰없이 요청을 하면 요청을 블락하기 때문에 비활성화
- authorizeRequests().anyMatchers()
  - css, js, img 를 포함한 정적 리소스와 화면에 대한 접근에 대해서는 권한 관계없이 접근 가능
  - api 에 대해서는 NORMAL 권한이 있는 사용자만 접근 가능
- anyRequest().authenticated()
  - 그외 모든 경로는 인증된 사용자만 접근 가능
- logout().logoutSuccessUrl()
  - 로그아웃을 하면 “/view/home” 으로 이동
- oauth2Login().defaultSuccessUrl()
  - 로그인 성공하면 “/view/home” 으로 이동
- userService(oauthService)
  - social login 성공 이후 작업을 진행할 수 있는 UserService 인터페이스 구현체
  - Resource Server 에서 Resource Owner 의 정보를 가져온 , 추가로 진행하고자 하는 작업 정의 가능

**CustomOAuth2UserService**

```java
@Slf4j
@Component
@RequiredArgsConstructor
public class CustomOAuth2UserService extends DefaultOAuth2UserService {

	private final UserRepository userRepository;
	private final HttpSession httpSession;

	@Override
	public OAuth2User loadUser(final OAuth2UserRequest userRequest) throws OAuth2AuthenticationException {
		final String registrationId = userRequest.getClientRegistration()
												 .getRegistrationId();
		final OAuth2User oAuth2User = super.loadUser(userRequest);
		final Map<String, Object> attributes = oAuth2User.getAttributes();
		HttpSessionUtil.getInstance().setSession(httpSession, registerUser(attributes, registrationId));

		return oAuth2User;
	}

	private User registerUser(final Map<String, Object> attributes, final String registrationId) {
		final UserEntity loginUser = UserEntity.builder()
											   .email((String) attributes.get("email"))
											   .provider(registrationId)
											   .name((String) attributes.get("name"))
											   .picture((String) attributes.get("picture"))
											   .userRole(UserRoleType.NORMAL)
											   .build();
		return User.from(userRepository.save(loginUser));
	}
	
}
```

- registrationId : Resource Server (예 : google)
- oAuth2User.getAttributes()
  - OAuth2UserService를 통해 가져온 OAuth2User의 attirbute

![1](https://raw.githubusercontent.com/lkhlkh23/lkhlkh23.github.io/master/images/2024-02-03/1.png)

**application.yaml**

```yaml
server:
  port: 8080

spring:
  mustache:
    suffix: .html
  profiles:
    include: oauth
  jpa:
    hibernate:
      ddl-auto: update
    open-in-view: true
    show-sql: true
  datasource:
    hikari:
      jdbc-url: jdbc:h2:~/oauth-db;DB_CLOSE_DELAY=-1;DB_CLOSE_ON_EXIT=FALSE
      username: sa
      password:
      driver-class-name: org.h2.Driver
  h2:
    console:
      enabled: true
      path: /h2-console
```

- spring.profiles.include
  - application-oauth.yaml 에 정의된 설정값들을 불러오기 위한 설정
  - application-oauth.yaml 에 정의된 client-id 와 client-secret 은 git 업로드하기에는 민간함 정보이기 때문에 .gitignore 에서 application-oauth.yaml 은 제외되도록 설정

**application-oauth.yaml**

```yaml
spring:
  security:
    oauth2:
      client:
        registration:
          google:
            client-id: XXXXX
            client-secret: XXXXX
            scope: profile,email
```

![3](https://raw.githubusercontent.com/lkhlkh23/lkhlkh23.github.io/master/images/2024-02-03/3.png)
- client-id 는 화면의 클라이언트 ID 를 입력하고, client-secret 은 화면의 클라이언트 보안 비밀번호를 입력

**login.html**

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Login Form</title>
    <!-- Bootstrap CSS -->
    <link rel="stylesheet" href="https://stackpath.bootstrapcdn.com/bootstrap/4.3.1/css/bootstrap.min.css">
</head>
<body>
    <div class="container mt-5">
        <h1>Welcome to the Login Page</h1>
        <p><a class="btn btn-danger" href="/oauth2/authorization/google">google</a></p>
    </div>
<script src="https://code.jquery.com/jquery-3.3.1.slim.min.js"></script>
<script src="https://cdnjs.cloudflare.com/ajax/libs/popper.js/1.14.7/umd/popper.min.js"></script>
<script src="https://stackpath.bootstrapcdn.com/bootstrap/4.3.1/js/bootstrap.min.js"></script>
</body>
</html>
```

그외 HandlerMethodArgumentResolver 를 이용해서 Session 관리하는 코드도 개발했다. 이 부분은 아래 GIT 을 확인하는 것을 권장한다.

- [OAuth Github Repository](https://github.com/lkhlkh23/practice-oauth)

이 포스팅이 완료되면, 운동을 하러가야한다. 인바디를 체크했는데, 체지방을 봤다.

내 전체 몸에서 20%가 지방으로 이루어졌다. 17%를 목표로… 가야겠다..

![4](https://raw.githubusercontent.com/lkhlkh23/lkhlkh23.github.io/master/images/2024-02-03/4.png)

과연.. 나이들고 병든 육체로 운동과 공부를 병행할 수 있을까?!
