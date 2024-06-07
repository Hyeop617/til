# Java hash code



### Identity hash code

> the identity hash code is a value decided by the JVM that's constant over the lifetime of an object and can not be influenced by Java code. (i.e. `hashCode()` has no effect on this) This value can be gotten by calling `System.identityHashCode(obj)`
>
> https://stackoverflow.com/questions/76219209/how-is-a-32-bit-hashcode-stored-in-a-25-bit-mark-word-in-java-without-data-loss

> 주어진 객체의 클래스가 hashCode() 를 재정의하는지 여부에 관계없이 기본 메서드 hashCode() 가 반환하는 것과 동일한 객체에 대해 동일한 해시 코드를 반환합니다. Null 참조의 해시 코드는 0입니다.
>
> javadoc

```java
public static native int identityHashCode(Object x);
```



identityHashCode는 JNI를 통해 함수를 호출하고 있다.
그렇다면 identityHashCode는 어떤 JNI 함수를 호출하고 있을까?

### [System.c](https://github.com/openjdk/jdk21/blob/890adb6410dab4606a4f26a942aed02fb2f55387/src/java.base/share/native/libjava/System.c#L53)

```c
JNIEXPORT jint JNICALL
Java_java_lang_System_identityHashCode(JNIEnv *env, jobject this, jobject x)
{
    return JVM_IHashCode(env, x);
}
```



### [jvm.cpp](https://github.com/openjdk/jdk21/blob/890adb6410dab4606a4f26a942aed02fb2f55387/src/hotspot/share/prims/jvm.cpp#L608)

```cpp
JVM_ENTRY(jint, JVM_IHashCode(JNIEnv* env, jobject handle))
  // as implemented in the classic virtual machine; return 0 if object is null
  return handle == nullptr ? 0 : ObjectSynchronizer::FastHashCode (THREAD, JNIHandles::resolve_non_null(handle)) ;
JVM_END
```

identityHashCode는 JVM_IHashCode를 호출하고, JVM_IHashCode 함수는 ObjectSynchronizer::FastHashCode를 호출하고 있다.


### [synchronizer.cpp](https://github.com/openjdk/jdk21/blob/890adb6410dab4606a4f26a942aed02fb2f55387/src/hotspot/share/runtime/synchronizer.cpp#L918)

hashcode는 5가 default. (runtime/globals.hpp에서 확인 가능)

```cpp
// hashCode() generation :
//
// Possibilities:
// * MD5Digest of {obj,stw_random}
// * CRC32 of {obj,stw_random} or any linear-feedback shift register function.
// * A DES- or AES-style SBox[] mechanism
// * One of the Phi-based schemes, such as:
//   2654435761 = 2^32 * Phi (golden ratio)
//   HashCodeValue = ((uintptr_t(obj) >> 3) * 2654435761) ^ GVars.stw_random ;
// * A variation of Marsaglia's shift-xor RNG scheme.
// * (obj ^ stw_random) is appealing, but can result
//   in undesirable regularity in the hashCode values of adjacent objects
//   (objects allocated back-to-back, in particular).  This could potentially
//   result in hashtable collisions and reduced hashtable efficiency.
//   There are simple ways to "diffuse" the middle address bits over the
//   generated hashCode values:
static inline intptr_t get_next_hash(Thread* current, oop obj) {
  intptr_t value = 0;
  if (hashCode == 0) {
    // This form uses global Park-Miller RNG.
    // On MP system we'll have lots of RW access to a global, so the
    // mechanism induces lots of coherency traffic.
    value = os::random();
  } else if (hashCode == 1) {
    // This variation has the property of being stable (idempotent)
    // between STW operations.  This can be useful in some of the 1-0
    // synchronization schemes.
    intptr_t addr_bits = cast_from_oop<intptr_t>(obj) >> 3;
    value = addr_bits ^ (addr_bits >> 5) ^ GVars.stw_random;
  } else if (hashCode == 2) {
    value = 1;            // for sensitivity testing
  } else if (hashCode == 3) {
    value = ++GVars.hc_sequence;
  } else if (hashCode == 4) {
    value = cast_from_oop<intptr_t>(obj);
  } else {
    // Marsaglia's xor-shift scheme with thread-specific state
    // This is probably the best overall implementation -- we'll
    // likely make this the default in future releases.
    unsigned t = current->_hashStateX;
    t ^= (t << 11);
    current->_hashStateX = current->_hashStateY;
    current->_hashStateY = current->_hashStateZ;
    current->_hashStateZ = current->_hashStateW;
    unsigned v = current->_hashStateW;
    v = (v ^ (v >> 19)) ^ (t ^ (t >> 8));
    current->_hashStateW = v;
    value = v;
  }

  value &= markWord::hash_mask;
  if (value == 0) value = 0xBAD;
  assert(value != markWord::no_hash, "invariant");
  return value;
}

// Can be called from non JavaThreads (e.g., VMThread) for FastHashCode
// calculations as part of JVM/TI tagging.
static bool is_lock_owned(Thread* thread, oop obj) {
  assert(LockingMode == LM_LIGHTWEIGHT, "only call this with new lightweight locking enabled");
  return thread->is_Java_thread() ? JavaThread::cast(thread)->lock_stack().contains(obj) : false;
}



intptr_t ObjectSynchronizer::FastHashCode(Thread* current, oop obj) {

  while (true) {
    ObjectMonitor* monitor = nullptr;
    markWord temp, test;
    intptr_t hash;
    markWord mark = read_stable_mark(obj);
    if (VerifyHeavyMonitors) {
      assert(LockingMode == LM_MONITOR, "+VerifyHeavyMonitors requires LockingMode == 0 (LM_MONITOR)");
      guarantee((obj->mark().value() & markWord::lock_mask_in_place) != markWord::locked_value, "must not be lightweight/stack-locked");
    }
    if (mark.is_neutral()) {               // if this is a normal header
      hash = mark.hash();
      if (hash != 0) {                     // if it has a hash, just return it
        return hash;
      }
      hash = get_next_hash(current, obj);  // get a new hash
      temp = mark.copy_set_hash(hash);     // merge the hash into header
                                           // try to install the hash
      test = obj->cas_set_mark(temp, mark);
      if (test == mark) {                  // if the hash was installed, return it
        return hash;
      }
      // Failed to install the hash. It could be that another thread
      // installed the hash just before our attempt or inflation has
      // occurred or... so we fall thru to inflate the monitor for
      // stability and then install the hash.
    } else if (mark.has_monitor()) {
      monitor = mark.monitor();
      temp = monitor->header();
      assert(temp.is_neutral(), "invariant: header=" INTPTR_FORMAT, temp.value());
      hash = temp.hash();
      if (hash != 0) {
        // It has a hash.

        // Separate load of dmw/header above from the loads in
        // is_being_async_deflated().

        // dmw/header and _contentions may get written by different threads.
        // Make sure to observe them in the same order when having several observers.
        OrderAccess::loadload_for_IRIW();

        if (monitor->is_being_async_deflated()) {
          // But we can't safely use the hash if we detect that async
          // deflation has occurred. So we attempt to restore the
          // header/dmw to the object's header so that we only retry
          // once if the deflater thread happens to be slow.
          monitor->install_displaced_markword_in_object(obj);
          continue;
        }
        return hash;
      }
      // Fall thru so we only have one place that installs the hash in
      // the ObjectMonitor.
    } else if (LockingMode == LM_LIGHTWEIGHT && mark.is_fast_locked() && is_lock_owned(current, obj)) {
      // This is a fast-lock owned by the calling thread so use the
      // markWord from the object.
      hash = mark.hash();
      if (hash != 0) {                  // if it has a hash, just return it
        return hash;
      }
    } else if (LockingMode == LM_LEGACY && mark.has_locker() && current->is_lock_owned((address)mark.locker())) {
      // This is a stack-lock owned by the calling thread so fetch the
      // displaced markWord from the BasicLock on the stack.
      temp = mark.displaced_mark_helper();
      assert(temp.is_neutral(), "invariant: header=" INTPTR_FORMAT, temp.value());
      hash = temp.hash();
      if (hash != 0) {                  // if it has a hash, just return it
        return hash;
      }
      // WARNING:
      // The displaced header in the BasicLock on a thread's stack
      // is strictly immutable. It CANNOT be changed in ANY cases.
      // So we have to inflate the stack-lock into an ObjectMonitor
      // even if the current thread owns the lock. The BasicLock on
      // a thread's stack can be asynchronously read by other threads
      // during an inflate() call so any change to that stack memory
      // may not propagate to other threads correctly.
    }

    // Inflate the monitor to set the hash.

    // An async deflation can race after the inflate() call and before we
    // can update the ObjectMonitor's header with the hash value below.
    monitor = inflate(current, obj, inflate_cause_hash_code);
    // Load ObjectMonitor's header/dmw field and see if it has a hash.
    mark = monitor->header();
    assert(mark.is_neutral(), "invariant: header=" INTPTR_FORMAT, mark.value());
    hash = mark.hash();
    if (hash == 0) {                       // if it does not have a hash
      hash = get_next_hash(current, obj);  // get a new hash
      temp = mark.copy_set_hash(hash)   ;  // merge the hash into header
      assert(temp.is_neutral(), "invariant: header=" INTPTR_FORMAT, temp.value());
      uintptr_t v = Atomic::cmpxchg((volatile uintptr_t*)monitor->header_addr(), mark.value(), temp.value());
      test = markWord(v);
      if (test != mark) {
        // The attempt to update the ObjectMonitor's header/dmw field
        // did not work. This can happen if another thread managed to
        // merge in the hash just before our cmpxchg().
        // If we add any new usages of the header/dmw field, this code
        // will need to be updated.
        hash = test.hash();
        assert(test.is_neutral(), "invariant: header=" INTPTR_FORMAT, test.value());
        assert(hash != 0, "should only have lost the race to a thread that set a non-zero hash");
      }
      if (monitor->is_being_async_deflated()) {
        // If we detect that async deflation has occurred, then we
        // attempt to restore the header/dmw to the object's header
        // so that we only retry once if the deflater thread happens
        // to be slow.
        monitor->install_displaced_markword_in_object(obj);
        continue;
      }
    }
    // We finally get the hash.
    return hash;
  }
}
```



mark header가 락이 걸리지 않았을 때, (mark.is_neutral()) mark.hash()를 통해서 해시 값을 얻는다.
만약 해시 값이 없다면 get_next_hash(current, obj)를 통해서 해시 값을 새로 얻는다.
그리고 get_next_hash로 얻은 해시값을 다시 copy_set_hash를 통해 mark header에 해시값을 저장한다.

get_next_hash는 Java 8 이후로 기본적으로 `Marsaglia's xor-shift scheme with thread-specific state`를 사용한다.
로컬 스레드와 XOR 알고리즘을 사용해서 해시 값을 얻고 있다.





### [markWord.hpp](https://github.com/openjdk/jdk21/blob/890adb6410dab4606a4f26a942aed02fb2f55387/src/hotspot/share/oops/markWord.hpp#L242)

```cpp


#include "runtime/globals.hpp"  -> #include "utilities/globalDefinitions.hpp"

class markWord {
 private:
  uintptr_t _value;
 public:
  
  // ...

  static const int age_bits                       = 4;
  static const int lock_bits                      = 2;
  static const int first_unused_gap_bits          = 1;
  static const int max_hash_bits                  = BitsPerWord - age_bits - lock_bits - first_unused_gap_bits;
  static const int hash_bits                      = max_hash_bits > 31 ? 31 : max_hash_bits;
  static const int second_unused_gap_bits         = LP64_ONLY(1) NOT_LP64(0);

  static const int age_shift                      = lock_bits + first_unused_gap_bits;
  static const int hash_shift                     = age_shift + age_bits + second_unused_gap_bits;
  
  static const uintptr_t hash_mask                = right_n_bits(hash_bits);
  
  // ...


  uintptr_t value() const { return _value; }

  intptr_t hash() const {
     return mask_bits(value() >> hash_shift, hash_mask);
	}
  
  markWord copy_set_hash(intptr_t hash) const {
    uintptr_t tmp = value() & (~hash_mask_in_place);
    tmp |= ((hash & hash_mask) << hash_shift);
    return markWord(tmp);
  }
  
}
```



### globalDefinitions.hpp

```cpp
inline intptr_t mask_bits      (intptr_t  x, intptr_t m) { return x & m; }
```

