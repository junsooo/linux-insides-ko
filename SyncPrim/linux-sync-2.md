리눅스 커널의 동기화 기본기능. Part 2.
================================================================================

Queued Spinlocks
--------------------------------------------------------------------------------

이 부분은 리눅스 커널의 동기화 기본기능을 설명하는 [챕터](https://0xax.gitbooks.io/linux-insides/content/SyncPrim/index.html) 의 두번째 부분입니다, 이 챕터의 첫번째 [부분](https://0xax.gitbooks.io/linux-insides/content/SyncPrim/linux-sync-1.html) 에서는 그 첫번째 기본 기능 ([spinlock](https://en.wikipedia.org/wiki/Spinlock) 을 소개했죠. 이 부분에서는 이 동기화 기본기능에 대해 계속 배워보겠습니다. 앞 부분을 읽어보셨다면, 일반적인 spinlock과 달리, 리눅스 커널은 특별한 종류의 `spinlock` - `queued spinlock` 을 제공한다고 한 것을 기억할 겁니다. 이 부분에서는 이 컨셉이 무엇을 나타내는지 이해해 보겠습니다.

앞 [부분](https://0xax.gitbooks.io/linux-insides/content/SyncPrim/linux-sync-1.html) 에서는 `spinlock` 의 [API](https://en.wikipedia.org/wiki/Application_programming_interface) 를 살펴봤습니다:

* `spin_lock_init` - 특정 `spinlock` 의 초기화를 제공;
* `spin_lock` - 특정 `spinlock` 을 획득;
* `spin_lock_bh` - 소프트웨어 [인터럽트](https://en.wikipedia.org/wiki/Interrupt) 를 비활성화 시키고 특정 `spinlock` 을 획득.
* `spin_lock_irqsave` 와 `spin_lock_irq` - 현재 프로세서에서의 인터럽트를 비활성화 시키고 `flags` 에 기존의 인터럽트 상태를 저장/비저장;
* `spin_unlock` - 특정 `spinlock` 을 해제;
* `spin_unlock_bh` - 특정 `spinlock` 을 해제하고 소프트웨어 인터럽트를 활성화;
* `spin_is_locked` - 특정 `spinlock` 의 상태를 리턴;
* 기타 등등.

또한 우리는 [include/linux/spinlock.h](https://github.com/torvalds/linux/blob/master/include/linux/spinlock.h) 헤더파일에 정의된 이 매크로들이 [include/asm-generic/qspinlock.h](https://github.com/torvalds/linux/blob/master/include/asm-generic/qspinlock.h) 에 있는 `arch_*` 프리픽스를 갖는 함수들의 호출로 확장된다는 걸 알고 있습니다:

```C
#define arch_spin_is_locked(l)          queued_spin_is_locked(l)
#define arch_spin_is_contended(l)       queued_spin_is_contended(l)
#define arch_spin_value_unlocked(l)     queued_spin_value_unlocked(l)
#define arch_spin_lock(l)               queued_spin_lock(l)
#define arch_spin_trylock(l)            queued_spin_trylock(l)
#define arch_spin_unlock(l)             queued_spin_unlock(l)
```

Queued spinlock 과 그 [API](https://en.wikipedia.org/wiki/Application_programming_interface) 가 어떻게 구현되어 있는지 알아보기 전에, 이론적 부분을 먼저 들여다 보겠습니다.

Queued spinlock 에 대한 소개
-------------------------------------------------------------------------------

Queued spinlock 은 리눅스 커널의 [락킹 메커니즘](https://en.wikipedia.org/wiki/Lock_%28computer_science%29) 으로 표준적 `spinlock` 의 대체품입니다. 이는 적어도 [x86_64](https://en.wikipedia.org/wiki/X86-64) 아키텍쳐에 대해서는 사실입니다. 다음 커널 설정 파일 - [kernel/Kconfig.locks](https://github.com/torvalds/linux/blob/master/kernel/Kconfig.lock_64) 을 보면, 아래와 같은 설정 항목을 볼 수 있습니다:

```
config ARCH_USE_QUEUED_SPINLOCKS
	bool

config QUEUED_SPINLOCKS
	def_bool y if ARCH_USE_QUEUED_SPINLOCKS
	depends on SMP
```

이는 `ARCH_USE_QUEUED_SPINLOCKS` 이 활성화 되어 있다면, `CONFIG_QUEUED_SPINLOCKS` 커널 설정 옵션이 활성화 될것임을 의미합니다. `x86_64` 를 위한 커널 설정 파일 - [arch/x86/Kconfig](https://github.com/torvalds/linux/blob/master/arch/x86/Kconfig) 에서 `ARCH_USE_QUEUED_SPINLOCKS` 이 기본으로 활성화 되어 있음을 볼 수 있습니다:

```
config X86
    ...
    ...
    ...
    select ARCH_USE_QUEUED_SPINLOCKS
    ...
    ...
    ...
```

Queued spinlock 의 개념을 보기 전에, `spinlock` 타입들을 봅시다. 먼저 어떻게 `평범한` spinlock 이 구현되는지 생각해 봅시다. 보통, `평범한` spinlock 의 구현은 [test and set](https://en.wikipedia.org/wiki/Test-and-set) 명령에 기반합니다. 이 명령이 하는 일은 상당히 간단합니다. 이 명령은 메모리의 특정 위치에 값을 쓰고 그 위치에 쓰여있던 기존 값을 리턴합니다. 이 두개의 일은 원자적으로 이루어집니다, 즉 인터럽트 받지 않는 명령입니다. 따라서 첫번째 쓰레드가 이 명령을 수행하기 시작했다면, 두번째 쓰레드는 첫번째 프로세서가 이 명령을 마무리할 때까지 기다립니다. 이 메커니즘 위에서 기본 락이 만들어집니다. 개요적으로는 아래와 같을 겁니다:

```C
int lock(lock)
{
    while (test_and_set(lock) == 1)
        ;
    return 0;
}

int unlock(lock)
{
    lock=0;

    return lock;
}
```

첫번째 쓰레드는 `lock` 을 `1` 로 설정하는 `test_and_set` 을 수행할 겁니다. 두번째 쓰레드가 `lock` 함수를 호출했을 때는, 첫번째 쓰레드가 `unlock` 함수를 호출하고 `lock` 이 `0` 이 될때까지 `while` 루프를 반복할 겁니다. 이 구현은 성능에 크게 좋지는 않은데, 두가지 문제가 있기 때문입니다. 첫번째 문제는 이 구현이 불공평할 수 있고 한 프로세서의 쓰레드가 락이 풀리길 기다리는 다른 쓰레드보다 먼저 `lock` 을 호출했다 해도 긴 대기 시간을 가질 수 있다는 것입니다. 두번째 문제는 락을 잡고자 하는 모든 쓰레드가 공유 메모리에 있는 변수에 `test_and_set` 과 같은 `atomic` 오퍼레이션을 많이 수행해야 한다는 겁니다. 이는 프로세서의 캐시가 `lock=1` 을 저장하기 하므로 캐시 무효화를 일으키게 되지만, 메모리 상의 `lock` 은 이 쓰레드가 이 락을 놓은 후에 `1` 일 수 있습니다.

이 파트의 주제는 `queued spinlock` 입니다. 이 접근법은 이 두 문제를 모두 해결할 수도 있습니다. `queued spinlock` 은 각 프로세서가 각자의 메모리 위치에서 기다릴 수 있게 합니다. Queue-based spinlock 의 기본 개념은 [MCS](http://www.cs.rochester.edu/~scott/papers/1991_TOCS_synch.pdf) 라고 불리는 고전적 quque-based spinlock 구현을 공부함으로써 가장 잘 이해될 수 있습니다. 리눅스 커널의 `queued spinlock` 구현을 보기에 앞서, `MCS` 락이 어떻게 동작하는지 이해해 봅시다.

`MCS` 락의 기본 아이디어는 앞의 문단에서 적은 바와 같습니다, 쓰레드는 지역 변수를 반복적으로 기다리며 시스템의 각 프로세서가 이 변수의 복사본을 각자 갖습니다. 달리 말하면 이 컨셉은 리눅스 커널의 [per-cpu](https://0xax.gitbooks.io/linux-insides/content/Concepts/linux-cpu-1.html) 변수 컨셉에 기반합니다.

첫번째 쓰레드가 락을 획득하고자 하면, 이 쓰레드는 스스로를 `queue` 에 등록합니다. 달리 말하면 특수한 `queue` 에 등록되고 나서 락을 획득합니다, 이 락은 지금은 열려 있으니까요. 첫번째 쓰레드가 이 락을 놓기 전에 두번째 쓰레드가 같은 락을 얻고자 한다면, 이 쓰레드는 이 락 변수의 복사본을 이 `queue` 에 넣습니다. 이 경우 첫번째 쓰레드는 두번째 쓰레드를 가리키는 `next` 필드를 가지고 있을 겁니다. 이 순간, 두번째 쓰레드는 첫번째 쓰레드가 자신의 락을 놓고 `next` 쓰레드에게 이를 알릴 때까지 기다립니다. 첫번째 쓰레드는 `queue` 에서 지워지고 두번째 쓰레드가 락을 얻게 됩니다.

개요적으로는 이렇게 나타낼 수 있습니다:

빈 queue:

```
+---------+
|         |
|  Queue  |
|         |
+---------+
```

첫번째 쓰레드가 락을 잡으려 함:

```
+---------+     +---------------------------+
|         |     |                           |
|  Queue  |---->| 첫번째 쓰레드가 락을 잡음 |
|         |     |                           |
+---------+     +---------------------------+
```

두번째 쓰레드가 락을 잡으려 함:

```
+---------+     +------------------------------------------+     +---------------------------+
|         |     |                                          |     |                           |
|  Queue  |---->|  두번째 쓰레드가 첫번째 쓰레드를 기다림  |<----| 첫번째 쓰레드가 락을 잡음 |
|         |     |                                          |     |                           |
+---------+     +------------------------------------------+     +---------------------------+
```

또는 수도코드로는:

```C
void lock(...)
{
    lock.next = NULL;
    ancestor = put_lock_to_queue_and_return_ancestor(queue, lock);

    // 앞선 존재가 있었다면, 락은 이미 잡혀있고 우린 그게 놓아질 때까지 기다림
    if (ancestor)
    {
        lock.is_locked = 1;
        ancestor.next = lock;

        while (lock.is_locked == true)
            ;
    }

    // 아니라면 우리가 락을 잡고 여기서 나감
}

void unlock(...)
{
    // 우리가 queue 에 혼자 있나, 아니면 누군가에게 알림을 줘야하나?
    if (lock.next != NULL) {
        // lock() 함수에서의 루프가 종료되게 함
        lock.next.is_locked = false;
    }

    // 이제, 락을 놓았음을 알려줘야 할 다음 쓰레드가 queue 에 없음. 락에 `0` 을
    // 넣고 queue 에서 우릴 제거한 후 나감.
}
```

이게 `queued spinlock` 에 대한 이론의 전부입니다, 이제 이 메커니즘이 리눅스 커널에 어떻게 구현되어 있는지 알아봅시다. 앞의 수도코드와 달리, `queued spinlock` 의 구현은 복잡하고 이리저리 얽혀있습니다. 하지만 주의 깊게 공부해 보면 될 겁니다.

API of queued spinlocks
-------------------------------------------------------------------------------

Now we know a little about `queued spinlocks` from the theoretical side, time to see the implementation of this mechanism in the Linux kernel. As we saw above, the [include/asm-generic/qspinlock.h](https://github.com/torvalds/linux/blob/master/include/asm-generic/qspinlock.h) header file provides a set of macro which represents API for a spinlock acquiring, releasing and etc:

```C
#define arch_spin_is_locked(l)          queued_spin_is_locked(l)
#define arch_spin_is_contended(l)       queued_spin_is_contended(l)
#define arch_spin_value_unlocked(l)     queued_spin_value_unlocked(l)
#define arch_spin_lock(l)               queued_spin_lock(l)
#define arch_spin_trylock(l)            queued_spin_trylock(l)
#define arch_spin_unlock(l)             queued_spin_unlock(l)
```

All of these macros expand to the call of functions from the same header file. Additionally, we saw the `qspinlock` structure from the [include/asm-generic/qspinlock_types.h](https://github.com/torvalds/linux/blob/master/include/asm-generic/qspinlock_types.h) header file which represents a queued spinlock in the Linux kernel:

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

The `val` field represents the state of a given `spinlock`. This `4` bytes field consists from following parts:

* `0-7` - locked byte;
* `8` - pending bit;
* `9-15` - not used;
* `16-17` - two bit index which represents entry of the `per-cpu` array of the `MCS` lock (will see it soon);
* `18-31` - contains number of processor which indicates tail of the queue.

Before we move to consider `API` of `queued spinlocks`, notice the `val` field of the `qspinlock` structure has type - `atomic_t` which represents atomic variable or one operation at a time variable. So, all operations with this field will be [atomic](https://en.wikipedia.org/wiki/Linearizability). For example let's look at the reading value of the `val` API:

```C
static __always_inline int queued_spin_is_locked(struct qspinlock *lock)
{
	return atomic_read(&lock->val);
}
```

Ok, now we know data structures which represents queued spinlock in the Linux kernel and now is the time to look at the implementation of the main function from the `queued spinlocks` [API](https://en.wikipedia.org/wiki/Application_programming_interface):

```C
#define arch_spin_lock(l)               queued_spin_lock(l)
```

Yes, this function is - `queued_spin_lock`. As we may understand from the function's name, it allows to acquire lock by the thread. This function is defined in the [include/asm-generic/qspinlock_types.h](https://github.com/torvalds/linux/blob/master/include/asm-generic/qspinlock_types.h) header file and its implementation looks:

```C
static __always_inline void queued_spin_lock(struct qspinlock *lock)
{
        u32 val;

        val = atomic_cmpxchg_acquire(&lock->val, 0, _Q_LOCKED_VAL);
        if (likely(val == 0))
                 return;
        queued_spin_lock_slowpath(lock, val);
}
```

Looks pretty easy, except the `queued_spin_lock_slowpath` function. We may see that it takes only one parameter. In our case this parameter will represent `queued spinlock` which will be locked. Let's consider the situation that `queue` with locks is empty for now and the first thread wanted to acquire lock. As we may see the `queued_spin_lock` function starts from the call of the `atomic_cmpxchg_acquire` macro. As you may guess from its name, it executes atomic [CMPXCHG](http://x86.renejeschke.de/html/file_module_x86_id_41.html) instruction. Ultimately, the `atomic_cmpxchg_acquire` macro expands to the call of the `__raw_cmpxchg` macro almost like the following:

```C
#define __raw_cmpxchg(ptr, old, new, size, lock)		\
({								\
	__typeof__(*(ptr)) __ret;				\
	__typeof__(*(ptr)) __old = (old);			\
	__typeof__(*(ptr)) __new = (new);			\
								\
	volatile u32 *__ptr = (volatile u32 *)(ptr);		\
	asm volatile(lock "cmpxchgl %2,%1"			\
		     : "=a" (__ret), "+m" (*__ptr)		\
		     : "r" (__new), "0" (__old)			\
		     : "memory");				\
								\
	__ret;							\
})
```

which compares the `old` with the value which the `ptr` points to and if they are identical, it stores the `new` in the memory location which is pointed by the `ptr` and returns the initial value in this memory location. In our case, 

Let's back to the `queued_spin_lock` function. Assuming that we are the first one who tried to acquire the lock, the `val` will be zero and we will return from the `queued_spin_lock` function:

```C
	val = atomic_cmpxchg_acquire(&lock-val, 0, _Q_LOCKED_VAL);
	if (likely(val == 0))
		return;
```

So far, we've considered uncontended case (i.e. fast-path). Now let's consider contended case (i.e. slow-path). Suppose that one thread tried to acquire a lock, but the lock is already held, then `queued_spin_lock_slowpath` will be called. The `queued_spin_lock_slowpath` function is defined in the [kernel/locking/qspinlock.c](https://github.com/torvalds/linux/blob/master/kernel/locking/qspinlock.c) source code file:

```C
void queued_spin_lock_slowpath(struct qspinlock *lock, u32 val)
{
	...
	...
	...
	if (val == _Q_PENDING_VAL) {
		int cnt = _Q_PENDING_LOOPS;
		val = atomic_cond_read_relaxed(&lock-val,
					       (VAL != _Q_PENDING_VAL) || !cnt--);
	}
	...
	...
	...
}
```

which wait for in-progress lock acquisition to be done with a bounded number of spins so that we guarantee forward progress. Above, we saw that the lock contains - pending bit. This bit represents thread which wanted to acquire lock, but it is already acquired by the other thread and `queue` is empty at the same time. In this case, the pending bit will be set and the `queue` will not be touched. This is done for optimization, because there are no need in unnecessary latency which will be caused by the cache invalidation in a touching of own `mcs_spinlock` array.

If we observe contention, then we have no choice other than queueing, so jump to `queue` label that we'll see later:

```C
	if (val & ~_Q_LOCKED_MASK)
		goto queue;
```

So, the lock is already held. That is, we set the pending bit of the lock:

```C
	val = queued_fetch_set_pending_acquire(lock);
```

Again if we observe contention, undo the pending and queue.

```C
	if (unlikely(val & ~_Q_LOCKED_MASK)) {
		if (!(val & _Q_PENDING_MASK))
			clear_pending(lock);
		goto queue;
	}
```

Now, we're pending, wait for the lock owner to release it.

```C
	if (val & _Q_LOCKED_MASK)
		atomic_cond_read_acquire(&)
```

We are allowed to take the lock. So, we clear the pending bit and set the locked bit. Now we have nothing to do with the `queued_spin_lock_slowpath` function, return from it.

```C
	clear_pending_set_locked(lock);
	return;
```

Before diving into queueing, we'll see about `MCS` lock mechanism first. As we already know, each processor in the system has own copy of the lock. The lock is represented by the following structure:

```C
struct mcs_spinlock {
       struct mcs_spinlock *next;
       int locked;
       int count;
};
```

from the [kernel/locking/mcs_spinlock.h](https://github.com/torvalds/linux/blob/master/kernel/locking/mcs_spinlock.h) header file. The first field represents a pointer to the next thread in the `queue`. The second field represents the state of the current thread in the `queue`, where `1` is `lock` already acquired and `0` in other way. And the last field of the `mcs_spinlock` structure represents nested locks. To understand what nested lock is, imagine situation when a thread acquired lock, but was interrupted by the hardware [interrupt](https://en.wikipedia.org/wiki/Interrupt) and an [interrupt handler](https://en.wikipedia.org/wiki/Interrupt_handler) tries to take a lock too. For this case, each processor has not just copy of the `mcs_spinlock` structure but array of these structures:

```C
static DEFINE_PER_CPU_ALIGNED(struct qnode, qnodes[MAX_NODES]);
```

This array allows to make four attempts of a lock acquisition for the four events in following contexts:

* normal task context;
* hardware interrupt context;
* software interrupt context;
* non-maskable interrupt context.

Notice that we did not touch `queue` yet. We no need in it, because for two threads it just leads to unnecessary latency for memory access. In other case, the first thread may release it lock before this moment. In this case the `lock->val` will contain `_Q_LOCKED_VAL | _Q_PENDING_VAL` and we will start to build `queue`. We start to build `queue` by the getting the local copy of the `qnodes` array of the processor which executes thread and calculate `tail` which will indicate the tail of the `queue` and `idx` which represents an index of the `qnodes` array:

```C
queue:
	node = this_cpu_ptr(&qnodes[0].mcs);
	idx = node->count++;
	tail = encode_tail(smp_processer_id(), idx);

	node = grab_mcs_node(node, idx);
```

After this, we set `locked` to zero because this thread didn't acquire lock yet and `next` to `NULL` because we don't know anything about other `queue` entries:

```C
	node->locked = 0;
	node->next = NULL;
```

We already touched `per-cpu` copy of the queue for the processor which executes current thread which wants to acquire lock, this means that owner of the lock may released it before this moment. So we may try to acquire lock again by the call of the `queued_spin_trylock` function:

```C
	if (queued_spin_trylock(lock))
		goto release;
```

It does the almost same thing `queued_spin_lock` function does.

If the lock was successfully acquired we jump to the `release` label to release a node of the `queue`:

```C
release:
	__this_cpu_dec(qnodes[0].mcs.count);
```

because we no need in it anymore as lock is acquired. If the `queued_spin_trylock` was unsuccessful, we update tail of the queue:

```C
	old = xchg_tail(lock, tail);
	next = NULL;
```

and retrieve previous tail. The next step is to check that `queue` is not empty. In this case we need to link previous entry with the new. While waitaing for the MCS lock, the next pointer may have been set by another lock waiter. We optimistically load the next pointer & prefetch the cacheline for writing to reduce latency in the upcoming MCS unlock operation:

```C
	if (old & _Q_TAIL_MASK) {
		prev = decode_tail(old);
		WRITE_ONCE(prev->next, node);

		arch_mcs_spin_lock_contended(&node->locked);
		
		next = READ_ONCE(node->next);
		if (next)
			prefetchw(next);
	}
```

If the new node was added, we prefetch cache line from memory pointed by the next queue entry with the [PREFETCHW](http://www.felixcloutier.com/x86/PREFETCHW.html) instruction. We preload this pointer now for optimization purpose. We just became a head of queue and this means that there is upcoming `MCS` unlock operation and the next entry will be touched.

Yes, from this moment we are in the head of the `queue`. But before we are able to acquire a lock, we need to wait at least two events: current owner of a lock will release it and the second thread with `pending` bit will acquire a lock too:

```C
	val = atomic_cond_read_acquire(&lock->val, !(VAL & _Q_LOCKED_PENDING_MASK));
```

After both threads will release a lock, the head of the `queue` will hold a lock. In the end we just need to update the tail of the `queue` and remove current head from it. 

That's all.

Conclusion
--------------------------------------------------------------------------------

This is the end of the second part of the [synchronization primitives](https://en.wikipedia.org/wiki/Synchronization_%28computer_science%29) chapter in the Linux kernel. In the previous [part](https://0xax.gitbooks.io/linux-insides/content/SyncPrim/linux-sync-1.html) we already met the first synchronization primitive `spinlock` provided by the Linux kernel which is implemented as `ticket spinlock`. In this part we saw another implementation of the `spinlock` mechanism - `queued spinlock`. In the next part we will continue to dive into synchronization primitives in the Linux kernel. 

If you have questions or suggestions, feel free to ping me in twitter [0xAX](https://twitter.com/0xAX), drop me [email](anotherworldofworld@gmail.com) or just create [issue](https://github.com/0xAX/linux-insides/issues/new).

**Please note that English is not my first language and I am really sorry for any inconvenience. If you found any mistakes please send me PR to [linux-insides](https://github.com/0xAX/linux-insides).**

Links
--------------------------------------------------------------------------------

* [spinlock](https://en.wikipedia.org/wiki/Spinlock)
* [interrupt](https://en.wikipedia.org/wiki/Interrupt)
* [interrupt handler](https://en.wikipedia.org/wiki/Interrupt_handler) 
* [API](https://en.wikipedia.org/wiki/Application_programming_interface)
* [Test and Set](https://en.wikipedia.org/wiki/Test-and-set)
* [MCS](http://www.cs.rochester.edu/~scott/papers/1991_TOCS_synch.pdf)
* [per-cpu variables](https://0xax.gitbooks.io/linux-insides/content/Concepts/linux-cpu-1.html)
* [atomic instruction](https://en.wikipedia.org/wiki/Linearizability)
* [CMPXCHG instruction](http://x86.renejeschke.de/html/file_module_x86_id_41.html) 
* [LOCK instruction](http://x86.renejeschke.de/html/file_module_x86_id_159.html)
* [NOP instruction](https://en.wikipedia.org/wiki/NOP)
* [PREFETCHW instruction](http://www.felixcloutier.com/x86/PREFETCHW.html)
* [x86_64](https://en.wikipedia.org/wiki/X86-64)
* [Previous part](https://0xax.gitbooks.io/linux-insides/content/SyncPrim/linux-sync-1.html)
