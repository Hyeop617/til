## 220430 - @Transactional

스프링을 사용하면서 무조건 사용하게되는 스프링의 Transactional 어노테이션.
이 어노테이션에 대해서 제대로 정리해본 적이 없어서 정리를 해보고자 한다.

영한님의 JPA책에 따르면 다음과 같다.

>스프링 프레임워크는 이 어노테이션이 붙어 있는 메소드나 클래스에 트랜잭션을 적용한다.
>외부에서 이 클래스의 메소드를 호출할 때 트랜잭션을 시작하고 메소드를 종료할 때 트랜잭션을 커밋한다. 
>만약 예외가 발생하면 트랜잭션을 롤백한다.
>
>@Transactional은 RuntimeException과 그 자식들인 Unchecked Exception만 롤백한다.
>만약 체크 예외가 발생해도 롤백하고 싶다면 @Transactional(rollbackFor = Exception.class)처럼 롤백할 예외를 지정해야한다.

스프링 공식 도큐멘테이션을 보면 다음과 같다.

스프링에서는 트랜잭션을 크게 선언적 방식과 프로그래밍 방식 2가지 방법으로 선언할 수 있다.

선언적 방식 (AOP)

	- @Transactional 어노테이션
	- XML 파일

프로그래밍 방식도 2가지가 있다.

- TransactionTemplate or TransactionOperator
- TransactionManager 구현체로 직접

****

일단 @Transactional 어노테이션을 뜯어보자.

```java
/**
 * Describes a transaction attribute on an individual method or on a class.
 *
 * <p>When this annotation is declared at the class level, it applies as a default
 * to all methods of the declaring class and its subclasses. Note that it does not
 * apply to ancestor classes up the class hierarchy; inherited methods need to be
 * locally redeclared in order to participate in a subclass-level annotation. For
 * details on method visibility constraints, consult the
 * <a href="https://docs.spring.io/spring-framework/docs/current/reference/html/data-access.html#transaction">Transaction Management</a>
 * section of the reference manual.
 *
 * <p>This annotation type is generally directly comparable to Spring's
 * {@link org.springframework.transaction.interceptor.RuleBasedTransactionAttribute}
 * class, and in fact {@link AnnotationTransactionAttributeSource} will directly
 * convert the data to the latter class, so that Spring's transaction support code
 * does not have to know about annotations. If no custom rollback rules apply,
 * the transaction will roll back on {@link RuntimeException} and {@link Error}
 * but not on checked exceptions.
 *
 * <p>For specific information about the semantics of this annotation's attributes,
 * consult the {@link org.springframework.transaction.TransactionDefinition} and
 * {@link org.springframework.transaction.interceptor.TransactionAttribute} javadocs.
 *
 * <p>This annotation commonly works with thread-bound transactions managed by a
 * {@link org.springframework.transaction.PlatformTransactionManager}, exposing a
 * transaction to all data access operations within the current execution thread.
 * <b>Note: This does NOT propagate to newly started threads within the method.</b>
 *
 * <p>Alternatively, this annotation may demarcate a reactive transaction managed
 * by a {@link org.springframework.transaction.ReactiveTransactionManager} which
 * uses the Reactor context instead of thread-local variables. As a consequence,
 * all participating data access operations need to execute within the same
 * Reactor context in the same reactive pipeline.
 *
 */
```

![스크린샷 2022-04-30 오후 2.53.20](https://tva1.sinaimg.cn/large/e6c9d24egy1h1rpdqg4zej20kb0beq55.jpg)

>개별 메서드 또는 클래스의 트랜잭션 속성을 설명합니다.
>
>**이 어노테이션이 클래스에서 선언되면 선언 클래스와 하위 클래스의 모든 메서드에 기본값으로 적용됩니다. 클래스 계층 구조의 조상 클래스에는 적용되지 않습니다; 서브클래스 레벨 주석에 참여하려면 상속된 메서드를 로컬로 다시 선언해야 합니다. 방법 가시성 제약에 대한 자세한 내용은 스프링 레퍼런스의 [트랜잭션 관리(섹션 관리)](https://docs.spring.io/spring-framework/docs/current/reference/html/data-access.html#transaction)를 참조하십시오.**
>
>이 어노테이션 유형은 일반적으로 org.springframework.transaction.interceptor.RuleBasedTransactionAttribute 클래스와 직접 비교할 수 있습니다. 그리고 사실 AnnotationTransactionAttributeSource는 데이터를 후자의 클래스로 직접 변환하므로 Spring의 트랜잭션 지원 코드는 주석에 대해 알 필요가 없습니다. 사용자 지정 롤백 규칙이 적용되지 않으면, 트랜잭션은 RuntimeException 및 Error에서 롤백되지만 확인된 예외에서는 롤백되지 않습니다.
>
>이 주석의 다른 속성의 의미론에 대한 구체적인 정보는 TransactionDefinition 및 TransactionAttribute javadocs를 참조하십시오.
>
>이 어노테이션은 일반적으로 PlatformTransactionManager가 관리하는 스레드 바인딩 트랜잭션에서 작동하여 트랜잭션을 현재 실행 스레드 내의 모든 데이터 액세스 작업에 노출합니다.
>**참고: 이것은 메서드 내에서 새로 시작된 스레드로 전파되지 않습니다.**
> 
>대안으로, 이 어노테이션은 thread-local 변수 대신 리액터 컨텍스트를 사용하는 ReactiveTransactionManager가 관리하는 반응형 트랜잭션을 구분할 수 있습니다. 결과적으로, 모든 참여 데이터 액세스 작업은 동일한 리액티브 파이프라인에서 동일한 리액터 컨텍스트 내에서 실행되어야 합니다.

이해가 안되지만 정리를 해보자.
PlatformTransactionManager에서 트랜잭션을 관리하는 것을 알 수 있었다.

그렇다면 여기서 TransactionManager에 대해서 한 번 알아보자.
일단 TransactionManager는 마커 인터페이스다. 따라서 실질적인 것은 TransactionManager의 부모로 올라가야한다.

![스크린샷 2022-05-01 오전 1.39.11](https://tva1.sinaimg.cn/large/e6c9d24egy1h1s81s0ydaj20m503p3yn.jpg)

다시 PlatformTransactionManager에 대해서 보자.

```java
/**
 * This is the central interface in Spring's imperative transaction infrastructure.
 * Applications can use this directly, but it is not primarily meant as an API:
 * Typically, applications will work with either TransactionTemplate or
 * declarative transaction demarcation through AOP.
 *
 * <p>For implementors, it is recommended to derive from the provided
 * {@link org.springframework.transaction.support.AbstractPlatformTransactionManager}
 * class, which pre-implements the defined propagation behavior and takes care
 * of transaction synchronization handling. Subclasses have to implement
 * template methods for specific states of the underlying transaction,
 * for example: begin, suspend, resume, commit.
 */
public interface PlatformTransactionManager extends TransactionManager {
		TransactionStatus getTransaction(@Nullable TransactionDefinition definition) throws TransactionException;
  
  	void commit(TransactionStatus status) throws TransactionException;

		void rollback(TransactionStatus status) throws TransactionException;
}
```

![스크린샷 2022-05-01 오전 12.32.13](https://tva1.sinaimg.cn/large/e6c9d24egy1h1s647rnolj20la0ap764.jpg)

![스크린샷 2022-05-01 오전 1.40.12](https://tva1.sinaimg.cn/large/e6c9d24egy1h1s82suaugj20l4079dgo.jpg)

PlatformTransactionManager를 보니 드디어 의미 있는 메소드가 나왔다.
getTransaction, commit, rollback은 말 그대로 트랜잭션을 얻고, 커밋하고, 롤백하는 메소드로 보인다.
PlatformTransactionManager의 구현체로는 AbstractPlatformTransactionManager와 ChainedTransactionManager가 있다.

트랜잭션 동기화 관리와 트랜잭션 전파 전략을 미리 구현한 AbstractPlatformTransactionManager와 관계가 있다고 한다.
AbstractPlatformTransactionManager를 보자.

(참고로 ChainedTransactionManager는 TransactionManager 여러 개를 묶어서 한 TransactionManager로 관리하는 Bean이다.)
(이와 관련해서는 JtaTransactionManager도 보면서 같이 비교해야한다. 간략한 두 방법의 차이는 Chained는 단순히 여러 TransactionManager에서 관리하는 트랜잭션을 순차적으로 처리하는 것이고 JTA는 실제로 한 트랜잭션에서 처리하게 해주는 것이다.)

```java
 * Abstract base class that implements Spring's standard transaction workflow,
 * serving as basis for concrete platform transaction managers like
 * {@link org.springframework.transaction.jta.JtaTransactionManager}.
 *
 * <p>This base class provides the following workflow handling:
 * <ul>
 * <li>determines if there is an existing transaction;
 * <li>applies the appropriate propagation behavior;
 * <li>suspends and resumes transactions if necessary;
 * <li>checks the rollback-only flag on commit;
 * <li>applies the appropriate modification on rollback
 * (actual rollback or setting rollback-only);
 * <li>triggers registered synchronization callbacks
 * (if transaction synchronization is active).
 * </ul>
 */
```

![스크린샷 2022-05-01 오전 1.27.07](https://tva1.sinaimg.cn/large/e6c9d24egy1h1s7p6us9hj20ji06ogm4.jpg)

우리가 알고 있는 트랜잭션의 관리를 실질적으로 이 AbstractPlatformTransactionManager에서 하는 것으로 보인다.
AbstractPlatformTransactionManager의 상속 클래스로는 또 다음과 같다.

![스크린샷 2022-05-01 오전 1.28.21](https://tva1.sinaimg.cn/large/e6c9d24egy1h1s7qhp6vkj20t405mmyp.jpg)

보통 JPA를 사용하고, PlatformManager에 대한 Bean을 따로 등록을 안한다면 JpaTransactionManager를 사용할 것이다.
실제로 그런지 확인해보자.

![스크린샷 2022-05-01 오전 1.30.37](https://tva1.sinaimg.cn/large/e6c9d24egy1h1s7su0a4nj20su0cjmz2.jpg)

PlatformTransactionManager 빈을 불러오니 JpaTransactionManager가 나오는 것을 확인하였다.
일단 그러면 기본적으로 JpaTransactionManager를 사용한다는 것은 확인하였다.
@Transactional을 붙이면 선언적으로 트랜잭션 관리를 하는 것도 알겠다.

실제로는 어떻게 트랜잭션을 관리하는 것일까?

다시 AbstractPlatformTransactionManager를 보자.
밑은 PlatformTransactionManager의 getTransaction 메소드를 구현한 코드다

![스크린샷 2022-05-01 오전 2.28.25](https://tva1.sinaimg.cn/large/e6c9d24egy1h1s9h1i2igj20ty0t5439.jpg)

TransactionDefinition을 파라미터로 받아 TransactionDefinition에 따라 트랜잭션을 처리하는 것을 알 수 있다.
그렇다면 TransactionDefinition에는 뭐가 있을까?
코드를 보면 유추가능하지만 Propagation 옵션과 Timeout 값, Isolation 레벨이 일단 있다.

![스크린샷 2022-05-01 오전 2.31.45](https://tva1.sinaimg.cn/large/e6c9d24egy1h1s9kg1nwaj20is09qaav.jpg)

여기서 확인 가능한 것은,

- propagation의 기본 값은 REQUIRED다.

- 기본 isolation 레벨은 ISOLATION_DEFAULT다.

  ```java
  /**
  	 * Use the default isolation level of the underlying datastore.
  	 * All other levels correspond to the JDBC isolation levels.
  	 * @see java.sql.Connection
  	 */
  	int ISOLATION_DEFAULT = -1;
  ```

  DB 설정을 따라간다고 한다.

- Timeout 설정은 TIMEOUT_DEFAULT다.

- ```java
  /**
   * Use the default timeout of the underlying transaction system,
   * or none if timeouts are not supported.
   */
  int TIMEOUT_DEFAULT = -1;
  ```

  이 것도 DB 설정을 따라간다고 한다.

그럼 AbstractPlatformTransactionManager를 상속받은 JpaTransactionManager를 보자.

***



>[`PlatformTransactionManager`](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/transaction/PlatformTransactionManager.html) implementation for a single JPA [`EntityManagerFactory`](https://docs.oracle.com/javaee/7/api/javax/persistence/EntityManagerFactory.html?is-external=true). Binds a JPA EntityManager from the specified factory to the thread, potentially allowing for one thread-bound EntityManager per factory. [`SharedEntityManagerCreator`](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/orm/jpa/SharedEntityManagerCreator.html) and `@PersistenceContext` are aware of thread-bound entity managers and participate in such transactions automatically. Using either is required for JPA access code supporting this transaction management mechanism.
>This transaction manager is appropriate for applications that use a single JPA EntityManagerFactory for transactional data access. JTA (usually through [`JtaTransactionManager`](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/transaction/jta/JtaTransactionManager.html)) is necessary for accessing multiple transactional resources within the same transaction. Note that you need to configure your JPA provider accordingly in order to make it participate in JTA transactions.

PlatformTransactionManager는 단일 JPA EntityManagerFactory를 구현한다.
JPA EntityManager를 지정된 팩토리에서 스레드로 바인딩하여 잠재적으로 팩토리당 하나의 스레드의 범위를 가지는 EntityManager를 허용합니다.

즉 PlatformTransactionManager에서 EntityManagerFactory는 1개를 갖고 있고,
EntityManagerFactory에서 스레드별로 범위를 갖는 EntityManager를 생성하는 것이다.

구조는 다음과 같겠다.
![스크린샷 2022-05-01 오후 2.09.49](https://tva1.sinaimg.cn/large/e6c9d24egy1h1stqsgrnuj20r10dhwfj.jpg)

트랜잭션 생성시를 좀 더 자세히 보자.

![스크린샷 2022-05-01 오후 2.10.30](https://tva1.sinaimg.cn/large/e6c9d24egy1h1strh970bj20ph0gh3zi.jpg)



getTransaction()을 호출할 때, 먼저 doGetTransaction()을 가져온다.
doGetTransaction()에서는 JpaTransactionObject 클래스를 생성하고, 여기에 EntityManagerHolder를 넣어주고, DataSource를 넣어준다.

EntityManagerHolder란?

> Resource holder wrapping a JPA EntityManager. JpaTransactionManager binds instances of this class to the thread, for a given javax.persistence.EntityManagerFactory.

EntityManager의 래핑 클래스다. 즉, JpaTransactionObject에 EntityManager를 멤버 변수로 주입해준다.


```java
class JpaTransactionManager {
	@Override
	protected boolean isExistingTransaction(Object transaction) {
		return ((JpaTransactionObject) transaction).hasTransaction();
	}
  
	
  class JpaTransactionObject {
  	public boolean hasTransaction() {
			return (this.entityManagerHolder != null && this.entityManagerHolder.isTransactionActive());
		}
	}
}

```

그리고 isExistingTransaction()은 JpaTransactionManager에서 다음과 같다.
JpaTransactionObject의 hasTransaction()을 호출하고, 여기서 EntityManager가 트랜잭션 진행 중인지를 리턴한다.

트랜잭션이 없다면 startTransaction()을 호출할 것이다.

```java
	/**
	 * Start a new transaction.
	 */
	private TransactionStatus startTransaction(TransactionDefinition definition, Object transaction,
					boolean debugEnabled, @Nullable SuspendedResourcesHolder suspendedResources) {

		boolean newSynchronization = (getTransactionSynchronization() != SYNCHRONIZATION_NEVER);
		DefaultTransactionStatus status = newTransactionStatus(
				definition, transaction, true, newSynchronization, debugEnabled, suspendedResources);
		doBegin(transaction, definition);
		prepareSynchronization(status, definition);
		return status;
	}
```

그리고 startTransaction()에서 doBegin()을 호출하며 트랜잭션이 시작된다.
