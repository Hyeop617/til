## 도메인

소프트웨어로 해결하고자 하는 문제 영역을 뜻함.<br>
도메인은 여러 하위 도메인으로 나눌 수 있음.

e.g.) 주문 도메인 하위에 결제 도메인, 배송 도메인, 상품 도메인 등이 있음.

도메인을 모두 직접 구현할 필요는 없음.<br>
외부 서비스(결제 혹은 물류)를 활용하여 도메인을 구성할 수 있음.

같은 도메인이더라도 상황에 따라 다르게 구성될 수 있음.<br>
e.g.) 같은 주문 도메인이더라도 상황에 따라 배송 도메인이 없을 수도 있음.


## 도메인 전문가와 개발자간 지식 공유

개발에 앞서 요구사항 분석은 매우 중요함.<br>
요구사항을 올바르게 이해하지 못한다면, 쓸모없거나 유용하지 못한 시스템이 될 수 있음.<br>

요구사항을 이해하기 제일 쉬운 방법은 직접 대화하는 것.<br>

도메인 전문가만큼은 아니나 개발자도 도메인 지식을 갖고 있어야 함.<br>
그래야 원하는 요구사항에 가까운 제품이 나옴.

그리고 정말로 **도메인 전문가가 원하는 것이 무엇인지 파악**하는 것이 중요함.<br>
전달받은 요구사항을 표면적으로 구현하는 것이 아니라, 어떤 목표로 인해 요구사항이 나왔는지 이해해야 함.

## 도메인 모델

특정 도메인을 개념적으로 표현한 것.<br>
도메인 모델을 사용하면 여러 관계자들이 동일한 모습으로 도메인에 대해 이해가 가능.<br>
또한 도메인 지식을 공유하는데도 도움이 됨.

도메인 모델을 표현하는 방법은 여러가지가 있음.<br>

도메인 모델은 기본 적으로 개념적인 모델이므로, 반드시 구현 모델과 같을 필요는 없음.

**여러 하위 도메인을 한 도메인 모델로 표현하면 안 됨.**<br>
같은 용어라도 각 도메인에 따라 의미가 다를 수 있기 때문.<br>
따라서 각 도메인마다 별도의 도메인 모델이 필요함.

## 도메인 모델 패턴

일반적인 서버 아키텍쳐는 다음과 같음.<br>
Presentation Layer -> Application Layer -> Domain Layer -> Infrastructure Layer

Presentation: 사용자의 요청을 받고, 응답을 반환하는 역할<br>
Application: 사용자의 요청을 실행. 로직을 직접 구현하지 않고 도메인 레이어를 조합하여 실행<br>
Domain: 시스템이 제공할 도메인 규칙을 구현<br>
Infrastructure: 데이터베이스, 메시지 큐 등 외부 시스템과의 연동 담당

도메인 모델 패턴은 도메인 레이어를 객체지향적으로 구현하는 패턴을 뜻함.<br>

```java
public class Order {
  private OrderState state;
  private ShippingInfo shippingInfo;
  
  public void changeShippingINfo(ShippingInfo info) {
    if (!isShippingChangeable()) {
      throw new IllegalStateException("can't change shipping in " + state);
    }
    this.shippingInfo = info;
  }
  
  private boolean isShippingChangeable() {
    return state == OrderState.PAYMENT_WAITING || state == OrderState.PREPARING;
  }
}

public enum OrderState {
  PAYMENT_WAITING, PREPARING, SHIPPED, DELIVERING, DELIVERY_COMPLETE;
}
```
배송지가 변경 가능한지 판단 여부를 Order 객체에서 판단함.<br>
즉, 주문과 관련된 중요한 규칙을 도메인 레이어에서 구현하고 있음.

## 도메인 모델 도출

도메인을 모델링할 때 먼저 요구사항으로부터 핵심 구성요소, 규칙, 기능 등을 정의해야 함.
e.g) 주문 도메인 이라면,
- 최소 한 종류 이상의 상품을 주문해야 함.
- 한 상품을 한 개 이상 주문할 수 있음
- 총 주문 금액으느 각 상품의 구매 가격 합을 모두 더한 금액
- 각 상품의 구매 가격 합은 상품 가격에 구매 개수를 곱한 값이다.
- 주문할 때 배송지 정보를 반드시 입력 해야 함.
- 배송지 정보는 받는 사람 이름, 전화번호, 주소 등으로 구성 됨.
- 출고하면 배송지를 변경할 수 없음.
- 출고 전에 주문은 취소될 수 있음.
- 고객이 결제를 완료하기 전, 상품을 미리 준비하지 않음.

등을 나열했을 때, 주문은 '출고 상태로 변경', '배송지 정보 변경', '주문 취소', '결제 완료' 등의 기능을 제공한다는 것을 알 수 있음.<br>
이 때 Order 클래스에 해당 메소드를 가져야 한다고 판단이 가능함.

또한 '출고하면 배송지를 변경할 수 없음'이라는 규칙에서 아래 처럼 코드를 구현 가능함.
```java
public class Order {
  private OrderState state;
  
  public void changeShippingInfo(ShippingInfo info) {
    verifyNotYetShipped();
    setShippingInfo(info);
  }
  
  private void verifyNotYetShipped() {
    if (state != OrderState.PAYMENT_WAITING || state != OrderState.PREPARING) {
      throw new IllegalStateException("already shipped");
    }
  }
}
```
`isShippingChangeable` 메소드를 `verifyNotYetShipped`로 변경했다.<br>
도메인을 모델링하면서 도메인에 대해 더 잘 알게 되었으므로, 메소드 이름도 더 명확하게 변경할 수 있었음.

코드를 작성할 때 단순히 동작이 되게끔이 아니라, 도메인에 대해 잘 표현할 수 있게 작성해야 함.<br>
그랬을 때 가독성이 높은 코드가 나올 수 있으며, 유지보수성도 높아짐. (문서화의 개념)


## 엔티티와 밸류

도메인 모델은 크게 엔티티(Entity)와 밸류(Value)로 나눌 수 있음.

### Entity
Entity의 큰 특징은 식별자(Identifier)를 가짐.<br>
예를 들어 주문 도메인에서 Identifier는 주문번호가 될 수 있음.<br>
Identifier는 각 Entity마다 Unique해야 하며, Immutable 해야 함.

식별자는 도메인의 특징과 기술에 따라 달라질 수 있음.<br>
e.g. UUID, DB Sequence, 주문번호 같은 특정 규칙에 따른 값 등.

### Value
```java
public class ShippingInfo {
  private String receiverName;
  private String receiverPhone;
}
```

ShippingInfo 클래스에는 받는이 이름과, 받는이 전화번호가 있음.<br>
위 2개의 필드로 ShippingInfo는 개념적으로 완전한 하나의 받는 사람이라는 의미를 가짐.<br>

이렇게 밸류 타입은 개념적으로 완전한 하나를 표현할 때 사용됨.

밸류 타입을 이용하면 다음과 같은 장점이 있음.
- 코드를 이해하는 데 더 도움이 됨 (명확하게 도메인을 표현 가능. 금액이라는 값을 int 대신 Money class)
- 클래스 내에 메소드를 통해 기능 추가가 가능.

밸류 타입은 Immutable 해야 함.<br>
즉, 밸류 타입의 모든 필드는 Immutable 해야 하며, 그 이유는 **엔티티와 달리 동일성을 비교할 때 식별자가 아닌 값 자체로 비교하기 때문**


### Setter 지양
Setter를 사용하게 되면, 도메인 의미를 표현하기 힘들어짐.<br>
단순하게 값을 지정하는 것에는 아무런 도메인 의미가 없기 때문임.<br>
또한 도메인 모델이 올바른 상태를 유지하지 않을 수 있음.<br>
(정해진 도메인 규약을 무시하고 값이 변경될 수 있기 때문)

## 도메인 용어와 유비쿼터스 용어

도메인에서 사용하는 용어를 코드에 최대한 반영해야 함.<br>
에릭 에반스는 이를 유비쿼터스 언어(Ubiquitous Language)라고 부름.<br>
도메인 전문가와 개발자가 관련된 공통의 언어를 사용하는 것을 뜻함.

