# 7장 - 도메인 서비스

## 여러 애그리거트가 필요한 기능

도메인 영역에서 한 애그리거트로 처리할 수 없는 기능이 있을 수 있다.<br>
예를 들어 결제 금액 계산 로직은 다음과 같을 것이다.<br>
- 상품 애그리거트: 상품의 가격, 배송비가 필요함
- 주문 애그리거트: 주문한 상품의 개수가 필요함
- 할인 쿠폰 애그리거트: 주문에 적용된 할인 쿠폰의 정보(정액이거나 정률이거나 기타 등등)가 필요함
- 회원 애그리거트: 회원 등급에 따른 할인이 적용될 수 있으므로 필요함

여기서 금액을 계산하는 주체를 결정하기엔 쉽지 않다.<br>
주문 애그리거트를 떠올릴 수 있으나 그렇다면 상품, 할인 쿠폰, 회원 애그리거트 등의 수정이 일어날 때 주문 애그리거트를 수정해야 한다.<br>
이는 주문 애그리거트의 책임을 벗어나는 코드를 작성하게 되고, 외부 의존도가 높아지며 유지보수를 어렵게 만든다.<br>
게다가 애그리거트의 범위를 벗어나는 도메인 로직이 애그리거트에 들어가게 된다.

이를 해결하는 가장 쉬운 방법은 도메인 기능을 별도의 서비스로 구현하는 것이다.<br>

## 도메인 서비스
도메인 서비스는 도메인 영역에 위치한 도메인 로직을 표현할때 주로 사용한다.<br>
주로 다음 상황에서 도메인 서비스를 사용한다.<br>
- 계산 로직: 여러 애그리거트가 필요한 계산 로직이나, 한 애그리거트에 넣기엔 복잡한 로직
- 외부 시스템 연동: 구현하기 위해 타 시스템을 사용해야 하는 도메인 로직

### 계산 로직과 도메인 서비스
할인 금액 규칙 계산같은 복잡한 계산은 애그리거트에 넣기 보단 도메인 서비스를 이용하면 된다.<br>
도메인 영역의 애그리거트나 밸류와 도메인 서비스의 차이는, 도메인 서비스에서는 상태 없이 로직만 구현한다는 점이다.<br>
(도메인 서비스에는 도메인에 대한 상태, 즉 도메인 구성 요소(필드)가 없다는 뜻.)<br>
도메인 서비스를 구현하는데 필요한 상태는 모두 파라미터로 전달받는다.<br>
```java
public class DiscountCalculationService {
  public Money calculateDiscountAmounts(List<OrderLine> orderLines, List<Coupon> coupons, MemberGrade grade) {
    Money couponDiscount = coupons.stream()
        .map(coupon -> calculateDiscount(coupon))
        .reduce(Money(0), (v1, v2) -> v1.add(v2));
    
    Money membershipDiscount = calculateDiscount(grade);
    
    return couponDiscount.add(membershipDiscount);
  }
  
  private Money calculateDiscount(Coupon coupon) {
    // 쿠폰 할인 금액 계산 로직 구현
  }
  
  private Money calculateDiscount(MemberGrade grade) {
    // 회원 등급 할인 금액 계산 로직 구현
  }
}
```
위 DiscountCalculationService를 사용하는 주체는 응용 서비스가 될 수도 있고 애그리거트가 될 수도 있다.<br>
DiscountCalculationService를 애그리거트의 결제 금액 계산 기능에 전달하면 사용 주체는 애그리거트가 된다.<br>
그리고 애그리거트 객체에 도메인 서비스를 전달하는 것은 응용 서비스 책임이다.
```java
public class Order {
  public void calculateAmounts(DiscountCalculationService discountCalculationService, MemberGrade grade) {
    Money totalAmounts = getTotalAmounts();
    Money discountAmounts = discountCalculationService.calculateDiscountAmounts(this.orderLines, this.coupons, grade);
    this.totalAmounts = totalAmounts.minus(discountAmounts);
  }
}
public class OrderService {
  private DiscountCalculationService discountCalculationService;
  
  public Order createOrder(OrderNo orderNo, OrderRequest orderRequest) {
    Member member = memberRepository.findById(orderRequest.getMemberId());
    Order order = new Order(
        orderNo,
        orderRequest.getOrderLines(),
        orderRequest.getCoupons(),
        cretaeOrderer(member),
        orderRequest.getShippingInfo()
    );
    order.calculateAmounts(discountCalculationService, member.getGrade());
    return order;
  }
}
```

> **애그리거트에 도메인 서비스를 주입하면 안 된다.**<br>
> 도메인 서비스를 애그리거트에 파라미터로 전달하므로, 애그리거트가 도메인 서비스에 의존하는 것을 뜻한다.<br>
> 스프링 프레임워크를 이용하므로 애그리거트에 도메인 서비스를 빈으로 주입하고 싶은 생각이 들 것이나 이는 지양해야 한다.<br>
> 
> 도메인 객체 입장에서 도메인 서비스는 도메인을 표현하는 것과 아무 상관이 없다.<br>
> 도메인 객체는 각 필드들이 모여 개념적으로 하나의 모델을 표현하는데, 도메인 서비스는 도메인 객체의 상태를 표현하는 구성 요소가 아니다.<br>
> 또한 도메인 객체의 모든 기능에서 도메인 서비스가 필요한 것도 아니기에, 도메인 서비스를 애그리거트에 주입해야하는 이유는 없다.<br>
 
 
반대로 도메인 서비스의 기능을 실행할 때 애그리거트를 전달하기도 한다.<br>
예를 들어 계좌 이체 기능이 있다고 하자.<br>
```java
public class TransferService {
  public void transfer(Account fromAccount, Account toAccount, Money amount) {
    fromAccount.withdraw(amount);
    toAccount.deposit(amount);
  }
}
```
위와 같이 TransferService 도메인 서비스를 만들면, 응용 서비스는 두 애그리거트를 조회한 뒤 도메인 서비스에 전달하여 계좌 이체를 수행할 수 있다.<br>
트랜잭션 처리 등은 응용 서비스의 책임이기에 도메인 서비스에서는 신경쓰지 않는다.<br>

만약 특정 기능이 응용 서비스인지 도메인 서비스인지 헷갈린다면, 아래 기준으로 확인해보면 좋다.<br>
- 애그리거트의 상태를 변경하는 기능
- 애그리거트의 상태 값을 계산하는 기능
계좌 이체는 계좌 애그리거트의 상태를 변경하고, 결제 계산 로직은 주문 애그리거트의 주문 금액을 계산한다.<br>
위 두 기능은 도메인 로직이면서 하나의 애그리거트에 넣기 힘드므로 도메인 서비스로 구현한 것이다.<br>
(계좌이체는 다른 애그리거트가 필요하므로 도메인 서비스에서 해당 기능을 구현)

### 외부 시스템 연동과 도메인 서비스

외부 시스템이나 타 도메인과의 연동 기능도 도메인 서비스가 될 수 있다.<br>
예를 들어 설문 조사 시스템과 사용자 역할 관리 시스템이 분리되어 있고, 설문조사를 생성할 때 역할 관리 시스템을 통해 권한을 조회해야 할 것이다.<br>
이런 외부 시스템은 HTTP 요청을 통해 연동되어 있을 수 있지만, 설문 조사 도메인 입장에서는 권한 조회 기능만 필요할 것이다.<br>
따라서 권한 조회 기능은 도메인 로직이므로 도메인 서비스로 구현해야 한다. 하지만 도메인 로직 관점에서 이를 인터페이스로 작성하자.<br>
```java
public interface SurveyPermissionChecker {
  boolean hasUserCreationPermission(String userId);
}

public class CreateSurveyService {
  private SurveyPermissionChecker surveyPermissionChecker;
  
  public Long createSurvey(CreateSurveyRequest request) {
    if (!surveyPermissionChecker.hasUserCreationPermission(request.getUserId())) {
      throw new UnauthorizedException("user has no permission to create survey");
    }
    
    // 설문조사 생성 로직 구현
  }
}
```
그리고 SurveyPermissionChecker의 구현체는 인프라 영역에 작성하면 된다.

### 도메인 서비스의 패키지 위치
도메인 서비스는 도메인 로직을 표현하므로 다른 도메인 구성요소와 같은 위치에 작성한다.<br>

### 도메인 서비스의 인터페이스와 클래스
도메인 서비스의 로직이 여러가지라면, 도메인 서비스를 인터페이스로 작성할 수 있다.<br>
특히 도메인 로직을 외부 시스템이나 별도의 엔진을 이용해야 한다면 인터페이스와 클래스로 분리하는게 낫다.<br>
그리고 그 구현체는 인프라 영역에 작성하면 된다.<br>
이러면 도메인 영역이 특정 외부 시스템에 종속되지 않고, 도메인 영역의 테스트도 용이해진다.<br>