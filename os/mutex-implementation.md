# newlib/glibc의 mutex 구현 비교

## 공통

lock의 status에 tid/thread 구조를 같이 저장한다. 여기에 추가적인 bit를 통해 lock의 상태를 나타냄.
따라서 compare-and-swap을 통해 tid나 다른 bit까지 한번에 atomic하게 처리가 가능하다.

## newlib

thread의 suspend/resume만을 사용해서 구현이 되어있다.

```c
  /* No luck, try once more or suspend. */

  do {
    oldstatus = lock->__status;
    successful_seizure = 0;

    if ((oldstatus & 1) == 0) {
      newstatus = oldstatus | 1;
      successful_seizure = 1;
    } else {
      if (self == NULL)
        self = thread_self();
      newstatus = (long) self | 1;
    }

    if (self != NULL) {
      THREAD_SETMEM(self, p_nextlock, (pthread_descr) (oldstatus & ~1L));
      /* Make sure the store in p_nextlock completes before performing
         the compare-and-swap */
      MEMORY_BARRIER();
    }
  } while(! __compare_and_swap(&lock->__status, oldstatus, newstatus));

  /* Suspend with guard against spurious wakeup.
     This can happen in pthread_cond_timedwait_relative, when the thread
     wakes up due to timeout and is still on the condvar queue, and then
     locks the queue to remove itself. At that point it may still be on the
     queue, and may be resumed by a condition signal. */

  if (!successful_seizure) {
    for (;;) {
      suspend(self);
      if (self->p_nextlock != NULL) {
        /* Count resumes that don't belong to us. */
        spurious_wakeup_count++;
        continue;
      }
      break;
    }
    goto again;
  }

  /* Put back any resumes we caught that don't belong to us. */
  while (spurious_wakeup_count--)
    restart(self);

  READ_MEMORY_BARRIER();
```

suspend/resume을 어떻게 구현했느냐에 따라 달라지겠지만, 일단 상대적으로 간단한 system call만으로 구현이 가능하다.
다만 A thread에서 `suspend(self);`를 실행하기 직전에 context switching이 일어나서 B thread에서 lock을 풀어버리면 A thread는 영원히 잠들 가능성이 있다.
이러한 상황을 방지하는것은 몇가지 방법이 있을것이다.

* spurious wakeup을 보장한다. (suspend된 thread를 강제로 주기적으로 깨운다) => ARINC653에서 suspend의 일반적인 expected behaviour가 아님.
* resume이 먼저 호출된경우 suspend를 그 횟수만큼 무시한다. => 역시 suspend의 의미가 조금 이상해짐.
* mutex 전용 suspend/resume을 만들어서 `thread->p_nextlock`이 NULL일 경우 suspend시키지 않는다. => 코드가 더러워짐

따라서 newlib의 구현방식을 따르는것은 별다른 이득이 없다고 생각됨.


## glibc

linux의 futex system call을 활용하게 되어 있다.
futex system call은 기본적으로 특정 메모리의 값이 바뀔때까지 sleep하는 것으로, 메모리의 값이 바뀐다고 자동으로 깨어나는것은 아니고 반드시 FUTEX\_WAKE로 futex system call을 불러줘야 한다.
futex는 또한 msb 2개를 status bit으로 쓰는데, 이러한 status bit을 보고 추가적으로 futex system call을 불러주는것은 user(library)의 몫이다. (0x80000000이 set되어 있지 않으면 waiter가 없는것이므로 FUTEX\_WAKE를 부를 필요가 없음.)
커널쪽의 구현은 특성상 hashmap을 써야 하는데, RTOS에서 space-time tradeoff를 어떻게 해결할지는 고민해볼만한 문제인거 같다.
