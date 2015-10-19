# ARINC653과 POSIX에서의 mutex/semaphore/condvar 비교

(ARINC653 part1 기준)

## 공통

* 당연하게도 ARINC653에서는 partition initialization 과정에서만 sem/condvar object를 생성 가능.
* ARINC653엔 mutex가 따로 없음.
* ARINC653에서는 unnamed따윈 존재하지 않음. 모든 object는 named임.
* ARINC653은 sem/condvar object의 갯수와 이름을 static하게 선언해줘야 함.
* semaphore/condvar를 구분할때 ARINC653은 numerical ID로만 구분함. posix는 object가 struct로 구성.

### Mutex

* ARINC653엔 존재하지 않음. maximum value가 1인 semaphore를 써야함.

### Semaphore

* ARINC653의 모든 semaphore는 named semaphore이고 name은 미리 정의되어야 함.
* P/V operation의 이름은 wait/signal vs wait/post.
* ARINC653은 초기화시 waiting threads의 queueing policy를 정할수 있음. spec상 FIFO/priority-based 두가지가 존재.


### Conditional variable(Event)

얘네는 비슷한듯 보이지만 실상은 거의 다른 형태의 locking임. 오히려 windows의 event와 비교해야 되지 않나 싶긴 하지만
윈도의 event api를 내가 구현할껀 아니니깐..

* posix에서는 condvar에 쓰이는 mutex를 프로그래머가 정의하고 함수에 넘겨줘야 함.
* 위에서 쓰이는 mutex는 pthread_cond_wait을 호출하기 전에 프로그래머가 lock을 걸어야 함.
* ARINC653에서는 waiting threads중에 하나만 깨울수 있는 방법이 없음. 무조건 broadcast이고 하나만 signal은 불가능.
* posix condvar는 auto-reset. ARINC653 event는 직접 api로 reset을 해줘야함.

더불어 [여기](http://stackoverflow.com/questions/178114/pthread-like-windows-manual-reset-event)를 보면 pthread condvar를
이용해서 event를 구현한 여러 방법이 있긴 한데, 개인적인 생각으로는 semaphore 구현을 건드릴수 있는 나같은 상황이라면
오히려 value가 inf or 0을 가지는 semaphore를 이용해서 event를 구현하는게 쉽지 않을까 싶긴 한데 고민해볼 문제다.
