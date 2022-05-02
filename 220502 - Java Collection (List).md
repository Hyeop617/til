## 220502 - Java Collection (List)

Java 콜렉션에 대해서 공부를 하고자 한다.
흔히 사용하는 List, Map, Set 부터 시작해보자.

java.util.List는 먼저 인터페이스다.
JDK 11(어댑티움 테무린) 기준으로 눈여겨볼 리스트의 구현체는 다음과 같겠다.

- java.util.ArrayList
- java.util.LinkedList
- java.util.Collections.SynchronizedList
- java.util.concurrent.CopyOnWriteArrayList
- java.util.Vector



각 리스트별 차이점을 알아보자.

ArrayList<E>는 임시 배열(elementData)을 내부적으로 생성해서 값을 저장한다.
처음 ArrayList의 elementData의 DEFAULT_CAPACITY는 10이다.
empty list에 addAll 등으로 컬렉션을 추가할 때, 해당 컬렉션 사이즈가 10보다 크다면, 해당 컬렉션의 사이즈만큼 ArrayList의 배열 사이즈를 지정한다.



다음은 14개의 콜렉션을 추가한 뒤, 한개의 Integer를 add하는 코드다.

![스크린샷 2022-05-02 오전 10.27.27](https://tva1.sinaimg.cn/large/e6c9d24egy1h1tsxppu2hj20i103m3yp.jpg)





```java
private static final int DEFAULT_CAPACITY = 10;

private static final int MAX_ARRAY_SIZE = Integer.MAX_VALUE - 8;
```



![스크린샷 2022-05-02 오전 10.27.09](https://tva1.sinaimg.cn/large/e6c9d24egy1h1tsxfsm0qj20l30hugnq.jpg)



일단 현재 배열은 비어있기에 oldCapacity, newCapacity는 모두 0에, minCapacity는 14가 들어온다.
따라서 Math.max(DEFAULT_CAPACITY, minCapacity)의 값은 14가 된다.

그리고 다음 3을 추가할 때를 보자.


![스크린샷 2022-05-02 오전 10.32.29](https://tva1.sinaimg.cn/large/e6c9d24egy1h1tt2yq8bzj20kv0b6dhb.jpg)

먼저 add로 한 개의 요소가 들어오면, grow() 함수를 호출해서 minCapacity는 List의 사이즈 + 1이 된다. (여기선 15)
newCapacity를 보면 oldCapacity(임시 배열의 크기)에 1.5를 곱한 것을 알 수 있다.
이런식으로 ArrayList는 elementData의 크기를 늘리면서 요소들을 관리한다.



***



LinkedList<E>는 내부적으로 Node를 사용해서 리스트를 관리한다.
또한 ArrayList는 MAX_ARRAY_SIZE가 있지만, LinkedList는 공간의 제약을 받지 않는다.
따라서, JVM이 허용하는 한(heap memory) 무제한 늘릴 수 있다.
그러나, LinkedList의 size는 int 자료형으로 관리하고 있기 때문에, overflow가 발생할 수 있다고 한다.

***

Vector<E>는 현재는 잘 사용하지 않는 콜렉션이다.
동작 자체는 ArrayList와 비슷하나, elementData의 증가 크기(capacityIncrement)를 Vector 인스턴스 생성 때 직접 지정할 수 있다.
(capacityIncrement의 default value는 0이며 이럴 땐 elementData는 2배씩 늘어난다.)
그리고 무엇보다 synchronized 키워드가 각 **메소드** 앞에 붙어있다.

즉 thread-safe한 자료구조다.
하지만 synchronozied 키워드 때문에 멀티 스레드가 필요하지 않을 땐 성능저하가 발생한다.



***

이젠 Collections.SynchronizedList와 CopyOnWriteArrayList를 동시에 보자.
둘다 thread-safe한 자료구조다.

CopyOnWriteArrayList의 이름에서 볼 수 있듯이, 이 자료구조는 add 시에 배열을 복사 후 복사한 배열에 집어 넣는다.

![스크린샷 2022-05-02 오후 12.27.36](https://tva1.sinaimg.cn/large/e6c9d24egy1h1twepfvsij20e708i0t3.jpg)



get()의 경우에는 lock이 안 걸려져있는 것을 확인할 수 있다.

![스크린샷 2022-05-02 오후 12.29.29](https://tva1.sinaimg.cn/large/e6c9d24egy1h1twgpqiv0j20bi03rjre.jpg)



그렇다면 SynchronizedList의 경우에는 어떨까?
![스크린샷 2022-05-02 오후 12.30.39](https://tva1.sinaimg.cn/large/e6c9d24egy1h1twhvlz08j20f60nnwgs.jpg)

get, set, add 등등 모두 lock이 걸려져 있는 것을 확인할 수 있다.
어.. 그러면 Vector와 비슷한 거 아닌가?
Vector는 메소드 앞에, SynchronizedList는 코드블록에 synchronized 키워드가 있다.
음.. 그런데 결국 메소드 전체 아닌가..?

구글링을 해보니 Vector는 동기적인 큐 기반이고, 배열 복사 방식이며
SynchronizedList는 각 List를 쓰레드 세이프하지 않는 큐에서 쓰레드 세이프한 큐로 옮기는 래퍼 클래스라고 한다.
그리고 Collections 클래스의 상속 유무도 큰 것 같다.



따라서 SynchronizedList는 읽기 작업시에는 CopyOnWriteArrayList보다 느리지만, (lock)
쓰기 작업시에는 CopyOnWriteArrayList 보다 빠르다. (배열 복사를 안해도 되므로)