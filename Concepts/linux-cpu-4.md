리눅스 커널의 알림 체인
================================================================================

소개
--------------------------------------------------------------------------------

리눅스 커널의 여러 하위시스템으로 구성된 거대한 [C](https://en.wikipedia.org/wiki/C_(programming_language))코드의 조각입니다. 각 서브 시스템은 다른 서브 시스템과 독립적인 고유한 목적을 가지고 있습니다. 그러나 종종 한 서브 시스템은 다른 서브 시스템에서 무언가를 알고 싶어합니다. 리눅스 커널에는 이 문제를 부분적으로 해결하는 것을 허용하는 특별한 메커니즘이 있습니다. 이 메커니즘의 이름은 `notification chains`으로 이것의 주 목적은 다른 서브시스템을 위해 또 다른 서브시스템의 비동기 이벤트를 구독할 수 있는 방법을 제공하는 것입니다. 이 메커니즘은 커널 내부의 소통에만 사용되지만, 커널과 사용자 공간 간의 소통에는 또 다른 메커니즘이 있습니다.

`notification chains` [API](https://en.wikipedia.org/wiki/Application_programming_interface)와 이 API의 구현을 생각하기 전에, 이 책의 다른 파트에서 한 것처럼 이론적 측면에서 메커니즘을 살펴봅시다. `notification chains`메커니즘과 관련된 모든 것은 [include/linux/notifier.h](https://github.com/torvalds/linux/blob/master/include/linux/notifier.h)헤더 파일 및 [kernel/notifier.c](https://github.com/torvalds/linux/blob/master/kernel/notifier.c)소스코드 파일에 있습니다. 파일을 열어 살펴봅시다.

알림 체인 관련 데이터 구조체
--------------------------------------------------------------------------------

관련 데이터 구조체에서 `notification chains`메커니즘을 고려해봅시다. 위에서 쓴 것처럼, 메인 데이터 구조체는 [include/linux/notifier.h](https://github.com/torvalds/linux/blob/master/include/linux/notifier.h)헤더 파일에 있어야하므로 리눅스 커널은 특정 아키텍처에 의존하지 않는 일반 API를 제공합니다. 일반적으로, `notification chains`메커니즘은 이벤트가 일어났을 때 수행되는 [callback](https://en.wikipedia.org/wiki/Callback_(computer_programming))함수의 리스트(`chains`라 명명된 이유)를 나타냅니다.

이러한 모든 콜백 함수는 리눅스 커널에서 `notifier_fn_t`형식으로 표시됩니다:

```C
typedef	int (*notifier_fn_t)(struct notifier_block *nb, unsigned long action, void *data);
```

따라서 다음 세 가지 인자가 필요하다는 것을 알 수 있습니다:

* `nb` - 연결된 함수 포인터의 리스트(이제 볼 것입니다);
* `action` - 이벤트 유형. 알림 체인은 여러 이벤트를 지원할 수 있으므로 다른 이벤트와 구별하려면 이 매개변수가 필요;
* `data` - 제한 정보를 위한 저장공간. 실제로 이벤트에 대한 추가적인 데이터 제공을 허용함.

추가적으로 `notifier_fn_t`이 정수값을 반환하는 것을 볼 수 있습니다. 이 정수 값은 다음 값중 하나일 수 있습니다:

* `NOTIFY_DONE` - 알림에 흥미가 없는 구독자;
* `NOTIFY_OK` - 알림이 올바르게 처리됨;
* `NOTIFY_BAD` - 무언가 잘못됨;
* `NOTIFY_STOP` - 알림이 완료됐지만, 이벤트에 대해 더 이상 콜백을 호출하지 않아야 함.

이러한 모든 결과는 [include/linux/notifier.h](https://github.com/torvalds/linux/blob/master/include/linux/notifier.h)헤더 파일에서 매크로로 정의됩니다:

```C
#define NOTIFY_DONE		0x0000
#define NOTIFY_OK		0x0001
#define NOTIFY_BAD		(NOTIFY_STOP_MASK|0x0002)
#define NOTIFY_STOP		(NOTIFY_OK|NOTIFY_STOP_MASK)
```

`NOTIFY_STOP_MASK`에서 다음과 같이 표시됩니다:

```C
#define NOTIFY_STOP_MASK	0x8000
```

매크로는 다음 알림 중에 콜백이 호출되지 않음을 의미합니다.

특정 이벤트에 대해 알림을 받으려는 리눅스 커널의 각 파트는 자체 `notifier_fn_t`콜백 함수를 제공해야합니다. `notification chains`메커니즘의 주 역할은 비동기 이벤트가 일어났을 때 특정 콜백을 호출하는 것입니다.

`notification chains`의 주요 구성 요소는 `notifier_block`구조체입니다:

```C
struct notifier_block {
	notifier_fn_t notifier_call;
	struct notifier_block __rcu *next;
	int priority;
};
```

[include/linux/notifier.h](https://github.com/torvalds/linux/blob/master/include/linux/notifier.h)파일에 정의되어 있습니다. 이 구조체는 콜백 함수`notifier_call`에 대한 포인터, 다음 알림 콜백 및 처음으로 실행되는 높은 우선순위의 함수와 같은 콜백 함수 `priority`에 대한 연결을 포함합니다.

리눅스 커널은 다음과 같은 네 가지 유형의 알림 체인을 제공합니다:

* 알리미 체인 차단;
* SRCU 알리미 체인;
* 어토믹 알리미 체인;
* 가공되지 않은 알리미 체인.

이러한 유형의 알림 체인을 모두 순서대로 고려해봅시다:

첫 번째 경우는 `blocking notifier chains`로, 콜백이 프로세스 컨텍스트에서 호출/실행됩니다. 이는 알림체인의 호출이 차단될 수 있다는 것을 의미합니다.

두 번째로 `SRCU notifier chains`는 `blocking notifier chains`의 다른 형식을 나타냅니다. 첫 번째 경우, 차단 알리미 체인은 `rw_semaphore`동기화 기본을 사용해 체인 링크를 보호합니다. `SRCU`알리미 체인은 프로세스 컨텍스트에서도 실행되지만, 읽기 측면에서 중요 섹션에서 차단할 수 있는 특별한 형식의 [RCU](https://en.wikipedia.org/wiki/Read-copy-update)메커니즘을 사용합니다.

세 번째 경우는 `atomic notifier chains`으로 인터럽트 또는 어토믹 컨텍스트에서 실행되고 [스핀락](https://0xax.gitbooks.io/linux-insides/content/SyncPrim/linux-sync-1.html)동기화 기본으로 보호됩니다. 마지막으로 `raw notifier chains`는 콜백에 대한 잠금 제한 없이 알리미 체인의 특별한 유형을 제공합니다.  이는 보호가 호출자의 사이드 숄더에 달려 있음을 의미합니다. 이것은 매우 특정한 잠금 메커니즘으로 체인을 보호하려고 할 때 매우 유용합니다.

`notifier_block`구조체의 구현을 살펴보면, 알림 체인 리스트에서 `next`요소에 대한 포인터를 포함한 것을 볼 수 있지만, 헤드가 없습니다. 실제로 이러한 목록의 헤드는 별도의 구조체로 되어 있으며 알림 체인의 유형에 따라 다릅니다. 예시로 `blocking notifier chains`대한 것입니다:

```C
struct blocking_notifier_head {
	struct rw_semaphore rwsem;
	struct notifier_block __rcu *head;
};
```

또는 `atomic notification chains`에 대한 것입니다:

```C
struct atomic_notifier_head {
	spinlock_t lock;
	struct notifier_block __rcu *head;
};
```

이제 우리는 `notification chains`메커니즘에 관해 조금 알았고 API의 구현을 고려합시다.

알림 체인
--------------------------------------------------------------------------------

일반적으로 발행/구독자 메커니즘에는 두 가지 측면이 있습니다. 한 측은 알림을 받고자 하고 다른 측은 이러한 알림을 생성합니다. 우리는 이 양 측면에서 알림 체인 메커니즘을 고려할 것입니다. 알림 체인의 다른 유형이 비슷하고 대부분 보호 메커니즘이 다르기 때문에 이 파트에서 `blocking notification chains`을 고려할 것입니다.

알림 생성자가 알림을 생성하기 전에, 먼저 알림 체인의 헤드를 초기하해야 합니다. 예를 들어 커널 [로드 가능 모듈](https://en.wikipedia.org/wiki/Loadable_kernel_module)과 관련된 알림 체인을 생각해봅시다. [kernel/module.c](https://github.com/torvalds/linux/blob/master/kernel/module.c)소스 코드 파일을 보면, 다음 정의를 볼 수 있습니다:

```C
static BLOCKING_NOTIFIER_HEAD(module_notify_list);
```

알림 체인을 차단하는 로드 가능한 모둘을 위한 헤드를 정의합니다. `BLOCKING_NOTIFIER_HEAD`매크로는 [include/linux/notifier.h](https://github.com/torvalds/linux/blob/master/include/linux/notifier.h)헤더 파일에서 정의되었으며 다음의 코드로 확장합니다:

```C
#define BLOCKING_INIT_NOTIFIER_HEAD(name) do {	\
		init_rwsem(&(name)->rwsem);	                            \
		(name)->head = NULL;		                            \
	} while (0)
```

따라서 알리미 체인을 차단하는 헤드의 이름의 이름을 얻고 읽기/쓰기 [semaphore](https://0xax.gitbooks.io/linux-insides/content/SyncPrim/linux-sync-3.html)를 초기화하고 헤드를 `NULL`로 설정하는 것을 볼 수 있습니다. `BLOCKING_INIT_NOTIFIER_HEAD`매크로 외에도, 리눅스 커널에서 추가적으로 `ATOMIC_INIT_NOTIFIER_HEAD`, `RAW_INIT_NOTIFIER_HEAD`매크로와 어토믹 초기화를 위한 `srcu_init_notifier`함수와 알림 체인의 다른 유형을 제공합니다.

알림 체인의 헤드를 초기화한 후에, 주어진 알림 체인에서 알림을 받길 원하는 서브시스템은 알림의 유형에 따른 특정 함수를 등록해야합니다. [include/linux/notifier.h](https://github.com/torvalds/linux/blob/master/include/linux/notifier.h)헤더 파일을 보면, 이에 대한 다음 네 가지 함수를 볼 수 있습니다:

```C
extern int atomic_notifier_chain_register(struct atomic_notifier_head *nh,
		struct notifier_block *nb);

extern int blocking_notifier_chain_register(struct blocking_notifier_head *nh,
		struct notifier_block *nb);

extern int raw_notifier_chain_register(struct raw_notifier_head *nh,
		struct notifier_block *nb);

extern int srcu_notifier_chain_register(struct srcu_notifier_head *nh,
		struct notifier_block *nb);
```

위에서 이미 쓴 것처럼, 우리는 이 파트에서 알림 체인을 차단하는 것만 다루므로, `blocking_notifier_chain_register`함수의 구현을 고려해봅시다. 이 함수의 구현은 [kernel/notifier.c](https://github.com/torvalds/linux/blob/master/kernel/notifier.c)소스 코드 파일에 있으며 `blocking_notifier_chain_register`이 두 가지 매개변수를 사용하는 것을 볼 수 있습니다:

* `nh` - 알림 체인의 헤드;
* `nb` - 알림 디스크립터.

이제 `blocking_notifier_chain_register`함수의 구현을 살펴봅시다:

```C
int raw_notifier_chain_register(struct raw_notifier_head *nh,
		struct notifier_block *n)
{
	return notifier_chain_register(&nh->head, n);
}
```

우리가 보는 것처럼 동일한 소스 코드 파일에서  `notifier_chain_register`함수의 결과를 반환하고 우리가 이해한 것처럼 이 함수는 우리를 위해 모든 기능을 수행합니다. `notifier_chain_register`함수의 정의를 봅시다:

```C
int blocking_notifier_chain_register(struct blocking_notifier_head *nh,
		struct notifier_block *n)
{
	int ret;

	if (unlikely(system_state == SYSTEM_BOOTING))
		return notifier_chain_register(&nh->head, n);

	down_write(&nh->rwsem);
	ret = notifier_chain_register(&nh->head, n);
	up_write(&nh->rwsem);
	return ret;
}
```

우리가 본 것처럼 `blocking_notifier_chain_register`의 구현은 매우 간단합니다. 우선 현재 시스템 상태를 확인하는 검사가 있으며 재부팅 상태의 서브시스템이면 `notifier_chain_register`를 호출합니다. 다른 방법으로 동일한 `notifier_chain_register`의 호출을 수행하지만 우리가 본 것처럼 이 호출은 읽기/쓰기 세마포어로 보호됩니다. 이제 `notifier_chain_register`함수의 구현을 봅시다:

```C
static int notifier_chain_register(struct notifier_block **nl,
		struct notifier_block *n)
{
	while ((*nl) != NULL) {
		if (n->priority > (*nl)->priority)
			break;
		nl = &((*nl)->next);
	}
	n->next = *nl;
	rcu_assign_pointer(*nl, n);
	return 0;
}
```

이 함수는 새로운 `notifier_block`(알림을 얻길 원하는 서브시스템에서 줌)을 알림 체인 리스트에 삽입합니다. 이벤트를 구독하는 것 외에도, 구독자는 `unsubscribe`함수의 설정으로 특정 이벤트를 구독 취소 할 수 있습니다:

```C
extern int atomic_notifier_chain_unregister(struct atomic_notifier_head *nh,
		struct notifier_block *nb);

extern int blocking_notifier_chain_unregister(struct blocking_notifier_head *nh,
		struct notifier_block *nb);

extern int raw_notifier_chain_unregister(struct raw_notifier_head *nh,
		struct notifier_block *nb);

extern int srcu_notifier_chain_unregister(struct srcu_notifier_head *nh,
		struct notifier_block *nb);
```

알림의 생산자가 구독자에게 이벤트에 대해 알리려고 할 때, `*.notifier_call_chain`함수가 호출됩니다. 이미 알고 있듯이 각 알림 체인의 유형은 알림을 생성하는 자체 함수를 제공합니다:

```C
extern int atomic_notifier_call_chain(struct atomic_notifier_head *nh,
		unsigned long val, void *v);

extern int blocking_notifier_call_chain(struct blocking_notifier_head *nh,
		unsigned long val, void *v);

extern int raw_notifier_call_chain(struct raw_notifier_head *nh,
		unsigned long val, void *v);

extern int srcu_notifier_call_chain(struct srcu_notifier_head *nh,
		unsigned long val, void *v);
```

`blocking_notifier_call_chain`함수의 구현을 고려해봅시다. 이 함수는 [kernel/notifier.c](https://github.com/torvalds/linux/blob/master/kernel/notifier.c)소스 코드 파일에서 정의되었습니다:

```C
int blocking_notifier_call_chain(struct blocking_notifier_head *nh,
		unsigned long val, void *v)
{
	return __blocking_notifier_call_chain(nh, val, v, -1, NULL);
}
```

보시다시피 `__blocking_notifier_call_chain`함수의 결과를 반환합니다. 우리가 볼 수 있듯이 `blocking_notifer_call_chain`은 세 매개변수를 취합니다:

* `nh` - 알림 체인 리스트의 헤드;
* `val` - 알림의 유형;
* `v` -  처리기가 사용할 수 있는 입력 매개변수.

그러나 이 `__blocking_notifier_call_chain`함수는 다섯 가지 매개변수를 취합니다:

```C
int __blocking_notifier_call_chain(struct blocking_notifier_head *nh,
				   unsigned long val, void *v,
				   int nr_to_call, int *nr_calls)
{
    ...
    ...
    ...
}
```

호출되는 알리미 함수의 수와 전송되는 알림의 수는 `nr_to_call`와 `nr_calls`에 있습니다. 추측한 것처럼 `__blocking_notifer_call_chain`함수와 다른 알림 유형을 위한 다른 함수의 주 목표는 이벤트가 일어났을 때 콜백 함수를 호출하는 것입니다. `__blocking_notifier_call_chain`의 구현은 매우 간단합니다. 읽기/쓰기 세마포어로 보호되는 동일한 소스코드 파일에서 `notifier_call_chain`함수를 호출합니다:

```C
int __blocking_notifier_call_chain(struct blocking_notifier_head *nh,
				   unsigned long val, void *v,
				   int nr_to_call, int *nr_calls)
{
	int ret = NOTIFY_DONE;

	if (rcu_access_pointer(nh->head)) {
		down_read(&nh->rwsem);
		ret = notifier_call_chain(&nh->head, val, v, nr_to_call,
					nr_calls);
		up_read(&nh->rwsem);
	}
	return ret;
}
```

그리고 결과를 반환합니다. 이 경우 모든 작업은 `notifier_call_chain`함수에 의해 수행됩니다. 이 함수의 주 목적은 등록된 알리미에게 비동기 이벤트에 관해 알려주는 것입니다:

```C
static int notifier_call_chain(struct notifier_block **nl,
			       unsigned long val, void *v,
			       int nr_to_call, int *nr_calls)
{
    ...
    ...
    ...
    ret = nb->notifier_call(nb, val, v);
    ...
    ...
    ...
    return ret;
}
```

그것이 전부입니다. 일반적으로 모든 것이 단순해 보입니다.

이제 [로드 가능 모듈](https://en.wikipedia.org/wiki/Loadable_kernel_module)과 관련된 간단한 예를 생각해봅시다. [kernel/module.c](https://github.com/torvalds/linux/blob/master/kernel/module.c)를 살펴볼 경우. 우리는 이미 이 파트에서 봤습니다:

```C
static BLOCKING_NOTIFIER_HEAD(module_notify_list);
```

[kernel/module.c](https://github.com/torvalds/linux/blob/master/kernel/module.c)소스 코드 파일에 `module_notify_list`의 정의가 있습니다. 아 정의는 커널 모듈의 차단 알리미 체인의 리스트의 헤드를 결정합니다. 다음과 같은 세 가지 이상의 이벤트가 있습니다:

* MODULE_STATE_LIVE
* MODULE_STATE_COMING
* MODULE_STATE_GOING

리눅스 커널의 일부 하위시스템에 관심을 가질 수 있습니다. 예를 들어 커널 모듈 상태의 추적이 있습니다. `atomic_notifier_chain_register`, `blocking_notifier_chain_register` 등의 직접 호출 대신, 대부분의 알림 체인은 그것들에 등록하는데 사용되는 래퍼의 모음과 함께 제공됩니다. 이러한 모듈 이벤트의 등록은 래퍼의 도움으로 진행됩니다:

```C
int register_module_notifier(struct notifier_block *nb)
{
	return blocking_notifier_chain_register(&module_notify_list, nb);
}
```

[kernel/tracepoint.c](https://github.com/torvalds/linux/blob/master/kernel/tracepoint.c)소스 코드 파일을 보면, [트레이스 포인트](https://www.kernel.org/doc/Documentation/trace/tracepoints.txt)의 초기화가 이뤄지는 동의안 등록을 볼 수 있습니다:

```C
static __init int init_tracepoints(void)
{
	int ret;

	ret = register_module_notifier(&tracepoint_module_nb);
	if (ret)
		pr_warn("Failed to register tracepoint module enter notifier\n");

	return ret;
}
```

`tracepoint_module_nb`가 콜백 함수를 제공하는 곳:

```C
static struct notifier_block tracepoint_module_nb = {
	.notifier_call = tracepoint_module_notify,
	.priority = 0,
};
```

`MODULE_STATE_LIVE`, `MODULE_STATE_COMING` 또는 `MODULE_STATE_GOING`중 하나의 이벤트가 일어났을 때. 예를 들어 `MODULE_STATE_LIVE` `MODULE_STATE_COMING`알림이 [init_module](http://man7.org/linux/man-pages/man2/init_module.2.html) [시스템 호출](https://0xax.gitbooks.io/linux-insides/content/SysCall/linux-syscall-1.html)의 실행 중에 전달됩니다. 또는 예를 들어 `MODULE_STATE_GOING`은 [delete_module](http://man7.org/linux/man-pages/man2/delete_module.2.html) `system call`의 실행 중에 전달됩니다:

```C
SYSCALL_DEFINE2(delete_module, const char __user *, name_user,
		unsigned int, flags)
{
    ...
    ...
    ...
    blocking_notifier_call_chain(&module_notify_list,
				     MODULE_STATE_GOING, mod);
    ...
    ...
    ...
}
```

따라서 이러한 시스템 호출 중 하나가 사용자 공간에서 호출되면, 리눅스 커널은 시스템 콜에 따르는 특정 알림을 보내고 `tracepoint_module_notify` 콜백 함수가 호출됩니다.

그것이 전부입니다.

Links
--------------------------------------------------------------------------------

* [C programming langauge](https://en.wikipedia.org/wiki/C_(programming_language))
* [API](https://en.wikipedia.org/wiki/Application_programming_interface)
* [callback](https://en.wikipedia.org/wiki/Callback_(computer_programming))
* [RCU](https://en.wikipedia.org/wiki/Read-copy-update)
* [spinlock](https://0xax.gitbooks.io/linux-insides/content/SyncPrim/linux-sync-1.html)
* [loadable modules](https://en.wikipedia.org/wiki/Loadable_kernel_module)
* [semaphore](https://0xax.gitbooks.io/linux-insides/content/SyncPrim/linux-sync-3.html)
* [tracepoints](https://www.kernel.org/doc/Documentation/trace/tracepoints.txt)
* [system call](https://0xax.gitbooks.io/linux-insides/content/SysCall/linux-syscall-1.html)
* [init_module system call](http://man7.org/linux/man-pages/man2/init_module.2.html)
* [delete_module](http://man7.org/linux/man-pages/man2/delete_module.2.html)
* [previous part](https://0xax.gitbooks.io/linux-insides/content/Concepts/linux-cpu-3.html)
