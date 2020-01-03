커널 초기화 Part 10
================================================================================

리눅스 커널 초기화 과정의 끝
================================================================================

리눅스 커널 [초기화 프로세스](https://0xax.gitbooks.io/linux-insides/content/Initialization/index.html)에 관한 챕터의 10번째 부분입니다. [이전 부분](https://0xax.gitbooks.io/linux-insides/content/Initialization/linux-initialization-9.html) [RCU](http://en.wikipedia.org/wiki/Read-copy-update)의 초기화를 봤고 `acpi_early_init` 함수 호출을 멈추었다. 이 부분은 [커널 초기화 프로세스](https://0xax.gitbooks.io/linux-insides/content/Initialization/index.html) 장의 마지막 부분이고 끝내 봅시다.

[init/main.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/init/main.c)에서 `acpi_early_init` 함수를 호출 한 후 다음 코드를 볼 수 있습니다:

```C
#ifdef CONFIG_X86_ESPFIX64
	init_espfix_bsp();
#endif
```

여기에서는 `CONFIG_X86_ESPFIX64` 커널 설정 옵션에 따른 `init_espfix_bsp` 함수의 호출을 볼 수 있습니다. 함수 이름에서 알 수 있듯이 스택과 관련이 있습니다. 이 함수는 [arch/x86/kernel/espfix_64.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/kernel/espfix_64.c)에 정의되어 있으며 16 비트 스택으로 복귀하는 동안 `esp` 레지스터의`31:16` 비트 손실을 방지합니다. 우선 `espfix` 페이지 상단 디렉토리를 `init_espfix_bs`의 커널 페이지 디렉토리에 설치합니다:

```C
pgd_p = &init_level4_pgt[pgd_index(ESPFIX_BASE_ADDR)];
pgd_populate(&init_mm, pgd_p, (pud_t *)espfix_pud_page);
```

`ESPFIX_BASE_ADDR`은 다음과 같습니다:

```C
#define PGDIR_SHIFT     39
#define ESPFIX_PGD_ENTRY _AC(-2, UL)
#define ESPFIX_BASE_ADDR (ESPFIX_PGD_ENTRY << PGDIR_SHIFT)
```

또한 [Documentation/x86/x86_64/mm](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/Documentation/x86/x86_64/mm.txt)에서 찾을 수 있습니다:

```
... unused hole ...
ffffff0000000000 - ffffff7fffffffff (=39 bits) %esp fixup stacks
... unused hole ...
```

`espfix` pud로 페이지 글로벌 디렉토리를 채운 후 다음 단계는 `init_espfix_random` 및 `init_espfix_ap` 함수의 호출입니다. 첫 번째 함수는 `espfix`페이지에 대한 임의의 위치를 반환하고 두 번째 함수는 현재 CPU에 대해 `espfix`를 활성화합니다. `init_espfix_bsp`가 작업을 마치면 [kernel/fork.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/kernel/fork.c)에 정의 된 `thread_info_cache_init` 함수의 호출을 볼 수 있습니다. `THREAD_SIZE`가 `PAGE_SIZE`보다 작은 경우 `thread_info`에 캐시를 할당합니다.

```C
# if THREAD_SIZE >= PAGE_SIZE
...
...
...
void thread_info_cache_init(void)
{
        thread_info_cache = kmem_cache_create("thread_info", THREAD_SIZE,
                                              THREAD_SIZE, 0, NULL);
        BUG_ON(thread_info_cache == NULL);
}
...
...
...
#endif
```

우리가 이미 알고 있듯이 `PAGE_SIZE`는 `(_AC (1, UL) << PAGE_SHIFT)` 또는 `4096` 바이트이고 `THREAD_SIZE`는 `x86_64`의 경우 `(PAGE_SIZE << THREAD_SIZE_ORDER)` 또는 `16384` 바이트입니다 . `thread_info_cache_init` 이후의 다음 함수는 [kernel/cred.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/kernel/cred.c)의 `cred_init`입니다. 이 함수는 자격 증명에 캐시를 할당합니다(`uid`,`gid` 등):

```C
void __init cred_init(void)
{
         cred_jar = kmem_cache_create("cred_jar", sizeof(struct cred),
                                     0, SLAB_HWCACHE_ALIGN|SLAB_PANIC, NULL);
}
```

자격 증명에 대한 자세한 내용은 [Documentation/security/credentials.txt](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/Documentation/security/credentials.txt)에서 확인할 수 있습니다. 다음 단계는 [kernel/fork.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/kernel/fork.c)의 `fork_init` 기능입니다. `fork_init` 함수는 `task_struct`에 캐시를 할당합니다. `fork_init`의 구현을 살펴 보자. 우선 `ARCH_MIN_TASKALIGN` 매크로의 정의와 task_structs가 할당 될 슬래브 생성을 볼 수 있습니다:

```C
#ifndef CONFIG_ARCH_TASK_STRUCT_ALLOCATOR
#ifndef ARCH_MIN_TASKALIGN
#define ARCH_MIN_TASKALIGN      L1_CACHE_BYTES
#endif
        task_struct_cachep =
                kmem_cache_create("task_struct", sizeof(struct task_struct),
                        ARCH_MIN_TASKALIGN, SLAB_PANIC | SLAB_NOTRACK, NULL);
#endif
```

보시다시피이 코드는 `CONFIG_ARCH_TASK_STRUCT_ACLLOCATOR` 커널 구성 옵션에 따라 다릅니다. 이 설정 옵션은 주어진 아키텍처에 대한 `alloc_task_struct`의 존재를 보여줍니다. `x86_64`에는 `alloc_task_struct` 기능이 없으므로이 코드는 작동하지 않으며 `x86_64`에서 컴파일되지 않습니다.

초기화 작업을 위한 캐시 할당
--------------------------------------------------------------------------------

그런 다음 `fork_init`에서 `arch_task_cache_init` 함수의 호출을 볼 수 있습니다:

```C
void arch_task_cache_init(void)
{
        task_xstate_cachep =
                kmem_cache_create("task_xstate", xstate_size,
                                  __alignof__(union thread_xstate),
                                  SLAB_PANIC | SLAB_NOTRACK, NULL);
        setup_xstate_comp();
}
```

`arch_task_cache_init`는 아키텍처 특정 캐시의 초기화를 수행합니다. 우리의 경우에는 `x86_64`이므로 `arch_task_cache_init`는 [FPU](http://en.wikipedia.org/wiki/Floating-point_unit) 상태를 나타내는 `task_xstate`에 캐시를 할당하는 것을 알 수 있습니다. `setup_xstate_comp` 함수를 호출하여 [xsave](http://www.felixcloutier.com/x86/XSAVES.html) 영역에서 모든 확장 상태의 오프셋과 크기를 설정합니다. `arch_task_cache_init` 이후에 다음을 사용하여 기본 최대 스레드 수를 계산합니다:

```C
set_max_threads(MAX_THREADS);
```

여기서 기본 최대 스레드 수는 다음과 같습니다:

```C
#define FUTEX_TID_MASK  0x3fffffff
#define MAX_THREADS     FUTEX_TID_MASK
```

`fork_init` 함수의 끝에서 [signal](http://www.win.tue.nl/~aeb/linux/lk/lk-5.html) 핸들러를 초기화합니다:

```C
init_task.signal->rlim[RLIMIT_NPROC].rlim_cur = max_threads/2;
init_task.signal->rlim[RLIMIT_NPROC].rlim_max = max_threads/2;
init_task.signal->rlim[RLIMIT_SIGPENDING] =
		init_task.signal->rlim[RLIMIT_NPROC];
```

우리가 알다시피 `init_task`는 `task_struct` 구조체의 인스턴스이므로 시그널 핸들러를 나타내는 `signal` 필드를 포함합니다. `struct signal_struct` 유형이 있습니다. 처음 두 줄에서 `리소스 제한`의 현재 및 최대 제한 설정을 볼 수 있습니다. 모든 프로세스에는 관련 리소스 제한이 있습니다. 이러한 제한은 현재 프로세스가 사용 할 수 있는 리소스 양을 지정합니다. 여기서 `rlimit`은 리소스 컨트롤 제한이며 다음과 같습니다:

```C
struct rlimit {
        __kernel_ulong_t        rlim_cur;
        __kernel_ulong_t        rlim_max;
};
```

[include/uapi/linux/resource.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/include/uapi/linux/resource.h)의 구조체. 우리의 경우 리소스는 사용자가 소유 할 수 있는 최대 프로세스 수인 `RLIMIT_NPROC`이고 대기중인 신호의 최대 수인`RLIMIT_SIGPENDING`입니다. 여기서 볼 수 있습니다:

```C
cat /proc/self/limits
Limit                     Soft Limit           Hard Limit           Units     
...
...
...
Max processes             63815                63815                processes
Max pending signals       63815                63815                signals   
...
...
...
```

캐시 초기화
--------------------------------------------------------------------------------

`fork_init`이후의 다음 함수는 [kernel/fork.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/kernel/fork.c)의 `proc_caches_init`입니다. 이 함수는 메모리 디스크립터 (또는`mm_struct` 구조체)에 캐시를 할당합니다. `proc_caches_init`의 시작 부분에서 우리는 `kmem_cache_create`를 호출하여 다른 [SLAB](http://en.wikipedia.org/wiki/Slab_allocation) 캐시의 할당을 볼 수 있습니다:

* `sighand_cachep` - 설치된 신호 처리기에 대한 정보를 관리;
* `signal_cachep` - 프로세스 신호 디스크립터에 대한 정보 관리;
* `files_cachep` - 열린 파일에 대한 정보 관리;
* `fs_cachep` - 파일 시스템 정보 관리.

그런 다음 우리는 `mm_struct` 구조체를 위해 `SLAB` 캐시를 할당합니다:

```C
mm_cachep = kmem_cache_create("mm_struct",
                         sizeof(struct mm_struct), ARCH_MIN_MMSTRUCT_ALIGN,
                         SLAB_HWCACHE_ALIGN|SLAB_PANIC|SLAB_NOTRACK, NULL);
```


그런 다음 커널이 가상 메모리 공간을 관리하는 데 사용하는 중요한 `vm_area_struct`에 대해 SLAB 캐시를 할당합니다:

```C
vm_area_cachep = KMEM_CACHE(vm_area_struct, SLAB_PANIC);
```

여기서는 `kmem_cache_create` 대신 `KMEM_CACHE` 매크로를 사용합니다. 이 매크로는 [include/linux/slab.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/include/linux/slab.h)에 정의되어 있으며 `kmem_cache_create` 호출로 확장됩니다:

```C
#define KMEM_CACHE(__struct, __flags) kmem_cache_create(#__struct,\
                sizeof(struct __struct), __alignof__(struct __struct),\
                (__flags), NULL)
```

`KMEM_CACHE`는 `kmem_cache_create`와 하나의 차이점이 있습니다. `__alignof__` 연산자를 살펴보십시오. `KMEM_CACHE` 매크로는 `SLAB`을 주어진 구조체의 크기에 맞추지만 `kmem_cache_create`는 주어진 값을 사용하여 공간을 정렬합니다. 그 후에 우리는 `mmap_init`와 `nsproxy_cache_init` 함수의 호출을 볼 수 있습니다. 첫 번째 함수는 가상 메모리 영역 `SLAB`을 초기화하고 두 번째 함수는 네임 스페이스에 대해 `SLAB`을 초기화합니다.

`proc_caches_init` 다음에 오는 함수는 `buffer_init`입니다. 이 함수는 [fs/buffer.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/fs/buffer.c) 소스 코드 파일에 정의되어 있으며 `buffer_head`에 캐시를 할당합니다. `buffer_head`는 [include/linux/buffer_head.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/include/linux/buffer_head.h)에 정의되어 있으며 버퍼 관리에 사용되는 특수 구조입니다. `buffer_init` 함수의 시작에서 우리는 이전 함수에서 했던 것처럼 `kmem_cache_create` 함수를 호출하여 `struct buffer_head` 구조체에 캐시를 할당합니다. 그리고 다음을 사용하여 메모리에서 버퍼의 최대 크기를 계산하세요:

```C
nrpages = (nr_free_buffer_pages() * 10) / 100;
max_buffer_heads = nrpages * (PAGE_SIZE / sizeof(struct buffer_head));
```

이는 `ZONE_NORMAL`의 `10 %`와 같습니다(x86_64의 4GB의 모든 RAM). `buffer_init` 다음에 오는 함수는 `vfs_caches_init`입니다. 이 함수는 다른 [VFS](http://en.wikipedia.org/wiki/Virtual_file_system) 캐시에 대해 `SLAB` 캐시와 해시 테이블을 할당합니다. 우리는 이미 리눅스 커널의 [초기화 과정](https://0xax.gitbooks.io/linux-insides/content/Initialization/linux-initialization-8.html) 8 번째 부분에서 `dcache`(또는 디렉토리 캐시) 와 [inode](http://en.wikipedia.org/wiki/Inode)캐시를 초기화하는 `vfs_caches_init_early` 함수를 보았습니다. `vfs_caches_init` 함수는 `dcache` 및 `inode` 캐시, 개인 데이터 캐시, 마운트 포인트에 대한 해시 테이블 등을 초기에 초기화합니다. [VFS](http://en.wikipedia.org/wiki/Virtual_file_system)에 대한 자세한 내용은 별도의 부분에서 설명합니다. 그 후에 우리는`signals_init` 함수를 볼 수 있습니다. 이 함수는 [kernel/signal.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/kernel/signal.c)에 정의되어 있으며 실시간 신호의 대기열을 나타내는 `sigqueue` 구조체에 대한 캐시를 할당합니다. 다음 함수는 `page_writeback_init`입니다. 이 기능은 더티 페이지의 비율을 초기화합니다. 모든 하위 레벨 페이지 항목에는 메모리에 로드된 후 페이지가 기록되었는지 여부를 나타내는 더티 `dirty` 비트가 포함됩니다.

procfs의 루트 생성
--------------------------------------------------------------------------------

이 모든 준비가 끝나면 [proc](http://en.wikipedia.org/wiki/Procfs) 파일 시스템의 루트를 만들어야합니다. [fs/proc/root.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/fs/proc/root.c)에서 `proc_root_init` 함수를 호출하면 됩니다. `proc_root_init` 함수의 시작에서 우리는 inode에 대한 캐시를 할당하고 다음과 같이 시스템에 새로운 파일 시스템을 등록합니다:

```C
err = register_filesystem(&proc_fs_type);
      if (err)
                return;
```

위에서 쓴 것처럼 [VFS](http://en.wikipedia.org/wiki/Virtual_file_system) 및 다른 파일 시스템에 대한 자세한 내용은 다루지 않지만 `VFS`에 대해서는 이 장에서 볼 것입니다. 시스템에 새 파일 시스템을 등록한 후 [fs /proc/self.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/fs/proc/self.c)에서 `proc_self_init` 함수를 호출하고 이 함수는`self(`/proc/self` 디렉토리는 `/proc` 파일 시스템에 액세스하는 프로세스를 나타냄)`에 `inode` 번호를 할당합니다. `proc_self_init` 다음 단계는 현재 스레드에 대한 정보를 포함하는 `/proc/thread-self` 디렉토리를 설정하는 `proc_setup_thread_self`입니다. 그후 우리는 다음 호출을 통해 마운트 포인트를 포함하는 `/proc/self/mounts` 심볼릭 링크를 생성합니다:

```C
proc_symlink("mounts", NULL, "self/mounts");
```

몇 개의 디렉토리는 다른 구성 옵션에 따라 다릅니다:

```C
#ifdef CONFIG_SYSVIPC
        proc_mkdir("sysvipc", NULL);
#endif
        proc_mkdir("fs", NULL);
        proc_mkdir("driver", NULL);
        proc_mkdir("fs/nfsd", NULL);
#if defined(CONFIG_SUN_OPENPROMFS) || defined(CONFIG_SUN_OPENPROMFS_MODULE)
        proc_mkdir("openprom", NULL);
#endif
        proc_mkdir("bus", NULL);
        ...
        ...
        ...
        if (!proc_mkdir("tty", NULL))
                 return;
        proc_mkdir("tty/ldisc", NULL);
        ...
        ...
        ...
```

`proc_root_init`의 끝에서 우리는 `/proc/sys` 디렉토리를 생성하고 [Sysctl](http://en.wikipedia.org/wiki/Sysctl)을 초기화하는 `proc_sys_init` 함수를 호출합니다.

`start_kernel` 함수의 끝입니다. `start_kernel`에서 호출되는 모든 함수를 설명하지는 않았습니다. 그것들은 일반적인 커널 초기화에 중요하지 않으며 다른 커널 구성만 따르기 때문에 생략했습니다. 작업 별 통계를 사용자 공간으로 내보내는 `taskstats_init_early`,`delayacct_init`- 작업 별 지연 계정 초기화,`key_init` 및`security_init` 다른 보안 항목 초기화, 'check_bugs'- 아키텍처에 따른 버그 수정, `ftrace_init` 함수는 [ftrace](https://www.kernel.org/doc/Documentation/trace/ftrace.txt)의 초기화를 실행,`cgroup_init`는 나머지 [cgroup](http://en.wikipedia.org/wiki/Cgroups) 하위 시스템 등의 초기화합니다. 더 많은 부분과 하위 시스템은 다른 장에서 설명됩니다.

그게 다입니다. 마지막으로 우리는 긴 `start_kernel` 함수를 통과했습니다. 그러나 리눅스 커널 초기화 프로세스의 끝은 아닙니다. 아직 첫 번째 프로세스를 실행하지 않았습니다. `start_kernel`의 끝에서 `rest_init` 함수의 마지막 호출을 볼 수 있습니다. 계속 갑시다.

start_kernel 이후 첫 단계
--------------------------------------------------------------------------------

`rest_init` 함수는 `start_kernel` 함수와 동일한 소스 코드 파일에 정의되어 있으며 이 파일은 [init/main.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/init/main.c)에 있습니다. `rest_init`의 시작 부분에서 우리는 다음 두 함수의 호출을 볼 수 있습니다:

```C
	rcu_scheduler_starting();
	smpboot_thread_init();
```

첫 번째 `rcu_scheduler_starting`은 [RCU](http://en.wikipedia.org/wiki/Read-copy-update) 스케줄러를 활성화하고 두 번째 `smpboot_thread_init`는 `smpboot_thread_notifier` CPU notifier를 등록합니다(자세한 내용은 [CPU hotplug documentation](https://www.kernel.org/doc/Documentation/cpu-hotplug.txt)에서 볼 수 있음). 이 후 다음과 같은 호출을 볼 수 있습니다:

```C
kernel_thread(kernel_init, NULL, CLONE_FS);
pid = kernel_thread(kthreadd, NULL, CLONE_FS | CLONE_FILES);
```

`kernel_thread` 함수([kernel/fork.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/kernel/fork.c)에 정의 된)는 새로운 커널 스레드를 생성합니다. `kernel_thread` 함수는 세 개의 인자를 취합니다:

* 새로운 스레드에서 실행될 함수;
* `kernel_init` 함수의 매개 변수;
* 플래그.

우리는 `kernel_thread` 구현에 대해 자세히 다루지 않을 것입니다 (스케줄러를 설명하는 장에서 그것을 볼 것입니다, `kernel_thread`가 [clone](http://www.tutorialspoint.com/unix_system_calls/clone.htm))을 호출한다고 보면 됩니다). 이제 우리는 `kernel_thread`함수를 사용하여 새로운 커널 스레드를 생성하고, 스레드의 부모와 자식은 파일 시스템에 대한 공유 정보를 사용하고 `kernel_init` 함수를 실행하기 시작한다는 것을 알아야 합니다. 커널 스레드는 커널 모드에서 실행되는 사용자 스레드와 다릅니다. 따라서 이 두 개의 `kernel_thread`호출로 `init`프로세스의 경우 `PID = 1`이고 `kthreadd`의 경우 `PID = 2`인 두 개의 새로운 커널 스레드를 만듭니다. 우리는 이미 `init`프로세스가 무엇인지 알고 있습니다. `kthreadd`를 봅시다. 커널의 다른 부분이 다른 커널 스레드를 작성하도록 관리하고 도와주는 특수 커널 스레드입니다. `ps` 유틸리티의 출력에서 볼 수 있습니다:

```C
$ ps -ef | grep kthreadd
root         2     0  0 Jan11 ?        00:00:00 [kthreadd]
```

이제 `kernel_init`와 `kthreadd`를 잠시 미루고 `rest_init`로 넘어 갑시다. 두 개의 새로운 커널 스레드를 생성 한 후 다음 단계에서 다음 코드를 볼 수 있습니다:

```C
	rcu_read_lock();
	kthreadd_task = find_task_by_pid_ns(pid, &init_pid_ns);
	rcu_read_unlock();
```

첫 번째 `rcu_read_lock` 함수는 [RCU](http://en.wikipedia.org/wiki/Read-copy-update)읽기 부분 중요 섹션의 시작을 나타내고 `rcu_read_unlock`은 RCU 읽기의 끝을 나타냅니다. `find_task_by_pid_ns`를 보호해야하기 때문에 이러한 함수를 호출합니다. `find_task_by_pid_ns`는 주어진 pid에 의해 `task_struct`에 대한 포인터를 리턴합니다. 그래서 여기에 우리는 `PID = 2`에 대한 `task_struct`에 대한 포인터를 얻습니다(우리는 `kernel_thread`로 `kthreadd`를 만든 후에 얻었습니다). 다음 단계에서 우리는 `complete` 함수를 호출합니다:

```C
complete(&kthreadd_done);
```

그리고 `kthreadd_done`의 주소를 전달합니다. `kthreadd_done`은 다음과 같이 정의됩니다:

```C
static __initdata DECLARE_COMPLETION(kthreadd_done);
```

`DECLARE_COMPLETION`가 다음에 정의되어있습니다:

```C
#define DECLARE_COMPLETION(work) \
         struct completion work = COMPLETION_INITIALIZER(work)
```

`completion` 구조체의 정의로 확장됩니다. 이 구조체는 [include/linux/completion.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/include/linux/completion.h)에 정의되어 있고 `completions`에 대한 개념이 있습니다. Completions는 일부 프로세스가 특정 지점이나 특정 상태에 도달 할 때까지 기다려야하는 스레드에 레이스 없는 솔루션을 제공하는 코드 동기화 메커니즘입니다. Completions을 사용하는 것은 세 부분으로 구성됩니다. 첫 번째는 `complete`구조체의 정의이며 `DECLARE_COMPLETION`으로 수행했습니다. 두 번째는 `wait_for_completion`의 호출입니다. 이 함수를 호출 한 후 호출 한 스레드는 계속 실행되지 않고 다른 스레드가 `complete`함수를 호출하지 않은 동안 대기합니다. `kernel_init_freeable`의 시작 부분에서 `kthreadd_done`으로 `wait_for_completion`을 호출합니다:

```C
wait_for_completion(&kthreadd_done);
```

그리고 마지막 단계는 위에서 본 것처럼 `complete`함수를 호출하는 것입니다. 그 후 `kernel_init_freeable` 기능은 실행되지 않지만 `kthreadd` 스레드는 설정되지 않습니다. `kthreadd`가 설정되면 `rest_init`에서 다음 세 가지 기능을 볼 수 있습니다:

```C
	init_idle_bootup_task(current);
	schedule_preempt_disabled();
    cpu_startup_entry(CPUHP_ONLINE);
```

[kernel/sched/core.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d93/kernel/sched/core.c)의 첫 번째 `init_idle_bootup_task`함수는 현재 프로세스의 스케줄링 클래스를 설정합니다(`idle`클래스):

```C
void init_idle_bootup_task(struct task_struct *idle)
{
         idle->sched_class = &idle_sched_class;
}
```

여기서 `idle`클래스는 낮은 작업 우선 순위이며 프로세서에 이 작업 외에 실행할 항목이 없는 경우에만 작업을 실행할 수 있습니다. 두 번째 함수 인 `schedule_preempt_disabled`는 `idle` 작업에서 선점을 비활성화합니다. 그리고 세 번째 함수 인 `cpu_startup_entry`는 [kernel/sched/idle.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/sched/idle.c)에 정의되어 있으며 [kernel/sched/idle.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/sched/idle.c)에서 `cpu_idle_loop`를 호출합니다. `cpu_idle_loop` 함수는 `PID = 0`의 프로세스로 작동하며 백그라운드에서 작동합니다. `cpu_idle_loop`의 주요 목적은 idle CPU 사이클을 소비하는 것입니다. 실행할 프로세스가 없으면 이 프로세스가 작동하기 시작합니다. 우리는 `idle` 스케줄링 클래스를 가진 하나의 프로세스를 가지고 있습니다(우리는 `init_idle_bootup_task` 함수를 호출하여 `current` 작업을 `idle`로 설정했습니다). 따라서 `idle` 스레드는 유용한 작업을 수행하지 않고 단지 다음으로 전환해야하는 작업이 있는지 체크합니다:

```C
static void cpu_idle_loop(void)
{
        ...
        ...
        ...
        while (1) {
                while (!need_resched()) {
                ...
                ...
                ...
                }
        ...
        }
```

이에 대한 자세한 내용은 스케줄러에 관한 장에 있습니다. 따라서 이 시점에서 `start_kernel`은 `rest_init` 함수를 호출하여 `init`(`kernel_init` 함수)프로세스를 생성하고 `idle` 프로세스 자체가 됩니다. 이제 `kernel_init`를 볼 차례입니다. `kernel_init` 함수의 실행은 `kernel_init_freeable` 함수의 호출에서 시작됩니다. `kernel_init_freeable` 함수는 먼저 `kthreadd`설정이 완료되기를 기다립니다. 이미 위에서 다루었습니다:

```C
wait_for_completion(&kthreadd_done);
```

그 후에 우리는 `gfp_allowed_mask`를 `__GFP_BITS_MASK`로 설정했습니다. 이것은 시스템이 이미 실행 중임을 의미하고, [cpus/mems](https://www.kernel.org/doc/Documentation/cgroups/cpusets.txt)를 `set_mems_allowed` 함수로 모든 CPU와 [NUMA](http://en.wikipedia.org/wiki/Non-uniform_memory_access) 노드에 설정, `set_cpus_allowed_ptr`로 CPU에서 `init` 프로세스가 실행되도록 허용, `cad` 또는 `Ctrl-Alt-Delete`는`smp_prepare_cpus`를 호출하여 다른 CPU를 부팅 할 준비를하고 `do_pre_smp_initcalls`로 일찍 [initcalls](http://kernelnewbies.org/Documents/InitcallMechanism)를 호출, `smp_init`로 `SMP`를 초기화하고 `lockup_detector_init`의 호출로 [lockup_detector](https://www.kernel.org/doc/Documentation/lockup-watchdogs.txt)를 초기화하고 `sched_init_smp`로 스케줄러를 초기화합니다.

그 후에 우리는 `do_basic_setup` 함수의 호출을 볼 수 있습니다. `do_basic_setup` 함수를 호출하기 전에 커널은 이미 이 시점에서 초기화되었습니다. 코멘트에 따르면:

```
이제 우리는 마침내 실제 작업을 시작할 수 있습니다..
```

`do_basic_setup`은 [cpuset](https://www.kernel.org/doc/Documentation/cgroups/cpusets.txt)를 활성 CPU로 다시 초기화하고 커널 내에서 사용자 공간을 호출하는데 사용되는 커널 스레드인 `khelper`를 초기화합니다. [tmpfs](http://en.wikipedia.org/wiki/Tmpfs)를 초기화하고, `drivers` 서브 시스템을 초기화하고, 사용자 모드 헬퍼 `workqueue`를 활성화하고 `initcalls`를 초기에 호출합니다. `do_basic_setup` 이후 `dev/console`이 열리고 `0`에서 `2`로 두 번 파일 디스크립터가 두 배로 증가하는 것을 볼 수 있습니다:

```C
if (sys_open((const char __user *) "/dev/console", O_RDWR, 0) < 0)
	pr_err("Warning: unable to open an initial console.\n");

(void) sys_dup(0);
(void) sys_dup(0);
```

우리는 `sys_open`과 `sys_dup`의 두 가지 시스템 호출을 사용하고 있습니다. 다음 장에서는 다른 시스템 호출에 대한 설명과 구현을 볼 수 있습니다. 초기 콘솔을 연 후,`rdinit =` 옵션이 커널 명령 행으로 전달되었는지 확인하거나 램 디스크의 기본 경로를 설정합니다:

```C
if (!ramdisk_execute_command)
	ramdisk_execute_command = "/init";
```

`ramdisk`에 대한 사용자의 권한을 확인하고 [init/do_mounts.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/init/do_mounts.c)에서 `prepare_namespace` 함수를 호출하십시오. [initrd](http://en.wikipedia.org/wiki/Initrd)를 마운트합니다:

```C
if (sys_access((const char __user *) ramdisk_execute_command, 0) != 0) {
	ramdisk_execute_command = NULL;
	prepare_namespace();
}
```

이것은 `kernel_init_freeable` 함수의 끝이며 `kernel_init`로 돌아 가야합니다. `kernel_init_freeable`이 실행을 마치면 다음 단계는 `async_synchronize_full`입니다. 이 함수는 모든 비동기 함수 호출이 완료 될 때까지 기다린 다음 `free_initmem`을 호출하여 `__init_begin`과 `__init_end` 사이에있는 초기화 항목이 차지하는 모든 메모리를 해제합니다. 그런 다음 `.rodata`를 `mark_rodata_ro`로 보호하고 시스템의 상태를 `SYSTEM_BOOTING`에서 다음으로 업데이트 합니다:

```C
system_state = SYSTEM_RUNNING;
```

그리고 `init` 프로세스를 실행하려고 시도합니다:

```C
if (ramdisk_execute_command) {
	ret = run_init_process(ramdisk_execute_command);
	if (!ret)
		return 0;
	pr_err("Failed to execute %s (error %d)\n",
	       ramdisk_execute_command, ret);
}
```

먼저 `kernel_init_freeable` 함수에서 설정 한 `ramdisk_execute_command`를 검사하고 `rdinit =`커널 명령 행 매개 변수의 값 또는 기본적으로 `/ init`와 같습니다. `run_init_process` 함수는 `argv_init` 배열의 첫 번째 요소를 채웁니다:

```C
static const char *argv_init[MAX_INIT_ARGS+2] = { "init", NULL, };
```

`init` 프로그램의 인자를 나타내고 `do_execve` 함수를 호출합니다:

```C
argv_init[0] = init_filename;
return do_execve(getname_kernel(init_filename),
	(const char __user *const __user *)argv_init,
	(const char __user *const __user *)envp_init);
```

`do_execve` 함수는 [include/linux/sched.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/include/linux/sched.h)에 정의되어 있으며 주어진 파일 이름과 인자로 프로그램을 실행합니다. 커널 명령 행에 `rdinit =`옵션을 전달하지 않으면, 커널은 `init =`커널 명령 행 매개 변수의 값과 동일한 `execute_command `를 검사하기 시작합니다.

```C
	if (execute_command) {
		ret = run_init_process(execute_command);
		if (!ret)
			return 0;
		panic("Requested init %s failed (error %d).",
		      execute_command, ret);
	}
```

우리가 `init =`커널 명령 행 매개 변수를 전달하지 않으면, 커널은 다음 실행 파일 중 하나를 실행하려고 시도합니다:

```C
if (!try_to_run_init_process("/sbin/init") ||
    !try_to_run_init_process("/etc/init") ||
    !try_to_run_init_process("/bin/init") ||
    !try_to_run_init_process("/bin/sh"))
	return 0;
```

그렇지 않으면 [panic](http://en.wikipedia.org/wiki/Kernel_panic)으로 끝납니다:

```C
panic("No working init found.  Try passing init= option to kernel. "
      "See Linux Documentation/init.txt for guidance.");
```

그게 다 입니다! 리눅스 커널 초기화 과정이 끝났습니다!

결론
--------------------------------------------------------------------------------

리눅스 커널 [초기화 과정](https://0xax.gitbooks.io/linux-insides/content/Initialization/index.html)에 대한 10 번째 부분이 끝났습니다. 10 번째 부분이자 리눅스 커널의 초기화를 설명하는 마지막 부분이기도합니다. 이 장의 [첫 번째 부분](https://0xax.gitbooks.io/linux-insides/content/Initialization/linux-initialization-1.html)에서 커널 초기화의 모든 단계를 거치게 된다고 썼 듯이 우리는 해냈습니다. 우리는 첫 번째 아키텍처 독립적 기능인 `start_kernel` 에서 시작했고 첫 번째 `init` 프로세스를 시작했습니다. 스케줄러, 인터럽트, 예외 처리 등을 다루지 않은 커널의 다른 하위 시스템에 대한 세부 정보를 건너 뛰었습니다. 다음 부분에서 다른 커널 하위 시스템으로 시작합니다. 재미있기를 바랍니다.

질문이나 제안이 있으면 댓글을 남기거나 [트위터](https://twitter.com/0xAX)로 알려주세요.

**모국어가 영어가 아니면 죄송합니다. 실수를 발견하면 PR을 [linux-insides](https://github.com/0xAX/linux-internals)로 보내주십시오.**

링크
--------------------------------------------------------------------------------

* [SLAB](http://en.wikipedia.org/wiki/Slab_allocation)
* [xsave](http://www.felixcloutier.com/x86/XSAVES.html)
* [FPU](http://en.wikipedia.org/wiki/Floating-point_unit)
* [Documentation/security/credentials.txt](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/Documentation/security/credentials.txt)
* [Documentation/x86/x86_64/mm](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/Documentation/x86/x86_64/mm.txt)
* [RCU](http://en.wikipedia.org/wiki/Read-copy-update)
* [VFS](http://en.wikipedia.org/wiki/Virtual_file_system)
* [inode](http://en.wikipedia.org/wiki/Inode)
* [proc](http://en.wikipedia.org/wiki/Procfs)
* [man proc](http://linux.die.net/man/5/proc)
* [Sysctl](http://en.wikipedia.org/wiki/Sysctl)
* [ftrace](https://www.kernel.org/doc/Documentation/trace/ftrace.txt)
* [cgroup](http://en.wikipedia.org/wiki/Cgroups)
* [CPU hotplug documentation](https://www.kernel.org/doc/Documentation/cpu-hotplug.txt)
* [completions - wait for completion handling](https://www.kernel.org/doc/Documentation/scheduler/completion.txt)
* [NUMA](http://en.wikipedia.org/wiki/Non-uniform_memory_access)
* [cpus/mems](https://www.kernel.org/doc/Documentation/cgroups/cpusets.txt)
* [initcalls](http://kernelnewbies.org/Documents/InitcallMechanism)
* [Tmpfs](http://en.wikipedia.org/wiki/Tmpfs)
* [initrd](http://en.wikipedia.org/wiki/Initrd)
* [panic](http://en.wikipedia.org/wiki/Kernel_panic)
* [Previous part](https://0xax.gitbooks.io/linux-insides/content/Initialization/linux-initialization-9.html)
