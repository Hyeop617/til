# 5장 - 스프링 데이터 JPA를 이용한 조회 기능

## 시작에 앞서
**CQRS**
- 명령(Command) 모델과 조회(Query) 모델을 분리하는 패턴
- 명령 모델은 상태를 변경할 때, 조회 모델은 상태를 읽을 때 사용

엔티티, 애그리거트, 리포지터리 등 앞에서 사용된 모델은 상태 변경할 때 주로 사용.<br>
즉, 도메인 모델은 명령 모델로 주로 사용됨.<br>
이 장에서는 리포지터리(도메인 모델에 속한)과 DAO(데이터 접근을 의미하는)을 혼용해서 사용할 것.

조회 모델을 구현할 때는 JPA, MyBatis, JdbcTemplate 등 다양한 방법이 있음.<br>

## 검색을 위한 스펙
검색 조건이 단순하면 다음과 같이 특정 조건으로 조회하는 기능을 만들면 됨.
```java
public interface OrderDataDao {
  Optional<OrderData> findById(OrderNo id);
  List<OrderData> findByOrderer(String ordererId, Date fromDate, Date toDate);
}
```
목록 조회 기능은 다양한 검색 조건을 조합해서 조회하는 경우가 있음.<br>
필요한 조합마다 find 메소드를 정의할 수 있으나, 그 대신 스펙(Specification)을 이용할 수 있음.<br>
스펙은 애그리거트가 특정 조건을 충족하는지 검사할 때 사용하는 인터페이스임.
```java
public inteface Specification<T> {
  public boolean isSatisfiedBy(T agg);
}
```
isSatisfiedBy 메소드의 agg 파라미터는 검사 대상이 되는 객체다.<br>
스펙을 리포지터리에 적용하면 agg는 애그리거트 루트가 될 것이고, DAO에 적용한다면 검색 결과로 리턴할 데이터 객체가 된다.<br>
isSatisfiedBy 메소드는 agg가 조건을 충족하면 true를, 그렇지 않으면 false를 리턴한다.<br>
Order 애그리거트 객체가 특정 고객의 주문인지 확인하는 스펙은 다음과 같다.
```java
@AllArgsConstructor
public class OrderSpec implements Specification<Order> {
  private String ordererId;
  
  public boolean isSatisfiedBy(Order agg) {
    return agg.getOrdererId().getMemberId().getId().equals(ordererId);
  }
}
```
리포지터리나 DAO는 검색 대상을 걸러내는 용도로 스펙을 사용한다.<br>
만약 리포티저리가 메모리에 모든 애그리거트를 가지고 있다면, 다음과 같이 스펙을 적용할 수 있다.
```java
public class MemoryOrderRepository implements OrderRepository {
  
  public List<Order> findAll(Specification<Order> spec) {
    List<Order> allOrders = this.findAll();
    return allOrders.stream()
        .filter(spec::isSatisfiedBy)
        .toList();
  }
}
```
리포지터리가 스펙을 이용해서 검색 대상을 걸러주므로, 원하는 스펙을 만들고 전달하면 특정 조건으로 검색이 가능하다.
```java
// 검색 조건을 표현하는 스펙을 작성해서
Specification<Order> ordererSpec = new OrdererSpec("madvirus");
// 리포지터리에 전달
List<Order> orders = orderRepository.findAll(ordererSpec);
```
하지만 위와 같이 인 메모리 레포지터리를 실제로 사용할 일은 적을 것이다.<br>

## 스프링 데이터 JPA를 이용한 스펙 구현
스프링 데이터 JPA에는 검색 조건을 표현하기 위한 Specification 인터페이스가 있음.<br>

```java
import java.io.Serializable;

public interface Specification<T> extends Serializable {
  // not, where, and, or 메소드 생략
  
  @Nullable
  Predicate toPredicate(Root<T> root, CriteriaQuery<?> query, CriteriaBuilder cb);
}
```
제네릭 파라미터 T는 JPA 엔티티 타입을 의미한다.<br>
toPredicate 메소드는 JPA Criteria API에서 조건을 표현하는 Predicate 객체를 생성한다.<br> 
예를 들어, 다음 조건은 이렇게 구현할 수 있다.
- 엔티티 타입이 OrderSummary 이다.
- ordererId 프로퍼티 값이 지정한 값과 동일하다.
```java
import org.springframework.data.jpa.domain.Specification;
import com.example.order.query.dto.OrderSummary;
import com.example.order.query.dto.OrderSummary_;

@AllArgsConstructor
public class OrdererIdSpec implements Specification<OrderSummary> {
  private String ordererId;
  
  @Override
  public Predicate toPredicate(Root<OrderSumamry> root, CriteriaQuery<?> query, CriteriaBuilder cb) {
    return cb.equal(root.get(OrderSummary_.ordererId), ordererId);
  }
}
```

### JPA 정적 메타 모델
OrderSummary_ 클래스는 JPA 정적 메타 모델을 정의한 코드다.<br>
정적 메타 모델 클래스는 다음과 같이 구현 가능하다.
```java
import jakarta.persistence.metamodel.StaticMetamodel;

@StaticMetamodel(OrderSummary.class)
public class OrderSummary_ {
  public static volatile SingularAttribute<OrderSummary, String> number;
  public static volatile SingularAttribute<OrderSummary, Long> version;
  public static volatile SingularAttribute<OrderSummary, String> ordererId;
  public static volatile SingularAttribute<OrderSummary, String> ordererName;
}
```
정적 메타 모델은 @StaticMetamodel 어노테이션을 사용해서 지정한다.<br>
메타 모델 클래스명은 모델 클래스의 이름 뒤에 `_` 문자를 붙여서 정의하는 것이 관례다.<br>

정적 메타 모델 클래스는 대상 모델의 각 프로퍼티와 동일한 이름을 갖는 정적 필드를 정의한다.<br>
이 정적 필드는 프로퍼티에 대한 메타 모델로써 프로퍼티 타입에 따라 SingularAttribute, ListAttribute, SetAttribute, MapAttribute 등을 사용한다.<br>

정적 메타 모델 대신 문자열을 프로퍼티에 넣어서 사용할 수 있지만, 오타 및 리팩토링시 안전하지 않으므로 권장되지 않는다.<br>
하이버네이트같은 JPA 구현체는 정적 메타 모델 클래스를 자동으로 생성하는 기능을 제공한다.<br>


스펙 구현 클래스를 개별적으로 만들지 않고, 한 클래스에 여러 스펙을 모아서 관리할 수도 있다.<br>
```java
public class OrderSummarySpecs {
  public static Specification<OrderSummary> ordererId(String ordererId) {
    return (Root<OrderSummary> root, CriteriaQuery<?> query, CriteriaBuilder cb) -> {
      return cb.equal(root.<String>get("ordererId"), ordererId);
    };
  }
  
  public static Specifcation<OrderSummary> orderDateBetween(LocalDateTime from, LocalDateTime to) {
    return (Root<OrderSummary> root, CriteriaQuery<?> query, CriteriaBuilder cb) -> {
      return cb.between(root.get(OrderSummary_.orderDate), from, to);
    };
  }
}
```
스펙 인터페이스는 함수형 인터페이스므로, 위처럼 람다식으로 구현할 수 있다. 

## 리포지터리/DAO에서 스펙 사용하기
스펙을 충족하는 엔티티를 검색하고 싶다면 findAll 메소드를 사용하면 된다.<br>
findAll 메소드는 스펙 인터페이스를 파라미터로 갖기에 다음처럼 사용할 수 있다.
```java
public interface OrderSummaryDao extends Repository<OrderSummary, String> {
  List<OrderSummary> findAll(Specification<OrderSummary> spec);
}

// 스펙 객체를 생성 한 뒤
Specification<OrderSummary> spec = new OrdererIdSpec("user1");
// findAll 메소드를 이용해서 검색
List<OrderSummary> results = orderSummaryDao.findAll(spec);
```

## 스펙 조합
스펙 인터페이스 중 and와 or 메소드를 이용해 스펙을 조합할 수 있다.<br>
```java
public interface Specification<T> extends Serializable {
  default Specification<T> and(@Nullable Specification<T> other);
  default Specification<T> or(@Nullable Specification<T> other);
  
  @Nullable
  Predicate toPredicate(Root<T> root, CriteriaQuery<?> query, CriteriaBuilder cb);
}
```
and와 or는 기본 구현을 가진 디폴트 메소드다.<br>
and 메소드는 두 스펙을 모두 충족하는 조건을 만들고, or 메소드는 두 스펙 중 하나라도 충족하는 스펙을 만든다.<br>
not을 이용해서 조건을 반대로 적용할 수도 있다.
```java
Specifictation<OrderSummary> spec = OrderSummarySpecs.ordererId("user1")
    .and(OrderSummarySpecs.orderDateBetween(from, to));
Specification<OrderSummary> spec2 = Specification.not(OrderSummarySpecs.ordererId("user1"));
```
또한 where를 이용하면 nullable한 스펙을 안전하고 편리하게 사용 가능하다.<br>
where 메소드는 인자로 null을 전달하면, 항상 true를 리턴하는 스펙을 생성한다.<br>
```java
Specification<OrderSummary> nullableSpec = createNullableSpec();
Specification<OrderSummary> otherSpec = createOtherSpec();

Specification<OrderSummary> spec = nullableSpec == null ? otherSpec : nullableSpec.and(otherSpec); // 이것 대신
Specification<OrderSummary> spec2 = Specification.where(nullableSpec).and(otherSpec); // 이것을 사용
```
## 정렬 지정하기
스프링 데이터 JPA는 두 가지 방법을 이용해서 정렬할 수 있다.
- 메소드 이름에 OrderBy를 사용해서 정렬 조건 지정
- Sort 객체를 인자에 전달

`findByOrdererIdOrderByNumberDesc` 메소드는 ordererId로 검색한 결과를 number 프로퍼티 기준으로 내림차순 정렬한다.<br>
`findByOrdererIdOrderByOrderDateDescNumberAsc` 메소드는 ordererId로 검색한 결과를 orderDate 기준으로 내림차순한 뒤, number 기준으로 오름차순 정렬한다.<br>
메소드 이름을 사용하면 간편하지만, 복잡한 정렬 조건을 지정하거나 동적인 정렬을 하는데 한계가 있다.<br>
이럴 때 Sort 객체를 이용하면 된다.
```java
public interface OrderSummaryDao extends Repository<OrderSummary, String> {
  List<OrderSummary> findByOrdererId(String ordererId, Sort sort);
  List<OrderSummary> findAll(Specification<OrderSummary> spec, Sort sort);
}

Sort sort1 = Sort.by("number").ascending();
Sort sort2 = Sort.by("orderDate").descending();
Sort complexSort = sort1.and(sort2);
Sort shorttenSort = Sort.by("number").ascending().and(Sort.by("orderDate").descending()); 
```

## 페이징 처리하기
스프링 데이터 JPA는 페이징 처리를 할때 Pageable 인터페이스를 사용한다.<br>
실제 Pageable 인터페이스 구현체는 PageRequest 클래스를 사용한다.<br>
```java
public interface MemberDataDao extends Repository<MemberData, String> {
  List<MemberData> findByNameLike(String name, Pageable pageable);
}

PageRequest pageRequest = PageRequest.of(1, 10); // 2페이지에서 10개를 조회하는 요청
List<MemberData> members = memberDataDao.findByNameLike("hyeop%", pageRequest);
PageRequest sortedPageRequest = PageRequest.of(2, 10, Sort.by("name").ascending());   // Sort 객체를 이용해 정렬이 가능
List<MemberData> members2 = memberDataDao.findByNameLike("hyeop%", sortedPageRequest);
```

Page 타입이 리턴형이면 전체 페이지 정보도 얻을 수 있다. (totalCount, totalPages 등)<br>
따라서 count 쿼리가 같이 실행된다.

```java
import java.awt.print.Pageable;

public interface MemberDataDao extends Repository<MemberData, String> {
  Page<MemberData> findByBlocked(boolean blocked, Pageable pageable);
  Page<MemberData> findAll(Specification<MemberData> spec, Pageable pageable);
}
```

그리고 페이징 처리가 아니라 상위 N개만 구하고 싶다면 `findTopN` 혹은 `findFirstN` 메소드를 사용하면 된다.<br>
```java
List<MemberData> findFirst3ByNameLikeOrderByName(String name);
List<MemberData> findTop3ByNameLikeOrderByName(String name);  // 둘 다 동일
```

### Page
`findBy프로퍼티` 메소드는 Pageable을 파라미터로 받더라도 리턴 타입이 List면 카운트 쿼리가 실행되지 않는다.<br>
따라서 페이징 처리가 필요하지 않다면 List를 리턴 타입으로 사용해서 불필요한 카운트 쿼리를 방지해야 한다. (2번째)<br>
```java
Page<MemberData> findByBlocked(boolean blocked, Pageable pageable);
List<MemberData> findByBlocked(boolean blocked, Pageable pageable);
```
하지만 스펙을 사용하는 findAll 메소드에 Pageable을 파라미터로 전달하면 리턴 타입이 Page가 아니여도 카운트 쿼리가 실행된다.<br>
즉, 다음 메소드는 카운트 쿼리가 같이 실행된다.
```java
List<MemberData> findAll(Specification<MemberData> spec, Pageable pageable);
```
이를 방지하려면 커스텀 리포지토리를 이용해서 직접 구현해야한다. [저자 블로그](https://javacan.tistory.com/entry/spring-data-jpa-range-query)

## 스펙 조합을 위한 스펙 빌더 클래스
스펙을 사용할 때 조건에 따라 스펙을 조합해서 사용할 때가 있다.<br>
```java
public void search(SearchRequest searchRequest) {
  Specification<MemberData> spec = Specification.where(null);
  if (searchRequest.isOnlyNotBlocked()) {
    spec = spec.and(MemberDataSpecs.nonBlocked());
  }
  if (StringUtils.hasText(searchRequest.getName())) {
    spec = spec.and(MemberDataSpecs.nameLike(searchRequest.getName()));
  }  
  List<MemberData> results = memberDataDao.findAll(spec, PageRequest.of(0, 5));
}
```
위 코드처럼 스펙을 조합하면, if문이 많아져 가독성이 떨어지고 실수하기 쉽다.<br>
이를 방지하기 위해 스펙 빌더를 직접 만들어서 사용하면 다음과 같이 사용할 수 있다.<br>
여기서는 ifTrue, ifHasText 등을 사용했는데 필요한 메소드가 있으면 직접 추가해서 사용하면 된다.

```java
public class SpecBuilder {
  public static <T> Builder<T> builder(Class<T> type) {
    return new Builder<T>();
  }

  public static class Builder<T> {
    private List<Specification<T>> specs = new ArrayList<>();

    public Builder<T> and(Specification spec) {
      specs.add(spec);
      return this;
    }

    public Builder<T> ifHasText(String string, Function<String, Specification<T>> specFunction) {
      if (StringUtils.hasText(stirng)) {
        specs.add(specFunction.apply(string));
      }
      return this;
    }

    public Builder<T> ifTrue(Boolean condition, Supplier<Specification<T>> specSupplier) {
      if (condition != null && condition) {
        specs.add(specSupplier.get());
      }
      return this;
    }

    public Specification<T> toSpec() {
      Specification<T> specification = Specifcation.where(null);
      for (Specification<T> spec : specs) {
        specification = specification.and(spec);
      }
      return specification;
    }
  }
}

public void search(SearchRequest searchRequest) {
  Specification<T> spec = SpecBuilder.builder(MemberData.class)
      .ifTrue(searchRequest.isOnlyNotBlocked(), () -> MemberDataSpecs.nonBlocked())
      .ifHasText(searchRequest.getName(), name -> MemberDataSpecs.nameLike(name))
      .toSpec();
  List<MemberData> results = memberDataDao.findAll(spec, PageRequest.of(0, 5));
}
```

## 동적 인스턴스 생성
JPA는 쿼리 결과에서 임의의 객체를 동적으로 생성할 수 있는 기능을 제공하고 있다.
```java
public interface OrderSummaryDao extends Repository<OrderSummary, String> {
  @Query("""
      SELECT new com.example.order.query.dto.OrderView(
        o.number, o.state, m.name, m.id, p.name
      )
      FROM Order o JOIN o.orderLines ol, Member m, Product p
      WHERE o.orderer.memberId.id = :ordererId
      AND o.orderer.memberId.id = m.id
      AND index(ol) = 0
      AND ol.productId.id = p.id
      ORDER BY o.number.number DESC
      """)
  List<OrderView> findOrderView(String ordererId);
}
```
이 코드에서 JPQL의 SELECT 절에는 new 키워드가 있다.<br>
new 키워드 뒤에는 생성할 클래스의 FQCN을 지정하고 생성자 안에 파라미터로 전달할 값을 지정한다.<br>

```java
public class OrderView {
  private final String number;
  private final OrderState state;
  private final String memberName;
  private final String memberId;
  private final String productName;
  
  public OrderView(OrderNo orderNo, OrderState state, String memberName, MemberId memberId, String productName) {
    this.number = orderNo.getNumber();
    this.state = state;
    this.memberName = memberName;
    this.memberId = memberId.getId();
    this.productName = productName;
  }
}
```
조회 전용 모델을 만드는 이유는 Presentation Layer을 통해 사용자에게 보여줄 데이터를 표현하기 위함이다.<br>
우리가 새로 추가한 밸류 타입을 알맞은 형식으로 표현할 수 없으므로, 기본 타입으로 변환해서 표현한다.<br>

동적 인스턴스의 장점은 JPQL을 그대로 사용하므로 객체 기준으로 쿼리를 작성하면서도 동시에 지연/즉시 로딩과 같은 고민 없이 원하는 모습으로 데이터를 조회 할 수 있다는 점이다.

## 하이버네이트 @SubSelect 사용
하이버네이트는 JPA 확장 기능으로 @SubSelect를 제공한다.<br>
@SubSelect 어노테이션을 이용하면 쿼리 결과를 엔티티로 매핑할 수 있다.<br>
```java
import org.hibernate.annotations.Immutable;
import org.hibernate.annotations.SubSelct;
import org.hibernate.annotations.Synchronize;

@Entity
@Immutable
@SubSelct("""
      SELECT 
        o.order_number as number,
        o.version,
        o.orderer_id,
        o.orderer_name,
        o.total_amounts,
        o.receiver_name,
        o.state, 
        o.order_date,
        p.product_id,
        p.name as product_name,
    FROM purchase_order o 
      INNER JOIN order_line ol ON o.order_number = ol.order_number 
      CROSS JOIN product p
    WHERE ol.line_idx = 0 AND ol.product_id = p.product_id
""")
@Synchronize({"purchase_order", "order_line", "product"})
public class OrderSummary {
  @Id
  private String number;
  private long version;
  private String ordererId;
  private String ordererName;
  private long totalAmounts;
  private String receiverName;
  @Enumerated(EnumType.STRING)
  private OrderState state;
  private LocalDateTime orderDate;
  private String productId;
  private String productName;
    
}
```
@Immutable, @SubSelect, @Synchronize는 하이버네이트 전용 어노테이션으로 이를 이용하면 쿼리 결과를 엔티티로 매핑할 수 있다.<br>

@SubSelect는 조회 쿼리를 값으로 갖는다.<br>
DBMS가 여러 테이블을 조인한 결과를 뷰를 통해 제공하는 것과 유사하게, 쿼리 실행 결과를 매핑할 테이블처럼 사용한다. (가상 테이블)<br>

뷰를 수정할 수 없듯 @SubSelect로 매핑한 엔티티도 수정할 수 없다.<br>
이를 update 하려고 하면 하이버네이트가 업데이트를 시도할 때 예외가 발생하므로, 근본적으로 막기 위하여 @Immutable 어노테이션을 사용한다.<br>

### @Immutable
@Immutable을 적용하면 하이버네이트는 해당 엔티티에 대해 스냅샷을 저장하지 않기에 변경 감지를 하지 않는다.<br>
@Immutable을 엔티티에 적용하면 update, delete는 무시된다.<br>
@OneToMany 같은 컬렉션에 적용하면 insert, update, delete 다 무시된다.<br>

```java
// order 테이블에서 조회
Order order = orderRepository.findById(orderNumber);
order.changeShippingInfo(newInfo);

// 변경 내역이 DB에 반영되지 않았는데 purchase_order 테이블에서 조회
List<OrderSummary> summaries = orderSummaryDao.findByOrdererId(userId);
```
위 코드는 Order의 상태를 변경 후 다시 OrderSummary를 조회하는 코드다.<br>

### @Synchronize
하이버네이트는 다음 상황에 flush를 한다.
- 트랜잭션이 커밋될 때
- JPQL or Criteria 쿼리가 실행되면 stale한 값을 볼 가능성이 있을 때 (쿼리에서 읽는 테이블이 dirty한 상태일 때)
- 명시적으로 flush() 메소드가 호출될 때
- native 쿼리가 실행될 때

여기서 2번째 조건인 JPQL 쿼리가 실행되면 stale한 값을 볼 가능성이 있다고 생각해, flush가 될 거라고 예상할 수 있다.<br>
그러나 OrderSummary 엔티티는 하이버네이트 입장에서는 OrderSummary 엔티티로 인식하기 때문에, Order 엔티티의 변경사항을 flush 해야한다고 판단할 수 없다.<br>
(@SubSelect 쿼리 안에는 Order, OrderLine, Product 테이블이 있지만, 하이버네이트 입장에선 단순히 OrderSummary 엔티티 이다.)<br>
이를 방지하기 위하여 @Synchronize 어노테이션을 이용해야 한다.<br>
@Synchronize 어노테이션에 'purchase_order' 테이블이 있으므로<br>
OrderSummary 엔티티를 조회하기 전에, purchase_order 테이블에 변경된 내용이 있다면, 이를 flush 하도록 하이버네이트에게 힌트를 준다.<br>
따라서 OrderSummary 엔티티를 조회할 땐 변경된 내용이 반영되어 조회된다.


@SubSelect를 사용하면 일반 @Entity와 같기 때문에 JPQL, Criteria API, 스펙, 페이징 등을 사용할 수 있다.<br>
또한 @SubSelect를 사용하면 실제 쿼리는 from절에 서브쿼리가 들어가게 된다.<br>
이를 피하려면 네이티브 쿼리를 이용하거나 쿼리 매퍼를 이용해서 직접 쿼리를 작성하면 된다.<br>

