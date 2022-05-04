## 220504 - ReentrantLock

```java
package java.util.concurrent.locks;

public class ReentrantLock implements Lock, Serializable {
  	private static final long serialVersionUID = 123123123123123123123;
  /** Synchronizer providing all implementation mechanics */
	  private final Sync sync;
  
  /**
     * Base of synchronization control for this lock. Subclassed
     * into fair and nonfair versions below. Uses AQS state to
     * represent the number of holds on the lock.
     */
    abstract static class Sync extends AbstractQueuedSynchronizer {
        private static final long serialVersionUID = -5179523762034025860L;
    }
  
  /**
     * Acquires the lock.
     *
     * <p>Acquires the lock if it is not held by another thread and returns
     * immediately, setting the lock hold count to one.
     *
     * <p>If the current thread already holds the lock then the hold
     * count is incremented by one and the method returns immediately.
     *
     * <p>If the lock is held by another thread then the
     * current thread becomes disabled for thread scheduling
     * purposes and lies dormant until the lock has been acquired,
     * at which time the lock hold count is set to one.
     */
    public void lock() {
        sync.acquire(1);
    }
  
   /**
     * Attempts to release this lock.
     *
     * <p>If the current thread is the holder of this lock then the hold
     * count is decremented.  If the hold count is now zero then the lock
     * is released.  If the current thread is not the holder of this
     * lock then {@link IllegalMonitorStateException} is thrown.
     *
     * @throws IllegalMonitorStateException if the current thread does not
     *         hold this lock
     */
    public void unlock() {
        sync.release(1);
    }
}


```

ReentrantLock은 Java에서 Lock을 하기 위해 사용되는 클래스다. (JDK 5에 추가됨)

이너 클래스로 AbstractQueuedSynchronizer를 상속받은 Sync 클래스가 있다.

```java
import java.util.concurrent.locks.ReentrantLock;

class Myclass {
	ReentrantLock reentrantLock = new ReentrantLock();
  reetrantLock.lock();
  try {
    // blah blah..
  } catch (Exception e){
    // blah blah..
  } finally {
    reetrantLock.unlock();  
  }


}
```



lock() 메소드를 이용해서 lock을 걸 수 있다.

락 걸은 스레드가 락을 해제하니까 Mutex 인가 싶다.

그러면 synchronized 키워드와 뭐가 다를까?



1. 명시적인 lock이 가능하다.
   - lock을 얻고 해제하는 것을 코드로 좀 더 명확하게 사용 할 수 있다.
2. Condition을 이용해서 각 스레드 풀을 관리 가능하다.
   - 임계 영역 부분을 진입할 때 서로 역할이 있는 경우, reentrantLock.getCondition();으로 각 스레드의 역할 지정 가능.
3. 더 유연하게 사용가능하다.
   - 락을 얻으려고 하는데 실패한경우, 락을 몇 초동안 얻으려고 시도하는지 등 더 자세한 세팅이 가능.



ReentrantLock에는 Condition을 이용해서 스레드 풀을 관리 가능하다.

Condition을 사용해서 관리해보자.

[JUNHYUNNY님의 블로그](https://junhyunny.blogspot.com/2019/02/lock-condtion.html)에 관련 코드가 설명이 잘 되어있어 참고하였다.



일단 자바 병렬 프로그래밍 도서가 오면 더 자세히 정리해보자...