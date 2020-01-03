인터럽트 및 인터럽트 처리. Part 10.
================================================================================

마지막 파트
-------------------------------------------------------------------------------

이것은 리눅스 커널에서 인터럽트 처리를 다루는 세 번째 파트입니다 [chapter](https://0xax.gitbooks.io/linux-insides/content/Interrupts/index.html)  그리고 이전 [파트](https://0xax.gitbooks.io/linux-insides/content/Interrupts/linux-interrupts-9.html)에서 우리는 지연된 인터럽트와 `softirq`, `tasklet` 및 `workqeue`와 같은 관련 개념에 대해 조금 보았습니다. 이 부분에서 우리는이 주제를 계속해서 살펴볼 것이며 이제 실제 하드웨어 드라이버를 살펴볼 차례입니다.

예를 들어 [StrongARM ** SA-110 / 21285 Evaluation Board](http://netwinder.osuosl.org/pub/netwinder/docs/intel/datashts/27813501.pdf) 보드의 직렬 드라이버를 고려해 보자. 이 드라이버는 [IRQ](https://en.wikipedia.org/wiki/Interrupt_request_%28PC_architecture%29) 줄을 요청합니다.
이 트리거의 소스 코드는 [drivers / tty / serial / 21285.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/drivers/tty/serial/21285.c)에 있습니다. 소스코드를 갖고 있으므로, 시작합시다!

커널 모듈의 초기화
--------------------------------------------------------------------------------

우리는 이 책에서 보았던 모든 새로운 개념으로 보통이 드라이버를 고려했을 것입니다. 우리는 그것을 초기화에서 고려하기 시작할 것입니다. 이미 알고 있듯이 Linux 커널은 드라이버 또는 커널 모듈의 초기화 및 마무리를 위한 두 가지 매크로를 제공합니다.

* `module_init`;
* `module_exit`.

드라이버 소스 코드에서 이러한 매크로의 사용법을 찾을 수 있습니다.

```C
module_init(serial21285_init);
module_exit(serial21285_exit);
```

장치 드라이버의 대부분은로드 가능한 커널 [module](https://en.wikipedia.org/wiki/Loadable_kernel_module) 또는 다른 방법으로 Linux 커널에 정적으로 링크 될 수 있습니다. 첫 번째 경우 장치 드라이버의 초기화는 [include / linux / init.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/include/linux/init.h)에 정의 된`module_init` 및`module_exit` 매크로를 통해 생성됩니다.  :

```C
#define module_init(initfn)                                     \
        static inline initcall_t __inittest(void)               \
        { return initfn; }                                      \
        int init_module(void) __attribute__((alias(#initfn)));

#define module_exit(exitfn)                                     \
        static inline exitcall_t __exittest(void)               \
        { return exitfn; }                                      \
        void cleanup_module(void) __attribute__((alias(#exitfn)));
```

[initcall](http://kernelnewbies.org/Documents/InitcallMechanism) 함수에 의해 호출됩니다.
:

* `early_initcall`
* `pure_initcall`
* `core_initcall`
* `postcore_initcall`
* `arch_initcall`
* `subsys_initcall`
* `fs_initcall`
* `rootfs_initcall`
* `device_initcall`
* `late_initcall`

[init / main.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/init/main.c)의 `do_initcalls`에서 호출됩니다. 그렇지 않으면 장치 드라이버가 Linux 커널에 정적으로 링크 된 경우 이러한 매크로의 구현은 다음과 같습니다.
```C
#define module_init(x)  __initcall(x);
#define module_exit(x)  __exitcall(x);
```

이런 식으로 [kernel / module.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/kernel/module.c) 소스 코드 파일에 배치 된 모듈 로딩 구현은`do_init_module` 함수에서 초기화됩니다. 이 장에서는 로드 가능한 모듈에 대해 자세히 다루지 않지만 Linux 커널 모듈을 설명하는 특별 장에서 볼 수 있습니다. 자, `module_init` 매크로는 하나의 매개 변수-우리의 경우 `serial21285_init`를 사용합니다. 함수 이름에서 알 수 있듯이이 함수는 드라이버 초기화와 관련된 작업을 수행합니다. 봅시다:

```C
static int __init serial21285_init(void)
{
	int ret;

	printk(KERN_INFO "Serial: 21285 driver\n");

	serial21285_setup_ports();

	ret = uart_register_driver(&serial21285_reg);
	if (ret == 0)
		uart_add_one_port(&serial21285_reg, &serial21285_port);

	return ret;
}
```

보다시피, 먼저 드라이버에 대한 정보를 커널 버퍼에 출력하고 `serial21285_setup_ports` 함수를 호출합니다. 이 함수는 `serial21285_port`장치의 기본 [uart](https://en.wikipedia.org/wiki/Universal_asynchronous_receiver/transmitter) 클락을 설정합니다.
```C
unsigned int mem_fclk_21285 = 50000000;

static void serial21285_setup_ports(void)
{
	serial21285_port.uartclk = mem_fclk_21285 / 4;
}
```

`serial21285`는 `uart` 드라이버를 설명하는 구조체입니다 :

```C
static struct uart_driver serial21285_reg = {
	.owner			= THIS_MODULE,
	.driver_name	= "ttyFB",
	.dev_name		= "ttyFB",
	.major			= SERIAL_21285_MAJOR,
	.minor			= SERIAL_21285_MINOR,
	.nr			    = 1,
	.cons			= SERIAL_21285_CONSOLE,
};
```

드라이버가 성공적으로 등록되면 [drivers/tty/serial/serial_core.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/drivers/tty/serial/serial_core.c)의 `uart_add_one_port` 함수로 드라이버 정의 포트 `serial21285_port` 구조체를 연결합니다  소스 코드 파일이며 `serial21285_init` 함수에서 반환됩니다.

```C
if (ret == 0)
	uart_add_one_port(&serial21285_reg, &serial21285_port);

return ret;
```

그게 전부입니다. 드라이버가 초기화되었습니다. [drivers/tty/serial/serial_core.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/drivers/tty/serial/serial_core.c)에서 `uart_open` 함수를 호출하여 `uart` 포트를 열 때 , `uart_startup` 함수를 호출하여 직렬 포트를 시작합니다. 이 함수는 `uart_ops` 구조의 일부인 `startup` 함수를 호출합니다. 각 `uart` 드라이버는 이 구조체에 대한 정의를 가지고 있습니다.


```C
static struct uart_ops serial21285_ops = {
	...
	.startup	= serial21285_startup,
	...
}
```

`serial21285` 구조. 보시다시피`serial21285_startup` 함수에서`.strartup` 필드 참조를 볼 수 있습니다. 이 함수의 구현은 인터럽트 및 인터럽트 처리와 관련되어 있기 때문에 매우 흥미 롭습니다.

irq 라인 요청
--------------------------------------------------------------------------------

`serial21285` 함수의 구현을 살펴 봅시다.

```C
static int serial21285_startup(struct uart_port *port)
{
	int ret;

	tx_enabled(port) = 1;
	rx_enabled(port) = 1;

	ret = request_irq(IRQ_CONRX, serial21285_rx_chars, 0,
			  serial21285_name, port);
	if (ret == 0) {
		ret = request_irq(IRQ_CONTX, serial21285_tx_chars, 0,
				  serial21285_name, port);
		if (ret)
			free_irq(IRQ_CONRX, port);
	}

	return ret;
}
```

우선`TX`와 `RX`에 대해. 장치의 직렬 버스는 두 개의 전선으로 구성됩니다. 하나는 데이터 전송 용이고 다른 하나는 수 신용입니다. 따라서 직렬 장치에는 두 개의 직렬 핀, 즉 수신기- `RX` 및 송신기-`TX`가 있어야합니다. 첫 번째 두 매크로 인 tx_enabled와 rx_enabled를 호출하여이 와이어를 활성화합니다. 이 기능의 다음 부분은 우리에게 가장 큰 관심사입니다. `request_irq` 함수에 주의하십시오. 이 함수는 인터럽트 핸들러를 등록하고 지정된 인터럽트 라인을 활성화합니다. 이 함수의 구현을 살펴보고 세부 사항을 살펴 보겠습니다. 이 함수는 [include/linux/interrupt.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/include/linux/interrupt.h) 헤더 파일에 정의되어 있으며 다음과 같습니다.

```C
static inline int __must_check
request_irq(unsigned int irq, irq_handler_t handler, unsigned long flags,
            const char *name, void *dev)
{
        return request_threaded_irq(irq, handler, NULL, flags, name, dev);
}
```

보시다시피`request_irq` 함수는 5 개의 매개 변수를 취합니다 :

* `irq`-요청 된 인터럽트 번호;
* `handler`-인터럽트 핸들러에 대한 포인터;
* `flags`-비트 마스크 옵션;
* `name`-인터럽트 소유자의 이름.
* `dev`-공유 인터럽트 라인에 사용되는 포인터.

이제 우리 예제에서`request_irq` 함수의 호출을 봅시다. 보시다시피 첫 번째 매개 변수는`IRQ_CONRX`입니다. 우리는 그것이 인터럽트의 수라는 것을 알고 있습니다. 그러나  무엇이 `CONRX`일까요? [arch/ arm / mach-footbridge / include / mach / irqs.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d93/arch/arm/mach-footbridge/include/mach/irqs.h) 헤더 파일에 정의 된이 매크로  `21285` 보드가 생성 할 수있는 전체 인터럽트 목록을 찾을 수 있습니다. `request_irq` 함수의 두 번째 호출에서 우리는`IRQ_CONTX` 인터럽트 번호를 전달합니다. 이 두 인터럽트는 드라이버에서`RX` 및`TX` 이벤트를 처리합니다. 이 매크로의 구현은 쉽습니다.
```C
#define IRQ_CONRX               _DC21285_IRQ(0)
#define IRQ_CONTX               _DC21285_IRQ(1)
...
...
...
#define _DC21285_IRQ(x)         (16 + (x))
```

이 보드의 [ISA] https://en.wikipedia.org/wiki/Industry_Standard_Architecture) IRQ는 `0`에서 `15`이므로 인터럽트는 `16`과 `17`의 첫 두 숫자를 갖습니다. `request_irq` 함수의 두 호출에 대한 두 번째 매개 변수는 `serial21285_rx_chars` 및 `serial21285_tx_chars`입니다. 이 함수들은 RX 또는 TX 인터럽트가 발생했을 때 호출됩니다. 이 장에서는 인터럽트와 인터럽트 처리를 다루지만 장치와 드라이버는 다루지 않기 때문에이 기능에 대한 자세한 내용은 다루지 않겠습니다. 다음 매개 변수 인`flags` 와 보시다시피 `request_irq` 함수의 두 호출에서 모두 0입니다. 허용되는 모든 플래그는 [include/linux/interrupt.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/include/linux/interrupt.h)에서`IRQF_ *`매크로로 정의됩니다 . 그 중 일부:

* `IRQF_SHARED`-여러 장치간에 irq를 공유 할 수 있습니다.
* `IRQF_PERCPU`-인터럽트는 CPU 당입니다.
* `IRQF_NO_THREAD`-인터럽트를 스레드 할 수 없습니다.
* `IRQF_NOBALANCING`-irq 밸런싱에서이 인터럽트를 제외합니다.
* `IRQF_IRQPOLL`-폴링에 인터럽트가 사용됩니다.
* 등

우리의 경우 `0`을 전달하므로 `IRQF_TRIGGER_NONE`이됩니다. 이 플래그는 모든 종류의 엣지 또는 레벨 트리거 인터럽트 동작을 의미하지 않습니다. 네 번째 매개 변수 (`name`)에 다음과 같이 정의 된 `serial21285_name`을 전달합니다.


```C
static const char serial21285_name[] = "Footbridge UART";
```

`/proc/interrupts`의 출력에 표시됩니다. 그리고 마지막 매개 변수에서 우리는 포인터를 기본 `uart_port` 구조로 전달합니다. 이제 우리는 `request_irq` 함수와 그 매개 변수에 대해 조금 알고 있습니다. 구현을 살펴 보겠습니다. 위에서 볼 수 있듯이 `request_irq` 함수는 내부에서 `request_threaded_irq` 함수를 호출합니다. [kernel/irq/manage.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d93/kernel/irq/manage.c) 소스 코드 파일에 정의 된 `request_threaded_irq` 함수. 이 함수를 살펴보면 `irqaction`과 `irq_desc`의 정의에서 시작합니다.

```C
int request_threaded_irq(unsigned int irq, irq_handler_t handler,
                         irq_handler_t thread_fn, unsigned long irqflags,
                         const char *devname, void *dev_id)
{
        struct irqaction *action;
        struct irq_desc *desc;
        int retval;
		...
		...
		...
}
```

We already saw the `irqaction` and the `irq_desc` structures in this chapter. The first structure represents per interrupt action descriptor and contains pointers to the interrupt handler, name of the device, interrupt number, etc. The second structure represents a descriptor of an interrupt and contains pointer to the `irqaction`, interrupt flags, etc. Note that the `request_threaded_irq` function called by the `request_irq` with the additional parameter: `irq_handler_t thread_fn`. If this parameter is not `NULL`, the `irq` thread will be created and the given `irq` handler will be executed in this thread. In the next step we need to make following checks:

```C
if (((irqflags & IRQF_SHARED) && !dev_id) ||
            (!(irqflags & IRQF_SHARED) && (irqflags & IRQF_COND_SUSPEND)) ||
            ((irqflags & IRQF_NO_SUSPEND) && (irqflags & IRQF_COND_SUSPEND)))
               return -EINVAL;
```

First of all we check that real `dev_id` is passed for the shared interrupt and the `IRQF_COND_SUSPEND` only makes sense for shared interrupts. Otherwise we exit from this function with the `-EINVAL` error. After this we convert the given `irq` number to the `irq` descriptor wit the help of the `irq_to_desc` function that defined in the [kernel/irq/irqdesc.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/kernel/irq/irqdesc.c) source code file and exit from this function with the `-EINVAL` error if it was not successful:

```C
desc = irq_to_desc(irq);
if (!desc)
    return -EINVAL;
```

The `irq_to_desc` function checks that given `irq` number is less than maximum number of IRQs and returns the irq descriptor where the `irq` number is offset from the `irq_desc` array:

```C
struct irq_desc *irq_to_desc(unsigned int irq)
{
        return (irq < NR_IRQS) ? irq_desc + irq : NULL;
}
```

As we have converted `irq` number to the `irq` descriptor we make the check the status of the descriptor that an interrupt can be requested:

```C
if (!irq_settings_can_request(desc) || WARN_ON(irq_settings_is_per_cpu_devid(desc)))
    return -EINVAL;
```

and exit with the `-EINVAL`otherways. After this we check the given interrupt handler. If it was not passed to the `request_irq` function, we check the `thread_fn`. If both handlers are `NULL`, we return with the `-EINVAL`. If an interrupt handler was not passed to the `request_irq` function, but the `thread_fn` is not null, we set handler to the `irq_default_primary_handler`:

```C
if (!handler) {
    if (!thread_fn)
        return -EINVAL;
	handler = irq_default_primary_handler;
}
```

In the next step we allocate memory for our `irqaction` with the `kzalloc` function and return from the function if this operation was not successful:

```C
action = kzalloc(sizeof(struct irqaction), GFP_KERNEL);
if (!action)
    return -ENOMEM;
```

More about `kzalloc` will be in the separate chapter about [memory management](https://0xax.gitbooks.io/linux-insides/content/MM/index.html) in the Linux kernel. As we allocated space for the `irqaction`, we start to initialize this structure with the values of interrupt handler, interrupt flags, device name, etc:

```C
action->handler = handler;
action->thread_fn = thread_fn;
action->flags = irqflags;
action->name = devname;
action->dev_id = dev_id;
```

In the end of the `request_threaded_irq` function we call the `__setup_irq` function from the [kernel/irq/manage.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/kernel/irq/manage.c) and registers a given `irqaction`. Release memory for the `irqaction` and return:

```C
chip_bus_lock(desc);
retval = __setup_irq(irq, desc, action);
chip_bus_sync_unlock(desc);

if (retval)
	kfree(action);

return retval;
```

Note that the call of the `__setup_irq` function is placed between the `chip_bus_lock` and the `chip_bus_sync_unlock` functions. These functions lock/unlock access to slow busses (like [i2c](https://en.wikipedia.org/wiki/I%C2%B2C)) chips. Now let's look at the implementation of the `__setup_irq` function. In the beginning of the `__setup_irq` function we can see a couple of different checks. First of all we check that the given interrupt descriptor is not `NULL`, `irqchip` is not `NULL` and that given interrupt descriptor module owner is not `NULL`. After this we check if the interrupt is nested into another interrupt thread or not, and if it is nested we replace the `irq_default_primary_handler` with the `irq_nested_primary_handler`.

In the next step we create an irq handler thread with the `kthread_create` function, if the given interrupt is not nested and the `thread_fn` is not `NULL`:

```C
if (new->thread_fn && !nested) {
	struct task_struct *t;
	t = kthread_create(irq_thread, new, "irq/%d-%s", irq, new->name);
	...
}
```

And fill the rest of the given interrupt descriptor fields in the end. So, our `16` and `17` interrupt request lines are registered and the `serial21285_rx_chars` and `serial21285_tx_chars` functions will be invoked when an interrupt controller will get event releated to these interrupts. Now let's look at what happens when an interrupt occurs. 

Prepare to handle an interrupt
--------------------------------------------------------------------------------

In the previous paragraph we saw the requesting of the irq line for the given interrupt descriptor and registration of the `irqaction` structure for the given interrupt. We already know that when an interrupt event occurs, an interrupt controller notifies the processor about this event and processor tries to find appropriate interrupt gate for this interrupt. If you have read the eighth [part](https://0xax.gitbooks.io/linux-insides/content/Interrupts/linux-interrupts-8.html) of this chapter, you may remember the `native_init_IRQ` function. This function makes initialization of the local [APIC](https://en.wikipedia.org/wiki/Advanced_Programmable_Interrupt_Controller). The following part of this function is the most interesting part for us right now: 

```C
for_each_clear_bit_from(i, used_vectors, first_system_vector) {
	set_intr_gate(i, irq_entries_start +
		8 * (i - FIRST_EXTERNAL_VECTOR));
}
```

Here we iterate over all the cleared bit of the `used_vectors` bitmap starting at `first_system_vector` that is:

```C
int first_system_vector = FIRST_SYSTEM_VECTOR; // 0xef
```

and set interrupt gates with the `i` vector number and the `irq_entries_start + 8 * (i - FIRST_EXTERNAL_VECTOR)` start address. Only one thing is unclear here - the `irq_entries_start`. This symbol defined in the [arch/x86/entry/entry_64.S](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/entry_entry_64.S) assembly file and provides `irq` entries. Let's look at it:

```assembly
	.align 8
ENTRY(irq_entries_start)
    vector=FIRST_EXTERNAL_VECTOR
    .rept (FIRST_SYSTEM_VECTOR - FIRST_EXTERNAL_VECTOR)
	pushq	$(~vector+0x80)
    vector=vector+1
	jmp	common_interrupt
	.align	8
    .endr
END(irq_entries_start)
```

Here we can see the [GNU assembler](https://en.wikipedia.org/wiki/GNU_Assembler) `.rept` instruction which repeats the sequence of lines that are before `.endr` - `FIRST_SYSTEM_VECTOR - FIRST_EXTERNAL_VECTOR` times. As we already know, the `FIRST_SYSTEM_VECTOR` is `0xef`, and the `FIRST_EXTERNAL_VECTOR` is equal to `0x20`. So, it will work:

```python
>>> 0xef - 0x20
207
```

times. In the body of the `.rept` instruction we push entry stubs on the stack (note that we use negative numbers for the interrupt vector numbers, because positive numbers already reserved to identify [system calls](https://en.wikipedia.org/wiki/System_call)), increase the `vector` variable and jump on the `common_interrupt` label. In the `common_interrupt` we adjust vector number on the stack and execute `interrupt` number with the `do_IRQ` parameter:

```assembly
common_interrupt:
	addq	$-0x80, (%rsp)
	interrupt do_IRQ
```

The macro `interrupt` defined in the same source code file and saves [general purpose](https://en.wikipedia.org/wiki/Processor_register) registers on the stack, change the userspace `gs` on the kernel with the `SWAPGS` assembler instruction if need, increase [per-cpu](https://0xax.gitbooks.io/linux-insides/content/Concepts/linux-cpu-1.html) - `irq_count` variable that shows that we are in interrupt and call the `do_IRQ` function. This function defined in the [arch/x86/kernel/irq.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/kernel/irq.c) source code file and handles our device interrupt. Let's look at this function. The `do_IRQ` function takes one parameter - `pt_regs` structure that stores values of the userspace registers:

```C
__visible unsigned int __irq_entry do_IRQ(struct pt_regs *regs)
{
    struct pt_regs *old_regs = set_irq_regs(regs);
    unsigned vector = ~regs->orig_ax;
    unsigned irq;

	irq_enter();
    exit_idle();
	...
	...
	...
}
```

At the beginning of this function we can see call of the `set_irq_regs` function that returns saved `per-cpu` irq register pointer and the calls of the `irq_enter` and `exit_idle` functions. The first function `irq_enter` enters to an interrupt context with the updating `__preempt_count` variable and the second function - `exit_idle` checks that current process is `idle` with [pid](https://en.wikipedia.org/wiki/Process_identifier) - `0` and notify the `idle_notifier` with the `IDLE_END`.

In the next step we read the `irq` for the current cpu and call the `handle_irq` function:

```C
irq = __this_cpu_read(vector_irq[vector]);

if (!handle_irq(irq, regs)) {
	...
	...
	...
}
...
...
...
```

The `handle_irq` function defined in the [arch/x86/kernel/irq_64.c](https://github.com/torvalds/linux/blob/arch/x86/kernel/irq_64.c) source code file, checks the given interrupt descriptor and call the `generic_handle_irq_desc`:

```C
desc = irq_to_desc(irq);
	if (unlikely(!desc))
		return false;
generic_handle_irq_desc(irq, desc);
```

Where the `generic_handle_irq_desc` calls the interrupt handler:

```C
static inline void generic_handle_irq_desc(unsigned int irq, struct irq_desc *desc)
{
       desc->handle_irq(irq, desc);
}
```

But stop... What is it `handle_irq` and why do we call our interrupt handler from the interrupt descriptor when we know that `irqaction` points to the actual interrupt handler? Actually the `irq_desc->handle_irq` is a high-level API for the calling interrupt handler routine. It setups during initialization of the [device tree](https://en.wikipedia.org/wiki/Device_tree) and [APIC](https://en.wikipedia.org/wiki/Advanced_Programmable_Interrupt_Controller) initialization. The kernel selects correct function and call chain of the `irq->action(s)` there. In this way, the `serial21285_tx_chars` or the `serial21285_rx_chars` function will be executed after an interrupt will occur.

In the end of the `do_IRQ` function we call the `irq_exit` function that will exit from the interrupt context, the `set_irq_regs` with the old userspace registers and return:

```C
irq_exit();
set_irq_regs(old_regs);
return 1;
```

We already know that when an `IRQ` finishes its work, deferred interrupts will be executed if they exist.

Exit from interrupt
--------------------------------------------------------------------------------

Ok, the interrupt handler finished its execution and now we must return from the interrupt. When the work of the `do_IRQ` function will be finsihed, we will return back to the assembler code in the [arch/x86/entry/entry_64.S](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/entry_entry_64.S) to the `ret_from_intr` label. First of all we disable interrupts with the `DISABLE_INTERRUPTS` macro that expands to the `cli` instruction and decreases value of the `irq_count` [per-cpu](https://0xax.gitbooks.io/linux-insides/content/Concepts/linux-cpu-1.html) variable. Remember, this variable had value - `1`, when we were in interrupt context:

```assembly
DISABLE_INTERRUPTS(CLBR_NONE)
TRACE_IRQS_OFF
decl	PER_CPU_VAR(irq_count)
```

In the last step we check the previous context (user or kernel), restore it in a correct way and exit from an interrupt with the:

```assembly
INTERRUPT_RETURN
```

where the `INTERRUPT_RETURN` macro is:

```C
#define INTERRUPT_RETURN	jmp native_iret
```

and

```assembly
ENTRY(native_iret)

.global native_irq_return_iret
native_irq_return_iret:
	iretq
```

That's all.

Conclusion
--------------------------------------------------------------------------------

It is the end of the tenth part of the [Interrupts and Interrupt Handling](https://0xax.gitbooks.io/linux-insides/content/Interrupts/index.html) chapter and as you have read in the beginning of this part - it is the last part of this chapter. This chapter started from the explanation of the theory of interrupts and we have learned what is it interrupt and kinds of interrupts, then we saw exceptions and handling of this kind of interrupts, deferred interrupts and finally we looked on the hardware interrupts and the handling of theirs in this part. Of course, this part and even this chapter does not cover full aspects of interrupts and interrupt handling in the Linux kernel. It is not realistic to do this. At least for me. It was the big part, I don't know how about you, but it was really big for me. This theme is much bigger than this chapter and I am not sure that somewhere there is a book that covers it. We have missed many part and aspects of interrupts and interrupt handling, but I think it will be good point to dive in the kernel code related to the interrupts and interrupts handling.

If you have any questions or suggestions write me a comment or ping me at [twitter](https://twitter.com/0xAX).

**Please note that English is not my first language, And I am really sorry for any inconvenience. If you find any mistakes please send me PR to [linux-insides](https://github.com/0xAX/linux-insides).**

Links
--------------------------------------------------------------------------------

* [Serial driver documentation](https://www.kernel.org/doc/Documentation/serial/driver)
* [StrongARM** SA-110/21285 Evaluation Board](http://netwinder.osuosl.org/pub/netwinder/docs/intel/datashts/27813501.pdf)
* [IRQ](https://en.wikipedia.org/wiki/Interrupt_request_%28PC_architecture%29)
* [module](https://en.wikipedia.org/wiki/Loadable_kernel_module)
* [initcall](http://kernelnewbies.org/Documents/InitcallMechanism)
* [uart](https://en.wikipedia.org/wiki/Universal_asynchronous_receiver/transmitter) 
* [ISA](https://en.wikipedia.org/wiki/Industry_Standard_Architecture) 
* [memory management](https://0xax.gitbooks.io/linux-insides/content/MM/index.html)
* [i2c](https://en.wikipedia.org/wiki/I%C2%B2C)
* [APIC](https://en.wikipedia.org/wiki/Advanced_Programmable_Interrupt_Controller)
* [GNU assembler](https://en.wikipedia.org/wiki/GNU_Assembler)
* [Processor register](https://en.wikipedia.org/wiki/Processor_register)
* [per-cpu](https://0xax.gitbooks.io/linux-insides/content/Concepts/linux-cpu-1.html)
* [pid](https://en.wikipedia.org/wiki/Process_identifier)
* [device tree](https://en.wikipedia.org/wiki/Device_tree)
* [system calls](https://en.wikipedia.org/wiki/System_call)
* [Previous part](https://0xax.gitbooks.io/linux-insides/content/Interrupts/linux-interrupts-9.html)
