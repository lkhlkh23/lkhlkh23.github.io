---
layout: post
title: AWS Summit 2025 - valkey
subtitle: redis vs valkey
excerpt_image: https://raw.githubusercontent.com/lkhlkh23/lkhlkh23.github.io/master/images/2025-05-14/banner.png
categories: infra
tags: [infra, cache, valkey, redis]
---
AWS Summit 2025 Session 에서 Valkey 라는 오픈소스가 잠시 소개되었다. `비용측면에서 Redis 를 대체`할 수 있다고 해서 현재 운영중인 서비스에 적용할 수 있는지 확인을 위해 간단하게 정리해봤다!

![0.jpeg](https://raw.githubusercontent.com/lkhlkh23/lkhlkh23.github.io/master/images/2025-05-14/0.jpeg)

### What is Valkey ..?!

Valkey 에 대해 알아보기 전에 Redis 의 시작을 먼저 알아야 할 필요가 있다. 초기 Redis 는 오픈소스로 시작되었기 때문에 자유롭게 사용이 가능했다. 그래서 다양한 Cloud 업체들이 Redis 를 이용해서 다양한 관리형 서비스를 출시했다. (예 : AWS ElasticCache, Azure Redis Cache ..)

하지만, 2024년 Redis 의 오픈소스는 결국 종료되었다. `오픈소스 → 누구나 코드를 보고 수정할 수 있지만, 이를 기반으로 한 유료 서비스 제공 금지 → Cloud 업체들이 Redis 를 이용해서 수익을 창출하는 것을 막겠다는 방향으로 정책 강화` 순서로 Redis 는 폐쇠적인 방향으로 변화했다.

이러한 변화속에서 Valkey 가 개발되었다. Valkey 는 Redis 7.2.4 의 오픈소스 기반으로 개발되었기 때문에 Redis 와 Valkey 는 근본적으로 유사하다. Valkey 도 메모리 내 캐시를 제공하기 때문에 디스크를 이용해서 데이터를 액세스하는 데이터베이스보다 훨씬 빠른 처리 속도를 지원한다. 그리고 Valkey 는 Linux Fondation 소속이기 때문에 오픈소스로서 안전성과 지속 가능성을 제공한다.

> **Linux Foundation**
> 
> 비영리 재단으로서 오픈소스 프로젝트를 보호, 관리, 성장시키기 위한 글로벌 조직
>

이러한 배경으로 AWS, Google, Oracle, Alibaba 를 포함한 대기업들도 Valkey 를 지원한다고 선언했다.

### Valkey vs Redis ..?!

**스레딩 아키텍처**

**Redis** 는 “Single-thread but fast” 를 지양한다. I/O 작업만 멀티 스레드를 지원하고, 모든 명령은 단일 스레드에서 이벤트 루프 모델을 사용한다. 스레드 동기화, Lock, 컨텍스트 스위칭과 같은 비용이 발생하지 않기 때문에 낮은 오버헤드라는 장점을 가지고 있다. 또한, 단순한 구조이고 빠른 응답속도를 지원하는 장점을 가지고 있다. 그러나 멀티코어 환경을 적절하게 활용하지 못한다. 또한, 느린 명령 하나가 전체 처리를 지연시킬 수 있는 단점이 있다.

반면, **Valkey (8.+)** 는 멀티코어 프로세서를 더욱 효과적으로 활용하는 향상된 I/O 멀티스레딩을 구현했다. 그렇기 때문에 멀티코어 환경을 적극적으로 활용 가능하고, 병렬처리를 통해 처리량이 향상되었다. Linux Foundation Member Summit 보고서에 따르면 이전 Valkey (7.x) 에 비해 약 3배 향상되었다고 한다. Valkey (8+) 는 최대 20%의 메모리 효율성을 개선했다. 반대로, 복잡도가 증가했다는 단점이 존재하기는 한다.

**확장성**

Redis 와 Valkey 는 동일한 클러스터 구조를 사용하기 때문에 큰 차이는 없다.

**장애 대응**

Redis 는 Master 노드에 장애가 발생하면 Slave 노드가 자동승격되는 방식으로 장애조치한다. Valkey 도 동일한 방식으로 장애를 조치하지만, 클러스터 노드간의 통신을 개선해서 Slave 승격속도가 향상되어 장애 복구 시간이 단축되었다.

### When should i use Valkey ..?!

AWS 의 ElasticCache 를 사용한다면, Redis 를 사용하기보다는 Valkey 로 전환을 권장한다. AWS 에서 제공하는 Redis 는 기본적으로 7.2.4 이하의 버전을 제공한다. Valkey 는 Redis 의 7.2.4 버전을 기반으로 개발되었기 때문에 Redis 와 다르지 않다. 그러나 Valkey 는 Redis 보다 비용절감의 장점이 있기 때문에 Valkey 전환을 권장하는 것이다. Redis 는 Redis.Inc 와 라이선스 비용을 분담해야 하기 때문에 Valkey 보다는 비용이 더 발생한다.

만약 Redis 7.2.4 이후의 버전을 이용한다면 Valkey 전환이 필요할까?! EC2 에 Redis 를 직접 설치해서 운영을 한다면 Redis 의 버전을 선택할 수 있다. 8.x 버전부터 Redis 와 Valkey 의 방향성의 차이가 발생하고 있다. Redis 는 AI 와 편의성에 집중하고, Valkey 는 메모리 및 성능 최적화에 집중하고 있다.

그렇다면 Redis 를 사용하는 나의 조직은 Valkey 전환이 필요할까?! 현재 조직은 AWS ElasticCache 를 사용하고 단순 캐시 수준의 기능만 사용하기 때문에 Valkey 전환이 비용 측면에서 더 효과적일 것이라고 생각한다.

### And I am ..

Redis 의 변화를 보면서 어떤 서비스가 생각이 났다. `무료 → 유료` 로 전환을 할 필요가 있었을까?! 유료 서비스로 전환을 제안했던 담당자는 현재 매출 결과를 예상했을까?! 친한 친구가 나에게 늘 말했다. 내팔내꼰이라고 …
