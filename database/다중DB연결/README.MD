# 다중 DB 연결

### 개발환경
- Java 11
- Spring
- Mysql
- Postgresql
- Data JPA
- QueryDSL 


스프링부트로 개발중 외부 DB를 접속해야 할 일이 생겼습니다.<br>
최근에는 대부분 API로 데이터를 주고 받는데 연결해야하는 시스템(그룹웨어)에서는 제공이 어렵다고 답변이 왔습니다. 

(오히려 좋아 짤)

현재 프로젝트에 구축되어있는 데이터베이스는 MySql인데 추가로 연결해야 하는 외부 DB는 Postgresql 입니다.

현재 프로젝트에서는 Spring Data JPA, QueryDSL을 사용중인데 어떻게 다중으로 데이터베이스에 접속을 했는지 알아보죠!

> build.gradle 의존성 추가

```
runtimeOnly 'org.postgresql:postgresql'
```

이부분은 사용 될 sql에 따라 달라집니다.

이번 포스팅에서는 postgresql이므로 의존성을 추가해줍니다.

> application.yml 추가

```
connpgsql:
    driver-class-name: org.postgresql.Driver
    url: jdbc:postgresql://접속주소:5432/do?charSet=utf-8&prepareThreshold=1
    username: DB접속계정 ID
    password: DB접속계정 패스워드
```

postgresql jdbc 드라이버를 추가합니다.


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

한개의 데이터베이스를 사용한다면 config를 굳이 만들지 않아도 됩니다.

하지만 두 개 이상의 데이터베이스를 사용한다면 어떤 Jdbc를 사용할지 명시를 해줘야 하기 때문에 config 파일로 분기 처리를 합니다.

먼저 mysql config 입니다.


> JdbcPgsqlConfig 클래스 추가

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

Postgresql의 config 코드입니다. 

각각 커스텀 해야 할 내용을 각각 넣어 줍시다.

이제 jdbc config 설정이 끝났으니 실제 사용을 하기 위해서

SpringDataJPA와 Repository 코드를 살펴보겠습니다.