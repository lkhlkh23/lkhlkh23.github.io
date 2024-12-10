---
layout: post
title: 간단한 Health Check Down 이슈 수정
subtitle: Health Check Down 이슈 수정
excerpt_image: https://raw.githubusercontent.com/lkhlkh23/lkhlkh23.github.io/master/images/2024-12-10/banner.png
categories: spring
tags: [springboot, java, actuator]
---

**“ADMIN이 멈췄다!”**

![0.png](https://raw.githubusercontent.com/lkhlkh23/lkhlkh23.github.io/master/images/2024-12-10/0.png)

아침부터 어수선했던 사무실이 갑작스러운 외침으로 더 깊은 혼란에 빠졌다. 고과 면담으로 꽉 찬 일정에 지친 표정들이 순식간에 긴장감으로 굳어졌다. 모두의 시선이 한 곳으로 모였다.

**“왜 안 되는 거죠?!”**

급박한 목소리와 함께 키보드 타이핑 소리가 사무실을 가득 메웠다. 화면 속 에러 메시지는 날카로운 단서처럼 눈길을 끌었다. 긴장이 팽팽히 감돌았다. 문제를 해결해야 했다. 그리고 그것도 빨리!

오늘 점심 메뉴는 1년에 몇 번 나오지 않는 특식. 문제를 늦게 해결하면 다 식어버린 음식뿐이다. 특식은 반드시 따뜻하게 먹어야 한다는 사명감. 그 사명감이 손가락 끝으로 전해지며, 속도를 더했다.

### 사건 접수

QAS 환경에서 ADMIN 이 정상적으로 동작하지 않는 현상을 접수했다. 인증 API  로부터 500 INTERNAL SERVER ERROR 를 응답받고 있었다. 그러나, 인증 API 에는 로그가 발생하지 않았다.

### 원인 분석

인증 API 가 Eureka 에 등록이 되지 않는 것을 확인했고, Health Check 결과도 DOWN 상태였다. 그렇기 때문에 인증 API 에 요청이 유입되지 않았기 때문에 로그도 발생하지 않고, 500 ERROR 만 발생한 것으로 보였다. Health Check 의 결과를 명확하게 확인하기 위해 아래와 같은 옵션을 추가했다.

```yaml
management:
	endpoint:
	  health:
	    show-details: always
```

결과는 아래와 같았다.

```json
{
  "status": "DOWN",
  "components": {
    "db": {
      "status": "UP",
      "details": {
        "database": "MySQL",
        "validationQuery": "isValid()"
      }
    },
    "discoveryComposite": {
      "status": "UP",
      "components": {
        "discoveryClient": {
          "status": "UP",
          "details": {
            "services": [
              ... ...
            ]
          }
        },
        "eureka": {
          "description": "Eureka discovery client has not yet successfully connected to a Eureka server",
          "status": "UP",
          "details": {
            "applications": {
              ... ...
            }
          }
        }
      }
    },
    "diskSpace": {
      "status": "UP",
      "details": {
        "total": 53619982336,
        "free": 52759257088,
        "threshold": 10485760,
        "path": "... ...",
        "exists": true
      }
    },
    "ping": {
      "status": "UP"
    },
    "redis": {
      "status": "DOWN",
      "details": {
        "error": "org.springframework.data.redis.RedisConnectionFailureException: Unable to connect to Redis"
      }
    },
    "refreshScope": {
      "status": "UP"
    }
  }
}
```

Health Check 결과 REDIS 를 연결하는 과정에서 예외가 발생했고, 결국은 Health Check 결과가 DOWN 으로 응답됬다. 문제는 REDIS 연결이었다. 

추가적으로 Health Check 는 아래와 같은 상황에서 DOWN 이 발생할 수 있다.

- 데이터베이스가 DOWN 되었거나, 네트워크 이슈로 접근 불가한 상황
- 디스크 공간이 부족한 상황
- 캐시 서비스가 DOWN 되었거나,응답하지 않는 상황
- 큐의 사용량이 초과한 상황

### 해결

build.gradle 에서 아래 의존성을 제거했다.

```groovy
implementation 'org.springframework.boot:spring-boot-starter-data-redis'
```

위와 같은 의존성이 설정이 되있다면 REDIS 에 연결을 시도한다. 그리고 연결을 시도하는 REDIS 에 대한 설정이 존재하지 않는다면 디폴트값을 이용해서 연결을 시도한다.

- host : localhost
- port : 6379

의존성을 제거하고 다시 Health Check 를 수행해보니, 아래와 같이 UP 상태를 확인할 수 있었다.

```json
{
  "status": "UP",
  "components": {
    "db": {
      "status": "UP",
      "details": {
        "database": "MySQL",
        "validationQuery": "isValid()"
      }
    },
    "discoveryComposite": {
      "status": "UP",
      "components": {
        "discoveryClient": {
          "status": "UP",
          "details": {
            "services": [
              ... ...
            ]
          }
        },
        "eureka": {
          "description": "Remote status from Eureka server",
          "status": "UP",
          "details": {
            "applications": {
              ... ...
            }
          }
        }
      }
    },
    "diskSpace": {
      "status": "UP",
      "details": {
        "total": 53619982336,
        "free": 52766064640,
        "threshold": 10485760,
        "path": "... ...",
        "exists": true
      }
    },
    "ping": {
      "status": "UP"
    },
    "refreshScope": {
      "status": "UP"
    }
  }
}
```

생각보다 매우 간단한 이슈였다. Health Check 의 응답만 봐도 쉽게 해결할 수 있는 이슈였다.

### 하지만…

Health Check 는 DOWN. 나는 Calm DOWN.

가장 큰 이슈는 Health Check DOWN 이 아니다. 내가 이제 DOWN 되었다. 나의 모든 서비스는 이제 DOWN 이다. 약속의 날이 더 빨리 올것 같은 느낌이 들었다.
