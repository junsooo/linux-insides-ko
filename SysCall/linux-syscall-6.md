Linux의 리소스 제한
================================================================================

시스템의 각 프로세스는 파일, CPU 시간, 메모리 등과 같은 특정 양의 서로 다른 리소스를 사용합니다.

이러한 자원은 무한하지 않으며 각 프로세스마다 이를 관리 할 도구가 있어야 합니다. 때로는 특정 자원에 대한 현재 제한을 알고 있거나 값을 변경하는 것이 유용합니다. 이 게시물에서는 프로세스 한계에 대한 정보를 얻고 그러한 한계를 늘리거나 줄일 수있는 도구를 고려할 것입니다.

사용자 공간보기에서 시작하여 Linux 커널에서 어떻게 구현되는지 살펴 보겠습니다.

프로세스에 대한 리소스 제한을 관리하기위한 세 가지 기본 [시스템 호출](https://en.wikipedia.org/wiki/System_call)이 있습니다:

  * `getrlimit`
  * `setrlimit`
  * `prlimit`

처음 두 개는 프로세스가 시스템 리소스에 대한 제한을 읽고 설정할 수 있도록합니다. 마지막은 이전 기능의 확장입니다. `prlimit`는 [PID](https://en.wikipedia.org/wiki/Process_identifier)에 의해 지정된 프로세스의 리소스 제한을 설정하고 읽을 수 있게 합니다. 이 함수의 정의는:

`getrlimit`는:

```C
int getrlimit(int resource, struct rlimit *rlim);
```

`setrlimit`는:

```C
int setrlimit(int resource, const struct rlimit *rlim);
```

그리고 `prlimit` 정의는:

```C
int prlimit(pid_t pid, int resource, const struct rlimit *new_limit,
            struct rlimit *old_limit);
```

처음 두 경우에 함수는 두 가지 매개 변수를 사용합니다:

  * `resource` - 자원 유형을 나타냅니다(나중에 사용 가능한 유형이 표시됨);
  * `rlim` - `soft` 와 `hard` 한계의 조합.

한계에는 두 가지 유형이 있습니다:

  * `soft`
  * `hard`

첫 번째는 프로세스 자원에 대한 실제 제한을 제공합니다. 두 번째는 `soft`한계의 상한값이며 수퍼 유저만 설정할 수 있습니다. 따라서 `soft`제한은 관련된 `hard`제한을 초과 할 수 없습니다.

이 두 값은 `rlimit` 구조체로 결합됩니다:

```C
struct rlimit {
    rlim_t rlim_cur;
    rlim_t rlim_max;
};
```

마지막 함수는 약간 복잡해 보이며 `4`개의 매개변수를 가집니다. 매개변수 `리소스` 제외하면 다음과 같습니다:

  * `pid` - `prlimit`를 실행해야하는 프로세스의 ID 를 지정합니다.;
  * `new_limit` - `NULL`이 아닌 경우 새로운 한계 값을 제공;
  * `old_limit` - `NULL`이 아닌 경우 현재 `soft` 및 `hard` 한계가 여기에 배치됩니다.

`prlimit` 기능은 [ulimit](https://www.gnu.org/software/bash/manual/html_node/Bash-Builtins.html#index-ulimit) util에서 사용합니다. [strace](https://linux.die.net/man/1/strace) util의 도움으로 이를 확인할 수 있습니다.

예를 들어:

```
~$ strace ulimit -s 2>&1 | grep rl

prlimit64(0, RLIMIT_NPROC, NULL, {rlim_cur=63727, rlim_max=63727}) = 0
prlimit64(0, RLIMIT_NOFILE, NULL, {rlim_cur=1024, rlim_max=4*1024}) = 0
prlimit64(0, RLIMIT_STACK, NULL, {rlim_cur=8192*1024, rlim_max=RLIM64_INFINITY}) = 0
```

여기에서 `prlimit64`는 볼 수 있지만 `prlimit`는 볼 수 없습니다. 사실 우리는 라이브러리 호출 대신 기본 시스템 호출을 볼 수 있습니다.

이제 사용 가능한 리소스 목록을 살펴 보겠습니다:

| Resource          | Description
|-------------------|------------------------------------------------------------------------------------------|
| RLIMIT_CPU        | 초 단위의 CPU 시간 제한                                                          |
| RLIMIT_FSIZE      | 프로세스가 생성 할 수있는 최대 파일 크기                                      |
| RLIMIT_DATA       | 프로세스 데이터 세그먼트의 최대 크기                                        |
| RLIMIT_STACK      | 바이트 단위의 프로세스 스택의 최대 크기                                           |
| RLIMIT_CORE       | [core](http://man7.org/linux/man-pages/man5/core.5.html) 파일의 최대 크기.     |
| RLIMIT_RSS        | RAM에서 프로세스에 할당 할 수있는 바이트 수                           |
| RLIMIT_NPROC      | 사용자가 만들 수있는 최대 프로세스 수                            |
| RLIMIT_NOFILE     | 프로세스가 열 수있는 파일 디스크립터의 최대 수                  |
| RLIMIT_MEMLOCK    | [mlock](http://man7.org/linux/man-pages/man2/mlock.2.html)에 의해 RAM에 잠길 수있는 최대 메모리 바이트 수.|
| RLIMIT_AS         | 가상 메모리의 최대 크기 (바이트)                                             |
| RLIMIT_LOCKS      | 최대 수 [flock](https://linux.die.net/man/1/flock) 및 잠금 관련 [fcntl](http://man7.org/linux/man-pages/man2/fcntl.2.html) 호출|
| RLIMIT_SIGPENDING | 호출 프로세스의 사용자를 위해 대기 할 수있는 최대 [신호](http://man7.org/linux/man-pages/man7/signal.7.html) 수|
| RLIMIT_MSGQUEUE   | [POSIX 메시지 대기열](http://man7.org/linux/man-pages/man7/mq_overview.7.html)에 할당 할 수있는 바이트 수  |
| RLIMIT_NICE       | 프로세스에 의해 설정 될 수있는 최대 [nice](https://linux.die.net/man/1/nice) 값  |
| RLIMIT_RTPRIO     | 최대 실시간 우선 순위 값                                                         |
| RLIMIT_RTTIME     | 시스템 호출을 차단하지 않고 실시간 예약 정책에 따라 프로세스를 예약 할 수있는 최대 마이크로 초 수|

오픈 소스 프로젝트의 소스 코드를 살펴보면 리소스 제한을 읽거나 업데이트하는 작업이 널리 사용됩니다.

예를 들어: [systemd](https://github.com/systemd/systemd/blob/01a45898fce8def67d51332bccc410eb1e8710e7/src/core/main.c)

```C
/* Don't limit the coredump size */
(void) setrlimit(RLIMIT_CORE, &RLIMIT_MAKE_CONST(RLIM_INFINITY));
```

또는 [haproxy](https://github.com/haproxy/haproxy/blob/25f067ccec52f53b0248a05caceb7841a3cb99df/src/haproxy.c):

```C
getrlimit(RLIMIT_NOFILE, &limit);
if (limit.rlim_cur < global.maxsock) {
	Warning("[%s.main()] FD limit (%d) too low for maxconn=%d/maxsock=%d. Please raise 'ulimit-n' to %d or more to avoid any trouble.\n",
		argv[0], (int)limit.rlim_cur, global.maxconn, global.maxsock, global.maxsock);
}
```

우리는 사용자 공간에서 리소스 제한 관련 사항에 대해 조금 보았습니다. 이제 Linux 커널에서 동일한 시스템 호출을 살펴 보겠습니다.

Linux 커널의 리소스 제한
--------------------------------------------------------------------------------

`getrlimit` 시스템 호출과 `setrlimit`의 구현은 비슷합니다. 둘 다 `prlimit` 시스템 호출의 핵심 구현 인 `do_prlimit` 함수를 실행하고 주어진 `rlimit`에서 사용자 공간으로 복사합니다:

`getrlimit`:

```C
SYSCALL_DEFINE2(getrlimit, unsigned int, resource, struct rlimit __user *, rlim)
{
	struct rlimit value;
	int ret;

	ret = do_prlimit(current, resource, NULL, &value);
	if (!ret)
		ret = copy_to_user(rlim, &value, sizeof(*rlim)) ? -EFAULT : 0;

	return ret;
}
```

`setrlimit`:

```C
SYSCALL_DEFINE2(setrlimit, unsigned int, resource, struct rlimit __user *, rlim)
{
	struct rlimit new_rlim;

	if (copy_from_user(&new_rlim, rlim, sizeof(*rlim)))
		return -EFAULT;
	return do_prlimit(current, resource, &new_rlim, NULL);
}
```

이러한 시스템 호출의 구현은 [kernel/sys.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/kernel/sys.c) 커널 소스 코드 파일에 정의되어 있습니다.

우선 `do_prlimit` 함수는 주어진 리소스가 유효한지 확인합니다:

```C
if (resource >= RLIM_NLIMITS)
	return -EINVAL;
```

실패하면 `-EINVAL` 오류를 반환합니다. 이 검사가 성공적으로 통과하고 새 한계가 `NULL`이 아닌 값으로 전달 된 후 다음 두 가지 검사 합니다:

```C
if (new_rlim) {
	if (new_rlim->rlim_cur > new_rlim->rlim_max)
		return -EINVAL;
	if (resource == RLIMIT_NOFILE &&
			new_rlim->rlim_max > sysctl_nr_open)
		return -EPERM;
}
```

주어진 `소프트`한계가 `하드`한계를 초과하지 않는지와 주어진 자원이 하드 디스크 한계가 `sysctl_nr_open`값보다 크지 않은 파일 디스크립터의 최대 수인 경우 점검하십시오. `sysctl_nr_open`의 값은 [procfs](https://en.wikipedia.org/wiki/Procfs)를 통해 찾을 수 있습니다:

```
~$ cat /proc/sys/fs/nr_open
1048576
```

이 모든 검사 후에 우리는 주어진 리소스에 대한 한계를 업데이트하는 동안 [signal]() 핸들러와 관련된 것들이 파괴되지 않도록 `tasklist`를 잠급니다:

```C
read_lock(&tasklist_lock);
...
...
...
read_unlock(&tasklist_lock);
```

`prlimit`시스템 호출을 통해 주어진 pid에 의해 다른 작업의 한계를 업데이트 할 수 있기 때문에 이를 수행해야 합니다. 작업 목록이 잠기면 주어진 프로세스의 주어진 리소스 제한을 담당하는 `rlimit` 인스턴스를 가져옵니다:

```C
rlim = tsk->signal->rlim + resource;
```

여기서 `tsk-> signal-> rlim`은 특정 자원을 나타내는 `struct rlimit`의 배열입니다. 그리고 만약 `new_rlim`이 `NULL`이 아니면 우리는 그 값을 업데이트합니다. `old_rlim`이 `NULL`이 아닌 경우 채웁니다:

```C
if (old_rlim)
    *old_rlim = *rlim;
```

그게 답니다.

결론
--------------------------------------------------------------------------------

Linux 커널에서 시스템 호출의 구현을 설명하는 두 번째 부분의 끝입니다. 질문이나 제안 사항이 있으면 Twitter [0xAX](https://twitter.com/0xAX)에서 핑(Ping)하거나 [이메일](anotherworldofworld@gmail.com)을 보내거나 [이슈](https://github.com/0xAX/linux-internals/issues/new)를 만드세요.

**영어는 모국어가 아니면 죄송합니다. 실수를 발견하면 PR을 [linux-insides](https://github.com/0xAX/linux-internals)로 보내주십시오.**

링크들
--------------------------------------------------------------------------------

* [system calls](https://en.wikipedia.org/wiki/System_call)
* [PID](https://en.wikipedia.org/wiki/Process_identifier)
* [ulimit](https://www.gnu.org/software/bash/manual/html_node/Bash-Builtins.html#index-ulimit)
* [strace](https://linux.die.net/man/1/strace)
* [POSIX message queues](http://man7.org/linux/man-pages/man7/mq_overview.7.html)
