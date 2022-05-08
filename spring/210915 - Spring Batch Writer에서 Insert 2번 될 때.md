## 210915 - Spring Batch Writer에서 Insert 2번 될 때

사내 배치를 구현 중 겪은 이슈다.<br>예를 들어 Account, AccountTransaction인 엔티티가 있다 치자.
그리고 엔티티 관계는 다음과 같다.

##### Account

```java
@Entity
@AllArgsConstructor
@NoArgsConstructor
@Builder
@Getter
@ToString(exclude = "accountTransactionSet")
@EqualsAndHashCode(of = {"accountId", "accountNumber"}, callSuper = false)
@Slf4j
public class Account extends BaseEntity {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long accountId;

    @OneToMany(fetch = FetchType.LAZY, cascade = {CascadeType.PERSIST, CascadeType.MERGE}, mappedBy = "account", orphanRemoval = true)
    @Builder.Default
    private final Set<AccountTransaction> accountTransactionSet = new LinkedHashSet<>();
}
```

##### AccountTransaction

```java
@Entity
@AllArgsConstructor
@NoArgsConstructor
@Builder
@Getter
@ToString
@EqualsAndHashCode(of = {"note", "transactionType" ,"amount", "transactionDate"}, callSuper = false)
public class AccountTransaction extends BaseEntity {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long accountTransactionId;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "account_id")
    private Account account;
}
```

보다시피 Account에 영속성 전이(Persist, Merge)를 달아놨다.<br>따라서 Account에 딸린 AccountTransaction 엔티티를 수정한 뒤, Account만 save, update를 해도 AccountTransaction이 같이 반영이 될 것이다.<br>이렇게 잘 사용하고 있다가.. 외부 API를 호출 한 다음, 거래내역을 수정하는 배치를 짤 일이 생겨서 스프링 배치로 해당 작업을 구현하였다.

```java
    private static final int CHUNK_SIZE = 10;

    @Bean
    public Job simpleBatchJob() {
        return jobBuilderFactory.get(SIMPLE_JOB.name())
                .listener(jobListener)
                .incrementer(new UniqueRunIdIncrementer())
                .start(simpleBatchStep())
                .build();
    }

    @Bean
    @JobScope
    public Step simpleBatchStep() {
        return stepBuilderFactory.get(SIMPLE_STEP.name())
                .transactionManager(apiTransactionManager)
                .<Account, Account>chunk(CHUNK_SIZE)
                .reader(simpleReader())
                .processor(simpleProcessor())
                .writer(simpleWriter())
                .build();
    }

				// Reader

		@Bean
    public ItemProcessor<Account, Account> simpleProcessor() {
      	// ...processor
    }

		@Bean
    public ItemWriter<Account> simpleWriter() {
        JpaItemWriter<Account> jpaItemWriter = new JpaItemWriter<>();
        jpaItemWriter.setEntityManagerFactory(entityManagerFactory);
        return jpaItemWriter;
    }
```

Reader에서는 Account을 읽은 뒤, <br>Processor에서 외부 API를 호출한 뒤 Account에 딸려있는 AccountTransaction에 추가하고<br>Writer에서는 다시 Account를 업데이트하는 그런 배치다.<br>배치를 돌린 뒤 디비를 보니.. 새로운 AccountTransaction이 들어가긴 하는데 문제가 두번씩 들어간다..

멘붕이 터졌다... 안 들어가는 것도 아니고 왜 두 번 들어가는걸까...<br>

처음에는 영속성 전이 쪽을 의심했다.. 왜냐면 아무래도 놓치기 쉬운 부분이니까. (사실 이 문제와 아무런 상관이 없다)<br>중간에 AccountTransaction을 추가할 때 만들어둔 Convenient Method를 안 썼나?<br>(Convenient Method가 무엇이고 왜 쓰는지 궁금할 수도 있다.. 기초적인 부분이지만 중요하다 생각해서 이 점에 대해서는 따로 정리를 해야겠다.)<br>보니까 너무 잘 쓰고 있었다.. 긴가민가해서 오히려 영속성을 끊고 돌려보았다 ㅋㅋㅋㅋ...

그러니 이번엔 account_id가 null인 채로 들어간다..

하... 결국 로그를 찍어보면서 디버그를 한 결과 결론을 내릴 수 있었다.<br>그것은 Processor에서 Account에 AccountTransaction을 추가하면서 insert 1번,<br>그리고 Writer에서 Account를 insert하면서 또 영속성 전이가 걸린 AccountTransaction이 같이 insert 되기에 2번<br>그래서 총 2번 insert가 되었었다..

어... 어떻게 해야할까...

그렇다면 Account를 Writer로 넘기면 안되고, AccountTransaction을 넘기면 되지 않을까?라는 생각이 들었다.

다행히 Account의 값은 배치를 돌리면서 변하지 않기에.. AccountTransaction에서 Account에 cascade 옵션을 줄 필요(수정할 필요)는 없었다.<br>문제는 일대다 관계의 AccountTransaction을 List로 넘겨야 하는 것인데.. 

그래서 일단 다음과 같이 수정해보았다.

```java
		@Bean
    @JobScope
    public Step simpleBatchStep() {
        return stepBuilderFactory.get(SIMPLE_STEP.name())
                .transactionManager(apiTransactionManager)
                .<Account, List<AccountTransaction>>chunk(CHUNK_SIZE)
                .reader(simpleReader())
                .processor(simpleProcessor())
                .writer(simpleWriter())
                .build();
    }

   // Reader

		@Bean
    public ItemProcessor<Account, List<AccountTransaction>> simpleProcessor() {
      	// ...processor
    }

		@Bean
    public ItemWriter<List<AccountTransaction>> simpleWriter() {
        JpaItemWriter<List<AccountTransaction>> jpaItemWriter = new JpaItemWriter<>();
        jpaItemWriter.setEntityManagerFactory(entityManagerFactory);
        return jpaItemWriter;
    }
```



컴파일도 잘 되었고.. 완벽하다라고 생각하던 그 때 런타임 에러가 터졌다..<br>

![image-20210922074304800](https://tva1.sinaimg.cn/large/008i3skNgy1gup0m7xoboj624y0cgjy202.jpg)

수정하면서 설마 이게 되나 싶었는데 역시나 안된다<br>방법이 없나 싶어서 구글링을 해보니 역시 갓졸두님이 관련해 글을 쓴게 보였다. [갓졸두님 게시글](https://jojoldu.tistory.com/140)

나름대로 정리하자면, ItemWriter 인터페이스의 write 메소드를 보면 다음과 같다<br>![스크린샷 2021-09-22 오전 7.33.43](https://tva1.sinaimg.cn/large/008i3skNgy1gup0cj15x2j611c0cymyi02.jpg)

ItemWriter는 제네릭 클래스인데 ItemWriter<T>는 파라미터로 T(혹은 T의 자식)의 리스트를 받는다.<br>따라서 ItemWriter<List<AccountTransaction>>은 저 items가 List<List<AccountTransaction>>일 것이다.



다음은 JpaItemWriter 클래스다.

![스크린샷 2021-09-22 오전 7.44.59](https://tva1.sinaimg.cn/large/008i3skNgy1gup0o841adj60u00u10wk02.jpg)

익셉션이 터진 113번 라인을 보면 다음과 같다.

> entityManager.contains(item) 

items는 여기서 List<List<AccountTransaction>>일테고, item은 그렇다면 List<AccountTransaction>일텐데..<br>또 EntityManager.contains 메소드를 보면 다음과 같다.

![스크린샷 2021-09-22 오전 7.47.00](https://tva1.sinaimg.cn/large/008i3skNgy1gup0qd3lwrj61680hedic02.jpg)

파라미터가 엔티티 객체여야한다고 한다.<br>그런데 item은 엔티티의 리스트였으니.. 엔티티매니저에서 오류가 났던 것이다.

원인은 알았으니.. 이제 이에 맞춰 수정을 해야한다.

JpaItemWriter를 상속받는 JpaItemListWriter를 만들자

```json
public class JpaItemListWriter<T> extends JpaItemWriter<List<T>> {
    private JpaItemWriter<T> jpaItemWriter;

    public JpaItemListWriter(JpaItemWriter<T> jpaItemWriter) {
        this.jpaItemWriter = jpaItemWriter;
    }

    @Override
    public void write(List<? extends List<T>> items) {
        List<T> collect = items.stream()
                .flatMap(Collection::stream)
                .collect(Collectors.toList());
        jpaItemWriter.write(collect);
    }

}
```

write 메소드를 상속받는 새로운 메소드를 만들자.<br>파라미터로 List<List<T>>를 받아서, 그 List를 flatMap으로 List<T>를 만들어주자.<br>그리고 생성자에서 받은 jpaItemWriter로 다시 write를 호출하면 된다.<br>Job의 Wrtier는 다음과 같다.

```java
		@Bean
    public JpaItemListWriter<AccountTransaction> simpleWriter() {
        JpaItemWriter<AccountTransaction> jpaItemWriter = new JpaItemWriter<>();
        jpaItemWriter.setEntityManagerFactory(entityManagerFactory);
        JpaItemListWriter<AccountTransaction> accountTransactionJpaItemListWriter = new JpaItemListWriter<>(jpaItemWriter);
        accountTransactionJpaItemListWriter.setEntityManagerFactory(entityManagerFactory);
        return accountTransactionJpaItemListWriter;
    }
```

JpaItemWriter를 새로 생성해 엔티티 매니저를 설정해준 뒤,<br>생성한 JpaItemWriter를 통해 JpaItemListWriter를 생성해준다.<br>그리고 JpaItemListWriter도 마찬가지로 엔티티 매니저를 설정한 뒤, 리턴하면 된다.