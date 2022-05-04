# 220504 - Spin Lock, Mutex, Semaphore

[쉬운코드님의 유튜브](https://www.youtube.com/watch?v=gTkvX2Awj6g)를 보고 정리한 글입니다.

동기화의 3대장

- Spin Lock
- Mutex
- Semaphore

에 대해서 정리를 해보자.



### Race Condition

- 여러 프로세스/스레드가 동시에 같은 데이터를 조작할 때 타이밍이나 접근 순서에 따라 결과가 달라질 수 있는 상황



### Synchronization

- 여러 프로세스/스레드를 동시에 실행해도 공유 데이터의 일관성을 유지하는 것



### Critical Section

- 공유 데이터의 일관성을 보장하기 위해 하나의 프로세스/스레드만 진입해서 실행가능한 영역



### Mutual Exclusion

- 하나의 프로세스/스레드만 진입해서 실행하는 것



그러면 어떻게 Mutual Exclusion을 보장해야할까?

-> Lock을 이용하자.



## Spin Lock



```c++
volatile int lock = 0;		// global

void critical() {
  while (test_and_set(&lock) == 1);							// lock 검사
  
  ... critical section
    
  lock = 0;
}

int test_and_set(int* lockPtr){
  int oldLock = *lockPtr;
  *lockPtr = 1;
  return oldLock;
}
```



T1이 먼저 critical()을 실행한다 해보자.
먼저 test_and_set에서 oldLock에 현재 global lock의 값인 0을 넣어준다.
그리고 global lock을 1로 바꿔줄 것이고, oldLock을 리턴한다.
반환 값이 0이기 때문에 while 문을 탈출한다.
그리고 critical section에 진입을 한다.

이 때 T2가 critical()을 실행한다고 해보자.
T2는 test_and_set에서 oldLock은 global Lock의 값인 1로 초기화를 한다.
그리고 global Lock을 1로 바꿔줄 것이다. (이미 1이지만)
그리고 oldLock을 리턴한다.
oldLock이 1이기 때문에 while문을 반복해서 돈다.
언제까지? T1이 critical section을 끝내고 lock을 0으로 바꿀 떄 까지.



즉 락을 건 스레드가 있을 것이다.
그 스레드가 직접 락을 해제할 때까지 다른 스레드는 계속 while loop을 돌면서 락을 획득하려고 시도한다.



*그런데 test_and_set을 동시에 여러 스레드가 접근하면?? 동시성이 깨지지 않나?*



test_and_lock은 CPU의 atomic 명령어다.

#### atomic

- 실행 중간에 간섭받거나 중단되지 않는다. (비선점)
- 같은 메모리 영역에 대해 동시에 실행되지 않는다.



Spin Lock의 단점은 Lock을 획득하려고 하는 동안 계속 while문을 돌면서 CPU의 자원을 낭비한다.
이를 개선하기 위해서 **Mutex**가 나왔다.





## Mutex



```c++
class Mutex {
  int value = 1;
  int guard = 0;
}

Mutex::lock() {
  while (test_and_set(&guard));
  
  if(value == 0){
    현재 스레드를 큐에 넣음
    guard = 0; & go to sleep
  } else {
    value = 0;
    guard = 0;
  }
}

Mutex::unlock() {
  while(test_and_set(&guard));
  
  if (큐에 하나라도 대기중이라면){
    그 중에 하나를 깨운다.
  } else {
    value = 1;
  }
  guard = 0;
}

int test_and_set(int* lockPtr){
  int oldLock = *lockPtr;
  *lockPtr = 1;
  return oldLock;
}

mutex->lock();
... critical section;
mutex->unlock();
```



Mutex의 value로 동기화 문제를 해결한 코드다.
그리고 Spin Lock처럼 한 스레드만 value를 수정할 수 있게끔 test_and_set에 guard를 넣어서 사용했다.
만약 value가 1이라면 lock을 건 후에 critical section으로 들어갈 것이고,
0이라면 큐에 들어간 다음 대기중일 것이다.
작업을 마친뒤엔 guard를 0으로 바꿔 다시 다른 스레드가 접근 가능하게 한다.

unlock을 할 땐 다른 스레드가 큐에 있어서 대기중이라면, 그 큐를 깨울 것 이고
큐가 비었다면 value를 1로 바꾸고 종료될 것이다.



두가지 포인트가 있다.

- Mutex의 value로 큐에 들어가거나 실행하거나를 결정한다.
- test_and_set에 Mutex의 guard를 넣어서 CPU의 atomic 명령어를 사용한다.



즉 락을 가질 때 까지 휴식하는 방식이다.



그러면 스핀락 보다 뮤텍스가 항상 좋냐?

-> 아니다.

멀티코어 환경에서 Critical Section에서의 작업이 Context Switiching보다 더 빨리 끝난다면 스핀락이 더 이점이 있다.





### Semaphore

Signal mechanism을 가진 하나 이상의 프로세스/스레드가 critical section에 접근 가능하도록 하는 장치



```C++
class Semaphore {
  int value = 1;
  int guard = 0;
}

Semaphore::wait() {
  while (test_and_set(&guard));
  
  if(value == 0){
    ... 현재 스레드를 큐에 넣음;
    guard = 0;
    &go to sleep
  } else {
    value -= 1;
    guard = 0;
  }
}

Semaphore::signal(){
  while(test_and_set(&guard));
  if (큐에 하나라도 대기중이라면){
    그 중에 하나를 깨운다.
  } else {
    value += 1;
  }
  guard = 0;
}

int test_and_set(int* lockPtr){
  int oldLock = *lockPtr;
  *lockPtr = 1;
  return oldLock;
}

semaphore->wait();
... critical section;
semaphore->signal();
```



세마포어는 순서를 정해줄 때 사용할 수 있다.



```c++
// Process 1
task1:
semaphore->signal();

// Process 2
task2:
semaphore->wait();
task3;

```

task3는 반드시 task1이 수행된 후에 실행된다.

P1이 먼저 실행되었을 때를 보자.

task1을 수행하고 signal()을 실행한다. Semaphore의 value는 1이 될 것이다.
그리고 P2가 task2를 수행하고 wait()에서 value를 0으로 바꿀 것이다.
그리고 task3이 수행된다.



P2가 먼저 실행될 떄를 보자.

task2를 수행하고 wait()이 실행하며 큐에 자신이 들어갈 것이다.
그리고 P1이 task1을 수행하고 signal()을 호출해서 P2를 깨울 것이다.
그리고 task3이 수행된다.



여기서 알 수 있는 점은 다른 프로세스가  singal, wait을 호출했다.

Mutex와 Binary Semaphore는 똑같은 걸까?

**아니다. Mutex는 락을 걸은 프로세스만이 락을 해제하지만, 세마포어는 그렇지 않다.**

그리고 또 뭐가 다른 것일까?

**Mutex는 priority inheritance 속성을 가지지만, 세마포어는 그 속성이 없다.**



priority inheritance가 뭘까..

### Priority inheritance

CPU의 스케쥴링 방식은 대체적으로 우선순위 방식을 따른다. (Round Robin과 섞어서 사용하는 것으로 알고 있다.)

이 때 우선순위가 높은 P1과 우선순위가 낮은 P2가 있다고 치자.
이럴 때 P2가 락을 획득하면 P1은 락을 얻을 수 있을 때 까지 계속 대기하게 된다.
다른 프로세스도 존재하는 상황이라면? (P3, P4, ...) P2의 차례가 올 때까지 계속 아무 작업도 못할 것이다.



이럴 때 Mutex는?
락을 건 프로세스만 락을 해제할 수 있으므로, 이런 상황에서 P2의 우선순위를 P1 만큼 올려버린다.
그러면 P2가 우선순위가 높으니 실행 될 것이고 빨리 critical section을 처리 후 락을 반환할 것이다.
이렇게 우선순위를 올리는 작업을 Priority Inheritance(우선순위 상속)이라고 한다.



### 정리하자면?

- ### 상호 배제가 필요하다면 Mutex

- ### 작업 간의 실행 순서 동기화가 필요하다면 Semaphore