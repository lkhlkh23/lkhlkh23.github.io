---
layout: post
title: gzip 압축된 post method payload 처리
subtitle: content-encoding gzip, filter
excerpt_image: https://raw.githubusercontent.com/lkhlkh23/lkhlkh23.github.io/master/images/2023-12-19/banner.png
categories: java
tags: [java, spring]
---

![banner](https://raw.githubusercontent.com/lkhlkh23/lkhlkh23.github.io/master/images/2023-12-19/banner.png)

최근에 ‘D회사’ 로부터 전달받은 데이터를 DB 에 저장하는 API 를 개발했다.
‘D회사’ 에서는 아래와 같은 curl 을 통해 API 를 호출했다.

```jsx
echo '["key":"value"]' | gzip | curl -X POST -H "Content-Encoding: gzip" -H "Content-Type: application/json" --data-binary @ [https://example-url/push](https://localhost:8080/push) 
```

하지만, 현재 개발된 API 에서는 500 오류가 발생하고, 데이터를 DB 에 저장하지 못했다.
이를 해결한 간단한 코드를 정리하려고 한다.

그전에 **content-encoding gzip**, **filter vs interceptor** 에 대해 간단하게 알아보려고 한다.

### content-encoding gzip

content-encoding 은 리소스 (텍스트, 미디어 …) 전송하는 동안 데이터를 압축하거나 인코딩하는 방법을 정의하는 헤더이다. 대역폭을 절약하고 웹 페이지 로딩 시간을 단축하는 데 도움이 된다.

- **gzip**
  - 텍스트 기반의 리소스를 압축하기 위해 사용
  - 서버는 리소스를 gzip으로 압축하여 클라이언트에게 전송하고, **클라이언트는 해당 리소스를 받은 후에 gzip을 decode 처리 후 사용**
- **deflate:** 텍스트 기반의 리소스를 압축하기 위해 사용
- **br**
  - 텍스트 및 압축 효율이 높은 리소스에 대한 압축하기 위해 사용 (gzip 보다 더 효과적)
- **identity:**
  - 인코딩이 적용되지 않았음을 의미

### Filter vs Interceptor

Filter 는 Dispatcher Servlet 에 요청이 전달되기 전 / 후에 URL 패턴에 맞는 모든 요청에 대해 부가 작업을 처리할 수 있는 기능을 제공한다. 또한, 스프링 컨테이너가 아닌 톰캣과 같은 웹 컨테이너에 의해 관리가 되는 것이고 스프링 범위 밖에서 처리된다.

Interceptor 는 Dispatcher Servlet 이 Controller 를 호출하기 전/후에 인터셉터가 끼어들어 요청과 응답을 참조하거나 가공할 수 있는 기능을 제공한다

> **Dispatcher Servlet**
HTTP 로 들어오는 모든 요청을 가장 먼저 받아 적합한 컨트롤러에 위임해주는 프론트 컨트롤러
>

Filter 와 Interceptor 의 가장 큰 차이는 Request 와 Response 에 대한 조작 유무의 차이이다. **Filter 는 Filter Chaining 통해서 원하는 Request 와 Response 객체를 전달할 수 있다.** Filter 는 조작이 가능하다!

### 오류 원인

오류의 원인은 간단하다. ‘D회사’ 에서 전달받은 Payload 는 gzip 으로 압축되었고, 해당 데이터는 Dispatcher Servlet 통해서 URL 패턴에 맞는 Controller 에게 전달되었다. gzip 으로 압축된 데이터는 Controller 의 DTO 형식에 맞지 않았고, 결국은 500 오류가 발생한 것이다.

### 해결 방법

결국은, gzip 으로 압축된 데이터는 decode 하고 Controller 에게 넘겨야 한다. 그러기 위해서 Filter 가 필요하다. Interceptor 로 개발을 한다면, Request 에서 전달받은 Payload 를 읽어서 decode 할 수 있지만, decode 한 데이터를 Request 에 넣을 수 없다. 왜냐하면?! Interceptor 는 Request 를 조작할 수 없기 때문이다.

결국은, Request 를 변경할 수 있는 Filter 를 이용해야 한다. gzip decode 처리한 데이터를 가지고 있는 새로운 Request 를 전달하는 방식으로 문제를 해결해야 한다.

### 해결 코드
```java
class GzipBodyDecodeFilter implements Filter {
	@Override
	public void doFilter(final ServletRequest servletRequest,
						 final ServletResponse servletResponse, FilterChain chain) throws IOException, ServletException {
		final HttpServletRequest request = ((ServletRequestAttributes)RequestContextHolder.currentRequestAttributes()).getRequest();
		final boolean isGzipped =
			request.getHeader(HttpHeaders.CONTENT_ENCODING) != null && request.getHeader(HttpHeaders.CONTENT_ENCODING)
																			  .contains("gzip");
		final boolean isPostMethod = "POST".equals(request.getMethod());
		// gzip 압축되었거나, POST 요청에 대해서만 gzip decode 수행!
		if (isGzipped && isPostMethod) {
			// client 로 부터 전달받은 request 가 아닌, gzip decode 된 데이터를 가지고 있는 신규 request 전달
			chain.doFilter(new GZIPRequestWrapper(request), servletResponse);
			return;
		}
		chain.doFilter(servletRequest, servletResponse);
	}

	private static class GZIPRequestWrapper extends HttpServletRequestWrapper {
		private final String decompressed;
		
		public GZIPRequestWrapper(HttpServletRequest request) throws IOException {
			super(request);
			final byte[] inputs = IoUtils.toByteArray(request.getInputStream());
			this.decompressed = decompress(inputs);
		}

		@Override
		public ServletInputStream getInputStream() {
			final ByteArrayInputStream byteArrayInputStream = new ByteArrayInputStream(
				decompressed.getBytes(StandardCharsets.UTF_8));
			return new ServletInputStream() {
				@Override
				public boolean isFinished() {
					return false;
				}

				@Override
				public boolean isReady() {
					return false;
				}

				@Override
				public void setReadListener(ReadListener listener) {
				}

				@Override
				public int read() {
					// 데이터를 읽을 때는 gzip decode 된 데이터를 읽을 수 있도록 오버라이딩
					return byteArrayInputStream.read();
				}
			};
		}

		@Override
		public BufferedReader getReader() {
			return new BufferedReader(new InputStreamReader(this.getInputStream()));
		}

		// gzip decode 메소드
		private String decompress(final byte[] value) {
			try (ByteArrayOutputStream outStream = new ByteArrayOutputStream(); GZIPInputStream gzipInStream = new GZIPInputStream(
				new ByteArrayInputStream(value))) {
				byte[] buffer = new byte[1024];
				int bytesRead;
				while ((bytesRead = gzipInStream.readNBytes(buffer, 0, buffer.length)) > 0) {
					outStream.write(buffer, 0, bytesRead);
				}
				return new String(outStream.toByteArray());
			} catch (Exception e) {
				return "";
		
```
