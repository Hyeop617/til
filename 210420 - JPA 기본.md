# JPA

### cascade

cascade는 영속성 전이 속성.

```java
@Transactional
@Override
public <S extends T> S save(S entity) {
	Assert.notNull(entity, "Entity must not be null.");
	if (entityInformation.isNew(entity)) {
		em.persist(entity);
		return entity;
	} else {
		return em.merge(entity);
	}
}
```

JPA에서 save()를 할 때 persist or merge를 날림.

따라서 굳이 JPA에서 delete()를 할 일이 없을 경우, 

> cascade = {CascadeType.PERSIST, CascadeType.MERGE}

정도면 충분하지 않을까 생각.



****

#### 04.11.21 추가

JPA의 상태에는 Transient(준영속)과 Persist(영속), Detached(비영속), Removed(분리)가 있다.

Transient란 영속성 컨텍스트를 한번도 거치지 않은 상태이다.

따라서 보통 새로운 엔티티를 디비에 저장할 때, 새로운 엔티티가 Transient 상태라고 생각하면 된다.

하지만 이때, 새로운 엔티티의 id가 있을 경우 JPA는 Detached 상태로 판단한다.

예를 들어서,

```java
Account account1 = Account.builder()
  			 .id(1L) 				// 1
  			 .amount(1000L)
  			 .build();

Account account2 = Account.builder()
  			.amount(2000L)
  			.build();

accountRepository.save(account1); 				// 2
accountRepository.save(account2); 				// 3
```

account1의 경우에는 1번에서 id를 직접 지정해주었고,

account2는 id를 직접 지정하지 않았다.

두 엔티티 다 새로운 엔티티이지만, JPA는 2번에서 account1을 detached로 판단하고 merge,

 3번에서는 transient로 판단하고 persist 실행을 한다.

-------------







그 외에 CascadeType.REFRESH, CascadeType.REMOVE, CascadeType.DETACH가 있음.

REFRESH : 부모 엔티티가 DB를 다시 조회해서 새로고침될 때, 같이 새로고침.

REMOVE : 부모 엔티티가 제거될 때, 같이 제거

DETACH : 부모 엔티티가 영속성 컨텍스트에 의해서 더이상 관리되지 않을때, 같이 DETACHED.



REMOVE는 delete쿼리를 지양하려고 설계했기 때문에 사용하지 않았고, 

설령 사용하더라도 의도하지 않게 작동할 위험(설계를 잘 하면 되겠지만)이 있다.

나머지는 영속성 컨텍스트가 있는데 왜라는 생각이 들어서 사용하지 않았으나

REFRESH 같은 경우는 좀 더 알아봐야 할 수도 있음.



### fetch

그냥 LAZY 쓰자.

한번에 가져오고 싶으면? 이럴때 주로 이것을 사용하였음.

```java
@EntityGraph(attributePaths = {"user"})
@Query("select DISTINCT u from UserInfo u where u.userInfoId = :userInfoId")
Optional<UserInfo> findByUserInfoId(Long userInfoId);
```



엔티티그래프를 선언해서 사용하였음.

어차피 논리적으로 같이 가져와야 하는 테이블일 경우에도? -> YES

EAGER 조인의 경우에도 N+1 문제 발생하기 때문.

DISTINCT 사용이유는 ? 카테시안 곱(outer join)이 발생하기 때문에 (실제로 해보자)

DISTINCT가 싫으면? Set(LinkedHashSet)을 사용하자.

