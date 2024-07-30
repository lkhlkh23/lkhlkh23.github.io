---
layout: post
title: Eureka Server 재시작 시, 서비스 등록 실패
subtitle: Eureka Client 버전 호환성 이슈 해결
excerpt_image: https://raw.githubusercontent.com/lkhlkh23/lkhlkh23.github.io/master/images/2024-07-30/banner.png
categories: spring
tags: [spring, eureka]
---
Eureka Server 재시작 시, Eureka Client 가 자동적으로 서비스 등록되지 않는 이슈가 발생했다. 모든 Eureka Client 가 서비스 등록에 실패하는 것이 아닌 특정 Eureka Client 대상으로만 발생했다.

오늘은 해당 이슈에 대한 원인과 해결방법에 대해 정리하려고 한다.

### Symptoms

이슈가 있는 서비스는 Eureka Client-2 이다. Eureka Server 재시작 시, Eureka Client-2 만 자동으로 서비스 등록이 되지 않는다.

- Eureka Server
  - SpringBoot : 3.2.1
  - SpringCloud : 2023.0.0
  - Java17
- Eureka Client-1 (coupon)
  - SpringBoot : 3.2.1
  - SpringCloud : 2023.0.0
  - Java17
- eureka client-2 (order) (이슈)
  - SpringBoot : 2.7.12
  - SpringCloud : 2021.0.3
  - Java17

그리고, 아래와 같은 오류가 발생한다.

```bash
2024-07-30 15:53:31.024  WARN 59292 --- [tbeatExecutor-0] c.n.d.s.t.d.RetryableEurekaHttpClient    : Request execution failed with message: Error while extracting response for type [class com.netflix.appinfo.InstanceInfo] and content type [application/json]; nested exception is org.springframework.http.converter.HttpMessageNotReadableException: JSON parse error: Root name ('timestamp') does not match expected ('instance') for type `com.netflix.appinfo.InstanceInfo`; nested exception is com.fasterxml.jackson.databind.exc.MismatchedInputException: Root name ('timestamp') does not match expected ('instance') for type `com.netflix.appinfo.InstanceInfo`
 at [Source: (org.springframework.util.StreamUtils$NonClosingInputStream); line: 1, column: 2] (through reference chain: com.netflix.appinfo.InstanceInfo["timestamp"])
2024-07-30 15:53:31.026 ERROR 59292 --- [tbeatExecutor-0] com.netflix.discovery.DiscoveryClient    : DiscoveryClient_ORDER/192.168.45.109:order:8082 - was unable to send heartbeat!
```

### Solutions

**원인은 Eureka Server 와 Eureka Client 의 버전 호환성으로 발생하는 이슈**이다. 해결책은 심플하다! **버전 호환성을 통일**하면 된다. 이렇게 간단한 이유라면 포스팅을 정리하지도 않았다.

현재 Eureka Client-2 는 QA 가 모두 완료되었기 때문에 버전을 변경하는 것은 현실적으로 어렵다. 그래서 Eureka Client 의 버전을 다운그레이드 하는 것은 어렵다. 제안하는 해결책은 2개이다.

**Solution-1. Eureka Server 버전 다운그레이드**

Eureka Server 를 아래와 같이 버전 다운그레이드 했다. 다운그레이드 이후에 Eureka Server 재시작을 해보니 정상적으로 모든 Eureka Client 가 등록되었다.

- SpringBoot ~~3.2.1~~ → 2.7.12
- SpringCloud : ~~2023.0.0~~ → 2021.0.3
- Java17

**Solution-2. ErrorController 구현**

Eureka Server 에 아래와 같이 Controller 를 추가한다면, 버전 변경없이 쉽게 해결할 수 있다.

```java
@RestController
public class EurekaErrorController implements ErrorController {

	@RequestMapping("/error")
	public ResponseEntity<Void> error() {
		return ResponseEntity.notFound().build();
	}
}
```

팀내 시니어 개발자분이 가이드 주신 방법이다. 감사합니다 :D

![banner.png](https://raw.githubusercontent.com/lkhlkh23/lkhlkh23.github.io/master/images/2024-07-30/banner.png)

내가 생각한 `Solution-1` 은 다운그레이드로 인한 사이드 이펙트가 발생할 수 있기 때문에 좋은 방법은 아니다. 그리고 업그레이드도 아닌 다운그레이드라니 …

### Cause

원인은 앞서 설명했듯이 **버전 호환성으로 인한 문제**이다. SpringBoot 2.x 와 SpringBoot 3.x 는 에러 응답 객체가 다르다.

- SpringBoot 2.x : Body 가 없는 응답
- SpringBoot 3.x : Body 가 있는 응답

그렇기 때문에 위 오류는 **에러 응답을 파싱하는 과정에서 발생**한 것이다. 내가 이 문제에서 가장 이해가 되지 않았던 부분은 Eureka Client 가 시작 시 Eureka Server 에 정상적으로 서비스 등록이 되는데, Eureka Server 가 재시작될 때는 왜?! 서비스 등록이 되지 않는지가 제일 이해가 되지 않았다.

그래서 Eureka Client 의 `DiscoveryClient` 클래스를 디비깅해봤다.

```java
private class HeartbeatThread implements Runnable {

        public void run() {
            if (renew()) {
                lastSuccessfulHeartbeatTimestamp = System.currentTimeMillis();
            }
        }
}
```

```java
boolean renew() {
        EurekaHttpResponse<InstanceInfo> httpResponse;
        try {
            httpResponse = eurekaTransport.registrationClient.sendHeartBeat(instanceInfo.getAppName(), instanceInfo.getId(), instanceInfo, null);
            logger.debug(PREFIX + "{} - Heartbeat status: {}", appPathIdentifier, httpResponse.getStatusCode());
            if (httpResponse.getStatusCode() == Status.NOT_FOUND.getStatusCode()) {
                REREGISTER_COUNTER.increment();
                logger.info(PREFIX + "{} - Re-registering apps/{}", appPathIdentifier, instanceInfo.getAppName());
                long timestamp = instanceInfo.setIsDirtyWithTime();
                boolean success = register();
                if (success) {
                    instanceInfo.unsetIsDirty(timestamp);
                }
                return success;
            }
            return httpResponse.getStatusCode() == Status.OK.getStatusCode();
        } catch (Throwable e) {
            logger.error(PREFIX + "{} - was unable to send heartbeat!", appPathIdentifier, e);
            return false;
        }
}
```

Eureka Client 는 주기적으로 HeartBeat 를 Eureka Server 에 전달한다. HeartBeat 전달하는 과정에서 404 오류가 발생을 하면, `register()` 메소드를 이용해서 Eureka Server 에 서비스 등록을 한다. 그런데 해결책을 적용하기 전에는 Eureka Server 로 부터 전달받은 응답을 파싱하는 과정에서 예외가 발생했기 때문에 `register()` 메소드를 호출하지 못해서 서비스 등록을 하지 못했다.

그래서 `Solution-2` 는 Eureka Server 에서 404 오류에 대해서는 Body 가 없는 응답값으로 커스텀해서 내려주는 방식으로 해결하고 있다. Body 가 없기 때문에 Eureka Client-2 에서는 응답을 정상적으로 파싱할 수 있게 되었고, 결국은 `register()` 메소드를 호출해서 서비스 등록을 할 수 있게 되었다.

`Solution-1` 은 Eureka Server 가 2.x 버전으로 다운그레이드 했기 때문에 Body 가 없는 형태로 응답을 하게 되었다. 그래서 Eureka Client-2 가 응답을 정상적으로 파싱할 수 있게 된 것이다.

결국, Eureka Client 가 시작 시 Eureka Server 에 정상적으로 서비스 등록이 되는데, Eureka Server 가 재시작될 때 서비스 등록이 되지 않았던 이유는 서비스 등록이 문제가 아니라, HeartBeat 처리하는 과정에서 오류가 발생했기 때문이었다.

### Conclusion

오류 덕분에 Eureka 에 대해 더 공부할 수 있게 되었다. 하지만 오류는 싫다. 긍정적인척을 했다. 순간 나의 이중성에 조금 놀랬다.

![0.png](https://raw.githubusercontent.com/lkhlkh23/lkhlkh23.github.io/master/images/2024-07-30/0.png)
