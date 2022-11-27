# 다중 DB 연결

### 개발환경
- Java 11
- Spring
- Mysql
- Postgresql
- Data JPA
- QueryDSL 

스프링부트로 개발중 외부 DB를 접속해야 할 일이 생겼다.<br>
구축되어있는 데이터베이스는 MySql인데 외부 DB는 Postgresql 이였다.

> build.gradle 의존성 추가

```
runtimeOnly 'org.postgresql:postgresql'
```

> application.yml 추가

```
connpgsql:
    driver-class-name: org.postgresql.Driver
    url: jdbc:postgresql://접속주소:5432/do?charSet=utf-8&prepareThreshold=1
    username: DB접속계정 ID
    password: DB접속계정 패스워드
```


> JdbcMysqlConfig 클래스 추가

```java
@Configuration
@EnableJpaRepositories
        (
                basePackages = "패키지명",
                entityManagerFactoryRef = "mysqlEntityManager",
                transactionManagerRef = "mysqlTransactionManager"
        )
@RequiredArgsConstructor
public class JdbcMysqlConfig {
    private final Environment environment;
    private final JpaProperties jpaProperties;
    private final HibernateProperties hibernateProperties;

    @Primary
    @Bean
    public DataSource mysqlDataSource() {
        DriverManagerDataSource dataSource = new DriverManagerDataSource();
        dataSource.setDriverClassName(environment.getProperty("spring.datasource.driver-class-name"));
        dataSource.setUrl(environment.getProperty("spring.datasource.url"));
        dataSource.setUsername(environment.getProperty("spring.datasource.username"));
        dataSource.setPassword(environment.getProperty("spring.datasource.password"));
        return dataSource;
    }

    @Primary
    @Bean
    public LocalContainerEntityManagerFactoryBean mysqlEntityManager(EntityManagerFactoryBuilder builder) {
        Map<String, Object> properties = hibernateProperties.determineHibernateProperties(jpaProperties.getProperties(), new HibernateSettings());
        return builder
                .dataSource(mysqlDataSource())
                .packages("패키지명")
                .persistenceUnit("mysqlDataSource")
                .properties(properties)
                .build();
    }

    @Primary
    @Bean
    public PlatformTransactionManager mysqlTransactionManager(@Qualifier(value = "mysqlEntityManager") EntityManagerFactory entityManagerFactory) {
        JpaTransactionManager jpaTransactionManager = new JpaTransactionManager();
        jpaTransactionManager.setEntityManagerFactory(entityManagerFactory);
        return jpaTransactionManager;
    }
}
```


JdbcPgsqlConfig 클래스 추가

```java
@Configuration
@EnableJpaRepositories
        (
                basePackages = "패키지명",
                entityManagerFactoryRef = "pgsqlEntityManager",
                transactionManagerRef = "pgsqlTransactionManager"
        )
@RequiredArgsConstructor
public class JdbcPgsqlConfig {
        private final Environment environment;
        private final JpaProperties jpaProperties;
        private final HibernateProperties hibernateProperties;

        @Bean
        public DataSource pgsqlDataSource() {
                DriverManagerDataSource dataSource = new DriverManagerDataSource();
                System.out.println("environment.getProperty(\"spring.connpgsql.driver-class-name\") = " + environment.getProperty("spring.connpgsql.driver-class-name"));
                dataSource.setDriverClassName(environment.getProperty("spring.connpgsql.driver-class-name"));
                dataSource.setUrl(environment.getProperty("spring.connpgsql.url"));
                dataSource.setUsername(environment.getProperty("spring.connpgsql.username"));
                dataSource.setPassword(environment.getProperty("spring.connpgsql.password"));
                return dataSource;
        }

        @Bean
        public LocalContainerEntityManagerFactoryBean pgsqlEntityManager(EntityManagerFactoryBuilder builder) {
                Map<String, Object> properties = hibernateProperties.determineHibernateProperties(jpaProperties.getProperties(), new HibernateSettings());
                return builder
                        .dataSource(pgsqlDataSource())
                        .packages("패키지명")
                        .persistenceUnit("pgsqlDataSource")
                        .properties(properties)
                        .build();
        }

        @Bean
        public PlatformTransactionManager pgsqlTransactionManager(@Qualifier(value = "pgsqlEntityManager") EntityManagerFactory entityManagerFactory) {
                JpaTransactionManager jpaTransactionManager = new JpaTransactionManager();
                jpaTransactionManager.setEntityManagerFactory(entityManagerFactory);
                return jpaTransactionManager;
        }
}
```
