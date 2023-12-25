---
layout: post
title: Transaction read only true 왜 써야할까?!
subtitle: Transaction read only true 는 필수일까?!
excerpt_image: https://raw.githubusercontent.com/lkhlkh23/lkhlkh23.github.io/master/images/2023-12-25/banner.jpeg
categories: jpa
tags: [jpa, transaction]
---

![banner](https://raw.githubusercontent.com/lkhlkh23/lkhlkh23.github.io/master/images/2023-12-25/banner.jpeg)

최근에 Pull Request 에 `@Transaction (readonly=true)` 를 적용했으면 좋겠다고 피드백을 남겼다. 이번에는 `@Transaction (readonly=true)` 적용했을 때, 장점을 간단하게 정리해보려고 한다.

### 가독성

대다수의 개발자들이 메소드 네이밍에 진심이기 때문에 메소드명으로만으로 충분히 메소드의 역할을 파악할 수 있다. 조회만 수행하는 메소드인지, 삽입을 수행하는 메소드인지 충분히 파악이 가능하다.  
하지만, `@Transaction (readonly=true)` 를 통해 메소드의 역할을 확실히 파악할 수 있다.

### 성능 효율

성능 효율전에 `변경감지, Dirty Checking` 을 먼저 이야기 하려고 한다.

Entity 조회 시, 영속성 컨텍스트는 Entity 초기 상태에 대한 Snapshot 을 저장한다. 트랜잭션이 commit 되었을 때, 초기 상태에 대한 Snapshot 과 현재 Entity 상태를 비교해서 변경된 내용에 대해 쓰기 지연 저장소에 저장한다. 그리고나서, 일괄적으로 쓰기 지연 저장소에 저장된 query 를 flush 하여 DB 에 반영한다.

이러한 일련의 과정을 `변경감지, Dirty Checking` 이라고 한다.

`@Transaction (readonly=true)` 는 flush mode 가 manual 로 설정되기 때문에 수동으로 flush() 메소드를 호출하지 않는한 DB에 반영이 되지 않는다.

> 
> manual mode : 수동으로 flush를 호출하지 않으면 flush가 자동으로 수행되지 않는 모드
>

결국은 `@Transaction (readonly=true)` 를 하면, 불필요한 `변경감지, Dirty Checking` 을 하지 않기 때문에 성능이 향상된다.

### DB 부하 조절

대부분의 서비스가 안전성 (장애 복구) 과 효율성 (부하 분산) 의 목적으로 Master DB 와 Replication DB 를 같이 운영하고 있다. `@Transaction (readonly=true)` 를 이용하면 데이터 조회는 Replication DB 로, 데이터 변경 작업은 Master DB 로 트래픽을 분산할 수 있다.

![banner](https://raw.githubusercontent.com/lkhlkh23/lkhlkh23.github.io/master/images/2023-12-25/0.png)

MySql Master DB 와 Replication DB 사이의 데이터 동기화 방식

1. 데이터의 변경이 발생하면, Master DB 에서는 Binary Log 에 모두 기록
2. Replication DB 의 I/O Thread 에서는 Binary Log 의 기록 조회하여 Slave DB 의 Relay Log 에 기록
3. SQL Thread 에서는 Relay Log 를 읽어서 Slave DB 에 쿼리 수행하여 동기화 진행

다음 포스팅에서는 Master DB 와 Replication DB 에 대한 트래픽 분산을 어떻게 수행하는지에 대해서 간단한 코딩을 통해 알아보려고 한다.

크리스마스에 쿠키로 내집 마련하였다. 위에 있는 배너 이미지가 내가 만든 쿠키집이다!
