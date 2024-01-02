---
layout: post
title: Transaction read only master-replication 분기 처리
subtitle: master, replication db 로 분기처리 로직 개발
excerpt_image: https://raw.githubusercontent.com/lkhlkh23/lkhlkh23.github.io/master/images/2023-12-26/banner.png
categories: jpa
tags: [jpa, transaction]
---

![banner](https://raw.githubusercontent.com/lkhlkh23/lkhlkh23.github.io/master/images/2023-12-26/banner.png)

`@Transaction (readonly=true)` 설정만 있다면, 짠! 하고 내부적으로 Replication DB 로 리디렉션 처리해줄 것으로 생각했다. 하지만, 내부적인 개발 설정이 별도로 필요하다. 관련된 코드는 [github](https://github.com/lkhlkh23/practice-transaction-route) 를 통해 확인 가능하다. 참고로 DB 는 Docker 를 이용했다.

Master DB 와 Replication DB 를 다음과 같이 설정을 했다.

```yaml
spring:
  datasource:
    master:
      hikari:
        username: root
        password: 1234
        driver-class-name: com.mysql.cj.jdbc.Driver
        jdbc-url: jdbc:mysql://localhost:3306/master_db
    slave:
      hikari:
        username: root
        password: 1234
        driver-class-name: com.mysql.cj.jdbc.Driver
        jdbc-url: jdbc:mysql://localhost:3306/slave_db
```

DataSource 에 대한 설정은 아래와 같다. 

```java
@Configuration
public class DataSourceConfig {

	@Bean("masterDataSource")
	@ConfigurationProperties(prefix = "spring.datasource.master.hikari")
	public DataSource masterDataSource() {
		return DataSourceBuilder.create()
								.type(HikariDataSource.class)
								.build();
	}

	@Bean("slaveDataSource")
	@ConfigurationProperties(prefix = "spring.datasource.slave.hikari")
	public DataSource slaveDataSource() {
		return DataSourceBuilder.create()
								.type(HikariDataSource.class)
								.build();
	}

	@Bean("switchDataSource")
	@DependsOn({"masterDataSource", "slaveDataSource"})
	public DataSource switchDataSource(@Qualifier("masterDataSource") final DataSource masterDataSource,
									                   @Qualifier("slaveDataSource") final DataSource slaveDataSource) {
		final DataSourceSwitch sourceSwitch = new DataSourceSwitch();
		final Map<Object, Object> dataSources = new HashMap<>() {
			{
				put("master", masterDataSource);
				put("slave", slaveDataSource);
			}
		};

		sourceSwitch.setTargetDataSources(dataSources);
		sourceSwitch.setDefaultTargetDataSource(masterDataSource);

		return sourceSwitch;
	}

	@Bean
	@Primary
	@DependsOn("switchDataSource")
	public LazyConnectionDataSourceProxy routingDataSource(@Qualifier("switchDataSource") final DataSource switchDataSource) {
		return new LazyConnectionDataSourceProxy(switchDataSource);
	}

	public static class DataSourceSwitch extends AbstractRoutingDataSource {

		@Override
		protected Object determineCurrentLookupKey() {
			return (TransactionSynchronizationManager.isCurrentTransactionReadOnly()) ? "slave" : "master";
		}
	}

}
```

여기서 중요하게 봐야할 것은 **AbstractRoutingDataSource** 를 상속받은 DataSourceSwitch 클래스이다.

AbstractRoutingDataSource 에서 Map 에 등록된 DataSource 중에서 원하는 DataSource 를 찾는 로직은 아래와 같이 구현되있다.

```
protected DataSource determineTargetDataSource() {
		Assert.notNull(this.resolvedDataSources, "DataSource router not initialized");
		Object lookupKey = determineCurrentLookupKey();
		DataSource dataSource = this.resolvedDataSources.get(lookupKey);
		if (dataSource == null && (this.lenientFallback || lookupKey == null)) {
			dataSource = this.resolvedDefaultDataSource;
		}
		if (dataSource == null) {
			throw new IllegalStateException("Cannot determine target DataSource for lookup key [" + lookupKey + "]");
		}
		return dataSource;
	}
```

determineCurrentLookupKey() 메소드는 Map 에 등록된 DataSource 를 get 할 수 있는 Key 를 가져오는 메소드이다. DataSourceSwitch 에서는 `@Transaction (readonly=true)` 가 되있는 설정에 대해서 Replication DB 를 이용해서 조회할 수 있도록 설정했다.

만약, Key 에 해당되는 DataSource 가 없다면 디폴트로 설정된 Master DB 를 이용하도록 설정했다.

다음으로 가장 중요한 것은 **LazyConnectionDataSourceProxy** 이다.

Spring 에서는 트랜잭션에 진입하는 순간, DataSource 의 커넥션을 가져온다.  **LazyConnectionDataSourceProxy** 이 없다면,  Replication DB 와 Master DB 를 써야할지 조건에 따라 결정하기도 전에, 이미 디폴트로 설정된 DataSource 를 결정하고 커넥션을 가져오게 된다는 것이다.

그렇기 때문에 **LazyConnectionDataSourceProxy** 이용해서 커넥션을 가져오는 시점을 늦춰야만 한다. 

다음과 같은 설정을 모두 완료하게 되면, 아래 Service 에서 내가 의도한 DataSource 를 통해 처리할 수 있게 된다. 테스트 코드를 통해서 확인을 했을 때, `@Transaction (readonly=true)` 가 있을 때, Replication DB 인 slave_db 를 통해서 조회한 것을 확인할 수 있었다. 반대로, false 경우에는 Master DB 를 통해서 처리된 것을 확인했다.

```java
@Service
public class UserServiceImpl implements UserService {

	@Autowired
	private UserRepository repository;

	@Override
	@Transactional(readOnly = true)
	public List<UserInfo> getUsers() {
		return repository.findAll();
	}

	@Override
	@Transactional(readOnly = false)
	public void addUser(final UserInfo user) {
		repository.save(user);
	}

}
```

옵션값 하나만 추가하면 자동으로 Master DB 와 Replication DB 를 분기처리할 수 있다고 생각했지만, 현실은 쉽지 않았다. 우리의 삶이란 늘 쉽지 않다.. 이 0 과 1 로 이루어진 세상에 삶이 잘 녹아져있다는 것을 또 한번 깨달았다.
