# Multi Datasource 적용기

회사에서 프로젝트를 하다가 Access Log 를 수집하는 것으로 결정했다.   
Access Log는 웹/앱 유의미한 모든 동작을 수집하는 데이터이기 떄문에 짧은 시간에 많은 데이터가 쌓인다.
따라서 별도의 Database에 쌓을려고 한다.

[Datasource 설정법](https://docs.spring.io/spring-boot/docs/3.2.0/reference/html/howto.html#howto.data-access.configure-custom-datasource)

Multi Datasource 를 적용하기 위해서는 수동으로 설정을 해주어야 한다.
기본설정처럼 datasource.url 로 설정을 하면
`Failed to instantiate [org.springframework.orm.jpa.LocalContainerEntityManagerFactoryBean]: Factory method 'entityManagerFactory' threw exception with message: dataSource or dataSourceClassName or jdbcUrl is required.`
jdbcUrl 이 필요하다는 에러가 발생한다.

```yaml
spring:
  original:
    datasource:
      jdbc-url: jdbc:h2:mem:original
      username: original
      password: original1234
  accessLog:
    datasource:
      jdbc-url: jdbc:h2:mem:accesslog
      username: accesslog
      password: accesslog1234
```

```Kotlin
@Configuration
@EnableTransactionManagement // JPA 트랜잭션 사용
@EnableJpaRepositories(basePackages = ["io.github.kimjunghak.member.repository"],
    entityManagerFactoryRef = "originalEntityManagerFactory",
    transactionManagerRef = "originalTransactionManager")
class OriginalDBConfig {

    @Bean
    @Primary
    @ConfigurationProperties(prefix = "spring.original.datasource")
    fun originalDatasource(): DataSource {
        return DataSourceBuilder.create().build()
    }

    @Bean
    @Primary
    fun originalEntityManagerFactory(entityManagerFactoryBuilder: EntityManagerFactoryBuilder,
                                     @Qualifier("originalDatasource") originalDataSource: DataSource): LocalContainerEntityManagerFactoryBean {
        val properties = HashMap<String, String>()
        properties["hibernate.hbm2ddl.auto"] = "create"
        return entityManagerFactoryBuilder
            .dataSource(originalDataSource)
            .packages("io.github.kimjunghak.member.entity")
            .properties(properties)
            .build()
    }

    @Bean
    @Primary
    fun originalTransactionManager(@Qualifier("originalEntityManagerFactory") originalEntityManagerFactory: LocalContainerEntityManagerFactoryBean): PlatformTransactionManager {
        return JpaTransactionManager(originalEntityManagerFactory.`object`!!)
    }
}

===

@Configuration
@EnableTransactionManagement // JPA 트랜잭션 사용
@EnableJpaRepositories(
    basePackages = ["io.github.kimjunghak.accesslog.repository"],
    entityManagerFactoryRef = "accessLogEntityManagerFactory",
    transactionManagerRef = "accessLogTransactionManager"
)
class AccessLogDBConfig {

    @Bean
    @ConfigurationProperties(prefix = "spring.access-log.datasource")
    fun accessLogDatasource(): DataSource {
        return DataSourceBuilder.create().build()
    }

    @Bean
    fun accessLogEntityManagerFactory(entityManagerFactoryBuilder: EntityManagerFactoryBuilder,
                                      @Qualifier("accessLogDatasource") accessLogDataSource: DataSource): LocalContainerEntityManagerFactoryBean {
        val properties = HashMap<String, String>()
        properties["hibernate.hbm2ddl.auto"] = "create"
        return entityManagerFactoryBuilder
            .dataSource(accessLogDataSource)
            .packages("io.github.kimjunghak.accesslog.entity")
            .properties(properties)
            .build()
    }

    @Bean
    fun accessLogTransactionManager(@Qualifier("accessLogEntityManagerFactory") accessLogEntityManagerFactory: LocalContainerEntityManagerFactoryBean): PlatformTransactionManager {
        return JpaTransactionManager(accessLogEntityManagerFactory.`object`!!)
    }
}
```

> 같은 타입의 Bean이 여러개 일 때 @Qualifier 를 잘 사용해야 한다. 그렇지 않으면 잘못된 것이 주입될 수 있으니 주의하자.

```Kotlin
val properties = HashMap<String, String>()
properties["hibernate.hbm2ddl.auto"] = "create"
```
Multi Datasource를 적용하면 spring.jpa.hibernate.ddl-auto가 먹히지 않는다. 그래서 따로 지정해 준 것

Spring Boot 2.0부터 사용되는 HikariCP([참고](https://github.com/spring-projects/spring-boot/wiki/Spring-Boot-2.0-Migration-Guide#configuring-a-datasource))는 jdbc-url을 필요로 한다.   
그래서 datasource.url 을 datasource.jdbc-url로 변경해야 한다.


> 다른 방법(url 그대로 사용하기)
> 
[Two Datasource 설정법](https://docs.spring.io/spring-boot/docs/3.2.0/reference/html/howto.html#howto.data-access.configure-two-datasources)

예전 기억을 더듬어 가며 하나씩 해보았는데 찾아보니 더 편한 방법이 있었다.   
DatasourceProperties를 활용하는 것이다.

```Kotlin
@Bean
@ConfigurationProperties(prefix = "spring.datasource")
fun datasourceProperties(): DataSourceProperties {
    return DataSourceProperties()
}

@Bean
fun datasource(): DataSource {
    return datasourceProperties().initializeDataSourceBuilder().type(HikariDataSource::class.java).build()
}
```
DatasourceProperties에서 url로 값을 주입받아서 HikariDatasource를 만들 때 Spring에서 주입해주는 방식인가보다.

처음 백엔드 개발 시작하고 몇달 안돼서 Multi Datasource 적용할 때는 엄청 어려워서 며칠이 걸렸는데 지금 다시 보니 그렇게 어렵지 않았네..   
확실히 성장했나보다.