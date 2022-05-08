# 220506 - Lazy Evaluation In Java



Lazy Evalualtion (느긋한 계산법)에 대해서 알아보자.



```java
public boolean isContainsUnderbar(String s) throws InterruptedException {
  Thread.sleep(1000);
  return s.contains("_");
}
```



해당 메소드는 파라미터로 받은 String s에 언더바가 포함되어 있는지 여부를 반환하는 메소드다.
이 메소드의 실행시간은 약 1초일 것이다.



다음 메소드를 보자.

```java
public String eagerContains(boolean b1, boolean b2) {
  return b1 && b2 ? "yes" : "no";
}
```



위 메소드는 b1과 b2를 바로 받아서 삼항 연산자로 판단하는 메소드다.
만약 b1이 false라면 뒤의 b2는 볼 필요가 없다.
따라서 해당 코드도 Lazy Evaluation의 일종이라고 볼 수 있다.



```java
boolean b1 = isContainsUnderbar("now");
boolean b2 = isContiansUnderbar("end");
eagerContains(b1, b2);
```



위의 코드를 실행하면 isContainsUnderbar를 2번 실행하므로 총 실행시간은 약 2초일 것이다.
b1이 false 이어도 b2는 실행이 될 것이다.
위와 같은 코드를 Eager Evaluation이라고 한다.



기본적으로 Java는 Eager Evaluation이다.
해당 부분을 Lazy Evaluation으로 변경하려면 어떻게 해야할까?



```java
public String lazyContains(String s1, String s2) {
  return isContainsUnderbar(s1) && isContainsUnderbar(s2) ? "yes" : "no";
}
```



이렇게 조건문에 직접 명시하는 방법이 있다.

만약 isContainsUnderbar(s1)이 false라면 약 1초만에 끝날 것이다.
하지만 해당 조건 자체를 재활용이 안된다는 단점이 있다.



Java 8 이후로는 Supplier를 이용해서 해결 할 수 있다.

```java
Supplier<Boolean> supplierA = () -> isContainsUnderbar("now");
Supplier<Boolean> supplierB = () -> isContainsUnderbar("end");
lazyContainsAfter8(supplierA, supplierB);

public String lazyContainsAfter8(Supplier<Boolean> sa, Supplier<Boolean> sb) {
  return sa.get() && sb.get() ? "yes" : "no";
}
```



이렇게 한다면 sa가 false라면 총 실행시간은 약 1초일 것이다.

이렇게 필요하지 않은 연산을 하지 않을 수 있다는 장점이 있다.

그렇다면 장점만 있는 것일까?

아니다. 단점도 있다.



단점의 경우에는 해당 Supplier가 한번에 많은 양의 계산을 할 때다.
한 번에 많은 양의 데이터를 불러온다면, 메모리 오버헤드가 심하게 된다.
Eager Evaluation이라면 미리 불러와서 계산을 하겠지만, 동적으로 많은 데이터를 불러오는 것은 부담을 줄 수 있다.



각 장단점을 잘 파악해서 사용하도록 하자.



참고 : [사바라다는 차곡차곡](https://sabarada.tistory.com/153)