어떻게 `open` 시스템 콜 작업을 수행하나요?
--------------------------------------------------------------------------------

소개
--------------------------------------------------------------------------------

이것은 리눅스 커널의 [시스템 콜](https://en.wikipedia.org/wiki/System_call) 메커니즘을 설명하는 이 장의 다섯 번째 부분입니다. 이 장의 이전 부분에서는 이 메커니즘을 일반적으로 설명했습니다. 이제 Linux 커널에서 다른 시스템 호출의 구현을 설명하려고 합니다. 이 장의 이전 부분과 이 책의 다른 장의 부분은 사용자 공간에서 희미하게 보이거나 완전히 보이지 않는 Linux 커널의 대부분을 설명합니다. 그러나 리눅스 커널 코드는 그 자체가 아닙니다. 광대 한 리눅스 커널 코드는 우리 코드에 능력을 제공합니다. 리눅스 커널로 인해 우리 프로그램은 파일을 읽고 쓸 수 있으며 섹터, 트랙 및 디스크 구조의 다른 부분에 대해 전혀 알지 못하고도 네트워크를 통해 데이터를 보낼 수 있으며 수동으로 캡슐화 된 네트워크 패킷을 만들 지 않아도 됩니다.

당신의 경우 어떤지 모르겠지만 운영 체제가 작동하는 방법뿐만 아니라 소프트웨어가 어떻게 상호 작용하는지는 흥미롭습니다. 아시다시피, 우리의 프로그램은 [시스템 호출](https://en.wikipedia.org/wiki/System_call)이라는 특수 메커니즘을 통해 커널과 상호 작용합니다. 그래서 저는 `읽기`, `쓰기`, `열기`, `닫기`, `dup` 등과 같은 매일 우리가 사용하는 시스템 호출의 구현과 동작을 설명하는 일련의 부분을 작성하기로 결정했습니다 .

[open](http://man7.org/linux/man-pages/man2/open.2.html) 시스템 호출에 대한 설명부터 시작하기로 결정했습니다. 적어도 하나의 `C` 프로그램을 작성했다면 파일로 다른 조작을 읽거나 쓰거나 실행하기 전에 `open` 함수로 파일을 열어야한다는 것을 알아야합니다.

```C
#include <fcntl.h>
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <sys/stat.h>
#include <sys/types.h>

int main(int argc, char *argv) {
        int fd = open("test", O_RDONLY);

        if fd < 0 {
                perror("Opening of the file is failed\n");
        }
        else {
                printf("file sucessfully opened\n");
        }

        close(fd); 
        return 0;
}
```

이 경우 열기는 표준 라이브러리의 함수이지만 시스템 호출은 아닙니다. 표준 라이브러리는 우리에게 관련 시스템 호출을 호출합니다. `open` 호출은 프로세스 내에서 열린 파일과 관련된 고유 번호 인 [file descriptor](https://en.wikipedia.org/wiki/File_descriptor)를 반환합니다. 이제 `open` 호출의 결과로 파일을 열고 파일 설명자를 얻었으므로 이 파일과 상호 작용을 시작할 수 있습니다. 프로세스에 의해 열린 파일의 목록은 [proc](https://en.wikipedia.org/wiki/Procfs) 파일 시스템을 통해 이용할 수 있습니다 :

```
$ sudo ls /proc/1/fd/

0  10  12  14  16  2   21  23  25  27  29  30  32  34  36  38  4   41  43  45  47  49  50  53  55  58  6   61  63  67  8
1  11  13  15  19  20  22  24  26  28  3   31  33  35  37  39  40  42  44  46  48  5   51  54  57  59  60  62  65  7   9
```

이 게시물의 사용자 공간보기에서 `오픈` 루틴에 대한 자세한 내용은 설명하지 않지만 대부분 커널 측에서 설명합니다. 잘 모른다면 [man page](http://man7.org/linux/man-pages/man2/open.2.html)에서 더 많은 정보를 얻을 수 있습니다.

이제 시작하겠습니다.

오픈 시스템콜의 정의
--------------------------------------------------------------------------------

[linux-insides](https:0xax.gitbooks.io/linux-insides/content/index.html)의 [네 번째 파트](https://github.com/0xAX/linux-insides/blob/master/SysCall/linux-syscall-4.md)를 읽은 경우  책에서 시스템 호출은`SYSCALL_DEFINE` 매크로의 도움으로 정의된다는 것을 알아야합니다. 따라서 `open`시스템 콜도 예외는 아닙니다.

`open` 시스템 콜의 정의는 [fs/open.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/fs/open.c) 소스 코드 파일에 있으며 꽤 작게 보입니다. 

첫 번째보기 :

```C
SYSCALL_DEFINE3(open, const char __user *, filename, int, flags, umode_t, mode)
{
	if (force_o_largefile())
		flags |= O_LARGEFILE;

	return do_sys_open(AT_FDCWD, filename, flags, mode);
}
```

짐작 하시겠지만, [동일](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/fs/open.c) 소스 코드 파일의 `do_sys_open` 기능이 주요 작업을 수행합니다. 그러나 이 함수가 호출되기 전에, `open` 시스템 호출의 구현이 시작되는 `if` 절을 고려해 봅시다 :

```C
if (force_o_largefile())
	flags |= O_LARGEFILE;
```

여기서 우리는 `force_o_largefile()`이 true를 반환 할 경우에 `O_LARGEFILE` 플래그를 `open` 시스템 콜에 전달 된 플래그에 적용합니다.
`O_LARGEFILE`은 무엇입니까? 우리는 `open(2)` 시스템 콜에 대한 [man page](http://man7.org/linux/man-pages/man2/open.2.html)에서 이것을 읽을 수 있습니다 :

> O_LARGEFILE
>
> (LFS) 크기를 off_t로 표현할 수 없지만 off64_t로 표현할 수있는 파일을 열 수 있습니다.

[GNU C Library Reference Manual](https://www.gnu.org/software/libc/manual/html_mono/libc.html#File-Position-Primitive)에서 읽을 수있는 바와 같이 :

> off_t
>
> 파일 크기를 나타내는 데 사용되는 부호있는 정수 유형입니다.
> GNU C 라이브러리에서이 유형은 int보다 좁지 않습니다.
> 소스가 _FILE_OFFSET_BITS == 64로 컴파일 된 경우
> type은 투명하게 off64_t로 대체됩니다.

그리고

> off64_t
>
>이 유형은 off_t와 유사하게 사용됩니다. 차이점은
> off_t 유형이 32 비트 인 32 비트 시스템에서도
> off64_t는 64 비트이므로 최대 2 ^ 63 바이트의 파일을 처리 할 수 있습니다.
> 길이. _FILE_OFFSET_BITS == 64로 컴파일 할 때이 유형
>는 off_t라는 이름으로 제공됩니다.

따라서 `off_t`, `off64_t` 및 `O_LARGEFILE`이 대략 파일 크기라고 추측하기 어렵지 않습니다. 리눅스 커널의 경우, `O_LARGEFILE`은 호출자가 파일을 여는 동안 `O_LARGEFILE` 플래그를 지정하지 않은 경우 32 비트 시스템에서 큰 파일을 열 수 없도록하는 데 사용됩니다. 64 비트 시스템에서는 오픈 시스템 콜에서 이 플래그를 사용합니다. 그리고 [include/linux/fcntl.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/include/linux/fcntl.h#L7) 리눅스 커널 헤더 파일의 `force_o_largefile` 매크로는 이것을 확인합니다 :

```C
#ifndef force_o_largefile
#define force_o_largefile() (BITS_PER_LONG != 32)
#endif
```

이 매크로는 예를 들어 [IA-64](https://en.wikipedia.org/wiki/IA-64) 아키텍처와 같이 아키텍처에 따라 다를 수 있지만이 경우에는 [x86_64](https://en.wikipedia.org/wiki/X86-64)는 `force_o_largefile`의 정의를 제공하지 않으며 [include/linux/fcntl.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/include/linux/fcntl.h#L7)에서 사용됩니다..

따라서 우리가 알 수 있듯이 `force_o_largefile`은 [x86_64](https://en.wikipedia.org/wiki/X86-64) 아키텍처의 경우 `true` 값으로 확장되는 매크로 일뿐입니다. 64 비트 아키텍처를 고려할 때 `force_o_largefile`은 `true`로 확장되고 `O_LARGEFILE` 플래그는 `open` 시스템 콜에 전달 된 플래그 세트에 추가됩니다.

이제 우리는 `O_LARGEFILE` 플래그와 `force_o_largefile` 매크로의 의미를 고려 했으므로 `do_sys_open` 함수의 구현을 고려할 수 있습니다. 위에서 쓴 것처럼이 함수는 [동일](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/fs/open.c) 소스 코드 파일에 정의되어 있으며 다음과 같습니다.

```C
long do_sys_open(int dfd, const char __user *filename, int flags, umode_t mode)
{
	struct open_flags op;
	int fd = build_open_flags(flags, mode, &op);
	struct filename *tmp;

	if (fd)
		return fd;

	tmp = getname(filename);
	if (IS_ERR(tmp))
		return PTR_ERR(tmp);

	fd = get_unused_fd_flags(flags);
	if (fd >= 0) {
		struct file *f = do_filp_open(dfd, tmp, &op);
		if (IS_ERR(f)) {
			put_unused_fd(fd);
			fd = PTR_ERR(f);
		} else {
			fsnotify_open(f);
			fd_install(fd, f);
		}
	}
	putname(tmp);
	return fd;
}
```

이제 `do_sys_open`이 단계별로 작동하는 방식을 이해해봅시다

open(2) flags
--------------------------------------------------------------------------------

알다시피 `open` 시스템 호출은 파일 열기를 제어하는 두 번째 인수로`flags`를, 파일이 작성된 경우 파일의 권한을 지정하는 세 번째 인수로 mode를 사용합니다. `do_sys_open` 함수는`build_open_flags` 함수의 호출에서 시작하는데,이 함수는 주어진 플래그 세트가 유효한지 확인하고 플래그와 모드의 다른 조건을 처리합니다.

`build_open_flags`의 구현을 살펴 봅시다. 이 함수는 [동일](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/fs/open.c) 커널 파일에 정의되어 있으며 세 가지 인수를 사용합니다.

* flags - 파일 열기를 제어하는 플래그.
* mode - 새로 작성된 파일에 대한 권한

마지막 인수 인 `op`는 `open_flags` 구조체로 표현됩니다 :

```C
struct open_flags {
        int open_flag;
        umode_t mode;
        int acc_mode;
        int intent;
        int lookup_flags;
};
```

이는 [fs/internal.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/fs/internal.h#L99) 헤더 파일에 정의되어 있으며 플래그 및 내부 커널 목적을 위한 액세스 모입니다. 이미 알고 있듯이 `build_open_flags` 함수의 주요 목표는 이 구조체의 인스턴스를 채우는 것입니다.

`build_open_flags` 함수의 구현은 지역 변수의 정의에서 시작하며 그중 하나는 다음과 같습니다.

```C
int acc_mode = ACC_MODE(flags);
```

이 지역 변수는 액세스 모드를 나타내며 초기 값은 확장 된 `ACC_MODE` 매크로의 값과 같습니다. 이 매크로는 [include/linux/ fs.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/include/linux/fs.h)에 정의되어 있으며 매우 흥미로워 보입니다.

```C
#define ACC_MODE(x) ("\004\002\006\006"[(x)&O_ACCMODE])
#define O_ACCMODE   00000003
```

`"\004\002\006\006"`는 4 개의 문자로 구성된 배열입니다.

```
"\004\002\006\006" == {'\004', '\002', '\006', '\006'}
```

따라서, `ACC_MODE` 매크로는 `[(x) & O_ACCMODE]` 인덱스에 의해 이 배열에 대한 접근으로 확장됩니다. 방금 보았 듯이 `O_ACCMODE`는 `00000003`입니다. `x & O_ACCMODE`를 적용하면 `읽기`, `쓰기`또는 `읽기/쓰기` 액세스 모드를 나타내는 최하위 비트 2 개를 가져옵니다.

```C
#define O_RDONLY        00000000
#define O_WRONLY        00000001
#define O_RDWR          00000002
```

계산 된 인덱스에 의해 배열에서 값을 얻은 후, `ACC_MODE`는 `MAY_WRITE`, `MAY_READ`및 기타 정보를 보유 할 파일의 액세스 모드 마스크로 확장됩니다.

초기 액세스 모드를 계산 한 후 다음과 같은 상태가 나타날 수 있습니다.

```C
if (flags & (O_CREAT | __O_TMPFILE))
	op->mode = (mode & S_IALLUGO) | S_IFREG;
else
	op->mode = 0;
```

열린 파일이 임시 파일이 아니고 생성을 위해 열리지 않은 경우 `open_flags` 인스턴스의 권한을 재설정합니다. 이 때문입니다:

> O_CREAT 또는 O_TMPFILE을 지정하지 않으면 모드가 무시됩니다.

다른 경우에는`O_CREAT` 또는`O_TMPFILE`이 전달 된 경우 [opendir](http://man7.org/linux/man-pages/man3/opendir3.html) 시스템콜로 디렉토리를 작성해야하기 때문에 일반 파일로 정규화 할 수 있습니다.


다음 단계에서 우리는 [fanotify](http://man7.org/linux/man-pages/man7/fanotify.7.html)를 통해 그리고 `O_CLOEXEC` 플래그없이 파일을 열려고 시도하지 않았는지 확인합니다 :

```C
flags &= ~FMODE_NONOTIFY & ~O_CLOEXEC;
```

우리는 [파일 디스크립터](https://en.wikipedia.org/wiki/File_descriptor)를 누설하지 않기 위해 이렇게합니다. 기본적으로, 새로운 파일 디스크립터는 `execve` 시스템 호출에서 열린 채로 유지되도록 설정되어 있지만, `open` 시스템 콜은 이 기본 동작을 변경하는 데 사용할  수 있는 `O_CLOEXEC` 플래그를 지원합니다. 따라서 우리는 하나의 스레드가 파일을 열어 `O_CLOEXEC` 플래그를 설정하고 동시에 두 번째 프로세스가 [fork](https://en.wikipedia.org/wiki/Fork_\(system_call\)) + [execve](https://en.wikipedia.org/wiki/Exec_\(system_call\))를 수행할 때 파일 디스크립터의 누출을 방지하기 위해 이를 수행합니다. 그리고 자식은 부모의 열린 파일 디스크립터 세트의 사본을 갖게 될 것입니다.

다음 단계에서 플래그에 `O_SYNC` 플래그가 포함되어 있으면 `O_DSYNC` 플래그도 적용되는지 확인합니다.

```
if (flags & __O_SYNC)
	flags |= O_DSYNC;
```

`O_SYNC` 플래그는 모든 데이터가 디스크로 전송되기 전에 모든 쓰기 호출이 반환되지 않도록합니다. `O_DSYNC`는 메타 데이터 (예 :`atime` , `mtime` 등)가 변경 될 때까지 기다릴 필요가 없다는 점을 제외하면 `O_SYNC`와 같습니다. 우리는 `O_DSYNC`를 `__O_SYNC`의 경우 리눅스 커널에서`__O_SYNC | O_DSYNC`로 구현하기 때문에 적용합니다.

그런 다음 사용자가 임시 파일을 만들려면 플래그에 'O_TMPFILE_MASK'가 포함되어야하거나 다른 말로 `O_CREAT`또는 `O_TMPFILE`또는 둘 다 포함해야하며 쓰기 가능해야합니다.

```C
if (flags & __O_TMPFILE) {
	if ((flags & O_TMPFILE_MASK) != O_TMPFILE)
		return -EINVAL;
	if (!(acc_mode & MAY_WRITE))
		return -EINVAL;
} else if (flags & O_PATH) {
       	flags &= O_DIRECTORY | O_NOFOLLOW | O_PATH;
        acc_mode = 0;
}
```

매뉴얼 페이지에 기록 된대로 :

> O_TMPFILE은 O_RDWR 또는 O_WRONLY 중 하나로 지정해야합니다.

임시 파일 생성을 위해 `O_TMPFILE`을 전달하지 않으면 다음 조건에서 `O_PATH` 플래그를 확인합니다. `O_PATH` 플래그를 사용하면 다음 두 가지 목적으로 사용될 수있는 파일 디스크립터를 얻을 수 있습니다.

* 파일 시스템 트리의 위치를 나타냅니다.
* 파일 설명자 수준에서 순수하게 작동하는 작업을 수행합니다.

따라서이 경우 파일 자체는 열리지 않지만 `dup`, `fcntl` 등의 작업을 사용할 수 있습니다. 따라서 `read`, `write` 및 기타와 같은 모든 파일 내용 관련 작업이 허용되지 않으면 `O_DIRECTORY | O_NOFOLLOW | O_PATH` 플래그를 사용할 수 있습니다. 우리는 이 순간에 대해 `build_open_flags`에서 이 순간에 대한 플래그로 끝났고, 우리는 `open_flags-> open_flag`를 그것들로 채울 수 있습니다 :

```C
op->open_flag = flags;
```

이제 파일 열기를 제어하는 플래그를 나타내는 `open_flag` 필드의 생성을 위해 파일을 열면 새 파일의 `umask`를 나타내는 `mode`가 채워져 있습니다. 우리의 `open_flags` 구조에서 마지막 플래그를 채워야합니다. 다음은 열린 파일에 대한 액세스 모드를 나타내는 `op-> acc_mode`입니다. `acc_mode` 지역 변수를 `build_open_flags` 시작 부분의 초기 값으로 이미 채웠으며 이제 액세스 모드와 관련된 마지막 두 플래그를 확인합니다.

```C
if (flags & O_TRUNC)
        acc_mode |= MAY_WRITE;
if (flags & O_APPEND)
	acc_mode |= MAY_APPEND;
op->acc_mode = acc_mode;
```

이 플래그는 - `O_TRUNC`로, 열린 파일이 존재하기 전에 열린 파일의 길이를 0으로 자르고, `O_APPEND` 플래그는 파일을 추가 모드에서 열 수 있도록 합니다. 따라서 열린 파일은 쓰기 중에 추가되지만 덮어 쓰지는 않습니다.

`open_flags` 구조체의 다음 필드는 `intent`입니다. 이를 통해 우리의 의도 또는 다른 말로 파일과 관련하여 실제로 하고 싶은 작업, 파일 열기, 생성, 이름 변경 등을 알 수 있습니다. 따라서 플래그에 `O_PATH` 플래그가 포함되어 있으면 이 플래그로 인해 파일 내용과 관련된 작업을 수행 할 수 없으므로 이를 0으로 설정합니다.

```C
op->intent = flags & O_PATH ? 0 : LOOKUP_OPEN;
```

또는 단지 `LOOKUP_OPEN` 의도로. 또한 새 파일을 만들고 `O_EXCL` 플래그를 사용하여 파일이 존재하지 않도록 하려면 `LOOKUP_CREATE` 의도를 설정합니다.

```C
if (flags & O_CREAT) {
	op->intent |= LOOKUP_CREATE;
	if (flags & O_EXCL)
		op->intent |= LOOKUP_EXCL;
}
```

`open_flags` 구조의 마지막 플래그는`lookup_flags`입니다.

```C
if (flags & O_DIRECTORY)
	lookup_flags |= LOOKUP_DIRECTORY;
if (!(flags & O_NOFOLLOW))
	lookup_flags |= LOOKUP_FOLLOW;
op->lookup_flags = lookup_flags;

return 0;
```

디렉토리를 열려면 `LOOKUP_DIRECTORY`로 채우고 (열기) [symlink](https://en.wikipedia.org/wiki/Symbolic_link)를 따르지 않으려면 `LOOKUP_DIRECTORY`로 채우고 `LOOKUP_FOLLOW`로 채 웁니다. 이것이 `build_open_flags` 함수의 끝입니다. `open_flags` 구조는 파일 열기를위한 모드와 플래그로 채워져 있으며 우리는 `do_sys_open`으로 되돌아 갈 수 있습니다.

파일의 실제 열기
--------------------------------------------------------------------------------

`build_open_flags` 함수가 완료된 후 다음 단계에서 우리는 파일에 플래그와 모드를 형성했습니다. 우리는 오픈 시스템 콜에 전달 된 파일 이름을 이용해 `getname` 함수의 도움으로 `filename` 구조체를 얻어야합니다.

```C
tmp = getname(filename);
if (IS_ERR(tmp))
	return PTR_ERR(tmp);
```

`getname` 함수는 [fs/namei.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/fs/namei.c) 소스 코드 파일에 정의되어 있으며 다음과 같습니다.

```C
struct filename *
getname(const char __user * filename)
{
        return getname_flags(filename, 0, NULL);
}
```

따라서 `getget_flags` 함수를 호출하고 결과를 반환합니다. `getname_flags` 함수의 주요 목표는 사용자 영역에서 커널 공간으로 주어진 파일 경로를 복사하는 것입니다. `filename` 구조는 [include/linux/fs.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/include/linux/fs.h) 리눅스 커널 헤더 파일에 정의되어 있으며 다음을 포함합니다. 

* name - 커널 공간에서 파일 경로를 가리키는 포인터
* uptr - 사용자 영역의 원래 포인터;
* 이름 - [audit](https://linux.die.net/man/8/auditd) 컨텍스트의 파일 이름;
* refcnt - 기준 카운터;
* iname - `PATH_MAX`보다 작은 경우 파일 이름.

위에서 이미 쓴 것처럼 `getname_flags` 함수의 주요 목표는 strncpy_from_user 함수를 사용하여 사용자 공간에서 커널 공간으로 `open` 시스템 호출에 전달 된 파일의 이름을 복사하는 것입니다. 파일 이름이 커널 공간으로 복사 된 다음 단계는 새로운 비 사용 파일 디스크립터를 얻는 것입니다.

```C
fd = get_unused_fd_flags(flags);
```

`get_unused_fd_flags` 함수는 현재 프로세스의 열린 파일 테이블, 시스템에서 파일 설명 자의 최소 (`0`) 및 최대 (`RLIMIT_NOFILE`) 가능한 수와 `open` 시스템 콜에 전달한 플래그를 취합니다. 파일 디스크립터를 할당하고 현재 프로세스의 파일 디스크립터 테이블에서 사용 중으로 표시합니다. `get_unused_fd_flags` 함수는 전달 된 플래그의 상태에 따라 O_CLOEXEC 플래그를 설정하거나 지웁니다.

`do_sys_open`의 마지막 단계는`do_filp_open` 함수입니다.

```C
struct file *f = do_filp_open(dfd, tmp, &op);

if (IS_ERR(f)) {
	put_unused_fd(fd);
	fd = PTR_ERR(f);
} else {
	fsnotify_open(f);
	fd_install(fd, f);
}
```

이 함수의 주요 목표는 주어진 경로 이름을 프로세스의 열린 파일을 나타내는 `file` 구조체로 해석하는 것입니다. 문제가 발생하여 `do_filp_open` 함수의 실행이 실패하면 `put_unused_fd`를 사용하여 새 파일 디스크립터를 해제하거나 다른 방법으로 `do_filp_open`에서 리턴 한 현재 프로세스의 `file` 구조체가 파일 디스크립터 테이블에 저장됩니다.

이제 `do_filp_open` 함수의 구현을 간단히 살펴 봅시다. 이 함수는 [fs/namei.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/fs/namei.c) 리눅스 커널 소스 코드 파일에 정의되어 있으며 `nameidata` 구조체의 초기화부터 시작합니다 이 구조체는 파일 [inode](https://en.wikipedia.org/wiki/Inode)에 대한 링크를 제공합니다. 실제로 이것은 `open` 시스템 콜에 주어진 파일 이름으로 `inode`를 얻는 `do_filp_open` 함수의 요점 중 하나입니다. `nameidata` 구조가 초기화되면 `path_openat` 함수가 호출됩니다 :

```C
filp = path_openat(&nd, op, flags | LOOKUP_RCU);

if (unlikely(filp == ERR_PTR(-ECHILD)))
	filp = path_openat(&nd, op, flags);
if (unlikely(filp == ERR_PTR(-ESTALE)))
	filp = path_openat(&nd, op, flags | LOOKUP_REVAL);
```

세 번 호출된다는 것을 기억하세요. 실제로 Linux 커널은 [RCU](https://www.kernel.org/doc/Documentation/RCU/whatisRCU.txt) 모드에서 파일을 엽니다. 이것이 파일을 여는 가장 효율적인 방법입니다. 이 시도가 실패하면 커널은 일반 모드로 들어갑니다. 세 번째 호출은 상대적으로 드물며 [nfs](https://en.wikipedia.org/wiki/Network_File_System) 파일 시스템에서만 사용됩니다. `path_openat` 함수는`path lookup`을 실행하거나, 즉 경로에 해당하는 `dentry` (리눅스 커널이 디렉토리의 파일 계층을 추적하기 위해 사용하는 것)를 찾으려고 시도합니다.

`path_openat` 함수는 시스템에서 열린 파일의 양을 초과하는지 여부 등과 같은 몇 가지 추가 검사와 함께 새로운 `file` 구조체를 할당하는 `get_empty_flip ()` 함수의 호출에서 시작합니다.우리가 새로운 `file` 구조체를 할당 한 후에 우리는 `open` 시스템 콜을 호출하는 동안 `O_TMPFILE |O_CREATE` 또는 `O_PATH` 플래그를 통과 한 경우에 `do_tmpfile` 또는`do_o_path` 함수를 호출합니다. 이 두 경우는 매우 구체적이므로 이미 존재하는 파일을 열고 파일에서 읽고 쓰기를 원할 때는 일반적인 경우를 고려합니다.

이 경우 `path_init` 함수가 호출됩니다. 이 기능은 실제 경로 조회 전에 준비 작업을 수행합니다. 여기에는 경로 탐색의 시작 위치 검색 및 경로의 `inode`, `dentry inode` 등과 같은 메타 데이터가 포함됩니다. 이는 `root` 디렉토리 - `/` 또는 현재 디렉토리가 될 수 있습니다. `AT_CWD`를 시작점으로합니다 (포스트 시작 부분의`do_sys_open` 호출 참조).

`path_init` 다음의 다음 단계는 `link_path_walk`와 `do_last`를 실행하는 [loop](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/fs/namei.c#L3457)입니다. 첫 번째 함수는 이름 확인을 실행합니다. 즉,이 함수는 주어진 경로를 따라 걷는 과정을 시작합니다. 파일 경로의 마지막 구성 요소를 제외한 모든 단계를 단계별로 처리합니다. 이 처리에는 권한 확인 및 파일 구성 요소 가져 오기가 포함됩니다. 파일 구성 요소를 가져 오면 `walk_component`로 전달되어 `dcache`에서 현재 디렉토리 항목을 업데이트하거나 기본 파일 시스템을 요청합니다. 모든 경로의 구성 요소가 이러한 방식으로 처리되지 않기 전에이 과정이 반복됩니다. `link_path_walk`가 실행 된 후 `do_last` 함수는 `link_path_walk`의 결과에 따라 `file` 구조를 채웁니다. 주어진 파일 경로의 마지막 구성 요소에 도달하면 `do_last`의 `vfs_open` 함수가 호출됩니다.

이 함수는 [fs/open.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/fs/open.c) 리눅스 커널 소스 코드 파일에 정의되어 있으며 이 함수의 주요 목표는 기본 파일 시스템의 `open` 연산을 호출입니다.

지금은 여기까지입니다. 우리는 `open` 시스템 호출의 **전체** 구현을 고려하지 않았습니다. 마운트 포인트가 다른 다른 파일 시스템에서 파일을 열고 심볼릭 링크를 해결하는 등의 처리 방법과 같은 일부 부분은 생략하지만 이 내용을 따르는 것이 어렵지 않아야합니다. 이 내용은 개방 시스템 호출의 **일반** 구현에는 포함되어 있지 않으며 기본 파일 시스템에 따라 다릅니다. 관심이 있다면 특정 [filesystem](https://github.com/torvalds/linux/tree/master/fs)에 대한 `file_operations.open` 콜백 함수를 찾아 볼 수 있습니다.

결론
--------------------------------------------------------------------------------

이것은 Linux 커널에서 다른 시스템 콜을 구현하는 다섯 번째 부분의 끝입니다. 질문이나 제안 사항이 있으면 Twitter [0xAX](https://twitter.com/0xAX)에 핑(Ping)을 보내거나 [email](anotherworldofworld@gmail.com)을 보내거나 [issue](https://github.com/0xAX/linux-internals/issues/new)를 만드세요. 다음 부분에서는 계속해서 Linux 커널에서 시스템 호출을 살펴보고 [read](http://man7.org/linux/man-pages/man2/read.2.html) 시스템의 구현을 확인합니다.

**Please note that English is not my first language and I am really sorry for any inconvenience. If you find any mistakes please send me PR to [linux-insides](https://github.com/0xAX/linux-internals).**

링크
--------------------------------------------------------------------------------

* [system call](https://en.wikipedia.org/wiki/System_call)
* [open](http://man7.org/linux/man-pages/man2/open.2.html)
* [file descriptor](https://en.wikipedia.org/wiki/File_descriptor)
* [proc](https://en.wikipedia.org/wiki/Procfs)
* [GNU C Library Reference Manual](https://www.gnu.org/software/libc/manual/html_mono/libc.html#File-Position-Primitive)
* [IA-64](https://en.wikipedia.org/wiki/IA-64) 
* [x86_64](https://en.wikipedia.org/wiki/X86-64)
* [opendir](http://man7.org/linux/man-pages/man3/opendir.3.html)
* [fanotify](http://man7.org/linux/man-pages/man7/fanotify.7.html)
* [fork](https://en.wikipedia.org/wiki/Fork_\(system_call\))
* [execve](https://en.wikipedia.org/wiki/Exec_\(system_call\))
* [symlink](https://en.wikipedia.org/wiki/Symbolic_link)
* [audit](https://linux.die.net/man/8/auditd)
* [inode](https://en.wikipedia.org/wiki/Inode)
* [RCU](https://www.kernel.org/doc/Documentation/RCU/whatisRCU.txt)
* [read](http://man7.org/linux/man-pages/man2/read.2.html)
* [previous part](https://0xax.gitbooks.io/linux-insides/content/SysCall/linux-syscall-4.html)
