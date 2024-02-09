---
layout: post
title: Rest API 로 Google Login 구현
subtitle: Rest API 로 Google Login 구현 (springboot)
excerpt_image: https://raw.githubusercontent.com/lkhlkh23/lkhlkh23.github.io/master/images/2024-02-09/banner.png
categories: spring
tags: [spring, google-login, oauth2.0]
---

이전에는 하나의 프로젝트에서 OAuth 에 대한 FrontEnd 와 BackEnd 를 모두 구현했다. 이번에는 F/E 와 B/E 를 각각 다른 프로젝트에서 호출하는 방식으로 구현하려고 한다. 아래는 각각의 Git Repository 이다.

- [OAuth-FrontEnd](https://github.com/lkhlkh23/practice-oauth-client)
- [OAuth-BackEnd](https://github.com/lkhlkh23/practice-oauth-server)

OAuth 2.0 은 아래와 같은 순서로 동작한다. 이제 개발 코드를 통해 아래 순서에 맞게 설명해보겠다.

![0.png](https://raw.githubusercontent.com/lkhlkh23/lkhlkh23.github.io/master/images/2024-02-09/0.png)

### Step1. 서비스 등록

![1.png](https://raw.githubusercontent.com/lkhlkh23/lkhlkh23.github.io/master/images/2024-02-09/1.png)

Google 에서 서비스 등록을 완료하면 위와 같은 정보를 얻을 수 있다.

- client id
- client secret
- redirect url

그리고, 위 정보들을 바탕으로 아래와 같은 component 를 구성했다.

```java
@Component
public class GoogleConfig {

	@Value("${spring.security.oauth2.client.registration.google.client-id}")
	private String clientId;

	@Value("${spring.security.oauth2.client.registration.google.client-secret}")
	private String clientSecret;

	@Value("${spring.security.oauth2.client.registration.google.redirect-url}")
	private String redirectUrl;

	@Value("${spring.security.oauth2.client.registration.google.scope}")
	private String scope;

	@Value("${spring.security.oauth2.client.registration.google.login-url}")
	private String loginUrl;

	public String getUrl() {
		final Map<String, Object> parameters = new HashMap<>() {{
			put("client_id", clientId);
			put("redirect_uri", redirectUrl);
			put("response_type", "code");
			put("scope", scope.replaceAll(",", "%20"));
		}};

		final String query = parameters.entrySet()
																   .stream()
																   .map(param -> param.getKey() + "=" + param.getValue())
														       .collect(Collectors.joining("&"));

		return String.format("%s/o/oauth2/v2/auth?%s", loginUrl, query);
	}

	public GoogleLoginAuthorizationRequest of(final String authCode) {
		return GoogleLoginAuthorizationRequest.builder()
											  .clientId(clientId)
											  .clientSecret(clientSecret)
											  .code(authCode)
											  .redirectUri(redirectUrl)
											  .grantType("authorization_code")
											  .build();
	}

}
```

여기서 `getUrl()` 메소드는 그림에서 2-1 에 해당하는 URL 이다. Resource Owner 가 Google Login 으로 이동하기 위한 URL 이다.

### Step2. Resource Owner 승인

아래 코드는 OAuth-FrontEnd 에서 로그인 버튼이 전시되는 화면이다. 로그인 버튼에는 client-id, scope, redirect-url 이 포함된 URL 이 아닌, OAuth-BackEnd 의 API 를 호출한다. 이유는 client-id 를 포함한 다른 정보를 화면에 노출하고 싶지 않기 때문이다.

```html
<body>
    <div class="container mt-5">
        <h1>Welcome to the Login Page</h1>
        <p>
					<a class="btn btn-danger" href="http://localhost:8080/login">
						google
					</a>
				</p>
    </div>
</body>
```

아래 코드는 OAuth-FrontEnd 에서 `/login` API 호출하면, `GoogleConfig.getUrl()` 메소드에서 만든 URL 로 리디렉션 처리를 한다. 리디렉션 처리가 되면 로그인 및 scope 에 대한 동의 화면이 나올 것이다.

```java
@RestController
@RequiredArgsConstructor
@Slf4j
public class LoginController {

	private final GoogleConfig googleConfig;
	private final GoogleLoginService googleLoginService;

	@GetMapping("/login")
	public ResponseEntity moveGoogleLogin() throws URISyntaxException {
		final HttpHeaders httpHeaders = new HttpHeaders();
		httpHeaders.setLocation(new URI(googleConfig.getUrl()));

		return new ResponseEntity<>(httpHeaders, HttpStatus.SEE_OTHER);
	}
	
}
```

### Step3. Authorization Server 승인

로그인 및 scope 에 대한 동의가 완료되면, Authorization Server 는 authCode 를 생성한다. 그리고, Resource Owner 가 생성된 authCode 를 Client 에게 리디렉션하도록 처리한다.

그러면 authCode 와 함께 리디렉션되는 URL 은 무엇일까?!

`GoogleConfig.getUrl()` 메소드에서 `redirect-url` 로 정의한 URL 이 authCode 와 함께 리디렉션되는 URL 이다.

```java
@RestController
@RequiredArgsConstructor
@Slf4j
public class LoginController {

	private final GoogleConfig googleConfig;
	private final GoogleLoginService googleLoginService;

	@GetMapping(value = "/login/oauth2/code/google")
	public ResponseEntity googleLoginCallback(@RequestParam(value = "code", required = false, defaultValue = "") final String authCode) throws
		URISyntaxException {
		final String email = googleLoginService.getSocialEmail(authCode);
		final HttpHeaders headers = new HttpHeaders();
		if(!googleLoginService.isRegistered(email)) {
			headers.setLocation(new URI("http://localhost:8081/view/signup?email=" + email));
			return new ResponseEntity<>(headers, HttpStatus.FOUND);
		}

		headers.setLocation(new URI("http://localhost:8081/view/home?email=" + email));
		return new ResponseEntity<>(headers, HttpStatus.FOUND);
	}

}
```

### Step4 + 5. Token 발급 & API 호출

Authorization Server 에서 승인이 완료되면, 이제는 Token 을 발급해야 한다.

`https://oauth2.googleapis.com/token` 을 아래 파라미터와 함께 POST 로 호출한다.

- client id
- client secret
- auth code
- redirect url
- grant type (authorization_code)

Authorization Server 를 통해 Token 까지 발급이 완료되면, 발급된 Token 이용해서 Resource Owner 에 대한 정보 (scope 범위 내에서) 를 얻을 수 있다.

```java
@FeignClient(name = "google", url = "${spring.security.oauth2.client.registration.google.auth-url}")
public interface GoogleClient {

	@PostMapping(value = "/token", consumes = MediaType.APPLICATION_JSON_VALUE, produces = MediaType.APPLICATION_JSON_VALUE)
	GoogleLoginAuthorizationV1 getLoginAuthorization(@RequestBody final GoogleLoginAuthorizationRequest request);

	@GetMapping(value = "/tokeninfo")
	GoogleLoginV1 getLoginToken(@RequestParam(name = "id_token") final String token);

}
```

그외 다양한 설정들이 필요할 것이다. 그것은 위에 명시된 레파지토리를 통해 확인하면 될 것 같다.

명절이다. 그러나 나는 아프다. 그것도 혼자 아프다.

그래도 아프다는 이야기를 듣고, 문앞에 간식을 배달해줘서 감동받았다. 외롭지만 따뜻한 명절이다.

![2.png](https://raw.githubusercontent.com/lkhlkh23/lkhlkh23.github.io/master/images/2024-02-09/2.png)

다음에는 Nexus 에 JWT 를 생성하는 라이브러리를 publish 하는 코드를 포스팅해야겠다.
