리눅스 커널의 시스템 호출. Part 4.
================================================================================

리눅스 커널이 프로그램을 실행하는 방법
--------------------------------------------------------------------------------

Linux 커널에서 [시스템 호출](https://en.wikipedia.org/wiki/System_call)을 설명하는 [챕터](https://0xax.gitbooks.io/linux-insides/content/SysCall/index.html)의 네 번째 부분이며, [이전 파트](https://0xax.gitbooks.io/linux-insides/content/SysCall/linux-syscall-3.html)의 결론에서 이야기한 대로 이번 파트는 이번챕터의 마지막입니다. 이전 파트에서는 다음과 같은 두 가지 새로운 개념에서 파트를 마쳤었습니다:

* `vsyscall`;
* `vDSO`;

이들은 시스템 호출 개념과 관련이 있고 또한 매우 유사합니다.

이 파트는 이 장의 마지막 부분으로, 파트 제목에서 알 수 있듯이 프로그램을 실행할 때 리눅스 커널에서 어떤 일이 발생하는지 살펴보겠습니다. 자, 그럼 시작해봅시다.

프로그램은 어떻게 시작될까요?
--------------------------------------------------------------------------------

사용자 관점에서 보면 애플리케이션을 시작하는 방법은 매우 다양합니다. 예를 들어 [셸](https://en.wikipedia.org/wiki/Unix_shell)에서 프로그램을 실행하거나 애플리케이션 아이콘을 두 번 클릭할 수 있습니다. 이건 중요하지 않습니다. 리눅스 커널은 우리가 응용 프로그램을 어떻게 시작하던 상관없이 응용 프로그램 실행을 처리합니다. 이 파트에서는 셸에서 애플리케이션을 시작하는 방법을 사용했을 때로 고려하겠습니다. 아시다시피 셸에서 애플리케이션을 시작하는 표준 방법은 다음과 같습니다: 우리가 [terminal emulator](https://en.wikipedia.org/wiki/Terminal_emulator) 응용 프로그램을 시작하고 그저 프로그램 이름을 쓰고 프로그램에 인수를 전달하거나 전달하지 않거나 합니다. 예를 들면 이렇습니다:

![ls shell](http://i66.tinypic.com/214w6so.jpg)

셸에서 애플리케이션을 시작할 때 발생하는 일, 셸이 프로그램 이름을 쓸 때 셸이 수행하는 작업, 리눅스 커널이 수행하는 작업 등을 살펴봅시다. 하지만 우리가 이 흥미로운 것들을 고려하기 전에, 저는 이 책이 리눅스 커널에 관한 것이라는 것을 환기하자 합니다. 즉, 우리는 이 파트에서 Linux kernel insides와 관련된 것들 위주로만 살펴볼 것이라는 겁니다. 우리는 쉘이 무엇을 하는지에 대해서는 자세히 고려하지 않을 것이며 서브셸(subshells)과 같이 복잡한 경우들을 고려하지 않을 것입니다.

제 기본 셸은 [bash](https://en.wikipedia.org/wiki/Bash_%28Unix_shell%29)입니다. 따라서 저는 bash 셸이 프로그램을 시작하는 방법을 고려할 것입니다. 그럼 시작해봅시다.  [C](https://en.wikipedia.org/wiki/C_%28programming_language%29) 프로그래밍 언어로 작성된 모든 프로그램은 [main](https://en.wikipedia.org/wiki/Entry_point) 함수에서 시작됩니다. `bash` 셸의 소스 코드를 살펴보면, [shell.c](https://github.com/bminor/bash/blob/bc00779f0e1362100375d952de4c62fb/shell.c#L357) 소스 코드 파일에 `main` 함수가 있습니다. 이 함수는 `bash`의 메인 스레드 루프가 작동하기 전에 많은 것들을 합니다. 예를 들어 이 함수는:

* `/dev/ty`를 확인하고 열기를 시도합니다.
- 디버그 모드에서 셸이 실행 중인 지 확인합니다.
- 명령줄 인수를 구문 분석(parse)합니다.
- 쉘 환경을 읽습니다.
- `.bashrc`, `.profile` 및 기타 구성 파일을 로드합니다.
- 그 외에도 많은 일들을 합니다.

이러한 모든 작업 후에 `reader_loop` 함수의 호출을 볼 수 있습니다. 이 함수는 [eval.c](https://gitub.com/bminor/bash/blob/bc007799f0e1362100375d952de4c62fb/eval.c#L67) 소스 코드 파일에 정의되어 있고 메인 스레드 루프를 나타내며 혹은 다른 말로 하자면 명령을 읽고 실행합니다. `reader_loop` 함수는 모든 확인을 하고 주어진 프로그램 이름과 인수를 읽으면서 [execute_cmd.c](https://github.com/bminor/bash/blob/bc007799f0e1362100375bb95d952d28de4c62fb/execute_cmd.c#L378) 소스 코드 파일에서 `execute_command` 함수를 호출합니다. 함수 호출의 체인에서 `execute_command` 함수는 다음과 같이:

```
execute_command
--> execute_command_internal
----> execute_simple_command
------> execute_disk_command
--------> shell_execve
```

`subshell`을 시작해야 하는지, 이것이 내장된 `bash` 함수(기능 or 명령어)는 아닌지 등 여러 가지 점검을 합니다. 위에서 이미 언급한 바와 같이 리눅스 커널과 관련이 없는 사항에 대한 모든 세부 사항은 고려하지 않을 것입니다. 이 프로세스를 마치면 `shell_execve` 함수는 `execve` 시스템 호출을 호출합니다:

```C
execve (command, args, env);
```

`execve` 시스템 호출은 다음과 같은 모습을 가지고:

```
int execve(const char *filename, char *const argv [], char *const envp[]);
```

그리고 지정된 인수와 [환경 변수](https://en.wikipedia.org/wiki/Environment_variable)를 사용하여 지정된 파일 이름으로 프로그램을 실행합니다. 우리의 경우에서 보면 이 시스템 호출은 처음이자 마지막 호출입니다. 예를 들면 다음과 같습니다:

```
$ strace ls
execve("/bin/ls", ["ls"], [/* 62 vars */]) = 0

$ strace echo
execve("/bin/echo", ["echo"], [/* 62 vars */]) = 0

$ strace uname
execve("/bin/uname", ["uname"], [/* 62 vars */]) = 0
```

따라서 사용자 애플리케이션(이 경우엔 `bash`)은 시스템 호출을 호출하고 아시다시피 그 다음 단계는 리눅스 커널입니다.

시스템 호출 실행
--------------------------------------------------------------------------------

우리는 이 챕터의 두 번째 [파트](https://0xax.gitbooks.io/linux-insides/content/SysCall/linux-syscall-2.html)에서 사용자 응용 프로그램에서 호출한 시스템 호출 이전의 준비과정과 시스템 호출 처리기(handler)가 작업을 완료한 후를 보았습니다. 방금 전 단락에서는 `execve` 시스템 호출을 받은 상태에서 멈췄습니다. 이 시스템 호출은 [fs/exec.c](https://gitub.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d18e0bd98d973/fs/exec.c) 소스 코드 파일에 정의되어 있으며, 이미 아시겠지만 다음과 같이 세 가지 인수를 필요로 합니다.

```
SYSCALL_DEFINE3(execve,
		const char __user *, filename,
		const char __user *const __user *, argv,
		const char __user *const __user *, envp)
{
	return do_execve(getname(filename), argv, envp);
}
```

`do_execve` 함수의 결과만 반환하는 것을 보실 수 있듯이 여긴 꽤 간단합니다. 동일한 소스 코드 파일에 정의된 `do_execve` 함수는 다음 작업을 수행합니다.

* 주어진 인수 및 환경 변수를 사용하여 사용자 공간(userspace) 데이터의 두 포인터를 초기화합니다;
* `do_execveat_common`의 결과를 반환합니다.

함수의 구현은 다음과 같습니다.

```C
struct user_arg_ptr argv = { .ptr.native = __argv };
struct user_arg_ptr envp = { .ptr.native = __envp };
return do_execveat_common(AT_FDCWD, filename, argv, envp, 0);
```

`do_execveat_common` 함수가 실행하는 주 작업은 새 프로그램을 실행하는 것입니다. 이 함수도 유사한 매개변수들을 사용하지만 보시다시피 3개 대신 5개의 인수가 필요합니다. 첫 번째 인수는 우리의 응용프로그램이 있는 디렉토리를 나타내는 파일 디스크립터입니다. 우리의 경우 `AT_FDCWD`는 지정된 경로명이 호출 프로세스의 현재 작업 디렉토리에 대해 해석된다는 것을 의미합니다. 다섯 번째 인자는 플래그입니다. 우리의 경우에는 `do_execveat_common`으로 0을 넘겼습니다. 이에 대해 자세한 것은 다음 단계에서 알아볼 예정이므로 지금은 넘어가겠습니다.

먼저 `do_execveat_common` 함수는 `filename` 포인터를 확인하고 `NULL`일 경우 리턴합니다. 이 후 실행 중인 프로세스의 한도가 다음을 초과하지 않는지 현재 프로세스의 플래그를 확인합니다.

```C
if (IS_ERR(filename))
	return PTR_ERR(filename);

if ((current->flags & PF_NPROC_EXCEEDED) &&
	atomic_read(&current_user()->processes) > rlimit(RLIMIT_NPROC)) {
	retval = -EAGAIN;
	goto out_ret;
}

current->flags &= ~PF_NPROC_EXCEEDED;
```

이 두 가지 검사를 통과하면 `execve`가 실패하지 않도록 현재 프로세스의 플래그에 있는 `PF_NPROC_EXCEED` 플래그를 비활성화합니다. 그리고 보시다시피 다음 단계에서는 [kernel/fork.c](https://gitub.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d18e0b9d973/kernel/fork.c)에 정의되어 있는 `unshare_files` 함수를 호출하고 현재 작업 파일을 해제(unshare)하고, 이 함수의 결과를 체크합니다.

```C
retval = unshare_files(&displaced);
if (retval)
	goto out_ret;
```

우리는 실행된 바이너리(Execve'd binary)의 [file descriptor](https://en.wikipedia.org/wiki/File_descriptor)가 유출될 가능성을 제거하기 위해 이 함수를 호출해야 합니다. 다음 단계에서는 ([include/linux/binfmts.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/linux/binfmts.h)에 정의된) `struct linux_binprm` 구조체로 나타내어지는 `bprm`의 준비를 시작합니다. `linux_binprm` 구조체는 이진 파일(binary)을 로드할 때 사용되는 인수(arguments)를 유지하는 데 사용됩니다. 예를 들면 이 구조체는 `vma` 필드를 가지고 있는데, 이 필드는 `vm_area_struct` 타입을 가지고, 애플리케이션이 로드 될 주어진 주소 공간에서 연속 된 간격으로 나눠진 영역 중 하나을 나타냅니다. `mm` 필드는 바이너리의 메모리 디스크립터이며, 메모리 상단을 포인팅하고 그 외에도 기타 여러 필드가 있습니다.

가장 먼저 `kzalloc` 함수로 이 구조체에 메모리를 할당하고 할당 결과를 확인합니다.

```C
bprm = kzalloc(sizeof(*bprm), GFP_KERNEL);
if (!bprm)
	goto out_files;
```

이 후, `prepare_bprm_creds` 함수의 호출로 `binprm` 자격 증명(credential)의 준비를 시작합니다.

```C
retval = prepare_bprm_creds(bprm);
	if (retval)
		goto out_free;

check_unsafe_exec(bprm);
current->in_execve = 1;
```

`binprm` 자격 증명을 초기화하는 것은 다르게 말하자면 즉 `linux_binprm` 구조체 내부에 저장된 `cred` 구조체 초기화입니다. `cred` 구조체는 작업의 [실제 uid](https://en.wikipedia.org/wiki/User_identifier#Real_user_ID), 작업의 실제 [guid](https://en.wikipedia.org/wiki/Globally_unique_identifier), [가상 파일 시스템](https://en.wikipedia.org/wiki/Virtual_file_system)을 위한 `uid`와 `guid` 같은 작업의 보안 컨텍스트를 가지고 있습니다. 다음 단계에선 `bprm` 자격 증명을 준비하면서 `check_unsafe_exec` 함수의 호출로 프로그램을 안전하게 실행할 수 있는지 확인하고 현재 프로세스를 `in_execve` 상태로 설정합니다.

이러한 작업들이 모두 끝나면 `do_execveat_common` 함수에 전달된 플래그를 확인하는 `do_open_execat` 함수(`flags`에 `0`이 들어있음을 기억하세요)를 호출하며 디스크에서 실행가능한 파일을 검색하고 열고,  `noexec` 마운트 지점에서 이진 파일을 로드하는지 확인하고 ([proc](https://en.wikipedia.org/wiki/Procfs) 또는 [sysfs](https://en.wikipedia.org/wiki/Sysfs)과 같은 실행 이진 파일이 들어 있지 않은 파일 시스템에서의 이진 파일 실행을 방지해야 함), `file` 구조체를 초기화하고 이 구조체에 포인터를 반환합니다. 다음으로 `sched_exec`의 호출을 확인할 수 있습니다:

```C
file = do_open_execat(fd, filename, flags);
retval = PTR_ERR(file);
if (IS_ERR(file))
	goto out_unmark;

sched_exec();
```

`sched_exec` 함수는 새 프로그램을 실행할 수 있는 최소 부하 프로세서(least loaded processor)를 결정하고 현재 프로세스를 이 프로세서로 옮기는(migrate) 데 사용됩니다.

이 후에는 [file descriptor](https://en.wikipedia.org/wiki/File_descriptor)에서 실행 가능한 이진 파일을 확인해야 합니다. 바이너리 파일의 이름이 `/` 기호로 시작하는지, 또는 지정된 바이너리 실행파일의 경로가 호출 프로세스의 현재 작업 디렉토리에 대해 해석되는지 (현재 작업 디렉토리에 존재하는지), 또는 다른 말로 파일 설명자가 `AT_FDCWD`인지(위 내용 참조) 확인합니다.

만약 이들 중 한가지의 확인이 성공적이면 binary parameter filename을 설정합니다.

```C
bprm->file = file;

if (fd == AT_FDCWD || filename->name[0] == '/') {
	bprm->filename = filename->name;
}
```

그렇지 않고 만약 파일 이름이 비어 있는 경우 binary parameter filename을 주어진 바이너리 실행파일의 파일명에 따라 `/dev/fd/%d` 또는 `/dev/fd/%d/%s`로 설정합니다. 이는 파일 설명자가 참조하는 파일을 실행한다는 의미입니다.

```C
} else {
	if (filename->name[0] == '\0')
		pathbuf = kasprintf(GFP_TEMPORARY, "/dev/fd/%d", fd);
	else
		pathbuf = kasprintf(GFP_TEMPORARY, "/dev/fd/%d/%s",
		                    fd, filename->name);
	if (!pathbuf) {
		retval = -ENOMEM;
		goto out_unmark;
	}

	bprm->filename = pathbuf;
}

bprm->interp = bprm->filename;
```

`bprm->filename` 뿐만 아니라 프로그램 인터프리터의 이름이 포함된 `bprm->interp`도 설정한 것에 주의하세요. 지금은 거기에 그냥 같은 이름을 쓸 뿐이지만, 나중에 프로그램의 이진 형식(binary format)에 따라 프로그램 인터프리터의 실제 이름으로 업데이트될 것입니다. 위에서 `linux_binprm`의 `cred`를 준비한 것을 보실 수 있습니다. 다음 단계는 `linux_binprm`의 다른 필드들의 초기화입니다. 우선, `bprm_mm_init` 함수를 호출하고, `bprm`을 그것에 전달합니다.

```C
retval = bprm_mm_init(bprm);
if (retval)
	goto out_unmark;
```

동일한 소스 코드 파일에 정의된 `bprm_mm_init`는 함수 이름으로 알 수 있다시피, 메모리 디스크립터를 초기화하고, 다른 말로 하자면 `bprm_mm_init` 함수는 `mm_struct` 구조체를 초기화합니다. 이 구조체는 [include/linux/mm_types.h](https://gitub.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d18e0bd98eb93d973/include/mm_types.h) 헤더 파일에 정의되어 있어며 프로세스의 주소 공간을 나타냅니다. 리눅스 커널의 메모리 관리자와 관련된 중요한 것들은 우리가 잘 모르기 때문에 `bprm_mm_init` 함수의 구현을 살펴보지는 않겠지만, 이 함수가 `mm_struct`를 초기화하여 임시 스택 `vm_area_struct`로 채우는 것만 알면 됩니다.

이 다음에는 실행 가능한 바이너리로 전달된 명령줄 인수(command line arguments) 수와 환경 변수의 수를 계산하여 각각 `bprm->argc` 및 `bprm->envc`로 설정합니다.

```C
bprm->argc = count(argv, MAX_ARG_STRINGS);
if ((retval = bprm->argc) < 0)
	goto out;

bprm->envc = count(envp, MAX_ARG_STRINGS);
if ((retval = bprm->envc) < 0)
	goto out;
```

보시다시피 우리는 [같은](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/fs/exec.c) 소스 코드 파일에 정의된`count`함수의 도움으로 이 함수를 실행하고 `arvg` 배열의 문자열(string)의 수를 계산합니다. `MAX_ARG_STRINGS` 매크로는 [include/uapi/linux/binfmts.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/include/uapi/linux/binfmts.h) 헤더 파일에 정의되어 있으며 이름에서 알 수 있듯이 이 매크로는 `execve` 시스템 호출로 전해지는 문자열의 최대 갯수를 나타냅니다. `MAX_ARG_STRINGS`의 값은 다음과 같습니다:

```C
#define MAX_ARG_STRINGS 0x7FFFFFFF
```

명령줄 인수 및 환경변수의 수를 계산한 후에는 `prepare_binprm` 함수를 호출합니다. 우리는 이미 이전에 비슷한 이름을 가진 함수를 호출한 적이 있습니다. `prepare_binprm_cred`라는 함수이며, `linux_bprm`에서 `cred` 구조체를 초기화하는 함수로 기억하고 있습니다. 이제 `prepare_binprm` 함수가:

```C
retval = prepare_binprm(bprm);
if (retval < 0)
	goto out;
```

`linux_binprm` 구조체를 [inode](https://en.wikipedia.org/wiki/Inode)의 `uid`로 채우고 이진 실행 파일에서 `128`바이트를 읽습니다. 실행 파일에서 처음 `128`만 읽는 것은 실행 파일의 유형을 확인하기 위함입니다. 실행 파일의 나머지는 이후 단계에서 읽을 것입니다. `linux_bprm` 구조체를 준비한 후, `copy_strings_kernel` 함수를 호출하여 바이너리 이진 파일명, 명령줄 인수 및 환경 변수를 `linux_bprm`에 복사합니다.

```C
retval = copy_strings_kernel(1, &bprm->filename, bprm);
if (retval < 0)
	goto out;

retval = copy_strings(bprm->envc, envp, bprm);
if (retval < 0)
	goto out;

retval = copy_strings(bprm->argc, argv, bprm);
if (retval < 0)
	goto out;
```

그리고 `bprm_mm_init` 함수에서 설정한 새 프로그램 스택의 맨 위로 포인터를 설정합니다.

```C
bprm->exec = bprm->p;
```

스택의 최상단에는 프로그램 파일 이름이 있으며, 이 파일 이름은 `linux_bprm` 구조체의 `exec` 필드에 저장됩니다.

이제 `linux_bprm` 구조를 채웠고 `exec_binprm` 함수를 호출합니다.

```C
retval = exec_binprm(bprm);
if (retval < 0)
	goto out;
```

먼저, 현재 작업의 [namespace](https://en.wikipedia.org/wiki/Cgroups)에서 볼 수 있는 [pid](https://en.wikipedia.org/wiki/Process_identifier)과 `pid`를 `exec_binprm`에 저장합니다.

```C
old_pid = current->pid;
rcu_read_lock();
old_vpid = task_pid_nr_ns(current, task_active_pid_ns(current->parent));
rcu_read_unlock();
```

그리고 아래의:

```C
search_binary_handler(bprm);
```

함수를 호출합니다. 이 함수는 다른 이진 형식(binary format)을 포함하는 핸들러 목록을 살펴봅니다. 현재 리눅스 커널은 다음과 같은 이진 형식을 지원합니다.

* `binfmt_script` - [#!](https://en.wikipedia.org/wiki/Shang_%28Unix%29) 라인에서 시작하는 해석된 스크립트(interpreted scripts)를 지원;
* `binfmt_misc` - 리눅스 커널의 런타임 구성에 따라 다양한 바이너리 형식을 지원;
* `binfmt_elf` - [elf](https://en.wikipedia.org/wiki/Executable_and_Linkable_Format) 형식을 지원;
* `binfmt_aout` - [a.out](https://en.wikipedia.org/wiki/A.out) 형식을 지원;
* `binfmt_flat` - [flat](https://en.wikipedia.org/wiki/Binary_file#Structure) 형식을 지원;
* `binfmt_elf_fdpic` - [elf](https://en.wikipedia.org/wiki/Executable_and_Linkable_Format) [FDPIC](http://elinux.org/UClinux_Shared_Library#FDPIC_ELF) 바이너리에 대한 지원;
* `binfmt_em86` - [Alpha](https://en.wikipedia.org/wiki/DEC_Alpha) 머신에서 동작하는 Intel [elf](https://en.wikipedia.org/wiki/Executable_and_Linkable_Format) 바이너리를 지원합니다 .

그래서, `search_binary_handler`는 `load_binary` 함수를 호출하여 `linux_binprm`을 전달합니다. 만약 바이너리 핸들러가 지정된 실행 파일 형식을 지원하는 경우 실행을 위해 바이너리 실행 파일 준비를 시작합니다.

```C
int search_binary_handler(struct linux_binprm *bprm)
{
	...
	...
	...
	list_for_each_entry(fmt, &formats, lh) {
		retval = fmt->load_binary(bprm);
		if (retval < 0 && !bprm->mm) {
			force_sigsegv(SIGSEGV, current);
			return retval;
		}
	}

	return retval;
```

 여기서 `load_binary`는 예를들어 [elf](https://en.wikipedia.org/wiki/Executable_and_Linkable_Format)의 경우, `linux_bprm` 버퍼에 있는 매직 넘버(각 `elf` 이진 파일은 헤더에 매직 넘버가 포함되어 있음)를 확인하고 (실행 가능한 이진 파일에서 첫 `128` 바이트를 읽었음을 기억하십시오) `elf` 이진 파일이 아닐 경우 종료합니다.

```C
static int load_elf_binary(struct linux_binprm *bprm)
{
	...
	...
	...
	loc->elf_ex = *((struct elfhdr *)bprm->buf);

	if (memcmp(elf_ex.e_ident, ELFMAG, SELFMAG) != 0)
		goto out;
```

지정된 실행 파일이 `elf` 형식이면 `load_elf_binary`가 계속 실행됩니다. `load_elf_binary`는 실행 파일 실행 준비와 관련하여 여러 가지 작업을 수행합니다. 예를 들어 실행 파일의 아키텍처와 유형을 확인합니다:

```C
if (loc->elf_ex.e_type != ET_EXEC && loc->elf_ex.e_type != ET_DYN)
	goto out;
if (!elf_check_arch(&loc->elf_ex))
	goto out;
```

그리고 만약 잘못된 아키텍처가 있거나 실행 파일이 실행 불가능하거나 공유되지 않았으면 종료합니다. 프로그램 헤더 테이블 로드를 시도합니다:

```C
elf_phdata = load_elf_phdrs(&loc->elf_ex, bprm->file);
if (!elf_phdata)
	goto out;
```

프로그램 헤더 테이블은 [segments](https://en.wikipedia.org/wiki/Memory_segmentation)에 대해 기술합니다. `program interpreter`와 우리의 실행 이진 파일과 연결된 라이브러리를 디스크에서 읽어 메모리에 로드합니다. `program interpreter`는 실행 파일의 `.interp` 섹션에서 지정되어 있으며 [Linkers](https://0xax.gitbooks.io/linux-insides/content/Misc/linux-misc-3.html)를 설명하는 파트에서 볼 수 있듯이 이는 `x86_64`에서 `/lib64/ld-linux-x86-64.so.2`입니다. 이것은 스택을 설정하고 `elf` 이진 파일을 메모리의 올바른 위치에 매핑합니다. 이것은 [bs](https://en.wikipedia.org/wiki/.bss) 및 [brk](http://man7.org/linux/man-pages/man2/sbrk.2.html) 섹션을 매핑하고 실행할 실행 파일을 준비하기 위해 여러 가지 다른 작업을 수행합니다.

`load_elf_binary`를 실행한 후에는 `start_thread` 함수를 호출하며 다음 세 가지 인수를 전달합니다:

```C
	start_thread(regs, elf_entry, bprm->p);
	retval = 0;
out:
	kfree(loc);
out_ret:
	return retval;
```

이 인수들은:

* 새 작업에 대한 [레지스터](https://en.wikipedia.org/wiki/Processor_register)들의 집합;
* 새 작업의 엔트리 포인트 주소;
* 새 작업의 스택 최상단의 주소입니다.

함수 이름에서 보이듯이 새로운 스레드가 시작될 것 같지만 사실 그렇지는 않습니다. `start_thread'`함수는 새 작업의 레지스터를 실행할 준비가 되도록 준비하기만 합니다. 이 함수의 구현을 살펴봅시다:

```C
void
start_thread(struct pt_regs *regs, unsigned long new_ip, unsigned long new_sp)
{
        start_thread_common(regs, new_ip, new_sp,
                            __USER_CS, __USER_DS, 0);
}
```

보시다시피 `start_thread` 함수는 그저 `start_thread_common` 함수를 호출하기만 하며, 우릴 위해 다음과 같은 모든 기능을 수행하는 것은 이 함수입니다.

```C
static void
start_thread_common(struct pt_regs *regs, unsigned long new_ip,
                    unsigned long new_sp,
                    unsigned int _cs, unsigned int _ss, unsigned int _ds)
{
        loadsegment(fs, 0);
        loadsegment(es, _ds);
        loadsegment(ds, _ds);
        load_gs_index(0);
        regs->ip                = new_ip;
        regs->sp                = new_sp;
        regs->cs                = _cs;
        regs->ss                = _ss;
        regs->flags             = X86_EFLAGS_IF;
        force_iret();
}
```

`start_thread_common` 함수는 `fs` 세그먼트 레지스터를 zero, `es` 및 `ds`를 데이터 세그먼트 레지스터로 채웁니다. 이 후 [instruction pointer](https://en.wikipedia.org/wiki/Program_counter), `cs` 세그먼트 등에 새 값을 설정합니다. `start_thread_common` 함수의 끝에서 `iret` 명령을 통해 시스템 호출 리턴을 강제하는 `force_iret` 매크로를 볼 수 있습니다. 좋습니다. 사용자 공간에서 실행할 새 스레드를 준비했고 이제 `exec_binprm`에서 다시 `do_execveat_common`으로 돌아갈 수 있습니다. `exec_binprm`이 실행을 마치면 이전에 할당된 구조체에 대한 메모리를 할당 해제하고 리턴합니다.

`execve` 시스템 콜 핸들러에서 돌아오면, 프로그램 실행이 시작됩니다. 이렇게 할 수 있는 것은 모든 컨텍스트 관련 정보가 이미 이러한 목적으로 구성되어 있기 때문입니다. 보시다시피 `execve` 시스템 호출은 프로세스로 제어권을 넘기지 않지만, 코드, 데이터 및 기타 호출 프로세스의 세그먼트는 프로그램 세그먼트를 덮어씁니다. 우리의 응용 프로그램으로부터의 탈출은 `exit` 시스템 호출을 통해 구현됩니다.

이것으로 지금부터 우리의 프로그램이 실행될 것입니다.

결론
--------------------------------------------------------------------------------

이것으로 리눅스 커널의 시스템 호출 개념에 대한 네 번째 파트는 끝입니다. 우리는 4개 파트에서 시스템 호출(`system call`) 개념과 관련된 거의 모든 것들을 보았습니다. `system call` 개념의 이해에서 출발하여, 그것이 무엇인지, 그리고 왜 사용자 애플리케이션이 이 개념을 필요로 하는지를 배웠습니다. 다음으로 리눅스가 사용자 응용 프로그램의 시스템 호출을 어떻게 처리하는지 살펴봤습니다. `system call` 개념과 유사한 두 가지 개념 `vsyscall`과 `vDSO`를 만났고 마지막으로 리눅스 커널이 사용자 프로그램을 어떻게 실행하는지를 알게 되었습니다.

만약 질문이나 의견이 있으시다면, 트위터 [0xAX](https://twitter.com/0xAX)에서 저를 핑해주시거나, [email](anotherworldofworld@gmail.com)을 보내주시거나, 아니면 [issue](https://github.com/0xAX/linux-insides/issues/new)를 생성해주세요.

**영어는 제 모국어가 아닙니다, 그리고 여타 불편하셨던 점에 대해서 정말로 사과드립니다. 만약 실수들을 찾아내셨다면 부디 [linux-insides 원본](https://github.com/0xAX/linux-internals)으로, 번역에 대해서는 [linux-insides 한국 번역](https://github.com/junsooo/linux-insides-ko)로 PR을 보내주세요.**

참고 링크
--------------------------------------------------------------------------------

* [System call](https://en.wikipedia.org/wiki/System_call)
* [shell](https://en.wikipedia.org/wiki/Unix_shell)
* [bash](https://en.wikipedia.org/wiki/Bash_%28Unix_shell%29)
* [entry point](https://en.wikipedia.org/wiki/Entry_point)
* [C](https://en.wikipedia.org/wiki/C_%28programming_language%29)
* [environment variables](https://en.wikipedia.org/wiki/Environment_variable)
* [file descriptor](https://en.wikipedia.org/wiki/File_descriptor)
* [real uid](https://en.wikipedia.org/wiki/User_identifier#Real_user_ID)
* [virtual file system](https://en.wikipedia.org/wiki/Virtual_file_system)
* [procfs](https://en.wikipedia.org/wiki/Procfs)
* [sysfs](https://en.wikipedia.org/wiki/Sysfs)
* [inode](https://en.wikipedia.org/wiki/Inode)
* [pid](https://en.wikipedia.org/wiki/Process_identifier)
* [namespace](https://en.wikipedia.org/wiki/Cgroups)
* [#!](https://en.wikipedia.org/wiki/Shebang_%28Unix%29)
* [elf](https://en.wikipedia.org/wiki/Executable_and_Linkable_Format)
* [a.out](https://en.wikipedia.org/wiki/A.out)
* [flat](https://en.wikipedia.org/wiki/Binary_file#Structure)
* [Alpha](https://en.wikipedia.org/wiki/DEC_Alpha)
* [FDPIC](http://elinux.org/UClinux_Shared_Library#FDPIC_ELF)
* [segments](https://en.wikipedia.org/wiki/Memory_segmentation)
* [Linkers](https://0xax.gitbooks.io/linux-insides/content/Misc/linux-misc-3.html)
* [Processor register](https://en.wikipedia.org/wiki/Processor_register)
* [instruction pointer](https://en.wikipedia.org/wiki/Program_counter)
* [Previous part](https://0xax.gitbooks.io/linux-insides/content/SysCall/linux-syscall-3.html)
