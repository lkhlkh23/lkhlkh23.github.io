---
layout: post
title: Jsoup SocketTimeoutException 에 대해서
subtitle: 해결되지 않는 SocketTimeoutException 에 대해서 나의 시도들
excerpt_image: https://raw.githubusercontent.com/lkhlkh23/lkhlkh23.github.io/master/images/2024-01-02/banner.png
categories: java
tags: [java, jsoup, crawling, socketTimeoutException]
---

![banner](https://raw.githubusercontent.com/lkhlkh23/lkhlkh23.github.io/master/images/2024-01-02/banner.png)

최근 VIP 에게 쇼핑몰 상품 이미지를 자동으로 다운받을 수 있는 기능을 개발해달라는 요청을 받았다.

Java **Jsoup** 라이브러리를 사용하면 손쉽게 크롤링할 수 있다고 생각해서 진행을 했다.
물론, 로컬 환경까지 너무나도 쉽게 크롤링이 완료되었고, 마침내 AWS 에 서비스를 배포했다.

하지만, 'Z 쇼핑몰' 과 'C 쇼핑몰' 크롤링은 정상적으로 수행되었지만, 'B 쇼핑몰' 은 실패했다.

나의 불행의 시작은 지금부터 시작되었다.

### Jsoup 소개

[Jsoup 라이브러리 소개](https://jsoup.org/) 링크로 들어가면 Jsoup 에 대한 소개 및 사용법에 대해 자세하게 나와있다.

참고하면 될 것 같다. 그럼에도 불구하고 한줄로 요약하자면 아래와 같다.

<aside>
💡 Java 로 HTML 파싱하여 특정 정보를 조회하고, 조작할 수 있는 강력한 기능을 제공하는 라이브러리

</aside>

### 발생 오류

다른 'Z 쇼핑몰' 과 'C 쇼핑몰' 크롤링은 정상적으로 수행되었지만, 'B 쇼핑몰' 크롤링하는 과정에서 아래와 같은 오류가 발생했다. 하지만, local 에서는 'B 쇼핑몰' 도 정상적으로 수행되었지만, AWS EC2 배포한 서비스에서만 아래와 같은 오류가 발생했다.

```
java.net.SocketTimeoutException: Read timed out
        at java.base/sun.nio.ch.NioSocketImpl.timedRead(NioSocketImpl.java:288)
        at java.base/sun.nio.ch.NioSocketImpl.implRead(NioSocketImpl.java:314)
        at java.base/sun.nio.ch.NioSocketImpl.read(NioSocketImpl.java:355)
```

문제가 되는 코드는 아래 로직이다. 디버깅을 해보니 'B 쇼핑몰' 에서 get() 과정에서 오류가 발생했다.

```
final String userAgent = "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/120.0.0.0 Safari/537.36";
final Document document = Jsoup.connect(url)
                                .userAgent(userAgent)
                                .get();
```

### 오류 해결

**시도-1**

Jsoup Issue 게시판을 검색해서 다음과 같은 [질문](https://github.com/jhy/jsoup/issues/1165)을 찾았다. 하지만 inbound, outbound 규칙 모두 확인했을 때, 제약없이 열려있었다. 그렇다면 더더욱 local 환경과 차이가 없을 것이다.

결국 이것은 시도해볼것도 없이 **실패**했다.

**시도-2**

timeout 시간은 default 가 3000ms 라서, timeout 시간을 5000ms 로 조정했다.

참고로 timeout 을 0 으로 설정하면 무한으로 설정할 수 있다.

```
final String userAgent = "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/120.0.0.0 Safari/537.36";
final Document document = Jsoup.connect(url)
                               .userAgent(userAgent)
                               .timeout(5 * 1000)
                               .get();
```

하지만, 결과는 동일하였고, **실패**했다.

**시도-3**

tomcat 의 connection-timeout 설정을 변경했다. 혹시 tomcat connection timeout 이 문제일까?! 라는 의문을 가졌다.

```yaml
server:
  port: 8080
  connection-timeout: 1800000
```

하지만, 결과는 동일하였고, **실패**했다.

**시도-3**

'B 쇼핑몰' 에서 프로그램을 통해 악의적인 크롤링을 하는 것으로 인지하고, 차단하는 것이라고 생각했다.

header, referrer, cookie, userAgent 를 추가해서 사용자의 접근과 비슷하게 보이도록 설정했다.

```
final String userAgent = "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/120.0.0.0 Safari/537.36";
final Document document = Jsoup.connect(url)
                               .userAgent(userAgent)
                               .method(Connection.Method.GET)
                               .ignoreContentType(true)
                               .referrer("http://www.google.com")
                               .header("Accept", "text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.7")
                               .header("Accept-Encoding", "gzip, deflate, br")
                               .header("Accept-Language", "ko-KR,ko;q=0.9,en-US;q=0.8,en;q=0.7")
                               .header("Cache-Control", "max-age=0")
                               .header("Sec-Ch-Ua-Mobile", "?0")
                               .header("Sec-Fetch-Dest", "document")
                               .header("Sec-Fetch-Mode", "navigate")
                               .header("Sec-Fetch-Site", "none")
                               .header("Sec-Fetch-User", "?1")
                               .header("Upgrade-Insecure-Requests", "1")
                               .cookies(getCookies())
                               .timeout(1000 * 30)
                               .get();

... ...

private Map<String, String> getCookies() {
	final Map<String, String> cookies = new HashMap<>();
	cookies.put("_evga_0c2d", "{%22uuid%22:%2........}");

... ...
        return cookies;
}
```

하지만, 결과는 동일하였고, **실패**했다.

**시도-4**

Jsoup Issue 게시판을 검색해서 다음과 같은 [질문](https://github.com/jhy/jsoup/issues/1740)을 찾았다. userAgent 를 변경해보라고 한다!

- as-is
  - Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/120.0.0.0 Safari/537.36
- to-be
  - WhatsApp/2.19.81 A

```
final String userAgent = "WhatsApp/2.19.81 A";
final Document document = Jsoup.connect(url)
                               .userAgent(userAgent)
                               .method(Connection.Method.GET)
                               .ignoreContentType(true)
                               .referrer("http://www.google.com")
                               .header("Accept", "text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.7")
                               .header("Accept-Encoding", "gzip, deflate, br")
                               .header("Accept-Language", "ko-KR,ko;q=0.9,en-US;q=0.8,en;q=0.7")
                               .header("Cache-Control", "max-age=0")
                               .header("Sec-Ch-Ua-Mobile", "?0")
                               .header("Sec-Fetch-Dest", "document")
                               .header("Sec-Fetch-Mode", "navigate")
                               .header("Sec-Fetch-Site", "none")
                               .header("Sec-Fetch-User", "?1")
                               .header("Upgrade-Insecure-Requests", "1")
                               .cookies(getCookies())
                               .timeout(1000 * 30)
                               .get();
```

하지만, 결과는 동일하였고, **실패**했다.

### 결론

해결하지 못했다. Jsoup get() 은 결국 HttpClient 와 같은 라이브러리로 통신을 해서 응답을 받고, 응답된 문서를 HTML 파싱하는 순서로 동작을 할 것이다. 그런데, 계속해서 실패하고 있다. local 아니기 때문에 디버깅에도 한계가 있고… 정말 야마가.. 돈다…

왜 그런것일까?!

| enviroment | B 쇼핑몰 | Z 쇼핑몰 |
| --- | --- | --- |
| local | success | success |
| aws | SocketTimeoutException | success |

정답을 아시는분이 있다면 메일 부탁드립니다!! (lkhlkh09@gmail.com)
