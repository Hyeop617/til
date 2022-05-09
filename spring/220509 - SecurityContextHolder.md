# 220509 - SecurityContextHolder

Spring Security에서는 SecurityContextHolder로 시큐리티를 관리한다.



```java
package org.springframework.security.core.context;


/**
 * Associates a given {@link SecurityContext} with the current execution thread.
 * <p>
 * This class provides a series of static methods that delegate to an instance of
 * {@link org.springframework.security.core.context.SecurityContextHolderStrategy}. The
 * purpose of the class is to provide a convenient way to specify the strategy that should
 * be used for a given JVM. This is a JVM-wide setting, since everything in this class is
 * <code>static</code> to facilitate ease of use in calling code.
 * <p>
 * To specify which strategy should be used, you must provide a mode setting. A mode
 * setting is one of the three valid <code>MODE_</code> settings defined as
 * <code>static final</code> fields, or a fully qualified classname to a concrete
 * implementation of
 * {@link org.springframework.security.core.context.SecurityContextHolderStrategy} that
 * provides a public no-argument constructor.
 * <p>
 * There are two ways to specify the desired strategy mode <code>String</code>. The first
 * is to specify it via the system property keyed on {@link #SYSTEM_PROPERTY}. The second
 * is to call {@link #setStrategyName(String)} before using the class. If neither approach
 * is used, the class will default to using {@link #MODE_THREADLOCAL}, which is backwards
 * compatible, has fewer JVM incompatibilities and is appropriate on servers (whereas
 * {@link #MODE_GLOBAL} is definitely inappropriate for server use).
 *
 * @author Ben Alex
 *
 */
public class SecurityContextHolder {}
```


현재 실행중인 스레드와 SecurityContext를 연결해주는 클래스다.

SecurityContextHolderStrategy를 strategy라는 내부 변수로 들고 있다고 한다.
먼저 그 친구를 보자.

```java
package org.springframework.security.core.context;

/**
 * A strategy for storing security context information against a thread.
 *
 * <p>
 * The preferred strategy is loaded by {@link SecurityContextHolder}.
 *
 * @author Ben Alex
 */
public interface SecurityContextHolderStrategy {
  
	/**
	 * Clears the current context.
	 */
	void clearContext();

	/**
	 * Obtains the current context.
	 * @return a context (never <code>null</code> - create a default implementation if
	 * necessary)
	 */
	SecurityContext getContext();

	/**
	 * Sets the current context.
	 * @param context to the new argument (should never be <code>null</code>, although
	 * implementations must check if <code>null</code> has been passed and throw an
	 * <code>IllegalArgumentException</code> in such cases)
	 */
	void setContext(SecurityContext context);

	/**
	 * Creates a new, empty context implementation, for use by
	 * <tt>SecurityContextRepository</tt> implementations, when creating a new context for
	 * the first time.
	 * @return the empty context.
	 */
	SecurityContext createEmptyContext();


}
```

스레드에 Security Context Information을 저장하는 전략이라고 한다.

그리고 내부적으로 SecurityContext를 관리하는 인터페이스 임을 알 수 있다.
여기서 알 수 있는 점은 SecurityContext는 절대 null이 아니라는 것이다.



![스크린샷 2022-05-09 오후 11.53.11](https://tva1.sinaimg.cn/large/e6c9d24egy1h22jkbo14dj20ju02ywey.jpg)</center>

구현체는 3개 클래스가 있다.
Global, InheritableThreadLocal, ThreadLocal 3가지 방식이다.	