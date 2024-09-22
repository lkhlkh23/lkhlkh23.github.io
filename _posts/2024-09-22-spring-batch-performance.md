---
layout: post
title: 스프링배치 성능 최적화 (by ifKakao 2022)
subtitle: if Kakao 2022 “Batch Performance 극한으로 끌어올리기 - 1억 건 데이터 처리를 위한 노력
excerpt_image: https://raw.githubusercontent.com/lkhlkh23/lkhlkh23.github.io/master/images/2024-09-22/banner.png
categories: springbatch
tags: [springbatch, performance, kakao-2022]
---

요즘 시간이 있을 때, 과거 컨퍼런스 영상을 찾아보고 있다. “최신 컨퍼런스 영상도 아닌, 오래된 컨퍼런스 영상을 보고 있냐?!” 라는 질문을 받을 수 있다. 그러나, 역시! 이유가 있다!

![0.png](https://raw.githubusercontent.com/lkhlkh23/lkhlkh23.github.io/master/images/2024-09-22/0.png)

내 업무에 적용해서, 개발뿐만 아니라 운영도 경험할 수 있는, 나에게 스토리를 줄 수 있는 인사이트를 얻고 싶어서이다. 최신 기술들은 현재 조직 환경에는 적합하지 않다. 그래서 과거 컨퍼런스 영상중에서 조직에서 사용하고 있는 기술과 연관된 영상을 우선적으로 찾아보고 있다.

그래서 현 조직에서 잘 사용하고 있는 SpringBatch 와 관련된 강의 [if Kakao 2022 “Batch Performance 극한으로 끌어올리기: 1억 건 데이터 처리를 위한 노력” 강의](https://www.youtube.com/watch?v=2IIwQDIi3ys) 를 듣고 내용 정리 및 나의 생각도 같이 담을 예정이다.

SpringBatch 를 개발을 위해서는 두가지 방식이 있다.

- Tasklet
- Chunk Oriented Processing

Chunk Oriented Processing 에서는 `reader → processor → writer` 구조로 처리를 한다. 해당 강의에서는 대용량 데이터를 처리하는 과정에서 각각의 구성요소에서 성능효율을 최대화할 수 있는 방법에 대해 소개하고 있다.

![1.png](https://raw.githubusercontent.com/lkhlkh23/lkhlkh23.github.io/master/images/2024-09-22/1.png)

참고로 Chunk Oriented Processing 이 아래와 같은 절차로 진행을 되고있다고 착각을 했다. 이유는 SpringBatch 4.0.4 버전의 공식문서에서 아래와 같이 나와있었다. 하지만 정확한 절차는 위 그림이다.

![2.png](https://raw.githubusercontent.com/lkhlkh23/lkhlkh23.github.io/master/images/2024-09-22/2.png)

### **Optimizing Reader Performance**

Reader 성능 최적화를 위해서 `적절한 PaginReader` 를 사용하는 전략이 있다.

Chunk Oriented Processing 에서 데이터를 조회하기 위해서 `PagingItemReader` 와 `CursorItemReade` 를 사용한다.

먼저 `PagingItemReader` 는 주로 `JpaPaginItemReader` 와 `RepositoryItemReader` 구현체를 사용한다. 이 두 가지 Reader는 페이징 방식으로 데이터를 읽어오지만, 데이터 양이 많아질수록 `LIMIT + OFFSET` 을 사용하는 쿼리 때문에 성능 저하가 발생할 수 있다.

> LIMIT + OFFSET 성능 저하 이유

페이징을 구현할 때 LIMIT 과 OFFSET 을 사용하면, 데이터베이스는 요청된 페이지를 가져오기 전에 이전의 모든 레코드를 스캔한다. 즉, OFFSET 값이 클수록 데이터베이스는 더 많은 데이터를 스캔해야 하고, 이로 인해 성능이 저하된다.
>

이를 해결하기 위해서 OFFSET 을 항상 0으로 유지하는, ZERO OFFSET 을 사용한다. `QuerydslZeroOffsetItemReader` 를 아래와 같이 커스텀하게 구현할 수 있다.

```java
public class QuerydslPagingItemReader<T> extends AbstractPagingItemReader<T> {

    protected final Map<String, Object> jpaPropertyMap = new HashMap<>();
    protected EntityManagerFactory entityManagerFactory;
    protected EntityManager entityManager;
    protected Function<JPAQueryFactory, JPAQuery<T>> queryFunction;
    protected boolean transacted = true; // default value

    protected QuerydslPagingItemReader() {
        setName(ClassUtils.getShortName(QuerydslPagingItemReader.class));
    }

    public QuerydslPagingItemReader(EntityManagerFactory entityManagerFactory,
                                    int pageSize,
                                    Function<JPAQueryFactory, JPAQuery<T>> queryFunction) {
        this(entityManagerFactory, pageSize, true, queryFunction);
    }

    public QuerydslPagingItemReader(EntityManagerFactory entityManagerFactory,
                                    int pageSize,
                                    boolean transacted,
                                    Function<JPAQueryFactory, JPAQuery<T>> queryFunction) {
        this();
        this.entityManagerFactory = entityManagerFactory;
        this.queryFunction = queryFunction;
        setPageSize(pageSize);
        setTransacted(transacted);
    }

    /**
     * Reader의 트랜잭션격리 옵션 <br/>
     * - false: 격리 시키지 않고, Chunk 트랜잭션에 의존한다 <br/>
     * (hibernate.default_batch_fetch_size 옵션 사용가능) <br/>
     * - true: 격리 시킨다 <br/>
     * (Reader 조회 결과를 삭제하고 다시 조회했을때 삭제된게 반영되고 조회되길 원할때 사용한다.)
     */
    public void setTransacted(boolean transacted) {
        this.transacted = transacted;
    }

    @Override
    protected void doOpen() throws Exception {
        super.doOpen();

        entityManager = entityManagerFactory.createEntityManager(jpaPropertyMap);
        if (entityManager == null) {
            throw new DataAccessResourceFailureException("Unable to obtain an EntityManager");
        }
    }

    @Override
    @SuppressWarnings("unchecked")
    protected void doReadPage() {
        EntityTransaction tx = getTxOrNull();

        JPQLQuery<T> query = createQuery()
                .offset(getPage() * getPageSize())
                .limit(getPageSize());

        initResults();

        fetchQuery(query, tx);
    }

    protected EntityTransaction getTxOrNull() {
        if (transacted) {
            EntityTransaction tx = entityManager.getTransaction();
            tx.begin();

            entityManager.flush();
            entityManager.clear();
            return tx;
        }

        return null;
    }

    protected JPAQuery<T> createQuery() {
        JPAQueryFactory queryFactory = new JPAQueryFactory(entityManager);
        return queryFunction.apply(queryFactory);
    }

    protected void initResults() {
        if (CollectionUtils.isEmpty(results)) {
            results = new CopyOnWriteArrayList<>();
        } else {
            results.clear();
        }
    }

    /**
     * where 의 조건은 id max/min 을 이용한 제한된 범위를 가지게 한다
     * @param query
     * @param tx
     */
    protected void fetchQuery(JPQLQuery<T> query, EntityTransaction tx) {
        if (transacted) {
            results.addAll(query.fetch());
            if(tx != null) {
                tx.commit();
            }
        } else {
            List<T> queryResult = query.fetch();
            for (T entity : queryResult) {
                entityManager.detach(entity);
                results.add(entity);
            }
        }
    }

    @Override
    protected void doJumpToPage(int itemIndex) {
    }

    @Override
    protected void doClose() throws Exception {
        entityManager.close();
        super.doClose();
    }
}
```

```java
/**
 * 배치의 형태가 삭제/수정으로 인해 조회 결과가 계속 동적으로 변경될때 사용할 Reader
 * 참고: https://jojoldu.tistory.com/337
 *
 */
public class QuerydslZeroPagingItemReader<T> extends QuerydslPagingItemReader<T> {

    public QuerydslZeroPagingItemReader() {
        super();
        setName(ClassUtils.getShortName(QuerydslZeroPagingItemReader.class));
    }

    public QuerydslZeroPagingItemReader(EntityManagerFactory entityManagerFactory,
                                        int pageSize,
                                        Function<JPAQueryFactory, JPAQuery<T>> queryFunction) {
        this();
        setTransacted(true);
        super.entityManagerFactory = entityManagerFactory;
        super.queryFunction = queryFunction;
        setPageSize(pageSize);
    }

    @Override
    @SuppressWarnings("unchecked")
    protected void doReadPage() {

        EntityTransaction tx = getTxOrNull();

        JPQLQuery<T> query = createQuery()
                .offset(0)
                .limit(getPageSize());

        initResults();

        fetchQuery(query, tx);
    }

}
```

`CursorItemReader` 를 사용한다면, OFFSET 에 의한 성능 저하를 걱정할 필요가 없다.

그러나, 구현체인 `JpaCursorItemReader` 는 권장하지 않는다. JpaCursorItemReader 는 데이터를 모두 조회한 후, 애플리케이션에서 직접 Cursor 하는 방식이다. 그렇기 때문에 데이터가 많을 경우 OOM 이 발생할 수 있다.

그러므로, `JdbcCursorItemReader` 와 `HibernateCursorItemReader` 을 권장한다. 두개의 Reader 는 DB 에서 Cursor 하는 방식이기 때문에 OOM 을 걱정하지 않아도 된다. 그러나, 개인적으로 선호하지 않는다. 이 두 Reader 는 데이터를 조회할 때 HQL이나 Native Query를 사용해야 하기 때문에, 코드의 가독성이 떨어지고, 유지보수가 어렵고, 데이터베이스 종속적이라는 문제가 있다.

영상에서는 JetBrain Kotlin 기반 ORM 프레임워크 `Exposed` 를 추천했지만, 오직 Kotlin 만 호환이 되기 때문에 현재 조직에서 사용하는 프로젝트에는 적합하지 않다. 그래서, 나는 `QuerydslZeroOffsetItemReader` 를 선호한다.

### **Optimizing Processor Performance**

Processor 성능 최적화를 위해서 `redis` 를 사용하는 전략이 있다.

강의에서는 reader 에서 `GROUP BY + SUM` 같은 집계 쿼리를 사용하는 것이 성능 저하의 원인이 될 수 있다고 지적했다. 데이터가 적을 때는 간단하게 처리할 수 있지만, 처리할 데이터가 많아질 경우 비효율적이다. 또한, 쿼리가 복잡해질수록 쿼리 튜닝도 복잡해지고 어려워진다. 이 과정에서 쿼리 성능을 최적화하기 위해 과도한 인덱스를 생성하게 되면, 이는 다시 INSERT 나 UPDATE 성능을 저하시킬 수 있다.

반대로, `GROUP BY + SUM` 같은 집계 쿼리없이 전체 데이터를 모두 애플리케이션으로 가져와 처리한다면, 메모리 부족 문제가 발생할 수 있고, 데이터베이스에서 직접 처리하는 것보다 처리 속도도 훨씬 느리다.

이러한 문제를 해결하기 위해 redis 가 사용된다. redis 를 단순히 캐시로만 생각할 수 있지만, 사실 중간 연산을 처리하는 장비로도 활용할 수 있다. redis 는 메모리에서 작업하기 때문에 연산 속도가 매우 빠르고, 메모리 공간도 충분히 확보할 수 있다. 또한, 인메모리 특성상 중간 연산된 데이터가 영구적으로 저장되지 않고 휘발되어 관리가 편하다.

물론, redis 에 데이터를 요청할 때 발생할 수 있는 네트워크 지연이 걱정될 수 있다. 그러나 이를 해결하기 위해 `redis pipeline` 을 사용할 수 있다. redis pipeline 을 사용하면 여러 요청을 한 번에 묶어서 처리할 수 있어, 요청 간의 네트워크 지연을 최소화할 수 있다.

### **Optimizing Writer Performance**

writer 성능 최적화를 위해서는 `명시적 쿼리 사용`, `Bulk Insert` 전략이 있다. 간단하게 알아보자!

**Bulk Insert - 네트워크 지연 최소화**

쿼리를 1개씩 DB 에 요청하는 것이 아닌, N개의 쿼리를 한번에 요청함으로써, `네트워크 지연을 최소화`한다. 하지만, chunk 크기를 너무 크게 설정하면 메모리 사용량이 급격히 증가할 수 있으므로, 애플리케이션의 메모리와 데이터베이스의 처리 능력을 고려하여 적절한 크기를 설정하는 것이 중요하다.

**명시적 쿼리를 사용 - only 필요한 컬럼 Update 처리**

불필요한 컬럼을 포함해서 업데이트를 하면 네트워크 전송량이 증가해 대역폭이 증가하고, `네트워크 지연이 더 발생`할 수 있다. 그리고, `I/O 비용도 증가` 한다.

Update 를 할 때, 데이터는 Lock 이 걸리는데, 불필요한 컬럼을 포함해서 업데이트를 하면 `넓은 범위의 Lock 이 발생`할 수 있기 때문에 동시성 처리에 불리할 수 있다.

그러나, 우리에게 너무 친숙한 JPA 는 위 두가지 전략을 지원하지 않기 때문에 SrpingBatch 에는 적합하지 않는다. 하지만, 여기서 JPA 를 조금 아는 사람은 “@DynamicUpdate 를 통해서 명시적 쿼리를 사용할 수 있다!” 라고 질문을 할 수 있다. 하지만, DynamicUpdate 는 동적쿼리를 생성하는 과정에서 성능저하를 발생할 수 있기 때문에 JPA 를 쓰는 것을 권장하지 않는다!

영상의 뒤에서는 Spring Cloud Data Flow 를 활용해서 효율적인 Batch 자원의 활용과 모니터링에 대해 소개하고 있지만, 나는 영상을 끝까지 들을 수 없었다. 조직에서 Spring Cloud Data Flow 를 적용했지만, 내 손으로 제거했다. 아직은 SCDF 를 도입하기에는 시기가 이른것 같다.

이제 알겠냐?! 내가 왜 과거 컨퍼런스 영상만 보고있는지?! 내 선택은 틀리지 않았어!

![3.png](https://raw.githubusercontent.com/lkhlkh23/lkhlkh23.github.io/master/images/2024-09-22/3.png)
