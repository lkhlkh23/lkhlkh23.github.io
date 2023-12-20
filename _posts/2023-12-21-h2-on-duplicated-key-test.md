---
layout: post
title: H2 ON DUPLICATED KEY 테스트
subtitle: MODE=MYSQL 로 DBMS 명시
excerpt_image: https://raw.githubusercontent.com/lkhlkh23/lkhlkh23.github.io/master/images/2023-12-21/banner.png
categories: jpa
tags: [jpa, h2]
---

![banner](https://raw.githubusercontent.com/lkhlkh23/lkhlkh23.github.io/master/images/2023-12-21/banner.png)


H2 환경에서 MYSQL ‘ON DUPLICATE KEY’ 쿼리를 테스트할 수 있을까?!

### 문제

MYSQL 에서 아래와 같은 문법을 사용하고 있다. H2 환경에서 테스트를 진행하려고 할 때 오류가 발생했다.

[오류 내용]

```sql
쿼리
```

### 원인

H2 는 데이터베이스마다 다르게 동작하고 있으며 기본적으로 ANSI SQL 표준을 지원한다. 하지만 옵션값에 따라 다른 데이터베이스의 호환을 지원한다.

> ANSI, American National Standards Institute(미국 표준 협회)
각기 다른 DBMS (Oracle, MySQL ..) 에서 공통적으로 사용할 수 있도록 고안한 표준 SQL문 작성방법
>

‘ON DUPLICATE KEY’ 문법은 MYSQL 에서만 지원하기 때문에 디폴트인 ANSI SQL 표준에서는 지원하지 않는다.

### 해결방법

**MODE=MySQL;** 추가 하면된다. 매우 간단하다.

[다음 문서](https://www.h2database.com/html/features.html#compatibility)를 보면 MYSQL 모드에서 지원되는 내용에 대해 상세하게 정리되있다.

```yaml
spring:
  datasource:
    driver-class-name: org.h2.Driver  
    url: jdbc:h2:~/test;MODE=MySQL;
    username: sa  
    password:  
```

### 결과

```java

```

[성공사진]

시간이 지날수록 실력이 늘어야하는데, 이제는 알고 있는 내용도 계속 잊어버리니…

뇌가 점점 쓰지 못할 것으로 변해가고 있다는 것이 너무 슬프다.
