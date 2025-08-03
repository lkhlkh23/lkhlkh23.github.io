---
layout: post
title: N+1 문제와 EntityGraph 로 현명하게 해결하기
subtitle: EntityGraph vs Join Fetch
excerpt_image: https://raw.githubusercontent.com/lkhlkh23/lkhlkh23.github.io/master/images/2025-08-03/banner.png
categories: java
tags: [java, jpa, n+1, entitygraph]
---
JPA 사용하는 개발자라면 한 번쯤 고민해봤을 **N+1 문제**와 이를 우아하게 해결하는 방법인 `EntityGraph` 에 대해 간단하게 포스팅해봤다!

### What is N+1 ..?!

N+1 문제는 JPA 에서 연관 관계를 지연 로딩(Lazy Loading) 으로 설정했을 때 발생하는 성능 문제이다. 예를 들어, `Team` 과  `Member` 가  `1:N`  관계일 때, 모든 `Team` 을 조회하고 각 `Team` 의 `Member` 목록을 조회하려고 한다. 지연로딩 설정이 되있기 때문에 프록시를 이용해서 아래와 같이 조회할 것이다.

```java
List<Team> teams = teamRepository.findAll(); // 쿼리 1회 (Team 조회)

for (Team team : teams) {
     // N번의 쿼리 (각 Team의 Members 조회)
    System.out.println(team.getMembers().size());
}
```

`findAll()`  메서드 호출 시 `Team` 엔티티를 조회하는 쿼리가 1번 발생한다. 그리고 `for` 루프 안에서 각 `Team` 의 `getMembers()` 를 호출할 때마다 `Member` 를 조회하는 쿼리가 추가로 발생한다.

만약 `Team` 이 100개라면, 총 `1 (Team) + 100 (Member)= 101 번의 쿼리` 가 실행되는 비효율적인 상황이 발생한다. 이것이 바로 N+1 문제이다.

### How to solve N+1 ..?!

**JOIN FETCH**

가장 보편적인 해결책은 JPQL 의 `JOIN FETCH` 을 활용하는 방법이다. JPQL `FETCH JOIN` 은 연관된 엔티티를 한 번의 쿼리로 함께 로딩하여 N+1 문제를 해결한다.

```java
// JPQL FETCH JOIN 예시
@Query("SELECT t FROM Team t JOIN FETCH t.members")
List<Team> findAllWithMembers();
```

`JOIN FETCH` 를 사용하면 JPA는 `Team` 과 `Member` 를 **단 한 번의 쿼리**로 조회한다. `Team` 엔티티를 조회하는 시점에 연관된 `Member` 엔티티까지 프록시가 아닌 실제 객체로 함께 로딩한다.

**Entity Graph**

`EntityGraph`는 **엔티티에 미리 로딩 전략을 정의해두고, 필요에 따라 선택적으로 적용**할 수 있는 기능이다.

```java
@Entity
@NamedEntityGraph(name = "teamWithMembers", attributeNodes = @NamedAttributeNode("members"))
public class Team {
    // ...
    @OneToMany(mappedBy = "team")
    private List<Member> members = new ArrayList<>();
}
```

`@NamedEntityGraph` 어노테이션으로 `teamWithMembers` 라는 이름의 `EntityGraph` 를 정의했다. `attributeNodes`를 통해 `members` 필드를 즉시 로딩하겠다고 선언한 것이다.

```java
public interface TeamRepository extends JpaRepository<Team, Long> {
    @EntityGraph("teamWithMembers")
    List<Team> findAll();
}
```

`findAll()`  메서드에 `@EntityGraph` 어노테이션을 붙이면, 이 메서드가 호출될 때 JPA는 `FETCH JOIN` 이 포함된 쿼리를 동적으로 생성하여 `Team`과 `Member`를 함께 로딩한다. 쿼리 로직을 리포지토리 메서드에 직접 작성하지 않고, 재사용 가능한 로딩 전략을 적용한 것이다.

### Entity Graph vs JPQL Join Fetch

| 특징 | JPQL Join Fetch  | Entity Graph |
| --- | --- | --- |
| **where 조건** | JPQL 쿼리 내에 where 절로 직접 작성 | JPA 리포지토리 메서드 쿼리로 추가 (findBy …) |
| **페이징** | 페이징과 함께 사용 시 오류 발생 가능성이 높아 비추천 | Pageable 함께 사용 시 JPA가 최적화된 쿼리를 생성하여 안전하게 사용 가능 |
| **작동 방식** | 쿼리에 Join Fetch 구문을 직접 작성 | 엔티티의 로딩 전략을 정의해두고 런타임에 동적으로 적용 |
| **권장 사용법** | 복잡하고 유연한 조건의 쿼리가 필요한 경우 | 페이징이 필요하거나, 재사용 가능한 로딩 전략이 많을 경우 |
| **장점** | 세밀한 쿼리 제어 가능 | 코드의 가독성 향상 및 로딩 전략 재사용 용이 |
| **단점** | 쿼리가 복잡해지기 쉬움 | 복잡한 쿼리에는 한계가 있음 |

`EntityGraph` 는 페이징과 함께 사용해야 하거나, 하나의 엔티티에 대해 다양한 로딩 전략을 미리 정의해두고 유연하게 사용하고 싶을 때 매우 유용하다. 반면, `FETCH JOIN` 은 일회성으로 복잡하고 특수한 쿼리를 작성할 때 강력한 도구이다. 두 기술의 장단점을 명확히 이해하고 상황에 맞게 적절히 사용한다면, N+1 문제 해결과 함께 애플리케이션의 성능을 향상시킬 수 있다.

### And I am …

![0.png](https://raw.githubusercontent.com/lkhlkh23/lkhlkh23.github.io/master/images/2025-08-03/0.png)

정말 … 조용한 날이 하루가 없다. 업무가 많은 것은 야근을 하면 된다. 그리고 퇴근을 하면 모든 업무를 회사에 두고 오기 때문에 퇴근 후 스트레스가 적다. 하지만 사람 스트레스는 그렇지 못하다. 정말 …

오늘의 퀴즈! 다음 이모티콘이 의미하는 한국 속담은 무엇일까요?!
```
🧍‍♂️🐟🌊💦❌
```


