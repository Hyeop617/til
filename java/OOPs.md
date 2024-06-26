# OOPs



## OOPs (Ordinary Object Pointer)

자바의 모든 객체는 OOP (Ordinary Object Pointer, 평범한 객체 포인터) 라고 불리는 객체로 표현됨.
Java SE의 표준 JVM 중 자주 쓰이는 HotSpot VM에서 OOP의 구조는 [oopDesc](https://github.com/openjdk/jdk21/blob/master/src/hotspot/share/oops/oop.hpp)라는 파일에 정의되어 있음.

JVM 내부에서 객체를 참조하는 포인터를 뜻함!
따라서 인스턴스가 생성될 때마다 인스턴스에 대한 정보를 담은 OOP도 같이 할당됨.
OOP는 JVM의 힙 메모리에서 객체를 가리키며, 객체의 실제 데이터와 메타 데이터를 접근하는데 사용됨.

> 그렇다면 인스턴스 생성시마다, OOP도 같이 생성되는데 성능 이슈는?

C에서는 malloc를 이용해 메모리 확보를 함. 이 때 시스템 콜이 발생해 오버헤드 발생.
그러나 자바에선 JVM이 확보한 힙 메모리에 할당하므로 시스템 콜 호출할 필요가 없음.

---

## instanceOOP

일반적인 인스턴스를 가리킬 때 사용되는 OOP.
예를 들어, `new Integer(42)`와 같은 객체 인스턴스는 `InstanceOOP`로 참조됨.

- mark word (4 bytes in 32bit, 8 bytes in 64bit)
- klass word (4 bytes in 32bit, 64bit w/ compressedOOPs, 8 bytes in 64bit)
- (Optional) padding (4 bytes in 64bit w/ compressedOOPs)
- data w/ padding

이렇게 Mark Word와 Klass Word라는 헤더와 실제 데이터 총 3가지로 구성

## arrayOOP

배열을 가리킬때 사용되는 OOP
예를 들어, `new int[10]`과 같은 배열은 arrayOOP로 참조됨.

- mark word
- klass word
- **array length (4 bytes)**
- array elements data w/ padding



### Mark Word

인스턴스의 메타데이터를 가리키는 포인터. cpp파일을 참조하면 uintptr_t를 사용하고 있음.
따라서 32비트에선 4바이트, 64비트에선 8바이트를 차지

##### 32bit (4 bytes)

######  jdk 8 normal state

`|---- identity hash code ----|-- age --|- biased_lock -|- lock -|`
`|-------------25-------------|----4----|-------1-------|---2----|`

age: GC에서 몇 번 살아남았는지에 대한 정보

###### jdk 21 normal state

`|---- identity hash code ----|-- age --|- unused_gap -|- lock -|`
`|-------------25-------------|----4----|------1-------|---2----|`



##### 64bit (8 bytes) w/o compressedOOPs

###### jdk 8 normal state

`|---- unused ----|---- identity hash code ----|- unused_gap -|-- age --|- biased_lock -|- lock -|`
`|-------25-------|-------------31-------------|-------1------|----4----|-------1-------|----2---|`

###### jdk 21 normal state

`|---- unused ----|---- identity hash code ----|- unused_gap -|-- age --|- unused_gap -|- lock -|`
`|-------25-------|-------------31-------------|-------1------|----4----|-------1------|----2---|`



##### 64bit (8 bytes) w/ compressedOOPs

###### jdk 8 normal state

`|---- unused ----|---- identity hash code ----|- cms_free -|-- age --|- biased_lock -|- lock -|`
`|-------25-------|-------------31-------------|------1-----|----4----|-------1-------|----2---|`

###### 





64 bit VM은 32비트보다 더 긴 해시값을 필요로 하지 않는다.
-> hash_mask가 32비트라고 가정하고 있기 때문 -> 해시 테이블의 크기가 2^32라고 가정하고 있음









[Hotspot Java 15 source](https://github.com/openjdk/jdk15/blob/e208d9aa1f185c11734a07db399bab0be77ef15f/src/hotspot/share/oops/oop.hpp#L56)

#### Klass Word

클래스의 메타 데이터를 가리키는 포인터. 클래스의 메타 정보에 대한 참조를 저장해둔 포인터.
자바 7까지는 Permanent Generation을 가리키고 있었음. 자바 8부터 metaspace를 가리키도록 변경

```java
// Entry 인스턴스는 32bit JVM에서 총 24 byte를 사용.
// Mark Word - 4 byte
// Klass Word - 4 byte
static class Entry<K,V> implements Map.Entry<K,V> {
    final K key;			// 4 byte reference
    V value;					// 4 byte reference
    Entry<K,V> next;	// 4 byte reference
    final int hash;		// 4 byte instance variable
}
```

JVM에서는 헤더 + 인스턴스 변수 + 패딩 정렬을 통해 메모리를 할당

> (Field Alignment) 패딩 정렬
>
> - JVM은 성능을 위해 필드들을 특정 바이트 경계에 정렬합니다. 따라서 실제로 사용되는 메모리보다 더 많은 메모리가 사용될 수 있습니다. 보통 4바이트 경계에 정렬됩니다.
> - 프로세서의 캐시는 메모리 블록 단위로 데이터를 저장. 패딩이 되어있지 않으면 여러 캐시 라인에 걸쳐서 데이터가 저장 될 수 있음 -> cache miss 발생 확률 올라감
> - JVM은 여러 플랫폼에서 동일하게 동작해아함. JVM은 메모리 정렬을 통해 객체의 배치를 표준화 함. (e.g. 32bit JVM은 4바이트 정렬이 기본값, 64bit JVM은 8바이트 정렬이 기본값)

```java
// Entry 인스턴스는 32bit JVM에서 총 24 byte를 사용.
// Mark Word - 4 byte
// Klass Word - 4 byte
static class Entry<K,V> implements Map.Entry<K,V> {
    final K key;			// 4 byte reference
    V value;					// 4 byte reference
    Entry<K,V> next;	// 4 byte reference
    final short hash;		// 2 byte instance variable, but 2 byte padding added.
}
```

```java
// Entry 인스턴스는 64bit JVM에서 총 48 byte를 사용.
// Mark Word - 8 byte
// Klass Word - 8 byte (w/o compressedOOPs)
static class Entry<K,V> implements Map.Entry<K,V> {
    final K key;			// 8 byte reference
    V value;					// 8 byte reference
    Entry<K,V> next;	// 8 byte reference
    final int hash;		// 4 byte instance variable, but 4 byte padding added.
}
```





### KlassOOP

Java 클래스를 가리킬때 사용되는 OOP. 각 Java 클래스는 JVM에서 메타데이터를 포함하는 객체로 표현되며, 이는 해당 클래스의 모든 인스턴스가 공유하고 있음.

- mark word (4 - 8 bytes)
- klass word (4 - 8 bytes)
- method vtable



클래스의 필드, 메소드, 인터페이스 정보등이 포함되어 있음.

인스턴스는 자신의 KlassOOP를 통해 클래스 메타데이터에 접근함.





## Compressed OOPs

64비트 시스템에선 java object reference의 크기가 32비트 보다 2배의 공간을 차지함.
따라서 많은 메모리를 차지하고, 이 것은 잦은 GC의 원인이 됨. -> 성능 감소





Reference

https://velog.io/@dev_dong07/Java-%EA%B0%9D%EC%B2%B4%EB%8A%94-%EC%96%B4%EB%96%BB%EA%B2%8C-%EC%9D%B4%EB%A3%A8%EC%96%B4%EC%A0%B8-%EC%9E%88%EC%9D%84%EA%B9%8C

ChatGPT

https://www.baeldung.com/java-memory-layout
https://www.baeldung.com/jvm-compressed-oops

https://www.logpresso.com/ko/blog/2017-01-12-auto-boxing-penalty