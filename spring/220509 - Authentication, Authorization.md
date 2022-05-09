# 220509 - Authentication, Authorization



### Authentication (인증)

사용자 본인이 맞는지 확인하는 절차.
Session이나 Token을 이용해서 확인할 수 있음.

스프링 시큐리티에서는 Authentication 인터페이스를 이용해서 사용.
일반적으로 스프링 시큐리티를 기본으로 사용하면 UsernamePasswordAuthenticationToken과 AnonymousAuthenticationToken를 사용함.

말 그대로, SecurityContext에 Authentication이 null이면 AnonymousAuthenticationToken을 SecurityContext에 넣어줌. (비 로그인 상태)
로그인을 했다면 (기본으로 ID/PW 방식) UsernamePasswordAuthenticationToken을 SecurityContext에 넣어줌.



### Authorization (인가)

사용자의 권한을 확인하는 절차.
@PreAuthorize 어노테이션이나 @Secured 어노테이션을 사용해서 확인
Proxy 객체를 사용한 AOP 처리가 일반적.



### @PreAuthorize vs @Secured

두 어노테이션 모두 권한을 체크하는 것은 동일.

```java
package org.springframework.security.access.annotation;


/**
 * Java 5 annotation for describing service layer security attributes.
 *
 * <p>
 * The <code>Secured</code> annotation is used to define a list of security configuration
 * attributes for business methods. This annotation can be used as a Java 5 alternative to
 * XML configuration.
 * <p>
 * For example:
 *
 * <pre>
 * @Secured({ "ROLE_USER" })
 * public void create(Contact contact);
 *
 * @Secured({ "ROLE_USER", "ROLE_ADMIN" })
 * public void update(Contact contact);
 *
 * @Secured({ "ROLE_ADMIN" })
 * public void delete(Contact contact);
 * </pre>
 * @author Mark St.Godard
 */
@Target({ ElementType.METHOD, ElementType.TYPE })
@Retention(RetentionPolicy.RUNTIME)
@Inherited
@Documented
public @interface Secured {
}
```



```java
package org.springframework.security.access.prepost;

/**
 * Annotation for specifying a method access-control expression which will be evaluated to
 * decide whether a method invocation is allowed or not.
 *
 * @author Luke Taylor
 * @since 3.0
 */
@Target({ ElementType.METHOD, ElementType.TYPE })
@Retention(RetentionPolicy.RUNTIME)
@Inherited
@Documented
public @interface PreAuthorize {
}
```



그리고 스프링 시큐리티 3.0 버전 이후에 나온 것이라 PreAuthorize는 Secured 보다 더 자유롭게 조건 설정이 가능함.

PreAuthorize의 경우 SpEL<sub>Spring Expression Language</sub>을 사용가능 함.

`@Secured("ROLE_USER")` `@PreAuthorize("hasRole('ROLE_USER')")` 처럼 ROLE_USER만 허용하는 경우에는 둘 다 사용 가능하나, 다음과 같은 표현은 불가능

`@PreAuthorize("isAuthenticated()")`
`@PreAuthorize("!isAnonymous() AND hasRole( 'ADMIN')")`	



