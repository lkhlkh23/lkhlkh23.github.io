---
layout: post
title: SpringBoot 3.x 로 왜 전환해야 하는가?!
subtitle: 하이리스크-로우리턴인데, SpringBoot 3.x 로 왜 전환해야해?!
excerpt_image: https://raw.githubusercontent.com/lkhlkh23/lkhlkh23.github.io/master/images/2024-10-27/banner.png
categories: springboot
tags: [springboot]
---
최근에 Sonarqube 를 도입하는 과정에서 SpringBoot 2.x 버전으로 사용하는 프로젝트를 다수 확인했다. 서비스의 시작 시점을 고려하면, SpringBoot 2.x 버전으로 개발한것에 대해서는 충분히 이해가 되었다. 하지만, 올해 생성한 프로젝트마저 SpringBoot 2.x 버전으로 개발되었다는 사실은 조금 혼란스러웠다.

그래서, 평가시즌이 끝나서 추가 개발이 빈번하지 않는 지금이 SpringBoot 3.x 버전으로 전환할 수 있는 가장 좋은 타이밍라고 생각한다. 그 누구도 시키지 않았지만 진행하려고 한다.

나는 SpringBoot 3.x 전환을 `하이리스크-로우리턴` 이라고 생각한다. 고려사항과 작업내역 그리고 이슈가 발생할 수 있는 가능성도 많다. 하지만, 전환이 완료됬을 때, 정량적으로도 가시적으로도 향상된 부분을 바로 체감할 수 없다. 그렇기 때문에, 전환이 이슈없이 성공하면 본전, 실패하면 인민재판이다.

![0.png](https://raw.githubusercontent.com/lkhlkh23/lkhlkh23.github.io/master/images/2024-10-27/0.png)

그래서, SpringBoot 3.x 로 전환의 정당성을 부여하기 위해 SpringBoot 3.x 로 전환해야하는 이유를 정리해보려고 한다.

### **최신 Java 표준 호환성**

Spring Boot 3.x 는 최소 Java 17+ 을 기본으로 지원한다. 반면, Spring Boot 2.x 는 Java 17+ 의 버전을 지원하지 않기 때문에 Java 17+ 가 제공하는 기능을 활용할 수 없는 한계가 있다.

또한, Spring Boot 3.x 이상에서는 Jakarta EE 9+ 를 채택하여 `javax.*` 대신 `jakarta.*` 네임스페이스로 완전히 전환되었다. 이는 Oracle 과의 상표권 이슈로 인해 `javax` 네임스페이스를 사용할 수 없게 되었기 때문이다. 따라서 `jakarta`는 `javax`의 최신 버전으로 이해하면 된다. 현재 `javax`는 더 이상 업데이트되지 않기 때문에 `jakarta`로의 전환이 필요하다.

### 최신 보안 표준 지원 - Spring Security 6

Spring Security 6 는 Java 17+ 에서 동작하도록 설계되었다. Spring Security 6 에서는 보안 취약점이 발견될 가능성이 있는 API 를 제거하고, 최신 보안 표준을 준수하는 API 만 유지하게 되었다.

EX) WebSecurityConfigurerAdapter 제거되고, Lambda 기반의 방식으로 변경

SprintBoot 2.x - WebSecurityConfigurerAdapter

```java
@EnableWebSecurity
public class SecurityConfig extends WebSecurityConfigurerAdapter {
    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http
            .authorizeRequests()
            .anyRequest().authenticated();
    }
}
```

SpringBoot 3.x - Lambda 방식

```java
@EnableWebSecurity
public class SecurityConfig {
    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests(auth -> auth
                .anyRequest().authenticated()
            );
        return http.build();
    }
}
```

또한, 보안 필터 체인 설정하는 방식이 더 간단하고, 유연하게 변경되었다. 그리고, CSRF 와 HTTP 헤더 보안, 인증 엔드포인트 보호 등 여러 보안 설정에서 기본값이 더 강화되었다. 이전에는 개발자가 보안 설정을 직접 구성해야 했던 부분들이 기본적으로 더 안전하게 설정되어 있어 보안성을 향상시켰다.

### 성능 향상 - GraalVM 네이티브 이미지

SpringBoot 2.x 에서는 `JIT (Just-In-Time) 컴파일러` 를 사용해 실행 중에 필요한 코드를 컴파일하고 최적화했다. 그러나, SpringBoot 3.x 에서는 `AOT 컴파일` 을 사용해 미리 코드 전체를 바이너리 파일로 컴파일한다. AOT 컴파일 방식 덕분에, 실행 시점에 컴파일 작업이 필요하지 않기 때문에 실행 시간이 빨라지고 메모리 사용이 최적화된다.

또한, GraalVM은 미리 컴파일하면서 불필요한 부분들을 걸러내는데, 이 과정을 Dead Code Elimination이라고고 한다. 예를 들어, 사용되지 않는 클래스나 메서드가 있다면 컴파일 과정에서 포함되지 않아 전체적인 애플리케이션 크기가 줄어들고, 필요 없는 코드가 실행되지 않기 때문에 성능이 최적화된다.

### 결론

SpringBoot 2.x 에서는 더이상 업데이트가 되지 않는 서비스가 존재하기 때문에 계속해서 애플리케이션을 운영해야한다면, SpringBoot 3.x 로 전환이 반드시 필요하다! 최신 보안 표준 및 성능 개선과 함께 장기적인 유지보수가 필요하다면 변경하자!

아직도 SpringBoot 3.x 전환이 고민된다면…?! 내가 만약 3년이내 퇴사가 예정되있다면, 그냥 SpringBoot 2.x 를 계속 사용해라!

### References

- [SpringBoot 3.x 전환 가이드](https://github.com/spring-projects/spring-boot/wiki/Spring-Boot-3.0-Migration-Guide)
- [SpringBoot 3.x 전환 필요 이유](https://velog.io/@ililil9482/Spring-3.x-Security-%EC%84%A4%EC%A0%95)
- [GraalVM](https://mangkyu.tistory.com/302)
- [Java 컴파일 동작원리](https://gyoogle.dev/blog/computer-language/Java/%EC%BB%B4%ED%8C%8C%EC%9D%BC%20%EA%B3%BC%EC%A0%95.html)
- [컴파일러 vs 인터프리터](https://hyeinisfree.tistory.com/26)
