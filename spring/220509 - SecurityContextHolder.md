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



ThreadLocal은 기본 SecurityContextHolder의 기본 세팅이다.
따라서 ThreadLocal을 먼저 보자.

```java
final class ThreadLocalSecurityContextHolderStrategy implements SecurityContextHolderStrategy {

	private static final ThreadLocal<SecurityContext> contextHolder = new ThreadLocal<>();

	@Override
	public void clearContext() {
		contextHolder.remove();
	}

	@Override
	public SecurityContext getContext() {
		SecurityContext ctx = contextHolder.get();
		if (ctx == null) {
			ctx = createEmptyContext();
			contextHolder.set(ctx);
		}
		return ctx;
	}

	@Override
	public void setContext(SecurityContext context) {
		Assert.notNull(context, "Only non-null SecurityContext instances are permitted");
		contextHolder.set(context);
	}

	@Override
	public SecurityContext createEmptyContext() {
		return new SecurityContextImpl();
	}

}
```

말 그대로 ThreadLocal을 통해서 SecurityContext를 저장한다.
ThreadLocal은 각 스레드에 연결돼어 스레드마다 다른 데이터를 저장할 수 있다.

즉 SecurityContext는 각 스레드마다 저장되어 다른 스레드에선 읽을 수 없다.



```java
final class InheritableThreadLocalSecurityContextHolderStrategy implements SecurityContextHolderStrategy {

	private static final ThreadLocal<SecurityContext> contextHolder = new InheritableThreadLocal<>();

	@Override
	public void clearContext() {
		contextHolder.remove();
	}

	@Override
	public SecurityContext getContext() {
		SecurityContext ctx = contextHolder.get();
		if (ctx == null) {
			ctx = createEmptyContext();
			contextHolder.set(ctx);
		}
		return ctx;
	}

	@Override
	public void setContext(SecurityContext context) {
		Assert.notNull(context, "Only non-null SecurityContext instances are permitted");
		contextHolder.set(context);
	}

	@Override
	public SecurityContext createEmptyContext() {
		return new SecurityContextImpl();
	}
}
```

InheritableThreadLocalSecurityContextHolderStrategy은 InheritableThreadLocal을 사용한다.
InheritableThreadLocal은 자식 스레드가 부모 스레드의 ThreadLocal에 접근이 가능하다.

하지만, 일반적으로 InheritableThradLocal을 Servlet에서 사용하는 것을 추천하지 않는다고 한다.

톰캣은 기본적으로 ThreadPool을 이용해서 요청을 처리한다.
즉 스레드를 생성하고 계속해서 재사용하는 것이다.

만약 자식 스레드는 아직 처리 중인데, 부모 스레드가 다른 Request를 받았다고 생각해보자.
그리고 다른 유저의 SecurityContext를 저장하면 자식 스레드에도 영향을 미칠 수 있다.
그리고 자식 스레드에서 사용을 하기 때문에 메모리 누수<sub>memory leak</sub>현상이 발생하기 쉽다고도 한다.

하지만 Async하게 처리를 한다면 스레드를 새로 만들어야 할테니..
SecurityContext를 자식 스레드까지 전파할 일이 있는지 잘 고민해보고 결정하자.



```java

/**
 * A <code>static</code> field-based implementation of
 * {@link SecurityContextHolderStrategy}.
 * <p>
 * This means that all instances in the JVM share the same <code>SecurityContext</code>.
 * This is generally useful with rich clients, such as Swing.
 *
 * @author Ben Alex
 */
final class GlobalSecurityContextHolderStrategy implements SecurityContextHolderStrategy {

	private static SecurityContext contextHolder;

	@Override
	public void clearContext() {
		contextHolder = null;
	}

	@Override
	public SecurityContext getContext() {
		if (contextHolder == null) {
			contextHolder = new SecurityContextImpl();
		}
		return contextHolder;
	}

	@Override
	public void setContext(SecurityContext context) {
		Assert.notNull(context, "Only non-null SecurityContext instances are permitted");
		contextHolder = context;
	}

	@Override
	public SecurityContext createEmptyContext() {
		return new SecurityContextImpl();
	}

}
```



마지막은 GlobalSecurityContextHolderStrategy다. 여기서는 JVM 전체에서 한 SecurityContext를 사용한다고 한다.
SecurityContextHolder의 javadoc에서 나왔듯이 이 전략은 서버보다는 클라이언트에서 유용하다.

>  whereas MODE_GLOBAL is definitely inappropriate for server use

웬만해서는 이 전략을 서버에서는 사용하지 말자.