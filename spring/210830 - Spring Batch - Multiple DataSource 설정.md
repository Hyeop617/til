## Spring Batch - Multiple Datasource일 때 설정

```java
@Configuration
@EnableTransactionManagement
@EnableJpaRepositories(
        basePackages = {"com.example.seoul.common.domain"},
        entityManagerFactoryRef = "apiEntityManager",
        transactionManagerRef = "apiTransactionManager"
)
@PropertySource({"classpath:application-${spring.profiles.active}.yml"})
@RequiredArgsConstructor
public class ApiDatabaseConfig {

    private final Environment env;

    @Bean
    @Primary
    @ConfigurationProperties(prefix="spring.datasource")
    public DataSource apiDataSource() {
        return DataSourceBuilder.create().build();
    }

    @Bean
    @Primary
    public LocalContainerEntityManagerFactoryBean apiEntityManager() {
        final LocalContainerEntityManagerFactoryBean em = new LocalContainerEntityManagerFactoryBean();
        em.setDataSource(apiDataSource());
        em.setPackagesToScan("com.example.seoul.common.domain");

        final HibernateJpaVendorAdapter vendorAdapter = new HibernateJpaVendorAdapter();
        em.setJpaVendorAdapter(vendorAdapter);
        final HashMap<String, Object> properties = new HashMap<String, Object>();
        properties.put("hibernate.hbm2ddl.auto", env.getProperty("spring.jpa.hibernate.ddl-auto"));
        properties.put("hibernate.dialect", "org.hibernate.dialect.PostgreSQL10Dialect");
        properties.put("hibernate.implicit_naming_strategy", "org.springframework.boot.orm.jpa.hibernate.SpringImplicitNamingStrategy");
        properties.put("hibernate.physical_naming_strategy", "org.springframework.boot.orm.jpa.hibernate.SpringPhysicalNamingStrategy");
        properties.put("hibernate.show_sql", true);
        properties.put("hibernate.format_sql", true);
//        properties.put("hibernate.generate_statistics", true);
        em.setJpaPropertyMap(properties);

        return em;
    }

    @Bean
    @Primary
    public PlatformTransactionManager apiTransactionManager() {
        final JpaTransactionManager transactionManager = new JpaTransactionManager();
        transactionManager.setEntityManagerFactory(apiEntityManager().getObject());
        return transactionManager;
    }
}

```

PlatformTransactionManager 설정을 한다 (Bean으로 등록)<br>여기까지는 기본 DataSource 설정과 같다.

```java
@Slf4j
@RequiredArgsConstructor
@Configuration
public class ExpiredPointJobExample extends DefaultBatchConfigurer {
  
    private final EntityManagerFactory apiEntityManager;
  
		@Bean
    @JobScope
    public Step expirePointStep() {
        return stepBuilderFactory.get("expiredPointStep")
                .transactionManager(getTransactionManager())				// 중요!!!!!!!
                .<PointHistory, PointHistory>chunk(CHUNK_SIZE)
                .reader(expiredPointReader(null))
                .processor(expiredPointProcessor())
                .writer(expiredPointWriter())
                .build();
    }
  	
  	// Job, reader, processor 생략...
  
  	@Bean
    public JpaItemWriter<PointHistory> expiredPointWriter() {
        JpaItemWriter<PointHistory> jpaItemWriter = new JpaItemWriter<>();
        jpaItemWriter.setEntityManagerFactory(apiEntityManager);
        return jpaItemWriter;
    }
  
    @Override
    public PlatformTransactionManager getTransactionManager() {
        return new JpaTransactionManager(apiEntityManager);
    }
}
```

DefaultBatchConfigurer의 getTransactionManager을 오버라이딩한다.<br>그리고 Step에서 StepBuilderFactory에 오버라이딩한 PlatformTransactionManager을 넣어줘야한다!!!!<br>저 PlatformTransactionManager을 주입안하니까 계속 writer에서 트랜잭션 유지가 안되는지 계속 이 오류가 떴었다.

```java
javax.persistence.TransactionRequiredException: no transaction is in progress
```

이거 하나 때문에 3시간을 삽질했다..<br>https://silvernine.me/wp/?p=857에 따르면 기본으로 transactionManager Bean을 주입받기 때문에 그렇다는데...<br>apiTransactionManager Bean에 @Primary 설정을 해서 상관없을 줄 알았다.