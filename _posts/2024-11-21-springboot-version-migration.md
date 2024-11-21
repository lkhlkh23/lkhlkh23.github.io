---
layout: post
title: SpringBoot 3.x 마이그레이션
subtitle: SpringBoot 3.x 마이그레이션을 하면서...
excerpt_image: https://raw.githubusercontent.com/lkhlkh23/lkhlkh23.github.io/master/images/2024-11-21/banner.png
categories: springboot
tags: [springboot, migration]
---
SpringBoot 3.x 마이그레이션 작업을 진행하고 있다. [이전 포스팅](https://lkhlkh23.github.io/springboot/2024/10/27/why-upgrade-springboot3.html)에서 3년이내 회사를 퇴사하지 않을 예정이라면 마이그레이션을 권장한다고 글을 작성했다. 그럼 난 마이그레이션 작업을 해야하는것일까?!

![0.png](https://raw.githubusercontent.com/lkhlkh23/lkhlkh23.github.io/master/images/2024-11-21/0.png)

그래도 시작한 마이그레이션 작업은 마저 끝내야하지 않는가?! 그래서 여러 포스팅과 공식문서를 참고해서 작업한 내용을 정리했다.

### 목표

- JDK 버전 업그레이드
    - jdk 11 → jdk 21
- SpringBoot 버전 업그레이드
    - 2.7.x → 3.2.x

### 작업 내역

**Step.1 JDK 버전 + SpringBoot 버전 변경**

SpringBoot 버전이 낮을 경우, 한번에 3.x 버전으로 변경하기보다는 점차적으로 변경하는 것을 권장한다. 하지만, 나는 위험을 감수하고 한번에 변경을 했다. 현재 검증 프로세스 진행 상황으로 보면, 해당 작업의 완료 시점이 보이지 않는다. 

- [1.5 → 2.0 마이그레이션 공식문서](https://github.com/spring-projects/spring-boot/wiki/Spring-Boot-2.0-Migration-Guide)
- [2.4 + config data 마이그레이션 공식문서](https://github.com/spring-projects/spring-boot/wiki/Spring-Boot-Config-Data-Migration-Guide)
- [2.7 → 3.x 마이그레이션 공식 문서](https://github.com/spring-projects/spring-boot/wiki/Spring-Boot-3.0-Migration-Guide)

SpringBoot 버전에 호환되는 SpringCloud 버전으로 변경이 필요하다. 

![1.png](https://raw.githubusercontent.com/lkhlkh23/lkhlkh23.github.io/master/images/2024-11-21/1.png)

- [SpringCloud 버전 호환성 공식문서](https://spring.io/projects/spring-cloud)

**as-is**

```groovy
plugins {
    id 'java'
    id 'org.springframework.boot' version '2.7.14'
    id 'io.spring.dependency-management' version '1.0.15.RELEASE'
    id "com.ewerk.gradle.plugins.querydsl" version "1.0.10"
}
 
sourceCompatibility = '11'
 
ext {
    set('springCloudVersion', "2021.0.8")
}
```

**to-be**

```groovy
plugins {
    id 'java'
    id 'org.springframework.boot' version '3.2.5'
    id 'io.spring.dependency-management' version '1.1.4'
}  
 
sourceCompatibility = '21'
targetCompatibility = '21'  
 
ext {
    set('springCloudVersion', "2023.0.0")
}
```

**Step.2 jakatra 변경**

intellij 에서 제공하는 기능을 이용해서 javax → jakatra 로 한번에 변경이 가능하다.

![2.png](https://raw.githubusercontent.com/lkhlkh23/lkhlkh23.github.io/master/images/2024-11-21/2.png)

**의존성 변경 내역**

- javax.validation.constraints.NotBlank → jakarta.validation.constraints.NotBlank
- javax.validation.constraints.NotNull → jakarta.validation.constraints.NotNull
- javax.validation.constraints.Size → jakarta.validation.constraints.Size
- javax.validation.constraints.PositiveOrZero → jakarta.validation.constraints.PositiveOrZero
- javax.validation.constraints.Email → jakarta.validation.constraints.Email
- javax.servlet.http.HttpServletRequest → jakarta.servlet.http.HttpServletRequest
- javax.servlet.http.HttpServletResponse → jakarta.servlet.http.HttpServletResponse
- javax.validation.ConstraintViolationException → jakarta.validation.ConstraintViolationException
- javax.transaction.Transactional → jakarta.transaction.Transactional
- javax.persistence.Column → jakarta.persistence.Column
- javax.persistence.Entity → jakarta.persistence.Entity
- javax.persistence.IdClass → jakarta.persistence.IdClass
- javax.persistence.GeneratedValue → jakarta.persistence.GeneratedValue
- javax.persistence.GenerationType → jakarta.persistence.GenerationType
- javax.persistence.Id → jakarta.persistence.Id
- javax.persistence.SequenceGenerator → jakarta.persistence.SequenceGenerator
- javax.persistence.Table → jakarta.persistence.Table
- javax.servlet.FilterChain → jakarta.servlet.FilterChain
- javax.servlet.ServletException → jakarta.servlet.ServletException
- javax.servlet.ServletContextEvent → jakarta.servlet.ServletContextEvent
- javax.servlet.ServletContextListener → jakarta.servlet.ServletContextListener
- javax.servlet.annotation.WebListener → jakarta.servlet.annotation.WebListener
- javax.persistence.EntityManager → jakarta.persistence.EntityManager
- javax.annotation.PostConstruct → jakarta.annotation.PostConstruct
- javax.persistence.AttributeConverter → jakarta.persistence.AttributeConverter
- javax.persistence.Converter → jakarta.persistence.Converter
- javax.annotation.PreDestroy → jakarta.annotation.PreDestroy

**Step.3 config 설정 변경 (yaml or properties)**

SpringBoot 버전이 변경됨에 따라 config 에 대한 설정도 변화가 있다. 아래 문서를 통해 프로젝트 내부 config 설정 확인이 필요하다. 내가 작업한 프로젝트는 2.7.x → 3.2.x 로 변경했기 때문에 아래 문서를 통해 설정의 변경을 확인했다. 

springboot 의 버전을 점진적으로 변경하라는 이유를 이때 공감했다!

- [springboot 2.7.6 ~ 3.0.0 공식문서](https://github.com/spring-projects/spring-boot/wiki/Spring-Boot-3.0-Configuration-Changelog)
- [springboot 3.0.7 ~ 3.1.0 공식문서](https://github.com/spring-projects/spring-boot/wiki/Spring-Boot-3.1-Configuration-Changelog)
- [springboot 3.1.6 ~ 3.2.0 공식문서](https://github.com/spring-projects/spring-boot/wiki/Spring-Boot-3.2-Configuration-Changelog)

**as-is**

```yaml
management:
  metrics:
    export:
      influx:
        enabled: false
```

**to-be**

```yaml
management:
  influx:
    metrics:
      export:
        enabled: false
```

**Step.4 Querydsl 변경**

**as-is**

```groovy
dependencies {
    implementation 'com.querydsl:querydsl-jpa:5.0.0'
    implementation 'com.querydsl:querydsl-apt:5.0.0'
    implementation 'com.querydsl:querydsl-core:5.0.0'
 
    implementation 'org.springframework.boot:spring-boot-starter-validation'
  ...
}
```

**to-be**

```groovy
dependencies {   
    implementation 'com.querydsl:querydsl-jpa:5.0.0:jakarta'
    annotationProcessor 'com.querydsl:querydsl-apt:5.0.0:jakarta'
    annotationProcessor 'jakarta.persistence:jakarta.persistence-api'
    implementation 'com.querydsl:querydsl-core'
    implementation 'com.querydsl:querydsl-collections'
    annotationProcessor 'jakarta.annotation:jakarta.annotation-api'
}
```

**Step.5 Hibernate 변경**

Hibernate 6.x 부터는 Jakatra Persistence API 3.x 를 사용한다. 

- [Hibernate 5.x → 6.x 공식문서](https://github.com/quarkusio/quarkus/wiki/Migration-Guide-3.0:-Hibernate-ORM-5-to-6-migration)

<aside>
💡

What is Hibernate

Hibernate는 Java 기반의 객체-관계 매핑 (ORM) 도구로, 객체 지향 도메인 모델을 관계형 데이터베이스에 매핑하는 기능을 제공한다. JDBC를 통해 관계형 데이터베이스와 상호 작용한다.

</aside>

Hibernate 6.x 부터 @Type, @TypeDef 를 미지원하기 때문에 AttributeConverter 로 변경이 필요하다. 현재 운영중인 서비스에서는 Postgresql 에서 배열 타입의 컬럼을 사용하고 있는데, AttributeConverter 를 생성해서 대체했다.

```java
@Converter
public class StringToStringArrayConverter implements AttributeConverter<String[], String> {
 
    @Override
    public String convertToDatabaseColumn(final String[] attribute) {
        if (attribute == null) {
            return null;
        }
        return String.join(",", attribute);
    }
 
    @Override
    public String[] convertToEntityAttribute(final String dbData) {
        if (dbData == null) {
            return null;
        }
        return dbData.split(",");
    }
 
}
```

**Step.6 SpringSecurity 변경**

SpringSecurity 5.7.0 이후부터 `WebSecurityConfigurerAdapter` 가 deprecated 가 되었고, 컴포넌트 기반으로 보안 설정을 하도록 변경되었다. 

현재 내가 작업하고 있는 프로젝트의 SpringSecurity 는 5.7.1 버전이기 때문에 `WebSecurityConfigurerAdapter` 는 사용하고 있지는 않지만, SpringSecurity 6.2.4 버전으로 마이그레이션을 하는 과정에서는 추가적인 수정이 필요하다.

1.  `@EnableSecurity` 에서 @Configuration 제거
→ SpringSecurity 설정을 정의하는 클래스에서 별도로 Bean 등록이 필요
2. `antMatchers`, `mvcMatchers`,  `regexMatchers` 가 제거되고, `requestMatchers` 로 변경
3. 람다 기반의 방식으로 변경

**as-is**

```java
@EnableWebSecurity
@RequiredArgsConstructor
public class SecurityConfigure {
	...
	
	@Bean
    protected SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        return http
                .httpBasic().disable()
                .csrf().disable()
                .cors()
                .and()
                .authorizeRequests()
                .antMatchers("/admin/docs/**", "/actuator/**").permitAll()
                .anyRequest().authenticated()
                .and()
                .exceptionHandling()
                .authenticationEntryPoint(jwtAuthenticationEntryPoint)
                .accessDeniedHandler(jwtAccessDeniedHandler)
 
                .and()
                .addFilterBefore(new JwtFilter(authClientService), UsernamePasswordAuthenticationFilter.class)
                .build();
    }
    
    ...
}
```

**to-be**

```java
@EnableWebSecurity
@RequiredArgsConstructor
@Configuration
public class SecurityConfigure {
    ...
 
    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .csrf(AbstractHttpConfigurer::disable)
            .cors(AbstractHttpConfigurer::disable)
            .httpBasic(AbstractHttpConfigurer::disable)
            .authorizeHttpRequests(authorize -> authorize
                .requestMatchers("/admin/docs/**", "/actuator/**").permitAll()
                .anyRequest().authenticated()
            )
            .exceptionHandling(exception -> exception
                .authenticationEntryPoint(jwtAuthenticationEntryPoint)
                .accessDeniedHandler(jwtAccessDeniedHandler)
            )
            .addFilterBefore(new JwtFilter(authClientService), UsernamePasswordAuthenticationFilter.class);
 
        return http.build();
    }
     
    ...
}
```

- [SpringSecurity 마이그레이션 공식문서](https://docs.spring.io/spring-security/reference/5.8/migration/servlet/config.html)

**Step.7 SpringCloud Sleuth**

SpringCloud Sleuth 는 분산추적을 위한 SpringBoot 자동 구성을 제공한다. SpringBoot 3.x 이상 버전부터는 SpringCloud Sleuth 를 지원하지 않는다. SpringCloud Sleuth 를 지원하는 마지막 버전은 2.x 이다.

현재 작업하고 있는 프로젝트에서는 Sleuth 를 사용하지 않기 때문에 추가적으로 작업할 내용은 없었다.  

- [SpringCloud Sleuth 공식문서](https://docs.spring.io/spring-cloud-sleuth/docs/current/reference/html/)

### 검증 내역

프로젝트의 모든 API 를 테스트하는것은 쉽지가 않다. 그래서 작업 내용에 영향을 받는 API 들만 별도로 샘플링해서 개발자 수준의 테스트를 진행했다. 만약, 테스트 코드가 있다면 테스트 코드를 실행시키는것만으로 검증할 수 있다. 하지만, 이 프로젝트는 그렇게 호락호락하지 않았다.

현재는 검증중이고, 12월에 배포할 예정이다. 끝났다! 하지만…

![3.png](https://raw.githubusercontent.com/lkhlkh23/lkhlkh23.github.io/master/images/2024-11-21/3.png)

고통은 끝나지 않았다. 하지만, 그 고통 이제 나눌 것이다.
