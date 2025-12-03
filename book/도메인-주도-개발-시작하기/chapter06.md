# 6장 - 응용 서비스와 표현 영역

## 표현 영역과 응용 영역

도메인 영역은 사용자의 요구를 충족하는 소프트웨어를 만들기 위해 중요한 영역이다.<br>
그러나 사용자와 도메인을 연결해주는 매개체는 표현 영역과, 응용 영역도 중요한 영역이다.<br>

표현 영역은 사용자의 요청을 해석한다.<br>
표현 영역에서는 요청 파라미터, URL, 쿠키, 헤더 등을 이용해서 사용자의 요청을 해석하고, 응용 서비스를 실행한다.<br>

사용자가 원하는 기능을 제공하는 곳은 응용 영역에 위치한 서비스다.<br>
사용자가 '회원가입'을 요청했다면 그 기능의 주체는 응용 영역에 위치한 서비스 (응용 서비스)가 된다.<br>
응용 서비스는 파라미터로 필요한 입력 값을 받고, 그 결과를 리턴한다.<br>

표현 영역에서 받은 사용자의 요청 데이터와 응용 서비스가 필요한 파라미터는 서로 다를 수 있다.<br>
따라서 표현 영역은 응용 서비스가 필요로 하는 요청 파라미터를 조합하여 응용 서비스에게 전달해야 한다.<br>

```java
@PostMapping("/member/join")
public ModelAndView join(HttpServletRequest request) {
  String email = request.getParameter("email");
  String password = request.getParameter("password");
  // 응용 서비스에 맞게 사용자의 요청을 변환
  JoinRequest req = new JoinRequest(email, password);
  // 변환된 객체를 이용하여 응용 서비스를 실행
  joinService.join(req);
  // ...
}
```
사용자와 상호작용은 표현 영역이 처리하기 때문에, 응용 서비스는 표현 영역에 의존하지 않는다.<br>
응용 영역은 사용자가 HTTP 요청을 했는지, TCP 소켓 요청인지 등을 알 필요가 없다.<br>

## 응용 서비스의 역할

응용 서비스는 사용자가 요청한 기능을 실행한다.<br>
사용자의 요청을 처리하기 위해 레포지토리에서 도메인 객체를 가져와서 사용한다.<br>

응용 서비스의 주 역할은 도메인 객체를 사용해 사용자의 요청을 처리하는 것이므로,<br>
표현 영역 입장에선 도메인 영역과 표현 영역을 연결해주는 창구 역할을 해주는 곳이다.<br>

응용 서비스는 주로 도매인 객체간의 흐름을 조절하는 역할을 하기에, 복잡하지 않다.<br>
```java
public Result doSomeFunc(SomeReq req) {
  // 1. 레포지토리에서 애그리거트를 구한다.
  SomeAgg agg = someAggRepository.findById(req.getId());
  checkNull(agg);
  
  // 2. 애그리거트의 도메인 기능을 실행한다.
  agg.doFunc(req.getValue());
  
  // 3. 결과를 리턴한다.
  return createSuccessResult(agg);
}

public Result doSomeCreation(CreateSomeReq req) {
  // 1. 데이터 중복 등 데이터가 유효한지 검사한다.
  validate(req);
  
  // 2. 애그리거트를 생성한다.
  SomeAgg agg = createSome(req);
  
  // 3. 리포지토리에 애그리거트를 저장한다.
  someAggRepository.save(agg);
  
  // 4. 결과를 리턴한다.
  return createSuccessResult(agg);
}
```
만약 응용 서비스가 복잡하다면 도메인 로직이 포함되었을 가능성이 높다.<br>
응용 서비스에 도메인 로직이 포함되면 코드 중복, 로직 분산 등 코드 퀄리티에 안좋은 영향을 미친다.<br>

응용 서비스는 트랜잭션 처리도 맡는다.<br>
도메인의 상태 변경을 트랜잭션으로 처리할 책임을 갖는다.<br>

트랜잭션 외에 주요 역할은 접근 제어와, 이벤트 처리가 있는데 이는 뒤에서 다루도록 한다.<br>

### 도메인 로직 넣지 않기
예를 들어 사용자의 암호 변경 기능이 있다고 해보자.<br>
```java
public class ChangePasswordService {
  public void changePassword(String memberId, String oldPassword, String newPassword) {
    Member member = memberRepository.findById(memberId);
    checkMemberExists(member);

    if (!passwordEncoder.match(oldPassword, member.getPassword())) {
      throw new IllegalArgumentException("old password not match");
    }
    
    member.changePassword(newPassword);
    memberRepository.save(member);
  }
}
```
비밀번호를 올바르게 입력했는지 확인하는 것은 도메인의 핵심 로직이기 때문에, 위와 같이 응용 서비스에 구현하면 안 된다.<br>
첫째, 코드의 응집성이 낮아진다.<br>
도메인 데이터와 도메인 로직이 서로 다른 곳에 위치하는 것은, 도메인 로직을 파악하기 위해 여러 곳을 봐야한다는 것을 의미하기 때문이다.<br>
둘째, 여러 응용 서비스에서 동일한 도메인 로직이 중복될 수 있다.<br>

다음과 같이 Member 도메인 객체에 암호 확인 기능을 구현하면 된다.<br>
```java
public class Member {
    private String password;
    
    public void changePassword(String oldPw, String newPw) {
      if (!matchPassword(oldPw)) {
        throw new IllegalArgumentException("old password not match");
      }
      setPassword(newPw);
    }
  
    public boolean matchPassword(String pw) {
      return passwordEncoder.match(pw, this.password);
    }
    
    public void setPassword(String newPw) {
      if(newPw == null || newPw.length() < 8) {
        throw new IllegalArgumentException("password must be at least 8 characters");
      }
      this.password = passwordEncoder.encode(newPw);
    }
}
```
일부 도메인 로직이 응용 서비스에 있으면 발생하는 문제(응집도가 떨어지고 코드 중복 발생)은 결과적으로 코드 변경을 어렵게 만든다.<br>
소프트웨어가 가져야 할 중요한 경쟁 요소 중 하나는 변경 용이성인데, 이는 그만큼 소프트웨어의 가치가 떨어진다는 것을 의미한다.<br>
소프트웨어의 가치를 높이려면 도메인 로직을 도메인 영역에 넣어야 한다.<br>

## 응용 서비스의 구현
응용 서비스는 디자인 패턴에서의 파사드와 같은 역할을 한다.<br>

### 응용 서비스의 크기
응용 서비스의 구현은 어렵지 않지만, 크기 관련해서 고려할 점이 있다.<br>
응용 서비스는 보통 아래 두 가지 방법으로 구현한다.
- 한 응용 서비스 클래스에 도메인의 모든 기능 구현하기
- 구분되는 기능별로 응용 서비스 클래스를 따로 만들기

첫 번째 방법은 관련된 기능을 모두 한 클래스에서 관리하므로, 동일 로직에 대한 중복을 줄일 수 있다.<br>
그러나 클래스의 크기가 너무 커질 수 있다는 단점이 있다.<br>
크기가 커지면 연관성이 적은 코드가 한 클래스에 같이 위치할 가능성이 높아지므로, 결과적으로 코드를 이해하는데 방해가 된다.<br>

두 번째 방법은 기능별로 클래스를 나누므로, 한 응용 서비스 클래스에서 2~3개의 기능을 구현한다.<br>
이 방식을 이용하면 클래스 개수는 많아지지만, 각 클래스의 크기가 작아지므로 코드 품질을 유지하기 쉬워진다.<br>
만약 각 기능별로 동일한 로직이 있다면, Helper 클래스를 만들어서 중복을 제거할 수 있다.<br>
```java
public final class MemberServiceHelper {
  public static Member findExistingMember(MemberRepository repo, String memberId) {
    Member member = repo.findById(memberId);
    if (member == null) {
        throw new IllegalArgumentException("member not found: " + memberId);
    }
    return member;
}
```
### 응용 서비스의 인터페이스와 클래스
응용 서비스를 만들때 인터페이스는 꼭 필요하지 않다.<br>
구현 클래스가 여러개인 경우는 필요하나, 응용 서비스의 구현체가 여러개인 경우는 거의 없기 때문이다.<br> 
테스트 용이성 또한 Mockito를 통해 Mock 객체를 만들 수 있기에, 인터페이스의 필요성은 거의 없다.

### 메소드 파라미터와 값 리턴
응용 서비스의 메소드 파라미터는 별도 클래스(DTO)를 이용하는 것이 좋다.<br>
응용 서비스의 결과값을 표현 영역에서 사용해야 한다면, 필요한 데이터를 리턴하면 된다.<br>
```java
@Service
public class OrderService {
  
  @Transactional
  public OrderNo placeOrder(OrderRequest orderRequest) {
    OrderNo orderNo = orderRepository.nextId();
    Order order = createOrder(orderNo, orderRequest);
    orderRepository.save(order);
    return orderNo;
  }
}

@Controller
public class OrderController {
  
  @PostMapping("/order/place")
  public String order(OrderRequest orderRequest, ModelMap model) {
    setOrderer(orderRequest);
    OrderNo orderNo = orderService.placeOrder(orderRequest);
    model.addAttribute("orderNo", orderNo.toString());
    return "order/success";
  }
}
```
만약 OrderNo가 아니라 Order 애그리거트를 리턴하면 편할 수 있지만 지양해야 한다.<br>
그러면 도메인 로직 실행을 표현 영역에서 할 수 있게 될 수 있기 때문이다.<br>
응용 서비스에서는 표현 영역에서 필요한 최소한의 데이터만 리턴하는 것이 좋다.<br>

### 표현 영역에 의존하지 않기
응용 서비스에서 표현 영역과 관련된 타입을 사용하거나 참조하면 안된다.<br>
예를 들어, HttpServletRequest, HttpSession, ResponseEntity 등이다.<br>
이를 참조한다면 응용 서비스의 단위 테스트가 어려워지고, 표현 영역이 수정되면 응용 서비스도 수정해야 하는 문제가 발생한다.<br>
이를 쉽게 해결하려면 파라미터와 리턴 타입에 표현 영역과 관련된 타입을 사용하지 않으면 된다.<br>

### 트랜잭션 처리
스프링같은 프레임워크가 제공하는 트랜잭션 관리 기능을 이용하여, 응용 서비스에서 트랜잭션 처리를 이용한다.<br>

## 표현 영역
표현 영역의 책임은 다음과 같다.
- 사용자가 시스템을 사용할 수 있는 흐름(화면)을 제공하고 제어한다.
- 사용자의 요청을 알맞은 응용 서비스에 전달하고, 결과를 사용자에게 제공한다.
- 사용자의 세션을 관리한다.

표현 영역의 첫 번째 책임은 사용자가 시스템을 사용할 수 있도록 알맞은 흐름을 제공하는 것이다.<br>
웹 서비스의 표현 영역은 사용자가 요청한 내용을 응답으로 제공한다.<br>
응답은 다음 화면으로 이동할 수 있는 링크나, 데이터를 입력할 수 있는 폼 등이 포함된다.<br>
예를 들어, 사용자가 게시글 쓰기를 요청하면, 게시글 작성 폼을 표현 영역은 제공할 수 있다.<br>
그리고 사용자가 폼에 데이터를 채워서 표현 영역에 제출하면, 표현 영역은 이를 응용 서비스에 전달한 뒤 그 결과를 사용자에게 제공한다.<br>

두 번쨰 책임은 사용자의 요청에 맞게 응용 서비스를 호출하는 것이다.<br>
표현 영역은 사용자의 요청을 응용 서비스가 사용할 수 있도록 변환한 뒤 응용 서비스를 호출한다.<br>
그리고 응용 서비스의 결과를 다시 사용자에게 알맞은 형태로 변환하여 제공한다.<br>
예를 들어, 응용 서비스에서 에러가 나면 표현 영역은 에러를 받아 사용자에게 알맞은 형태로 에러를 처리한다.<br>

또 다른 역할은 세션 관리다.<br>
웹은 쿠키나 서버 세션을 이용해서 사용자의 연결 상태를 관리한다.<br>
세션 관리는 곧 사용자의 권한 검사와도 연결된다.<br>

## 값 검증
값 검증은 표현 영역과 응용 영역 두 곳에서 모두 수행할 수 있다.<br>
원칙적으로는 응용 서비스에서 모든 값을 검증해야 한다.<br>

그러나 표현 영역에서 필수 값을 검증하면, 응용 서비스에서는 ID 중복 여부 등의 논리적인 검증만 수행할 수 있다.<br>
표현 영역에서는 필수 값, 값의 형식, 범위 등을 검증하고, 응용 서비스에서는 데이터 존재 유무 같은 논리적 오류를 검증할 수 있다.<br> 
이는 의견이 갈리는 문제며 저자는 응용 서비스에서 모든 값을 검증하는 것을 선호한다고 한다. (응용 서비스의 코드가 늘어나지만, 완성도가 높아지는 장점이 있다고 함)

## 권한 검사
스프링 시큐리티를 이용하여 권한 검사를 수행할 수 있으나, 스프링 시큐리티는 러닝 커브가 높으므로 이해가 부족하면 유지보수가 어려워질 수 있다.<br>
따라서 상황에 맞게 권한 검사를 직접 구현할 수도 있다.<br>
권한 검사는 보통 아래 부분에서 수행할 수 있다.

- 표현 영역
- 응용 서비스
- 도메인

표현 영역에서 할 수 있는 기본적인 검사는, 사용자의 인증이다.<br>
예를 들어 회원 정보 변경 기능이라면 다음과 같이 접근 제어를 할 수 있다.
- 컨트롤러에 웹 요청을 전달하기 전에 인증 여부를 검사해서 인증된 사용자의 요청만 컨트롤러에 전달한다.
- 인증된 사용자가 아니면 로그인 화면으로 리다이렉트 시킨다.
위 방법을 처리하기 좋은 위치는 서블릿 필터다.<br>
서블릿 필터에서 인증 여부를 검사하고 인증된 사용자가 아니면 로그인 화면으로 리다이렉트 시키면 된다.<br>

위 방법으로 접근 제어를 모두 충족하기 어려운 경우, 응용 서비스의 메소드 단위로 권한 검사를 수행해야 한다.<br>
응용 서비스에서 코드를 직접 작성하지 않고, AOP를 이용하여 권한 검사를 구현할 수 있다.<br>
스프링 시큐리티의 `@PreAuthorize("hasRole('ADMIN')")` 등을 이용하면 쉽게 구현가능하다.<br> 

만약 개별 도메인 객체 단위로 검증을 수행해야한다면 더 복잡해진다.<br>
예를 들어 게시글 수정 기능이 있고, '게시글 작성자'와 '어드민'이 수정 가능하다고 해보자.<br>
그러면 게시글 애그리거트를 먼저 가져온 뒤, 권한 검사를 수행해야 할 것이다.<br> 
```java
public class DeleteArticleService {
  public void delete(String userId, Long articleId) {
    Article article = articleRepository.findById(articleId);
    checkArticleExistence(article);
    permissionService.checkDeletePermission(userId, article); // 권한 검사
    article.markDeleted();
  }
}
```
스프링 시큐리티와 같은 보안 프레임워크를 확장해 도메인 수준의 권한 검사도 가능하다.<br>
하지만 해당 프레임워크의 높은 이해가 없다면 권한 검사를 직접 구현하는 것이 낫다.<br>

## 조회 전용 기능과 응용 서비스
서비스에서 조회 전용 모델과 DAO(조회를 위한)을 사용하면 서비스 코드가 단순해질 수 있다.<br>
```java
public class OrderListService {
  public List<OrderView> getOrderList(String ordererId) {
    return orderViewDao.selectByOrderer(ordererId);
  }
}
```
위 로직은 단순히 조회만 하기에 트랜잭션이 필요하지도 않다.<br>
이런 경우라면 응용 서비스를 만들지 않고, 표현 영역에서 DAO를 바로 사용해도 될 것이다.<br>
응용 서비스를 만들지 않는 것이 이상할 수 있지만, 응용 서비스가 별다른 역할이 없으므로 굳이 서비스를 만들지 않아도 된다.<br>