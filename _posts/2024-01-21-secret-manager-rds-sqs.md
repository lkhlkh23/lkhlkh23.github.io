---
layout: post
title: Secret Manager, RDS, SQS 연동
subtitle: Secret Manager 로테이션 개발 및 검증
excerpt_image: https://raw.githubusercontent.com/lkhlkh23/lkhlkh23.github.io/master/images/2024-01-21/banner.png
categories: aws
tags: [aws, secret-manager, sqs, rds, spring]
---

![banner](https://raw.githubusercontent.com/lkhlkh23/lkhlkh23.github.io/master/images/2024-01-21/banner.png)

개인 프로젝트에서 DB 에 대한 정보를 Jasypt 이용해서 암호화를 했다. 
암호화를 했지만, 복호화가 너무 쉽기 때문에 보안 이슈가 있을 것이라고 생각해서 다른 방식을 고안하게되었다. 
물론, 이러한 이유로 Secrets Manager 를 고민하고, 공부한 것이 아니다. 회사 업무를 하면서 정리가 필요하다고 생각해서 정리를 진행하는 것이다!

### Secrets Manager 정의

DB, Application, API Key 및 기타 암호를 관리, 검색, 교체할 수 있는 서비스이다. 
Secrets Manager 를 사용하면 application.yaml 파일 또는 코드 내부에 하드코딩하지 않아도 된다. 
하드코딩된 보안 인증 정보를 Secrets Manager 서비스에 대한 런타임 호출로 대체하여 필요할 때 동적으로 보안 인증 정보를 검색한다.

그러므로, 보안 인증 정보에 대한 자동 교체 일정 때 서비스에 대해서 수정/배포 진행없이 바로 교체가 가능하다.

### Secrets Manager 동작 순서

![0](https://raw.githubusercontent.com/lkhlkh23/lkhlkh23.github.io/master/images/2024-01-21/0.png)

1. DBA 가 Secrets Manager 에 보안암호 (mbti/mysql) 를 생성한다.
2. 보안암호 (mbti/mysql) 를 생성하면 Key Management Service 이용해서 보안암호 텍스트가 생성된다.
3. APP 에서 DB 와 연결할 때, Secrets Manager 에서 보안암호 보안암호 (mbti/mysql) 쿼리한다.
4. Secrets Manager 는 IAM 자격증명이 있는지 확인한 후, 보안암호 텍스트를 복호화해서 APP 에 전달한다.
5. APP 은  Secrets Manager 를 통해 받은 보안암호 텍스트를 처리해서 DB 와 연결한다.

### Secrets Manager 생성

아래 이미지 순서대로 진행하면 보안암호 생성이 가능하다.

![1](https://raw.githubusercontent.com/lkhlkh23/lkhlkh23.github.io/master/images/2024-01-21/1.png)
![2](https://raw.githubusercontent.com/lkhlkh23/lkhlkh23.github.io/master/images/2024-01-21/2.png)
![3](https://raw.githubusercontent.com/lkhlkh23/lkhlkh23.github.io/master/images/2024-01-21/3.png)
![4](https://raw.githubusercontent.com/lkhlkh23/lkhlkh23.github.io/master/images/2024-01-21/4.png)
![5](https://raw.githubusercontent.com/lkhlkh23/lkhlkh23.github.io/master/images/2024-01-21/5.png)
### Secrets Manager 개발

Secrets Manager 에서 교체주기를 설정해서 RDS 암호를 4시간 마다 교체할 수 있도록 설정했다.

그리고 4시간마다 정상적으로 교체가 되어도 서비스에는 문제가 없는지 아래와 같은 구조로 검증하려고 한다.

![6](https://raw.githubusercontent.com/lkhlkh23/lkhlkh23.github.io/master/images/2024-01-21/6.png)

요약하면, 주기적으로 RDS 조회하고, SQS 에 정상적으로 메세지를 적재하는지를 통해서 서비스에 문제없이 정상적으로 동작하는지 확인하려고 한다.

우선 코드는 아래 Git 에 개발했다.

- [practice-secret-manager](https://github.com/lkhlkh23/practice-secret-manager)

```
  /* SQS */
	implementation group: 'org.springframework.cloud', name: 'spring-cloud-starter-aws', version: '2.2.1.RELEASE'
	implementation group: 'org.springframework.cloud', name: 'spring-cloud-aws-messaging', version: '2.2.3.RELEASE'

	/* Secrets Manager */
	implementation 'org.springframework.cloud:spring-cloud-starter-bootstrap:3.1.3'
	implementation 'org.springframework.cloud:spring-cloud-starter-aws-secrets-manager-config:2.2.6.RELEASE'
	implementation 'com.amazonaws.secretsmanager:aws-secretsmanager-jdbc:1.0.12'
```

Secrets Manager 와 SQS 에 필요한 dependency 는 위와 같이 진행하면 된다.

```java
@Configuration
@Slf4j
public class SecretManagerConfig {

	@Value("${cloud.aws.region.static}")
	private String region;

	@Bean("awsSecretsManager")
	@Primary
	public AWSSecretsManager awsSecretsManager() {
		final String profile = System.getProperty("spring.profiles.active");
		log.info("profile : {}", profile);

		if("local".equals(profile)) {
			final String accessKey = System.getProperty("aws.accessKeyId");
			final String secretKey = System.getProperty("aws.secretKey");

			return AWSSecretsManagerClientBuilder.standard()
												 .withRegion(region)
												 .withCredentials(new AWSStaticCredentialsProvider(new BasicAWSCredentials(accessKey, secretKey)))
												 .build();
		}

		final SystemPropertiesCredentialsProvider provider = new SystemPropertiesCredentialsProvider();
		log.info("access : {}", provider.getCredentials().getAWSAccessKeyId());
		log.info("secret : {}", provider.getCredentials().getAWSSecretKey());

		return AWSSecretsManagerClientBuilder.standard()
											 .withRegion(region)
											 .withCredentials(provider)
											 .build();
	}

}
```

Secrets Manager 에 대한 코드는 위와 같이 개발했다. access key 와 secret key 를 코드에 노출하고 싶지 않아서, vm option 으로 넣었다.

```yaml
spring:
  jpa:
    database-platform: org.hibernate.dialect.MySQL8Dialect
  datasource:
    type : com.zaxxer.hikari.HikariDataSource
    url: ENC(4CIfOizxicwmRRmoCP3LYgW3D7YBpCrSh97m475iI7iCrLv0vetQDPtBaLgo5BWdv1GYowOB/qOsHzLiSWwRVVDjkudzwV4K4Vexin8DlRc6D/6Vifbf6btvSgONtS8yVCvcSrCWwB+lmGOvzZfqwSXuNJOU5k5YfDizQySzcYZwujaGRg20DOaVR7DoQw/KAFlUMVhWu/fYr4gkM4SAIffvH+o4vwIa)
    driver-class-name: com.amazonaws.secretsmanager.sql.AWSSecretsManagerMySQLDriver
    username: ENC(xUnnmvH0sxzt57EgLfrVViS7HhLBQcbCW70i5gQT/bY=)
```

secrets manager 에 대한 설정은 위와 같이 개발했다. 
`username` 은 aws 에서 생성한 보안암호 이름을 넣으면 된다. 
나는 url 과 username 을 노출하고 싶지 않아서, `jasypt` 이용해서 암호화를 했다. 
또한, 암호화에 필요한 key 는 코드에 노출하고 싶지 않아서 vm option 으로 넣었다.

- as-is
  - username : db 계정 ID
  - password : db 계정 암호
- to-be
  - username : Secrets Manager 보안암호이름
  - password : 불필요

```java
@Slf4j
@Configuration
public class SqsConfig {

	@Value("${cloud.aws.region.static}")
	private String region;

	@Primary
	@Bean
	public AmazonSQSAsync amazonSQSAws() {
		final String profile = System.getProperty("spring.profiles.active");
		log.info("profile : {}", profile);

		if("local".equals(profile)) {
			final String accessKey = System.getProperty("aws.accessKeyId");
			final String secretKey = System.getProperty("aws.secretKey");

			return AmazonSQSAsyncClientBuilder.standard()
											  .withRegion(region)
											  .withCredentials(new AWSStaticCredentialsProvider(new BasicAWSCredentials(accessKey, secretKey)))
											  .build();
		}

		final SystemPropertiesCredentialsProvider provider = new SystemPropertiesCredentialsProvider();
		log.info("access : {}", provider.getCredentials().getAWSAccessKeyId());
		log.info("secret : {}", provider.getCredentials().getAWSSecretKey());

		return AmazonSQSAsyncClientBuilder.standard()
									  .withRegion(region)
									  .withCredentials(provider)
									  .build();
	}

}
```

```yaml
cloud:
  aws:
    region:
      static: ap-northeast-2
    stack:
      auto: false
    sqs:
      queue:
        name: practice
        url: https://sqs.ap-northeast-2.amazonaws.com/624849568220/practice
```

cloud.aws.sqs.url 에는 sqs queue 의 url 넣으면 된다. 암호화를 할걸 그랬다.. 깜빡했네..

```java
@Component
@RequiredArgsConstructor
public class SqsSender {

	@Value("${cloud.aws.sqs.queue.url}")
	private String url;

	private final AmazonSQS sqs;
	private final SurveyEntityRepository repository;

	public void sendWhenErrorOccurred() {
		try {
			final int count = repository.findAll().size();
		} catch (Exception e) {
			sqs.sendMessage(url, String.format("message : %s, time : %s", e.getMessage(), LocalDateTime.now()));
		}
	}

	public void sendAlways() {
		try {
			final int count = repository.findAll().size();
			sqs.sendMessage(url, String.format("count : %d, time : %s", count, LocalDateTime.now()));
		} catch (Exception e) {
			sqs.sendMessage(url, String.format("message : %s, time : %s", e.getMessage(), LocalDateTime.now()));
		}
	}

}
```

SQS 에 메세지를 보내는 것은 위와 같이 굉장히 간단하다.

### Secrets Manager 검증

자동 교체 주기를 4시간으로 설정했는데, EC2 에 배포를 하고 이미 시간이 많이 흘렀다.

교체를 할 때, 문제가 있다면 SQS 메세지로 예외 메세지가 적재되야 한다. SQS 메세지를 봤을 때, 없다!

성공적이다!

이번주부터 운동을 하려고 아울렛에서 운동복과 신발을 샀다. 신발끈을 잘 묶지 못하는 나에게 반스 신발은 정말 최적화된 상품이었다. 
시작이 반이라고 했다.운동복을 샀으니 반을 한것이다. 한번더 운동복을 사면 이제 다 한거나 마찬가지겠지?!
