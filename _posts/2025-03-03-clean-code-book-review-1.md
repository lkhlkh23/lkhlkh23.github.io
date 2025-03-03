---
layout: post
title: 만들면서 배우는 클린 아키텍처 책를 신규 프로젝트에 적용하기!  
subtitle: 만들면서 배우는 클린 아키텍처 리뷰
excerpt_image: https://raw.githubusercontent.com/lkhlkh23/lkhlkh23.github.io/master/images/2025-03-03/banner.png
categories: java
tags: [java]
---
서비스의 가용성, 장애격리, 코드베이스의 복잡성 감소를 위해서 신규 서비스를 신규 프로젝트에 개발하기로 결정했다. 기존 조직의 코드는 깨진 유리창이 아닌, 유리창이 없다. 그래서 신규 프로젝트는 기존 코드와 같은 결과가 되지 않기를 바랜다. 그래서 `만들면서 배우는 클린 아키텍처` 의 내용을 참고해서 아키텍처를 잡으려고 한다.

이번 포스팅에서는 아키텍처와 매핑전략을 정리했다!

기존 코드는 가장 많이 사용하는 계층형 아키텍처로 개발되었다. 이 책에서는 계층형 구조가 나쁜 코드 습관을 쉽게 스며들게 만든다고 지적한다. 현재 운영하고 있는 프로젝트에서 이러한 사례를 많이 경험했기 때문에 동감한다! 계층형 아키텍처를 잘못 사용할 때 발생할 수 있는 문제를 알아보자!

**계층형 아키텍처는 데이터베이스 설계를 유도한다!**

웹 계층은 도메인 계층에 의존하고, 도메인 계층은 영속성 계층에 의존하기 때문에 **데이터베이스에 의존**하게 된다.  상태를 바꾸는 주체는 행동이기 때문에 상태가 아닌 행동을 중심으로 모델링을 해야한다. 그렇기 때문에 가장 먼저 도메인 로직을 설계/개발후에 웹 및 영속성 계층을 개발해야 한다. 실제로 도메인 우선적으로 설계를 하지 않고, 화면기준으로 데이터베이스를 우선적으로 설계하는 모습을 많이 봤다.

계층은 아래 방향으로만 접근가능하기 때문에 도메인 계층은 엔티티에 접근할 수 있고, **영속성 계층과 도메인 계층의 강합 결합**이 발생한다. 영속성 코드가 도메인 코드에 녹아들어가면 변경에 유연한 구조가 될 수 없다.

**테스트가 어려워진다!**

웹 계층에서 영속성 계층에 직접 접근하는 경우가 있다. 도메인 계층에 대한 고려 및 개발을 건널뛸 수 있기 때문에 기능 개발이 더 빠를 수 있다. 하지만, 테스트의 복잡성을 증가시킨다. 웹 계층에서 도메인 계층뿐만 아니라, 영속성 계층에 대한 mocking 을 해야한다. 이렇게 된다면 단위 테스트의 복잡도가 증가한다. 실제로 이러한 코드들이 많이 있다. 심지어 테스트코드도 없다.

**유지보수의 어려움!**

계층에 대한 명확한 분리가 없이 로직들이 각 계층에 흩어져있다면, 새로운 기능을 추가할 적당한 위치를 찾는 것이 어려워진다. 또한, 적당한 위치가 찾기 어려워서 넓은 서비스를 생성한다. 실제로 내가 운영하는 프로젝트에서 UserService 는 회원가입, 회원탈퇴, 회원정보 조회, 비밀번호 찾기 등 회원과 관련된 모든 로직이 있다.

![0.png](https://raw.githubusercontent.com/lkhlkh23/lkhlkh23.github.io/master/images/2025-03-03/0.png)

넓은 서비스는 영속성에 대한 의존성도 많아지고 테스트도 어려운 구조이다. 또한, SRP 에 위반되기 때문에 변경에 대한 영향도가 크다. 또한 넓은 서비스는 동시작업에 대한 어려움이 있다. 회원로직 개발을 한다고할 때, 모두가 UserService 를 개발한다면 GIT 병합이 복잡해질 것이다. 서비스뿐만 아니라 웹과 영속성 계층도 같은 이유로 세분화가 필요하다.

![1.png](https://raw.githubusercontent.com/lkhlkh23/lkhlkh23.github.io/master/images/2025-03-03/1.png)

계층형 아키텍처는 가장 널리 사용되는 아키텍처이다. 계층형 아키텍처는 잘못이 없다. 계층형 아키텍처의 경계를 고민하지 않고 사용하는 사람이 문제이다!  계층에 대한 고민없이 개발을 하면 신규 개발은 빠를 수 있지만, 유지보수에서 어려움을 준다. 서비스의 전체 사이클에서 신규 개발은 굉장히 적은 비중을 차지한다. 그렇기 때문에 계층에 대한 명확한 분리가 필요하다.

**단일 책임 원칙 → 단일 변경이유 원칙**

사람들이 가장 잘못알고 있는 SOLID 원칙 중 하나가 SRP 이다. 대부분의 사람들은 책임을 역할로 해석한다. 역할로 해석하는 것은 문제가 아니다. 역할보다는 변경이유로 해석하는 것이 더 적합하다고 생각한다. 왜냐하면 변경의 이유가 하나라는 것은 하나의 역할만 가지고 있다고 해석할 수 있기 때문이다. 포괄적으로 해석한다면 변경이유가  더 적합하다!

**의존성 역전 원칙**

계층형 아키텍처의 의존성의 방향은 아래 계층을 가리킨다. 영속성 계층을 수정하면 도메인 계층을 수정해야한다. 하지만 도메인 코드는 애플리케이션에서 가장 중요한 코드이다. 그렇기 때문에 의존성을 제거해야 한다.

![2.png](https://raw.githubusercontent.com/lkhlkh23/lkhlkh23.github.io/master/images/2025-03-03/2.png)

```java
@Service
class OrderService {
    private final OrderRepository orderRepository;

    public OrderService(OrderRepository orderRepository) {
        this.orderRepository = orderRepository;
    }

    public void processOrder(Long orderId) {
        OrderEntity order = orderRepository.findById(orderId).orElseThrow();
        order.setStatus("COMPLETED");
        orderRepository.save(order);
    }
}
```

위와 같은 구조는 도메인 계층과 영속성 계층의 분리가 명확하지 않기 때문에 도메인 계층이 영속성 계층에 의존적이다. 그렇기 때문에 아래와 같이 구조를 변경해야한다.

![3.png](https://raw.githubusercontent.com/lkhlkh23/lkhlkh23.github.io/master/images/2025-03-03/3.png)

```java
@Service
class OrderService {
    private final OrderRepository orderRepository;

    public OrderService(OrderRepository orderRepository) {
        this.orderRepository = orderRepository;
    }

    public void processOrder(Long orderId) {
        Order order = orderRepository.findById(orderId).orElseThrow();
        order.completeOrder();  // 도메인 객체에서 상태 변경
        orderRepository.save(order);
    }
}

class Order {
    private Long id;
    private String status;

    public Order(Long id, String status) {
        this.id = id;
        this.status = status;
    }

    public void completeOrder() {
        this.status = "COMPLETED";
    }
}

interface OrderRepository {
    Optional<Order> findById(Long id);
    void save(Order order);
}

@Entity
@Table(name = "orders")
class OrderEntity {
    @Id @GeneratedValue
    private Long id;
    private String status;

    public Order toDomain() {
        return new Order(id, status);
    }

    public static OrderEntity fromDomain(Order order) {
        OrderEntity entity = new OrderEntity();
        entity.id = order.getId();
        entity.status = order.getStatus();
        return entity;
    }
}

@Repository
class JpaOrderRepository implements OrderRepository {
    private final JpaRepository<OrderEntity, Long> jpaRepository;

    public JpaOrderRepository(JpaRepository<OrderEntity, Long> jpaRepository) {
        this.jpaRepository = jpaRepository;
    }

    @Override
    public Optional<Order> findById(Long id) {
        return jpaRepository.findById(id).map(OrderEntity::toDomain);
    }

    @Override
    public void save(Order order) {
        jpaRepository.save(OrderEntity.fromDomain(order));
    }
}

```

도메인 로직과 영속성 로직이 명확하게 분리가 되었다. 영속성의 엔티티 계층은 데이터베이스와 직접 매핑된 ORM 엔티티를 의미하고, 도메인 계층의 엔티티는 비지니스 로직을 포함하고 있는 엔티티이다.  결과적으로 도메인 로직에서 외부로 향하는 의존성을 모두 제거했다. 도메인 로직은 어떤 영속성 프레임워크를 사용하는지, 어떤 UI 프레임워크가 사용하는지를 알 수 없기 때문에 오직 비지니스에만 집중할 수 있다.

그러나, 도메인 계층과 영속성 계층이 데이터를 주고받을 때, 두 엔티티를 변환해야하는 단점이 있다. 하지만 이러한 단점은 Mapstruct 를 통해 얼마든지 해결을 할 수 있다. 그리고 비슷한 필드를 보유하고 있는 두 클래스의 네이밍이 조금 어렵다는 단점이 있다. 그건, 영속성의 엔티티와 도메인의 엔티티의 네이밍을 구분할 수 있는 규칙을 넣기만 하면된다. 나같은 경우는 도메인 계층은 Order, 영속성 계층은 OrderEntity 로 Entity를 Suffix 로 넣는 방법으로 이 문제를 해결했다.

이제는 이 책의 핵심인 핵사고날 아키텍처가 나온다! 아래 그림은 대중적인 헥사고날 아키텍처의 모습이다.

![4.png](https://raw.githubusercontent.com/lkhlkh23/lkhlkh23.github.io/master/images/2025-03-03/4.png)

뭔가 복잡한 구성도로 보이지만, 좀더 구체적으로 그리면 아래와 같은 모습이다. 지금부터는 각각의 구성요소의 역할에 대해 정리해보겠다.

![5.png](https://raw.githubusercontent.com/lkhlkh23/lkhlkh23.github.io/master/images/2025-03-03/5.png)

모두 그런적이 있지 않나?! 이 기능은 웹 어댑터가 수행해야할까?! 유스케이스가 수행해야할까?! 혹은 이 코드는 어느 패키지에 위치하는 것이 적합할까?! 이 클래스의 이름은 어떻게 네이밍하지?! 그렇기때문에 각 계층별 역할, 위치, 네이밍 규칙을 설정하는 것이 중요하다고 생각한다.

### 웹 어댑터의 역할

- HTTP 요청을 자바 객체로 매핑
- 권한 검사
- 입력 유효성 검증
- 입력을 유스케이스의 입력모델로 매핑
- 유스케이스 호출
- 유스케이스의 출력을 HTTP 로 매핑
- HTTP 응답

### 유스케이스의 역할

- 비지니스 규칙 검증
- 도메인 엔티티 조작
- 출력 반환

### 영속성 어댑터의 역할

- 입력을 데이터베이스 포맷으로 매핑
- 입력을 데이터베이스로 전달
- 데이터베이스 출력을 애플리케이션 포맷으로 매핑
- 출력 반환

지금까지 아키텍처에 대해 설명을 했다. 그러나 여기에 매핑전략도 같이 녹아져있다.

지금까지 소개했던 아키텍처는  `완전 매핑 전략` 이다. 그리고 개인적으로 완전 매핑 전략을 선호한다. 완전 매핑 전략의 단점은 계층 사이에서의 매핑 `오버헤드`가 발생한다는 단점이 있다. 하지만 이러한 오버헤드는 Mapstruct 로 해결 가능하다. 그러나 변경이 하나의 계층에서만 발생한다면 다른 계층을 수정할 필요가 없기 때문에 `계층간의 독립성`이 유지되는 장점이 있다. 그리고 `도메인 모델을 보호`하고 `API 변경의 영향을 최소화`할 수 있다.

개인적으로 가장 선호하지 않는 방식의 전략은 `매핑하지 않기 전략` 이다. 모든 계층이 정확히 같은 정보를 필요하다면 계층간의 매핑은 불필요한 오버헤드가 될 수 있다. 하지만, 도메인 엔티티가 웹, 서비스, 도메인, 영속성 등 너무 많은 역할을 수행한다. 그리고 가장 핵심적이고 변화가 적어야하는 도메인 엔티티가 변화에 취약한 구조가 될 수 있다.

이제는 지금까지 정리한 내용을 공유하고! 조직 내부 규칙으로 적용할 일만 남았다!

![6.png](https://raw.githubusercontent.com/lkhlkh23/lkhlkh23.github.io/master/images/2025-03-03/6.png)

다음은 테스트 전략에 대해 정리하겠다!

![7.png](https://raw.githubusercontent.com/lkhlkh23/lkhlkh23.github.io/master/images/2025-03-03/7.png)

### And I am ..

이제는 시간이 없다. 연휴동안 스터디 계획을 수립했으니, 천천히 수행해야겠다!
