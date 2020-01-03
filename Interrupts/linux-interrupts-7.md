인터럽트와 인터럽트 처리. Part 7.
================================================================================

외부 인터럽트 소개
--------------------------------------------------------------------------------

이것은 리눅스 커널 [챕터](https://0xax.gitbooks.io/linux-insides/content/Interrupts/index.html)v에서 인터럽트 및 인터럽트 처리의 일곱 번째 부분이며 이전 [파트](https://0xax.gitbooks.io/linux-insides/content/Interrupts/linux-interrupts-6.html)에서는 프로세서에 의해 발생한 예외로 끝났습니다. 이번 파트에서 우리는 인터럽트 처리를 계속해서 진행할 것이며 외부 하드웨어 인터럽트 처리부터 시작할 것입니다. 아시다시피, 앞 부분에서 우리는 [arch/x86/kernel/trap.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/kernel/traps.c)의 `trap_init` 함수를 마쳤으며 다음 단계는 [init/main.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/init/main.c)의 `early_irq_init` 함수를 호출하는 것입니다.

인터럽트는 하드웨어나 소프트웨어에 의해 [IRQ](https://en.wikipedia.org/wiki/Interrupt_request_%28PC_architecture%29) 또는 '인터럽트 요청 라인'을 통해 전송되는 신호입니다. 키보드, 마우스 등과 같은 장치는 외부 하드웨어 인터럽트를 통해 프로세서의 주의가 필요함을 나타냅니다. 프로세서가 인터럽트 요청을 받으면 실행중인 프로그램의 실행을 일시적으로 중지하고 인터럽트에 따라 특별한 루틴을 호출합니다. 우리는 이미 이 루틴을 인터럽트 핸들러라고 부르고 있습니다(이 파트에서 이를 `ISR` 또는 `인터럽트 서비스 루틴`이라고 부르기도 합니다). `ISR` 또는 `Interrupt Handler Routine`은 메모리의 고정 주소에있는 인터럽트 벡터 테이블에서 찾을 수 있습니다. 인터럽트가 처리 된 후 프로세서는 인터럽트 된 프로세스를 다시 시작합니다. 부팅/초기화시 Linux 커널은 머신의 모든 장치를 식별하고 적절한 인터럽트 핸들러가 인터럽트 테이블에 로드됩니다. 이전 부분에서 보았듯이 대부분의 예외는 [Unix signal](https://en.wikipedia.org/wiki/Unix_signal)을 중단 된 프로세스로 보내는 것만으로 처리됩니다. 이것이 커널이 예외를 빠르게 처리 할 수있는 이유입니다. 불행히도 외부 하드웨어 인터럽트에는 이 방법을 사용할 수 없습니다. 외부 하드웨어 인터럽트는 종종 관련된 프로세스가 중단 된 후에 도착하기도합니다. 따라서 현재 프로세스에 Unix 신호를 보내는 것은 의미가 없습니다. 외부 인터럽트 처리는 인터럽트 유형에 따라 다릅니다. :

* `I/O` interrupts;
* Timer interrupts;
* Interprocessor interrupts.

이 책에서 모든 유형의 인터럽트에 대해 설명할 것 입니다.

일반적으로 `I/O` 인터럽트 핸들러는 여러 장치를 동시에 서비스 할 수 있을 정도로 유연해야합니다. 예를 들어 [PCI](https://en.wikipedia.org/wiki/Conventional_PCI) 버스 아키텍처에서는 여러 장치가 동일한 `IRQ` 라인을 공유 할 수 있습니다. 가장 간단한 방법으로 리눅스 커널은 `I/O` 인터럽트가 발생했을 때 다음과 같은 작업을 해야합니다 :

* `IRQ`의 값과 레지스터의 내용을 커널 스택에 저장합니다.;
* `IRQ` 회선을 서비스하는 하드웨어 컨트롤러에 승인을 보냅니다.
* 장치와 관련된 인터럽트 서비스 루틴(이것을 `ISR`이라고 할 것입니다)을 실행합니다.
* 레지스터를 복원하고 인터럽트에서 복귀합니다.

자, 우리는 약간의 이론을 알았으니 이제 `early_irq_init` 함수로 시작하겠습니다. `early_irq_init` 함수의 구현은 [kernel/irq/irqdesc.c]에 있습니다. 이 함수는 `irq_desc` 구조체를 초기에 초기화합니다. `irq_desc` 구조체는 리눅스 커널에서 인터럽트 관리 코드의 기초입니다. 구조체와 이름이 같은 이 구조체의 배열 `irq_desc`은 리눅스 커널의 모든 인터럽트 요청 소스를 추적합니다. 이 구조체는 [include/linux/irqdesc.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/include/linux/irqdesc.h)에 정의되어 있으며, 여기서 알 수 있듯이 `CONFIG_SPARSE_IRQ` 커널 구성 옵션에 따라 다릅니다. 이 커널 구성 옵션은 Sparse IRQ의 지원을 가능하게 합니다. `irq_desc` 구조체는 많은 다른 파일들을 포함합니다 :

* `irq_common_data` - chip 함수로 전달된 각각의 chip 데이터와 irq;
* `status_use_accessors` - [include/linux/irq.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/include/linux/irq.h)의 `enum`에서 얻은 값 및 동일한 소스 코드 파일에 정의 된 다른 매크로의 조합인 인터럽트 소스의 상태를 포함;
* `kstat_irqs` - 각 cpu의 irq 통계;
* `handle_irq` - 하이레벨 irq-events 핸들러;
* `action` - [IRQ](https://en.wikipedia.org/wiki/Interrupt_request_%28PC_architecture%29)가 발생할 때 호출 될 인터럽트 서비스 루틴을 식별;
* `irq_count` - IRQ 라인에서의 인터럽트 발생 카운터;
* `depth` - IRQ 라인이 활성화 된 경우 '0', 적어도 한 번 비활성화 된 경우 양수 값;
* `last_unhandled` - 처리되지 않은 카운트를 위한 aging 타이머;
* `irqs_unhandled` - 처리되지 않은 인터럽트의 수;
* `lock`  - IRQ 디스크립터에 대한 액세스를 직렬화하는 데 사용되는 스핀 잠금;
* `pending_mask` - 보류중인 재조정 인터럽트;
* `owner` - 인터럽트 디스크립터의 소유자. 인터럽트 디스크립터는 모듈에서 할당 될 수 있음. 이 필드는 인터럽트를 제공하는 모듈에 대한 참조 횟수를 증명해야함.
* and etc.

물론 이 구조체의 각 필드를 설명하기에는 너무 길기 때문에 이것이 `irq_desc` 구조체의 모든 필드는 아니지만 곧 모두 다루게 될 것입니다. 이제 `early_irq_init` 함수의 구현에 대해 알아 보겠습니다.

초기 외부 인터럽트 초기화
--------------------------------------------------------------------------------

이제 `early_irq_init` 함수의 구현을 살펴 봅시다. `early_irq_init`함수의 구현은 `CONFIG_SPARSE_IRQ` 커널 구성 옵션에 따라 다릅니다. `CONFIG_SPARSE_IRQ` 커널 설정 옵션이 설정되어 있지 않은 경우에는 `early_irq_init` 함수의 구현을 고려합니다. 이 함수는 `irq` 디스크립터 카운터, 루프 카운터, 메모리 노드 및 `irq_desc` 디스크립터와 같은 변수의 선언에서 시작합니다. :

```C
int __init early_irq_init(void)
{
        int count, i, node = first_online_node;
        struct irq_desc * desc; //띄어쓰기없애주기
		...
		...
		...
}
```

`node`는 온라인 [NUMA](https://en.wikipedia.org/wiki/Non-uniform_memory_access) 노드이며 `CONFIG_NODES_SHIFT` 커널 구성 매개 변수에 따라 달라지는 MAX_NUMNODES 값에 의존합니다.

```C
#define MAX_NUMNODES    (1 << NODES_SHIFT)
...
...
...
#ifdef CONFIG_NODES_SHIFT
    #define NODES_SHIFT     CONFIG_NODES_SHIFT
#else
    #define NODES_SHIFT     0
#endif
```

이미 쓴 것처럼 `first_online_node` 매크로의 구현은 `MAX_NUMNODES` 값에 따라 다릅니다. :

```C
#if MAX_NUMNODES > 1
  #define first_online_node       first_node(node_states[N_ONLINE])
#else
  #define first_online_node       0  
```

`node_states`는 [include/linux/nodemask.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/include/linux/nodemask.h)에 정의 된 [enum](https://en.wikipedia.org/wiki/Enumerated_type)이며 일련의 노드 상태를 나타냅니다. 우리의 경우 온라인 노드를 검색하고 있으며, MAX_NUMNODES가 1 또는 0 인 경우 `0`이 됩니다. 만약 `MAX_NUMNODES`가 1보다 크면 `node_states[N_ONLINE]`은 `1`을 리턴하고, `first_node` 매크로는 `__first_node` 함수의 호출로 확장되어 `최소` 또는 첫 온라인 노드를 리턴합니다:

```C
#define first_node(src) __first_node(&(src))

static inline int __first_node(const nodemask_t *srcp)
{
        return min_t(int, MAX_NUMNODES, find_first_bit(srcp->bits, MAX_NUMNODES));
}
```

이에 대한 자세한 내용은 `NUMA`에 관한 다른 챕터에서 다룰 것입니다. 이러한 지역 변수 선언 후 다음 단계는 다음 함수를 호출하는 것입니다. :

```C
init_irq_default_affinity();
```

동일한 소스 코드 파일에 정의 된 `init_irq_default_affinity` 함수는 `CONFIG_SMP` 커널 구성 옵션에 따라 주어진 [cpumask](https://junsoolee.gitbook.io/linux-insides-ko/summary/concepts/linux-cpu-2) 구조를 할당합니다 (여기에서는 `irq_default_affinity`) :

```C
#if defined(CONFIG_SMP)
cpumask_var_t irq_default_affinity;

static void __init init_irq_default_affinity(void)
{
        alloc_cpumask_var(&irq_default_affinity, GFP_NOWAIT);
        cpumask_setall(irq_default_affinity);
}
#else
static void __init init_irq_default_affinity(void)
{
}
#endif
```

우리는 디스크 컨트롤러나 키보드와 같은 하드웨어가 프로세서의 주의를 필요로 할 때 인터럽트가 발생한다는 것을 알고 있습니다. 인터럽트는 프로세서에 어떤 일이 발생했는지와, 프로세서가 현재 프로세스를 중단하고 들어오는 이벤트를 처리해야 함을 알려줍니다. 여러 장치가 동일한 인터럽트를 전송하지 못하도록 하기 위해 [IRQ](https://en.wikipedia.org/wiki/Interrupt_request_%28PC_architecture%29) 시스템은 컴퓨터 시스템의 각 장치에 고유한 특수 IRQ 가 할당되어 인터럽트가 고유하도록 설정되었습니다. 리눅스 커널은 특정 `IRQ`를 특정 프로세서에 할당 할 수 있습니다. 이것을 `SMP IRQ 선호도`라고 하며, 시스템이 다양한 하드웨어 이벤트에 응답하는 방법을 제어 할 수 있습니다 (따라서 `CONFIG_SMP` 커널 구성 옵션이 설정된 경우에만 특정 구현이 수행됩니다). `irq_default_affinity` cpumask를 할당 한 후, `printk` 출력을 볼 수 있습니다 :

```C
printk(KERN_INFO "NR_IRQS:%d\n", NR_IRQS);
```

which prints `NR_IRQS`:

```C
~$ dmesg | grep NR_IRQS
[    0.000000] NR_IRQS:4352
```

`NR_IRQS`는`irq` 디스크립터의 최대 수 또는 다른 말로 최대 인터럽트 수입니다. 값은 `CONFIG_X86_IO_APIC` 커널 구성 옵션의 상태에 따라 다릅니다. `CONFIG_X86_IO_APIC`가 설정되어 있지 않고 Linux 커널이 오래된 [PIC](https://en.wikipedia.org/wiki/Programmable_Interrupt_Controller) 칩을 사용하는 경우 `NR_IRQS`는 다음과 같습니다. :

```C
#define NR_IRQS_LEGACY                    16

#ifdef CONFIG_X86_IO_APIC
...
...
...
#else
# define NR_IRQS                        NR_IRQS_LEGACY
#endif
```

다른 방법으로, `CONFIG_X86_IO_APIC` 커널 설정 옵션이 설정되면, `NR_IRQS`는 프로세서의 수와 인터럽트 벡터의 수에 의존합니다 :

```C
#define CPU_VECTOR_LIMIT               (64 * NR_CPUS)
#define NR_VECTORS                     256
#define IO_APIC_VECTOR_LIMIT           ( 32 * MAX_IO_APICS )
#define MAX_IO_APICS                   128

# define NR_IRQS                                       \
        (CPU_VECTOR_LIMIT > IO_APIC_VECTOR_LIMIT ?     \
                (NR_VECTORS + CPU_VECTOR_LIMIT)  :     \
                (NR_VECTORS + IO_APIC_VECTOR_LIMIT))
...
...
...
```

이전 파트들에서 우리는 `CONFIG_NR_CPUS` 설정 옵션으로 리눅스 커널 설정 과정에서 설정할 수있는 프로세서의 수를 기억합니다 :

![kernel](http://oi60.tinypic.com/1zdm1dt.jpg)

첫 번째 경우(`CPU_VECTOR_LIMIT> IO_APIC_VECTOR_LIMIT`)에서 `NR_IRQS`는 `4352`가 되고, 두 번째 경우(`CPU_VECTOR_LIMIT <IO_APIC_VECTOR_LIMIT`)에서 `NR_IRQS`는 `768`이 됩니다. 필자의 경우 저의 설정에서 볼 수 있듯이 `NR_CPUS`는 `8`이며 `CPU_VECTOR_LIMIT`는 `512`이고, `IO_APIC_VECTOR_LIMIT`는`4096`입니다. 따라서 제 설정에 대한 `NR_IRQS`는 `4352`입니다. :

```
~$ dmesg | grep NR_IRQS
[    0.000000] NR_IRQS:4352
```

다음 단계에서는 IRQ 디스크립터의 배열을 `irly_irq_init` 함수의 시작에서 정의한 `irq_desc` 변수에 할당하고 `ARRAY_SIZE` 매크로를 사용하여 `irq_desc` 배열의 개수를 계산합니다. :

```C
desc = irq_desc;
count = ARRAY_SIZE(irq_desc);
```

`irq_desc` 배열은 동일한 소스 코드 파일에 정의되어 있으며 다음과 같습니다. :

```C
struct irq_desc irq_desc[NR_IRQS] __cacheline_aligned_in_smp = {
        [0 ... NR_IRQS-1] = {
                .handle_irq     = handle_bad_irq,
                .depth          = 1,
                .lock           = __RAW_SPIN_LOCK_UNLOCKED(irq_desc->lock),
        }
};
```

`irq_desc`는 `irq` 디스크립터의 배열입니다. 이미 초기화 된 세 개의 필드가 있습니다. :

* `handle _irq`- 위에서 이미 작성했듯이 이 필드는 irq-event 처리기입니다. 우리의 경우에는 [kernel/irq/handle.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/kernel/irq/handle.c) 소스 코드 파일에 정의 된 `handle_bad_irq` 함수를 이용하여 초기화되었고, 가짜와 처리되지 않은 irq를 처리합니다.
* `depth`- IRQ 라인이 활성화 된 경우 `0`, 적어도 한 번 비활성화 된 경우 양수 값.
* `lock`- IRQ 디스크립터에 대한 액세스를 직렬화하는 데 사용되는 스핀 잠금.

인터럽트 수를 계산하고 `irq_desc`배열을 초기화하면 루프에서 디스크립터를 채우기 시작합니다. :

```C
for (i = 0; i < count; i++) {
    desc[i].kstat_irqs = alloc_percpu(unsigned int);
    alloc_masks(&desc[i], GFP_KERNEL, node);
    raw_spin_lock_init(&desc[i].lock);
    lockdep_set_class(&desc[i].lock, &irq_desc_lock_class);
	desc_set_defaults(i, &desc[i], node, NULL);
}
```

모든 인터럽트 디스크립터를 살펴보고 다음을 수행합니다.

우선 우리는 `alloc_percpu` 매크로로 `irq` 커널 통계를 위한 [percpu](https://junsoolee.gitbook.io/linux-insides-ko/summary/concepts/linux-cpu-1) 변수를 할당합니다. 이 매크로는 시스템의 모든 프로세서에 대해 지정된 유형의 객체 인스턴스를 하나만 할당합니다. `/proc/stat`를 통해 사용자 공간에서 커널 통계에 접근 할 수 있습니다 :

```
~$ cat /proc/stat
cpu  207907 68 53904 5427850 14394 0 394 0 0 0
cpu0 25881 11 6684 679131 1351 0 18 0 0 0
cpu1 24791 16 5894 679994 2285 0 24 0 0 0
cpu2 26321 4 7154 678924 664 0 71 0 0 0
cpu3 26648 8 6931 678891 414 0 244 0 0 0
...
...
...
```

여섯 번째 열은 서비스 인터럽트입니다. 그런 다음 주어진 irq 디스크립터 affinity에 [cpumask](https://junsoolee.gitbook.io/linux-insides-ko/summary/concepts/linux-cpu-2)를 할당하고 주어진 인터럽트 디스크립터에 대한 [spinlock](https://en.wikipedia.org/wiki/Spinlock)을 초기화합니다. [critical section](https://en.wikipedia.org/wiki/Critical_section) 이전에는 `raw_spin_lock`의 호출로 잠금을 얻게 되고, `raw_spin_unlock`의 호출로 잠금이 해제됩니다. 다음 단계에서는 주어진 인터럽트 디스크립터의 잠금을 위해 [Lock validator]`irq_desc_lock_class` 클래스를 설정하는 `lockdep_set_class` 매크로를 호출합니다. `lockdep`,`spinlock` 및 기타 동기화 프리미티브에 대한 자세한 내용은 별도의 coqxj에서 설명합니다.

루프가 끝나면 [kernel/irq/irqdesc.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/kernel/irq/irqdesc.c)에서 `desc_set_defaults` 함수를 호출합니다. 이 함수는 다음과 같은 네 가지 매개 변수를 사용합니다. :

* irq의 수;
* 인터럽트 디스크립터;
* 온라인 `NUMA` 노드;
* 인터럽트 디스크립터의 소유자. 인터럽트 디스크립터는 모듈에서 할당 될 수 있습니다. 이 필드는 인터럽트를 제공하는 모듈에 대한 참조 횟수를 증명해야합니다;

나머지 `irq_desc` 필드를 채웁니다. `desc_set_defaults` 함수는 인터럽트 번호, `irq` 칩, 칩 형식에 대한 플랫폼 별 칩 각각의 개인 데이터, `irq_chip` 방식에 대한 각각의 IRQ 데이터 ,각 `irq` 와 `irq` 칩 데이터에 대한 [MSI](https://en.wikipedia.org/wiki/Message_Signaled_Interrupts) 디스크립터를 채웁니다. :

```C
desc->irq_data.irq = irq;
desc->irq_data.chip = &no_irq_chip;
desc->irq_data.chip_data = NULL;
desc->irq_data.handler_data = NULL;
desc->irq_data.msi_desc = NULL;
...
...
...
```

`irq_data.chip` 구조체는 irq 컨트롤러 [드라이버](https://github.com/torvalds/linux/tree/master/drivers/irqchip)에 대한 `irq_set_chip`, `irq_set_irq_type` 등과 같은 일반적인 `API` 를 제공합니다. [kernel/irq/chip.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/kernel/irq/chip.c) 소스 코드 파일에서 찾을 수 있습니다.

그런 다음 주어진 디스크립터에 대한 접근 자의 상태를 설정하고 인터럽트의 비활성화 상태를 설정합니다. :

```C
...
...
...
irq_settings_clr_and_set(desc, ~0, _IRQ_DEFAULT_INIT_FLAGS);
irqd_set(&desc->irq_data, IRQD_IRQ_DISABLED);
...
...
...
```

다음 단계에서 우리는 하드웨어 인터럽트가 아직 초기화되지 않았기 때문에, 가짜와 처리되지 않은 irq 들을 처리하는 `handle_bad_irq`를 사용하여 높은 수준의 인터럽트 핸들러를 설정하고, 'irq_desc.desc'를 '1'로 설정했습니다. `IRQ`가 비활성화 된 경우 처리되지 않은 인터럽트와 인터럽트의 수를 재설정합니다.

```C
...
...
...
desc->handle_irq = handle_bad_irq;
desc->depth = 1;
desc->irq_count = 0;
desc->irqs_unhandled = 0;
desc->name = NULL;
desc->owner = owner;
...
...
...
```

그런 다음 [for_each_possible_cpu](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/include/linux/cpumask.h#L714) 헬퍼와 함께 모든 [가능한](https://junsoolee.gitbook.io/linux-insides-ko/summary/concepts/linux-cpu-2) 프로세서를 살펴보고 주어진 인터럽트 디스크립터에 대해 `kstat_irqs`를 0으로 설정합니다 :

```C
	for_each_possible_cpu(cpu)
		*per_cpu_ptr(desc->kstat_irqs, cpu) = 0;
```

그리고, 주어진 인터럽트 디스크립터의 NUMA 노드를 초기화하고, 기본 SMP affinity를 설정하고 주어진 인터럽트 디스크립터의 `pending_mask`를 지우는 [kernel/irq/irqdesc.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/kernel/irq/irqdesc.c)에서 `desc_smp_init` 함수를 호출합니다. `CONFIG_GENERIC_PENDING_IRQ` 커널 설정 옵션의 값 :

```C
static void desc_smp_init(struct irq_desc *desc, int node)
{
        desc->irq_data.node = node;
        cpumask_copy(desc->irq_data.affinity, irq_default_affinity);
#ifdef CONFIG_GENERIC_PENDING_IRQ
        cpumask_clear(desc->pending_mask);
#endif
}
```

`early_irq_init` 함수의 끝에서 우리는 `arch_early_irq_init` 함수의 리턴값을 반환합니다 :

```C
return arch_early_irq_init();
```

이 함수는 [kernel/apic/vector.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/kernel/apic/vector.c)에 정의되어 있으며 [kernel/apic/io_apic.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/kernel/apic/io_apic.c)에서 `arch_early_ioapic_init` 함수를 한 번만 호출합니다. `arch_early_ioapic_init` 함수 이름에서 알 수 있듯이이 함수는 [I/O APIC](https://en.wikipedia.org/wiki/Advanced_Programmable_Interrupt_Controller)를 초기에 초기화합니다. 우선 `nr_legacy_irqs` 함수를 호출하여 레거시 인터럽트 수를 확인합니다. [Intel 8259](https://en.wikipedia.org/wiki/Intel_8259) 프로그래밍가능(programmable)한 인터럽트 컨트롤러로 레거시 인터럽트가 없는 경우 `io_apic_irqs`를`0xfffffffffffffffffff`로 설정합니다.

```C
if (!nr_legacy_irqs())
	io_apic_irqs = ~0UL;
```

그런 다음 모든 `I/O APIC`를 거치고 `alloc_ioapic_saved_registers`를 호출하여 레지스터를 위한 공간을 할당합니다.

```C
for_each_ioapic(i)
	alloc_ioapic_saved_registers(i);
```

그리고 `arch_early_ioapic_init` 함수의 끝에서 우리는 루프에서 모든 레거시 irq (IRQ0에서 IRQ15까지)를 거치고 주어진 `NUMA` 노드에서 irq의 구성을 나타내는 `irq_cfg`를 위한 공간을 할당합니다.:

```C
for (i = 0; i < nr_legacy_irqs(); i++) {
    cfg = alloc_irq_and_cfg_at(i, node);
    cfg->vector = IRQ0_VECTOR + i;
    cpumask_setall(cfg->domain);
}
```

끝났습니다.

Sparse IRQs
--------------------------------------------------------------------------------

이 파트의 시작 부분에서 이미 `early_irq_init` 함수의 구현은 `CONFIG_SPARSE_IRQ` 커널 구성 옵션에 의존한다는 것을 알았습니다. 이전에는 `CONFIG_SPARSE_IRQ` 구성 옵션이 설정되지 않은 경우의 `early_irq_init` 함수의 구현을 보았으므로 이제 이 옵션이 설정 된 경우의 구현을 살펴 보겠습니다. 이 함수의 구현은 매우 유사하지만 약간 다릅니다. `early_irq_init` 함수의 시작 부분에서 동일한 변수의 정의와 `init_irq_default_affinity`의 호출을 볼 수 있습니다. :

```C
#ifdef CONFIG_SPARSE_IRQ
int __init early_irq_init(void)
{
    int i, initcnt, node = first_online_node;
	struct irq_desc *desc;

	init_irq_default_affinity();
	...
	...
	...
}
#else
...
...
...
```

하지만 이후에 우리는 다음의 호출을 볼 수 있습니다. :

```C
initcnt = arch_probe_nr_irqs();
```

[arch/x86/kernel/apic/vector.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/kernel/apic/vector.c)에 정의 된 `arch_probe_nr_irqs` 함수는 미리 할당 된 irq의 개수를 계산하고 그 수로 `nr_irqs`를 업데이트합니다. 여기서 사전 할당 된 irq가있는 이유는 무엇일까요? [PCI](https://en.wikipedia.org/wiki/Conventional_PCI)에서 사용할 수있는 [Message Signaled Interrupts]라는 다른 형태의 인터럽트가 있습니다. 이 장치는 인터럽트 요청의 고정 된 수를 할당하는 것 대신에, RAM의 특정 주소, 실제로 [Local APIC](https://en.wikipedia.org/wiki/Advanced_Programmable_Interrupt_Controller#Integrated_local_APICs)에 표시되는 메시지를 기록 할 수 있습니다. `MSI`는 장치가 `1`,`2`,`4`,`8`,`16` 또는 `32` 인터럽트를 할당하도록 허용하고, `MSI-X`는 장치가 최대 `2048` 인터럽트를 할당하도록 허용합니다. 이제 우리는 irq를 미리 할당 할 수 있다는 것을 알고 있습니다. `MSI`에 대한 자세한 내용은 다음 파트에 있으므로, 여기서는 `arch_probe_nr_irqs` 함수를 살펴 보겠습니다. 시스템의 각 프로세서에 대한 인터럽트 벡터의 수가 `nr_irqs`보다 더 큰지 확인할 수 있고, `MSI` 인터럽트 수를 나타내는 `nr`을 계산할 수 있습니다. :

```C
int nr_irqs = NR_IRQS;

if (nr_irqs > (NR_VECTORS * nr_cpu_ids))
	nr_irqs = NR_VECTORS * nr_cpu_ids;

nr = (gsi_top + nr_legacy_irqs()) + 8 * nr_cpu_ids;
```

`gsi_top` 변수를 살펴봅시다. 각 `APIC`는 자체 `ID`와 `IRQ`가 시작되는 오프셋으로 식별됩니다. 이를 `GSI` 또는 `Global System Interrupt`라고합니다. 따라서 `gsi_top`은 이를 나타냅니다. [MultiProcessor Configuration Table](https://en.wikipedia.org/wiki/MultiProcessor_Specification) 테이블에서 `Global System Interrupt` 기반을 얻습니다 (Linux Kernel 초기화 프로세스 챕터의 여섯 번째 [파트](https://junsoolee.gitbook.io/linux-insides-ko/summary/initialization/linux-initialization-6)에서 이 테이블을 파싱했음을 기억할 수 있습니다).

그런 다음 `nr`을 업데이트하면 `gsi_top`의 값에 따라 달라집니다 :

```C
#if defined(CONFIG_PCI_MSI) || defined(CONFIG_HT_IRQ)
        if (gsi_top <= NR_IRQS_LEGACY)
                nr +=  8 * nr_cpu_ids;
        else
                nr += gsi_top * 16;
#endif
```

`nr`보다 작은 경우 `nr_irqs`를 업데이트하고 legasy irq의 수를 리턴하십시오. :

```C
if (nr < nr_irqs)
    nr_irqs = nr;

return nr_legacy_irqs();
}
```

`arch_probe_nr_irqs` 다음에 `IRQs` 수에 대한 정보가 출력됩니다.

```C
printk(KERN_INFO "NR_IRQS:%d nr_irqs:%d %d\n", NR_IRQS, nr_irqs, initcnt);
```

[dmesg] 출력에서 이를 볼 수 있습니다. :

```
$ dmesg | grep NR_IRQS
[    0.000000] NR_IRQS:4352 nr_irqs:488 16
```

그런 다음 우리는 `nr_irqs`와 `initcnt` 값이 최대 허용할 수 있는 `irqs`수보다 크지 않은지 확인합니다 :

```C
if (WARN_ON(nr_irqs > IRQ_BITMAP_BITS))
    nr_irqs = IRQ_BITMAP_BITS;

if (WARN_ON(initcnt > IRQ_BITMAP_BITS))
    initcnt = IRQ_BITMAP_BITS;
```

여기서 `CONFIG_SPARSE_IRQ`가 설정되어 있지 않고 `NR_IRQS + 8196`가 아닌 경우, `IRQ_BITMAP_BITS`는 `NR_IRQS`와 같습니다. 다음 단계에서는 루프안에서 할당될 필요가 있는 모든 디스크립터를 살펴보고, 그 디스크립터를 위한 공간을 할당하고 `irq_desc_tree` [radix tree](https://junsoolee.gitbook.io/linux-insides-ko/summary/datastructures/linux-datastructures-2)에 삽입합니다. :

```C
for (i = 0; i < initcnt; i++) {
    desc = alloc_desc(i, node, NULL);
    set_bit(i, allocated_irqs);
	irq_insert_desc(i, desc);
}
```

`early_irq_init` 함수의 끝에서 `CONFIG_SPARSE_IRQ` 옵션이 설정되지 않았을 때, 이전 변형에서 했던 것처럼 `arch_early_irq_init` 함수의 호출 값을 리턴합니다 :

```C
return arch_early_irq_init();
```

이상입니다.

결론
--------------------------------------------------------------------------------

[인터럽트와 인터럽트 처리](https://junsoolee.gitbook.io/linux-insides-ko/summary/interrupts) 챕터의 일곱 번째 파트가 끝났고, 이 파트에서 외부 하드웨어 인터럽트에 대해 알아보기 시작했습니다. 외부 인터럽트에 대한 설명을 하고, irq 액션 목록, 인터럽트 핸들러에 대한 정보, 인터럽트 소유자, 처리되지 않은 인터럽트 수 등의 정보를 포함하는 `irq_desc`구조체의 초기 초기화를 보았습니다. 다음 부분에서는 외부 인터럽트를 계속 살펴봅시다.

질문이나 제안 사항이 있으면 [twitter](https://twitter.com/0xAX)에 의견이나 핑을 남겨주세요.

**영어는 제 모국어가 아닙니다. 그리고 여타 불편하셨던 점에 대해서 정말로 사과드립니다. 만약 실수를 찾아내셨다면 부디 [linux-insides 원본](https://github.com/0xAX/linux-internals)으로, 번역에 대해서는 [linux-insides 한글 번역](https://github.com/junsooo/linux-insides-ko)으로 PR을 보내주세요.**

Links
--------------------------------------------------------------------------------

* [IRQ](https://en.wikipedia.org/wiki/Interrupt_request_%28PC_architecture%29)
* [numa](https://en.wikipedia.org/wiki/Non-uniform_memory_access)
* [Enum type](https://en.wikipedia.org/wiki/Enumerated_type)
* [cpumask](https://0xax.gitbooks.io/linux-insides/content/Concepts/linux-cpu-2.html)
* [percpu](https://0xax.gitbooks.io/linux-insides/content/Concepts/linux-cpu-1.html)
* [spinlock](https://en.wikipedia.org/wiki/Spinlock)
* [critical section](https://en.wikipedia.org/wiki/Critical_section)
* [Lock validator](https://lwn.net/Articles/185666/)
* [MSI](https://en.wikipedia.org/wiki/Message_Signaled_Interrupts)
* [I/O APIC](https://en.wikipedia.org/wiki/Advanced_Programmable_Interrupt_Controller)
* [Local APIC](https://en.wikipedia.org/wiki/Advanced_Programmable_Interrupt_Controller#Integrated_local_APICs)
* [Intel 8259](https://en.wikipedia.org/wiki/Intel_8259)
* [PIC](https://en.wikipedia.org/wiki/Programmable_Interrupt_Controller)
* [MultiProcessor Configuration Table](https://en.wikipedia.org/wiki/MultiProcessor_Specification)
* [radix tree](https://0xax.gitbooks.io/linux-insides/content/DataStructures/linux-datastructures-2.html)
* [dmesg](https://en.wikipedia.org/wiki/Dmesg)
