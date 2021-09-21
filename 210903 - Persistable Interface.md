## 210903 - Persistable Interface

JPA를 이용하면서 보통은 @Id에 @GeneretedValue 어노테이션을 달것이다.<br>그리고 insert할 땐 Id에 아무런 값을 넣지 않는게 일반적이다.

그런데 이번에 insert 할 때에도 Id에 값을 넣어야 하는 경우가 생겼다.<br>아무런 의심없이 Id에 값을 넣고 insert를 하는데 다음과 같은 오류가 났다.

> ### javax.persistence.EntityNotFoundException: Unable to find Entity with id 1

보자마자 생각난 것은,<br>*아... update 쿼리를 날렸구나* 였다.<br>원인을 알았으니 해결책을 찾아야 한다.<br>구글링을 하다가 [갓영한님의 답변](https://www.inflearn.com/questions/197446)을 보았다.

> @Id를 직접 할당하는 전략을 사용한다면 스프링 데이터 JPA가 제공하는 Persistable 인터페이스를 구현해서 최초에 생성된 객체인지 아닌지 스프링 데이터 JPA에 알려주어야 합니다. 이 부분은 Persistable로 검색해보시면 방법을 찾으실 수 있을거에요.

Persistable 인터페이스라는 힌트를 얻었으니 관련해 검색해보았다.<br>[skyer0](https://www.skyer9.pe.kr/wordpress/?p=1515)님의 블로그를 보고 다음과 같은 힌트를 얻었다.<br>예제 코드는 다음과 같다.

```java
@Setter
@Getter
@NoArgsConstructor
@Entity
@Table(name = "tbl_current_stock_summary", catalog = "db_summary")
public class CurrentStockSummary extends BaseTimeEntity implements Persistable<String> {

    @Id
    private String skuCd;

    // ......

    @Override
    public String getId() {
        return skuCd;
    }

    @Override
    public boolean isNew() {
        return getRegdate() == null;
    }
}
```

Persistable 인터페이스를 구현해서 getId와 isNew를 구현한 코드다.<br>직관적으로 isNew()를 사용해서 insert, update를 구분할 것이다.<br>현재 Jpa Auditing을 이용해서 insert일 때 시간과 같은 값을 넣어주고 있다.

```java
@Getter
@MappedSuperclass
@EntityListeners(AuditingEntityListener.class)
public abstract class BaseEntity {

    private String systemDiv = "HELLO";
    
    private String regUserId;
    
    private String regDt;
    
    private String lastChangeUserId;
    
    private String lastChangeDt;
    
    private String createDt;

    @PrePersist
    public void init(){
        regUserId = "SYSTEM";
        String now = DateUtil.yyyyMMddHHmmss();
        this.regDt = now;
        createDt = now;
        lastChangeUserId = "SYSTEM";
        lastChangeDt = now;
    }

    @PreUpdate
    public void initUpdate(){
        lastChangeUserId = "SYSTEM";
        lastChangeDt = DateUtil.yyyyMMddHHmmss();
    }
}
```

@PrePersist 에서 사용하고 있는 값으로 판단하면 된다는 생각이 들었다.<br>그 중 regUserId의 값이 null인지 판단해서 isNew()를 구현하였다.

```java
	 	@Override
    public boolean isNew() {
        return this.getRegUserId() == null;                   // "SYSTEM" 없으면 insert, 있으면 update
    }
```

그리고 다시 시도하니..<br>제대로 insert가 된다.<br>끝!