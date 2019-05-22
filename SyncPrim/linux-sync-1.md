리눅스 커널의 동기화 기본기능. Part 1.
================================================================================

소개
--------------------------------------------------------------------------------

이 파트는 [linux-insides](https://0xax.gitbooks.io/linux-insides/content/) 책의
새로운 챕터를 시작합니다. 앞의
[chapter](https://0xax.gitbooks.io/linux-insides/content/Timers/index.html)
에서는 타이머와 시간 관리에 대한 내용을 다뤘습니다. 이제 다음으로 넘어갑시다.
이 파트의 제목에서 이미 이해하셨겠지만, 이 챕터는 리눅스 커널의
[동기화](https://en.wikipedia.org/wiki/Synchronization_%28computer_science%29)
기본기능들을 설명합니다.

항상 그랬듯, 동기화에 관련된 뭔가를 고려하기 전에, `동기화 기본기능` 이
일반적으로 무엇인가에 대해 알아보겠습니다. 사실, 동기화 기본기능은 두개 이상의
[병렬](https://en.wikipedia.org/wiki/Parallel_computing) 프로세스나 쓰레드가
특정 코드 영역을 동시에 수행하지 못하게 하는 소프트웨어 메커니즘입니다. 예를
들어,
[kernel/time/clocksource.c](https://github.com/torvalds/linux/master/kernel/time/clocksource.c)
파일의 다음 코드를 봅시다:

```C
mutex_lock(&clocksource_mutex);
...
...
...
clocksource_enqueue(cs);
clocksource_enqueue_watchdog(cs);
clocksource_select();
...
...
...
mutex_unlock(&clocksource_mutex);
```

이 코드는 특정
[clocksource](https://0xax.gitbooks.io/linux-insides/content/Timers/linux-timers-2.html)
를 clock source 리스트에 추가하는 `__clocksource_register_scale` 함수에서
가져온 겁니다. 이 함수는 등록된 clock source 를 가지고 있는 리스트에 여러
연산을 수행합니다. 예를 들어, `clocksource_enqueue` 함수는 주어진 clock source
를 등록된 clocksource를 가지고 있는 리스트 - `clocksource_list` 에 추가합니다.
이 코드는 두 함수로 싸여져 있음을 알아두시기 바랍니다: 하나의 패러미터 (여기선
`clocksource_mutex`) 를 받는 `mutex_lock` 과 `mutex_unlock` 입니다.

이 함수들은 [mutex](https://en.wikipedia.org/wiki/Mutual_exclusion) 동기화
기본기능에 기반한 locking 과 unlocking 을 나타냅니다.  `mutex_lock` 이
수행되면, 이 함수는 우리가 두개 이상의 쓰레드가 이 mutex 소유자가
`mutex_unlock` 을 수행하기 전까지는 이 코드를 동시에 수행하는 걸 막을 수 있게
해줍니다. 달리 말하면, 우리는 `clocksource_list` 의 병렬 연산을 방지합니다.
여기서 `mutex` 가 필요한 이유가 뭘까요? 두개의 병렬 프로세스가 하나의 clock
source 를 등록하려 하면 어떻게 될까요. 우리가 이미 알고 있듯,
`clocksource_enqueue` 함수는 주어진 clock source 를 `clocksource_list` 리스트에
가장 큰 rating 을 갖는 clock source (시스템에서 가장 높은 frequency 를 갖는
등록된 clock source) 바로 뒤에 추가시킵니다:

```C
static void clocksource_enqueue(struct clocksource *cs)
{
	struct list_head *entry = &clocksource_list;
	struct clocksource *tmp;

	list_for_each_entry(tmp, &clocksource_list, list) {
		if (tmp->rating < cs->rating)
			break;
		entry = &tmp->list;
	}
	list_add(&cs->list, entry);
}
```

만약 두개의 병렬 프로세스가 이걸 동시에 수행하면, 두 프로세스 모두 같은 `entry`
를 보게 되어 [race condition](https://en.wikipedia.org/wiki/Race_condition) 을
일으킬 수 있는데 이를 달리 말하면, 두번째 프로세스가 `list_add` 를 수행함으로써
첫번째 쓰레드의 clock source 를 덮어쓰게 될겁니다.

이 간단한 예제 외에, 동기화 기본기능은 리눅스 커널의 모든 곳에 있습니다. 앞의
[chapter](https://0xax.gitbooks.io/linux-insides/content/Timers/index.html)
또는 다른 챕터를 다시 보거나 일반적인 리눅스 커널 솟 크도르르 보게 되면 이런
것들을 많이 볼 수 있을 겁니다. 우린 리눅스 커널에서 `mutex` 가 어떻게 구현되어
있는지는 고려하지 않겠습니다. 사실, 리눅스 커널은 다양한 동기화 기본기능들을
제공합니다:

* `mutex`;
* `semaphore`;
* `seqlock`;
* `atomic operation`;
* 기타 등등.

우린 이 챕터를 `spinlock` 으로 시작하겠습니다.

리눅스 커널의 spinlock.
--------------------------------------------------------------------------------

`spinlock` 간단히 말해 은 두개의 상태를 가질 수 있는 변수를 갖는 낮은 단계의
동기화 메커니즘입니다:

* `획득됨 (acquired)`;
* `해제됨 (released)`.

`spinlock` 을 획득하고자 하는 각 프로세스는 `spinlock 획득됨` 을 의미하는 값을
이 변수에 써야하고 이후에는 `spinlock 해제됨` 상태를 이 변수에 써야 합니다.
만약 어떤 프로세스가 `spinlock` 으로 보호되는 코드를 수행하려 하면, 이
프로세스는 이 락을 잡고 있는 프로세스가 그 락을 놓기 전까지 멈춰 있게 됩니다.
이 경우 모든 관련된 연산은 [원자적
(atomic)](https://en.wikipedia.org/wiki/Linearizability) 이어서 [race
condition](https://en.wikipedia.org/wiki/Race_condition) 상태를 방지할 수
있어야 합니다. 이 `spinlock` 은 리눅스 커널의 `spinlock_t` 타입으로 표현됩니다.
우리가 리눅스 커널 코드를 보려 한다면, 이 타입이
[광범위하게](http://lxr.free-electrons.com/ident?i=spinlock_t) 사용되는 걸 볼
수 있을 겁니다. 이 `spinlock_t` 는 다음과 같이 정의되어 있으며:

```C
typedef struct spinlock {
        union {
              struct raw_spinlock rlock;
 
#ifdef CONFIG_DEBUG_LOCK_ALLOC
# define LOCK_PADSIZE (offsetof(struct raw_spinlock, dep_map))
                struct {
                        u8 __padding[LOCK_PADSIZE];
                        struct lockdep_map dep_map;
                };
#endif
        };
} spinlock_t;
```

[include/linux/spinlock_types.h](https://github.com/torvalds/linux/master/include/linux/spinlock_types.h)
헤더 파일에 있습니다. 이 구현은 `CONFIG_DEBUG_LOCK_ALLOC` 커널 설정 옵션의
상태에 종속적임을 알 수 있을 겁니다. 이건 지금은 건너뛸텐데, 모든 디버깅 관련된
것들은 이 파트의 끝에서 다룰 것이기 때문입니다. 따라서,
`CONFIG_DEBUG_LOCK_ALLOC` 커널 설정 옵션은 비활성화 되어 있다면, 이
`spinlock_t` 는 `raw_spinlock` 이라는 하나의 필드를 갖는
[union](https://en.wikipedia.org/wiki/Union_type#C.2FC.2B.2B) 만을 갖습니다:

```C
typedef struct spinlock {
        union {
              struct raw_spinlock rlock;
        };
} spinlock_t;
```

이 `raw_spinlock` 구조체는
[같은](https://github.com/torvalds/linux/master/include/linux/spinlock_types.h)
헤더 파일에 정의되어 있으며 `일반' 스핀락의 구현을 나타냅니다. `raw_spinlock`
구조체가 어떻게 정의되어 있는지 봅시다:

```C
typedef struct raw_spinlock {
        arch_spinlock_t raw_lock;
#ifdef CONFIG_DEBUG_SPINLOCK
	unsigned int magic, owner_cpu;
	void *owner;
#endif
#ifdef CONFIG_DEBUG_LOCK_ALLOC
	struct lockdep_map dep_map;
#endif
} raw_spinlock_t;
```

`arch_spinlock_t` 는 아키텍쳐에 특수한 `spinlock` 구현을 나타냅니다. 앞서
언급되었듯, 디버깅 커널 설정 옵션은 건너뛰겠습니다. 이 책은
[x86_64](https://en.wikipedia.org/wiki/X86-64) 아키텍쳐에 집중되어 있으므로,
우리가 보고자 하는 `arch_spinlock_t` 는
[include/asm-generic/qspinlock_types.h](https://github.com/torvalds/linux/master/include/asm-generic/qspinlock_types.h)
헤더 파일에 있으며 다음과 같습니다:

```C
typedef struct qspinlock {
        union {
		atomic_t val;
		struct {
			u8	locked;
			u8	pending;
		};
		struct {
			u16	locked_pending;
			u16	tail;
		};
        };
} arch_spinlock_t;
```

지금은 이 구조체를 더 들여다 보지 않겠습니다. 스핀락을 사용하는 연산을
알아봅시다. 리눅스 커널은 `spinlock` 에 대해 다음과 같은 연산들을 제공합니다:

* `spin_lock_init` - 특정 `spinlock` 의 초기화를 수행합니다;
* `spin_lock` - 특정 `spinlock` 을 획득합니다;
* `spin_lock_bh` - 소프트웨어
  [인터럽트](https://en.wikipedia.org/wiki/Interrupt) 를 불능화 시키고 특정
  `spinlock` 을 획득합니다.
* `spin_lock_irqsave` 와 `spin_lock_irq` - 이 프로세서에서의 인터럽트를 불능화
  시키고 앞의 인터럽트 상태를 `flags` 에 보존하거나/하지 않습니다;
* `spin_unlock` - 특정 `spinlock` 을 해제합니다;
* `spin_unlock_bh` - 특정 `spinlock` 을 해제하고 소프트웨어 인터럽트를 활성화 시킵니다;
* `spin_is_locked` - 특정 `spinlock` 의 상태를 리턴합니다;
* 그리고 그 외에 기타등등.

`spin_lock_init` 매크로의 구현을 들여다 봅시다. 앞서 썼듯, 이것 등의 매크로는
[include/linux/spinlock.h](https://github.com/torvalds/linux/master/include/linux/spinlock.h)
헤더 파일에 있으며 `spin_lock_init` 매크로는 아래와 같습니다:

```C
#define spin_lock_init(_lock)			\
do {						\
	spinlock_check(_lock);		        \
	raw_spin_lock_init(&(_lock)->rlock);	\
} while (0)
```

보듯이, `spin_lock_init` 매크로는 `spinlock` 을 받아서 두개의 연산을
수행합니다: 해당 `spinlock` 을 체크하고 `raw_spin_lock_init` 을 수행합니다.
`spinlock_check` 의 구현은 상당히 간단한데, 이 함수는 단지 주어진 `spinlock` 의
`raw_spinlock_t` 를 리턴해서 우리가 정확히 `평범한` raw spinlock 을 가졌음을
확신할 수 있게 합니다:

```C
static __always_inline raw_spinlock_t *spinlock_check(spinlock_t *lock)
{
	return &lock->rlock;
}
```

`raw_spin_lock_init` 매크로입니다:

```C
# define raw_spin_lock_init(lock)		\
do {						\
    *(lock) = __RAW_SPIN_LOCK_UNLOCKED(lock);	\
} while (0)					\
```

이 매크로는 `__RAW_SPIN_LOCK_UNLOCKED` 값을 주어진 `spinlock` 의
`raw_spinlock_t` 에 저장합니다. `__RAW_SPIN_LOCK_UNLOCKED` 매크로의 이름에서
유추할 수 있듯이, 이 매크로는 주어진 `spinlock` 을 초기화 하고 이를 `해제된`
상태로 설정합니다. 이 매크로는
[include/linu/spinlock_types.h](https://github.com/torvalds/linux/master/include/linux/spinlock_types.h)
헤더 파일에 있으며 다음 매크로로 확장됩니다:

```C
#define __RAW_SPIN_LOCK_UNLOCKED(lockname)      \
         (raw_spinlock_t) __RAW_SPIN_LOCK_INITIALIZER(lockname)

#define __RAW_SPIN_LOCK_INITIALIZER(lockname)			\
         {                                                      \
             .raw_lock = __ARCH_SPIN_LOCK_UNLOCKED,             \
             SPIN_DEBUG_INIT(lockname)                          \
             SPIN_DEP_MAP_INIT(lockname)                        \
         }
```

앞에서 설명했듯, 우린 동기화 기본 기능의 디버깅에 관련된 것들은 고려하지
않겠습니다. 이 경우 우린 `SPIN_DEBUG_INIT` 과 `SPIN_DEP_MAP_INIT` 매크로를
무시합니다. 따라서 `__RAW_SPINLOCK_UNLOCKED` 매크로는 아래와 같이 확장됩니다:

```C
*(&(_lock)->rlock) = __ARCH_SPIN_LOCK_UNLOCKED;
```

여기서 `__ARCH_SPIN_LOCK_UNLOCKED` 는
[x86_64](https://en.wikipedia.org/wiki/X86-64) 에서 아래와 같습니다:

```C
#define __ARCH_SPIN_LOCK_UNLOCKED       { { .val = ATOMIC_INIT(0) } }
```

따라서, `spin_lock_init` 매크로의 확장 후에는, 주어진 `spinlock` 이 초기화 되고
그 상태는 `해제됨` 이 됩니다.

이제 우리는 `spinlock` 을 어떻게 초기화 하는지 알았으니, 리눅스 커널이
`spinlock` 을 조정하기 위해 제공하는
[API](https://en.wikipedia.org/wiki/Application_programming_interface) 를
알아봅시다. 첫번째는 스핀락을 `획득` 할 수 있게 하는 함수입니다:

```C
static __always_inline void spin_lock(spinlock_t *lock)
{
	raw_spin_lock(&lock->rlock);
}
```

이 `raw_spin_lock` 매크로는 같은 헤더파일 내에 정의되어 있으며 `_raw_spin_lock`
으로 확장됩니다:

```C
#define raw_spin_lock(lock)	_raw_spin_lock(lock)
```

`_raw_spin_lock` 은 `CONFIG_SMP` 옵션이 설정되어 있는지 그리고
`CONFIG_INLINE_SPIN_LOCK` 옵션이 설정되어 있는지에 종속적으로 정의되어
있습니다. 만약 [SMP](https://en.wikipedia.org/wiki/Symmetric_multiprocessing)
이 비활성화 되어 있다면, `_raw_spin_lock` 은
[include/linux/spinlock_api_up.h](https://github.com/torvalds/linux/blob/master/include/linux/spinlock_api_up.h)
헤더 파일에 정의되어 있습니다:

```C
#define _raw_spin_lock(lock)	__LOCK(lock)
```

SMP 가 활성화 되어 있고 `CONFIG_INLINE_SPIN_LOCK` 이 설정되어 있다면,
[include/linux/spinlock_api_smp.h](https://github.com/torvalds/linux/blob/master/include/linux/spinlock_api_smp.h)
헤더 파일에 다음과 같이 정의되어 있습니다:

```C
#define _raw_spin_lock(lock) __raw_spin_lock(lock)
```

만약 SMP 가 활성화 되어 있고 `CONFIG_INLINE_SPIN_LOCK` 이 설정되어 있지 않다면,
[kernel/locking/spinlock.c](https://github.com/torvalds/linux/blob/master/kernel/locking/spinlock.c)
에 다음과 같이 정의되어 있습니다:

```C
void __lockfunc _raw_spin_lock(raw_spinlock_t *lock)
{
	__raw_spin_lock(lock);
}
```

여기선 뒤쪽의 `_raw_spin_lock` 형태를 고려하겠습니다. `__raw_spin_lock` 함수는
아래와 같습니다:

```C
static inline void __raw_spin_lock(raw_spinlock_t *lock)
{
        preempt_disable();
        spin_acquire(&lock->dep_map, 0, 0, _RET_IP_);
        LOCK_CONTENDED(lock, do_raw_spin_trylock, do_raw_spin_lock);
}
```

볼 수 있듯, 여기선 먼저
[include/linux/preempt.h](https://github.com/torvalds/linux/blob/master/include/linux/preempt.h)
의 `preempt_disable` 매크로를 호출해서 (더 자세한 건 리눅스 커널 초기화
프로세스 챕터의 아홉번째
[part](https://0xax.gitbooks.io/linux-insides/content/Initialization/linux-initialization-9.html)
를 참고하세요)
[preemption](https://en.wikipedia.org/wiki/Preemption_%28computing%29) 을
불능화 시킵니다.  이 `spinlock` 을 해제할 때 preemption 은 다시 활성화
될겁니다:

```C
static inline void __raw_spin_unlock(raw_spinlock_t *lock)
{
        ...
        ...
        ...
        preempt_enable();
}
```

락을 잡기 위해 spin 하고 있는 사이에 다른 프로세스가 이 프로세스를 preempt
하는걸 막기 위해 이걸 해야 합니다. `spin_acquire` 매크로는 다른 연결을 통해
다음과 같이 확장됩니다:

```C
#define spin_acquire(l, s, t, i)                lock_acquire_exclusive(l, s, t, NULL, i)
#define lock_acquire_exclusive(l, s, t, n, i)           lock_acquire(l, s, t, 0, 1, n, i)
```

이 `lock_acquire` 함수는:

```C
void lock_acquire(struct lockdep_map *lock, unsigned int subclass,
                  int trylock, int read, int check,
                  struct lockdep_map *nest_lock, unsigned long ip)
{
         unsigned long flags;

         if (unlikely(current->lockdep_recursion))
                return;
 
         raw_local_irq_save(flags);
         check_flags(flags);
 
         current->lockdep_recursion = 1;
         trace_lock_acquire(lock, subclass, trylock, read, check, nest_lock, ip);
         __lock_acquire(lock, subclass, trylock, read, check,
                        irqs_disabled_flags(flags), nest_lock, ip, 0, 0);
         current->lockdep_recursion = 0;
         raw_local_irq_restore(flags);
}
```

앞에서 이야기했듯 디버깅이나 트레이싱에 관련된 것들은 다루지 않겠습니다.
`lock_acquire` 함수의 중요 포인트는 `raw_local_irq_save` 매크로를 호출함으로써
하드웨어 인터럽트를 불능화 시키는 것으로, 주어진 spinlock 은 활성화 된 하드웨어
인터럽트에서 획득될 수도 있기 때문입니다. 이런 방법으로 이 프로세스는 preempt
되지 않게 됩니다. `lock_acquire` 함수의 마지막에서 `raw_local_irq_restore`
매크로를 통해 하드웨어 인터럽트를 다시 활성화 시킴을 알아두시기 바랍니다.
짐작했겠지만, `__lock_acquire` 함수의 주요 부분은
[kernel/locking/lockdep.c](https://github.com/torvalds/linux/blob/master/kernel/locking/lockdep.c)
소스 코드 파일에 있습니다.

이 `__lock_acquire` 함수는 좀 커 보입니다. 우린 이 함수가 무슨 일을 하는지
이해하려 노력해 보겠지만, 여기서는 아닙니다. 사실 이 함수는 리눅스 커널
[lock validator](https://www.kernel.org/doc/Documentation/locking/lockdep-design.txt)
와 연관되어 있으며 그건 이 파트의 주제가 아닙니다. 다시 `__raw_spin_lock`
함수로 돌아가서, 결국 아래 정의를 보게 됩니다:

```C
LOCK_CONTENDED(lock, do_raw_spin_trylock, do_raw_spin_lock);
```

이 `LOCK_CONTENDED` 매크로는
[include/linux/lockdep.h](https://github.com/torvalds/linux/blob/master/include/linux/lockdep.h)
헤더 파일에 정의되어 있는데 주어진 `spinlock` 을 가지고 특정 함수를 호출할
뿐입니다:

```C
#define LOCK_CONTENDED(_lock, try, lock) \
         lock(_lock)
```

우리의 경우, 이 `lock` 은
[include/linux/spinlock.h](https://github.com/torvalds/linux/blob/master/include/linux/spnlock.h)
의 `do_raw_spin_lock` 함수이고 `_lock` 은 주어진 `raw_spinlock_t` 입니다:

```C
static inline void do_raw_spin_lock(raw_spinlock_t *lock) __acquires(lock)
{
        __acquire(lock);
         arch_spin_lock(&lock->raw_lock);
}
```

여기서의 `__acquire` 는 그저 [Sparse](https://en.wikipedia.org/wiki/Sparse) 에
연관된 매크로이고 우린 지금은 여기엔 관심 없습니다. `arch_spin_lock` 매크로는
[include/asm-generic/qspinlock.h](https://github.com/torvalds/linux/blob/master/include/asm-generic/qspinlocks.h)
헤더 파일에 다음과 같이 정의되어 있습니다:

```C
#define arch_spin_lock(l)               queued_spin_lock(l)
```

이 파트는 여기서 멈춥니다. 다음 파트에서, 우린 queued spinlock 이 어떻게
동작하는지 알아보고 관련된 컨셉들을 알아봅니다.

결론
--------------------------------------------------------------------------------

이 섹션은 리눅스 커널의 동기화 기본 기능에 대해 다루는 첫번째 파트를 마칩니다.
이 파트에서, 우린 리눅스 커널에 의해 제공되는 첫번째 동기화 기본 기능인
`spinlock` 을 알아봤습니다. 다음 파트에서 우린 이 흥미로운 주제에 더 깊이
들어가보고 다른 `동기화` 관련된 것들을 알아보겠습니다.

질문이나 제안이 있다면, 제게 트위터 [0xAX](https://twitter.com/0xAX) 로 연락
주시거나 [email](anotherworldofworld@gmail.com) 을 보내주시거나
[issue](https://github.com/0xAX/linux-insides/issues/new) 를 만들어 주시기
바랍니다.


링크
--------------------------------------------------------------------------------

* [Concurrent computing](https://en.wikipedia.org/wiki/Concurrent_computing)
* [Synchronization](https://en.wikipedia.org/wiki/Synchronization_%28computer_science%29)
* [Clocksource framework](https://0xax.gitbooks.io/linux-insides/content/Timers/linux-timers-2.html)
* [Mutex](https://en.wikipedia.org/wiki/Mutual_exclusion)
* [Race condition](https://en.wikipedia.org/wiki/Race_condition)
* [Atomic operations](https://en.wikipedia.org/wiki/Linearizability)
* [SMP](https://en.wikipedia.org/wiki/Symmetric_multiprocessing)
* [x86_64](https://en.wikipedia.org/wiki/X86-64) 
* [Interrupts](https://en.wikipedia.org/wiki/Interrupt)
* [Preemption](https://en.wikipedia.org/wiki/Preemption_%28computing%29) 
* [Linux kernel lock validator](https://www.kernel.org/doc/Documentation/locking/lockdep-design.txt)
* [Sparse](https://en.wikipedia.org/wiki/Sparse)
* [xadd instruction](http://x86.renejeschke.de/html/file_module_x86_id_327.html)
* [NOP](https://en.wikipedia.org/wiki/NOP)
* [Memory barriers](https://www.kernel.org/doc/Documentation/memory-barriers.txt)
* [Previous chapter](https://0xax.gitbooks.io/linux-insides/content/Timers/index.html)
