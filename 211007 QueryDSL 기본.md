## 211007 QueryDSL 기본

### RepositoryImpl

QueryDSL을 사용하는 방법은 여러가지가 있다.<br>기본적으로 다음과 같은 JpaRepository가 있다고 하자.<br>

```java
public interface ProductRepository extends JpaRepository<Long, Product>{
  
}
```

첫번째로는 JPAQueryFactory를 멤버 변수로 갖는 ProductRepositorySupport를 만드는 것이다.

```java
@Repository
public class ProductRepositorySupport(){
  public JPAQueryFactory queryFactory;
  
  @Autowired
  public ProductRepositorySupport(JPAQueryFactory jpaQueryFactory) {
    super(Product.class);
    this.queryFactory = jpaQueryFactory;
  }
}
```

하지만 위 방법의 단점은 Service에서 ProductRepository와 ProductRepositorySupport 두 빈 모두 주입받아야 하는 단점이 있다.<br>findById나 findAll와 같은 JpaRepository(혹은 CrudRepository)의 메소드를 ProductRepositorySupport에서는 따로 구현하지 않는한 사용할 수 없기 때문이다.

```java
@Service
@RequiredArgsConstructor
public class ProductService(){
  private final ProductRepository productRepository;
  private final ProductRepositorySupport productRepositorySupport;
  
  ...
}
```

두번째로는 CustomRepository 인터페이스를 만들어서, 구현 후 JpaRepository 구현체에서 사용하는 것이다.<Br>[jojoldu님 블로그](https://jojoldu.tistory.com/372)의 사진을 보면 다음과 같다.

![diagram](https://t1.daumcdn.net/cfile/tistory/997199476003C3E50B)

다음처럼  ProductRepositoryCustom 인터페이스를 만들자.

```java
public interface ProductRepositoryCustom{
  List<Product> findAllByAuthor(Author author);
}
```

그리고 이 인터페이스의 구현체인 ProductRepositoryImpl를 만들자.<br>네이밍 규칙은 기본적으로 RepositoryImpl로 끝나야한다. (자세한 내용은 [스프링 Data JPA 레퍼런스](https://docs.spring.io/spring-data/jpa/docs/2.1.3.RELEASE/reference/html/#repositories.custom-implementations)를 보면 된다.)

```java
@RequiredArgsConstructor
public class ProductRepositoryImpl implements ProductRepositoryCustom{
  private final JPAQueryFactory queryFactory;
  
  List<Product> findAllByAuthor(Author author){
    queryFactory
      .selectFrom(product)
      // ...
  }
}
```

그리고 ProductRepository (JpaRepository 상속받는 인터페이스)는 다음과 같이 바꾸자

```java
public interface ProductRepository extends JpaRepository<Long, Product>, ProductRepositoryCustom{
  
}
```

그러면 ProductService에서는 ProductRepository만 주입받아 사용하면 된다.

세번째 방법은 [우아콘에서 이동욱님의 강연](https://www.youtube.com/watch?v=zMAX7g6rO_Y&ab_channel=%EC%9A%B0%EC%95%84%ED%95%9CTech)을 보고 깨달은 방법이다.<br>위 방법의 단점은 언제나 Custom한 레포지토리와 그 Custom 레포지토리의 Impl이 필요하다.<br>그래서 다음과 같이 JPAQueryFactory만 주입받아 레포지토리를 생성해서 사용하는 방법도 있다.<br>

![스크린샷 2021-10-07 14.24.54](https://tva1.sinaimg.cn/large/008i3skNgy1gv6oj2dypyj61hl0u0tdt02.jpg)

```java
@RequiredArgsConstructor
@Repository
public class ProductQueryRepository {
  private final JPAQueryFactory queryFactory;
  
  // ...
}
```

위 방법의 단점으로는 첫번째 방법처럼 JpaRepository의 기본 메소드를 사용하지 못해,<br>서비스단에서 ProductQueryRepository와 ProductRepository를 모두 주입받아야 한다.<br>이런 단점때문에 첫번째 방법과 크게 다른게 있나? 싶었는데 유튜브 댓글을 보니 나와 비슷한 생각을 가진 사람이 있었다.![스크린샷 2021-10-07 14.31.21](https://tva1.sinaimg.cn/large/008i3skNgy1gv6opodybsj60yo0gqdj202.jpg)

확실히 이동욱(jojoldu)님의 말씀처럼 각 엔티티의 책임 구분이 명확하지 않을 때 (A 엔티티와 B 엔티티 둘 다 중요한 경우) 해당 방법을 이용하면 좋을 것 같다는 생각이 들었다.<br>이렇게 모호한 경우 각각 엔티티에 종속적이기 보단, 서비스 그 자체에 중점을 두고 레퍼지토리를 만드는게 더 나은 구조일수 있겠다 싶었다.