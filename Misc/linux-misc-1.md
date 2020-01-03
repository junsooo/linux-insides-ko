리눅스 커널 발전
================================================================================

도입부
--------------------------------------------------------------------------------

이미 알고 계실 수도 있는데, 저는 작년에 `x86_64` 아키텍처를 위한 어셈블러 프로그래밍에 대한 [블로그 포스팅](https://0xax.github.io/categories/assembler/) 시리즈를 시작했습니다. 저는 지금까지 대학교에서`'Hello World` 예제로 놀아본 것 외에는 지금까지 저수준의 코드 한 줄도 써보지 않았었습니다. 얼마 전부터 그런 것들에 관심을 갖게되었습니다. 프로그램을 작성할 수 있다는 것은 이해했지만, 실제로 프로그램이 어떻게 구성되어 있는지는 아직 이해하지 못했습니다.

어셈블러 코드를 작성한 후 **대략** 컴파일 후 프로그램이 어떻게 보이는지 이해하기 시작했습니다. 하지만 어쨌든, 다른 많은 것을 이해하지는 못했습니다. 예를 들어: 어셈블러에서 `syscall` 명령이 실행될 때 발생하는 일,`printf` 기능이 작동하기 시작할 때 발생하는 일 또는 네트워크를 통해 프로그램이 다른 컴퓨터와 어떻게 통신 할 수 있을까? [어셈블러](https://en.wikipedia.org/wiki/Assembly_language#Assembler) 프로그래밍 언어는 제 질문에 대한 답을주지 못했기에 더 깊이 연구해보기로 결정했습니다. 리눅스 커널의 소스 코드를 배우기 시작했고 제가 관심있는 것들을 이해하려고 노력했습니다. 리눅스 커널의 소스 코드는 제 **모든** 질문에 대한 답변을 주지 않았지만, 이제 저는 리눅스 커널과 그 프로세스에 대한 지식이 훨씬 좋아졌습니다.

저는 리눅스 커널의 소스 코드를 배우고 이 책의 첫 파트를 출판 한 지 9 개월 반 후에 이 부분을 쓰고 있습니다. 이제 40 개의 파트가 포함되어 있으며, 아직 끝이 아닙니다. 저는 리눅스 커널에 대해 이 시리즈를 작성하기로 결정했습니다. 아시다시피 Linux 커널은 매우 거대한 코드 조각이므로 Linux 커널에서 이 부분이 무엇을 하는지, 그 부분의 의미는 무엇인지, 어떻게 구현하는지 등을 잊어 버리기 쉽습니다. 하지만 곧 [linux-insides](https://github.com/0xAX/linux-insides) 레포가 인기를 얻었고 9개월 후에는 `9096`개의 별을 갖게되었습니다. :

![github](http://i63.tinypic.com/2lbgc9f.png)

사람들은 리눅스 커널 내부에 관심이 있는 것 같습니다. 이 외에도, 제가 `linux-insides`를 쓰고있는 동안, 저는 다른 사람들로부터 리눅스 커널에 기여하기를 시작하는 방법에 대해 많은 질문을 받았습니다. 일반적으로 사람들은 오픈 소스 프로젝트에 기여하는 데에 관심이 있으며 Linux 커널도 예외는 아닙니다. :

![google-linux](http://i64.tinypic.com/2j4ot5e.png)

따라서 사람들은 Linux 커널 개발 프로세스에 관심이 있는 것 같습니다. 리눅스 커널에 관한 책에 리눅스 커널 개발에 참여하는 방법을 설명하는 부분이 포함되어 있지 않으면 이상하다고 생각했기 때문에 작성하기로 결정했습니다. 이 파트에서 리눅스 커널에 기여해야하는 이유에 대한 정보는 찾을 수 없습니다. 그러나 리눅스 커널 개발을 시작하는 방법에 관심이 있다면 이 파트가 도움이됩니다.

시작해봅시다.

리눅스 커널을 시작하는 방법
---------------------------------------------------------------------------------

우선, 리눅스 커널을 얻고, 빌드하고, 실행하는 방법을 봅시다. 다음 두 가지 방법으로 리눅스 커널의 사용자 정의 빌드를 실행할 수 있습니다.

* 가상 머신에서 리눅스 커널을 실행하세요.
* 실제 하드웨어에서 리눅스 커널을 실행하세요.

저는 두 방법 모두에 대한 설명을 제공할 것입니다. 우리는 리눅스 커널로 무엇인가를 시작하기 전에 리눅스 커널을 얻어야합니다. 용도에 따라 몇 가지 방법이 있습니다. 컴퓨터에서 현재 버전의 리눅스 커널을 업데이트하려는 경우 당신의 Linux [distro](https://en.wikipedia.org/wiki/Linux_distribution) 관련 명령어들을 사용할 수 있습니다.

첫 번째의 경우 [package manager](https://en.wikipedia.org/wiki/Package_manager)를 사용하여 새로운 버전의 리눅스 커널을 다운로드하면 됩니다. 예를 들어, [Ubuntu (Vivid Vervet)](http://releases.ubuntu.com/15.04/)에서 리눅스 커널 버전을 `4.1`로 업그레이드 하려면 다음 명령을 실행하면 됩니다. :

```
$ sudo add-apt-repository ppa:kernel-ppa/ppa
$ sudo apt-get update
```

이 명령을 실행 한 후 :

```
$ apt-cache showpkg linux-headers
```

그리고 관심있는 리눅스 커널 버전을 선택하세요. 마지막으로 다음 명령을 실행하고 `${version}`을 이전 명령의 출력에서 선택한 버전으로 바꿉니다. :

```
$ sudo apt-get install linux-headers-${version} linux-headers-${version}-generic linux-image-${version}-generic --fix-missing
```

그리고 시스템을 재부팅하세요. 재부팅 후 [grub](https://en.wikipedia.org/wiki/GNU_GRUB) 메뉴에 새 커널이 표시됩니다.

다른 방법으로 리눅스 커널 개발에 관심이 있다면 리눅스 커널의 소스 코드를 얻어야합니다. [kernel.org](https://kernel.org/) 웹 사이트에서 찾을 수 있으며 Linux 커널 소스 코드가 포함 된 아카이브를 다운로드하세요. 실제로 리눅스 커널 개발 프로세스는 `git` [version control system](https://en.wikipedia.org/wiki/Version_control)을 중심으로 완전히 빌드되어 있습니다. `kernel.org`에서 `git`으로 얻을 수 있습니다 :

```
$ git clone git://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git
```

당신은 어떨지 모르겠지만, 저는 `github`를 선호합니다. 리눅스 커널 기본 저장소에는 [mirror](https://github.com/torvalds/linux)가 있으므로 다음을 사용하여 복제 할 수 있습니다. :
```
$ git clone git@github.com:torvalds/linux.git
```

개발을 위해 제 자신의 [fork](https://github.com/0xAX/linux)를 사용하고 주 레포지토리에서 업데이트를 가져오려면 다음 명령을 실행하면 됩니다. :

```
$ git checkout master
$ git pull upstream master
```

메인 레포지토리의 리모트 이름은 `upstream`입니다. 기본 리눅스 레포지토리에 새 리모트를 추가하려면 다음을 수행하세요. :

```
git remote add upstream git@github.com:torvalds/linux.git
```

이 수행 후에 여러분은 두 개의 리모트를 가지고 있을 것입니다. :

```
~/dev/linux (master) $ git remote -v
origin	git@github.com:0xAX/linux.git (fetch)
origin	git@github.com:0xAX/linux.git (push)
upstream	https://github.com/torvalds/linux.git (fetch)
upstream	https://github.com/torvalds/linux.git (push)
```

하나는 당신의 포크 (`origin`)이고 다른 하나는 메인 저장소 (`upstream`)입니다.

이제 리눅스 커널 소스 코드의 로컬 사본이 있으므로 이를 설정하고 빌드해야합니다. 리눅스 커널은 다른 방법으로 설정할 수 있습니다. 가장 간단한 방법은 `/boot` 디렉토리에있는 이미 설치된 커널의 설정 파일을 복사하는 것입니다. :

```
$ sudo cp /boot/config-$(uname -r) ~/dev/linux/.config
```

현재 여러분의 리눅스 커널이 `/proc/config.gz` 파일에 대한 액세스를 지원하도록 빌드된 경우 다음 명령을 사용하여 실제 커널 설정 파일을 복사 할 수 있습니다. :

```
$ cat /proc/config.gz | gunzip > ~/dev/linux/.config
```

배포판 관리자가 제공한 표준 커널 설정에 만족하지 못한다면 리눅스 커널을 수동으로 설정 할 수 있습니다. 몇 가지 방법이 있습니다. 리눅스 커널 루트 [Makefile](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/Makefile)은 이를 설정 할 수있는 일련의 타겟들을 제공합니다. 예를 들어 `menuconfig`는 커널 설정을 위한 메뉴 중심 인터페이스를 제공합니다. :

![menuconfig](http://i64.tinypic.com/zn5zbq.png)

`defconfig` 매개변수는 현재 아키텍처에 대한 기본 커널 설정 파일, 예를 들어 [x86_64 defconfig](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/configs/x86_64_defconfig)을 생성합니다. 주어진 아키텍처에 대해 `ARCH` 명령 행 매개변수를 `make`에 전달하여 `defconfig`를 빌드 할 수 있습니다. :

```
$ make ARCH=arm64 defconfig
```

`allnoconfig`, `allyesconfig`, `allmodconfig` 매개변수를 사용하면 모든 옵션이 개별적으로 모든 옵션이 비활성화되고, 활성화되고, 또는 모듈을 이용하여 개별적으로 활성화 되는 새 설정 파일을 생성 할 수 있습니다. 리눅스 커널을 설정하기 위한 메뉴가 있는 `ncurses` 기반 프로그램을 제공하는 `nconfig` 명령 행 매개변수 :

![nconfig](http://i68.tinypic.com/jjmlfn.png)

게다가 `randconfig`은 랜덤 리눅스 커널 설정 파일을 생성합니다. 저는 두 가지 이유로 리눅스 커널을 구성하는 방법이나 어떤 옵션을 사용할 수 있는지에 대해서는 다루지 않을 것입니다. 첫번째로 제가 여러분의 하드웨어를 알지 못하고, 두번째로 여러분이 하드웨어를 안다면 유일하게 남은 작업은 커널 설정을 위한 프로그램을 사용하는 방법을 알아내는 것 뿐이며 모든 것들은 사용하기가 매우 쉽습니다.

자, 우리는 지금 리눅스 커널의 소스 코드를 가지고 있고 그것을 설정했습니다. 다음 단계는 리눅스 커널의 컴파일입니다. 리눅스 커널을 컴파일하는 가장 간단한 방법은 바로 실행하는 것입니다 :

```
$ make
scripts/kconfig/conf  --silentoldconfig Kconfig
#
# configuration written to .config
#
  CHK     include/config/kernel.release
  UPD     include/config/kernel.release
  CHK     include/generated/uapi/linux/version.h
  CHK     include/generated/utsrelease.h
  ...
  ...
  ...
  OBJCOPY arch/x86/boot/vmlinux.bin
  AS      arch/x86/boot/header.o
  LD      arch/x86/boot/setup.elf
  OBJCOPY arch/x86/boot/setup.bin
  BUILD   arch/x86/boot/bzImage
  Setup is 15740 bytes (padded to 15872 bytes).
System is 4342 kB
CRC 82703414
Kernel: arch/x86/boot/bzImage is ready  (#73)
```

커널 컴파일 속도를 높이려면 `-jN` 명령 행 매개변수를 `make`에 전달하면 됩니다. 여기서 `N`은 동시에 실행할 명령 수를 지정합니다. :

```
$ make -j8
```

현재와 다른 아키텍처에 대한 리눅스 커널을 빌드하려는 경우 가장 간단한 방법은 두 가지 매개변수를 전달하는 것입니다.

* `ARCH` 명령 행 매개변수와 대상 아키텍처의 이름;
* `CROSS_COMPILER` 명령 행 매개변수와 cross-comppile 툴 프리픽스;

예를 들어 기본 커널 설정 파일을 사용하여 [arm64](https://en.wikipedia.org/wiki/ARM_architecture#AArch64_features)에 대한 Linux 커널을 컴파일하려면 다음 명령을 실행해야합니다. :

```
$ make -j4 ARCH=arm64 CROSS_COMPILER=aarch64-linux-gnu- defconfig
$ make -j4 ARCH=arm64 CROSS_COMPILER=aarch64-linux-gnu-
```

컴파일 결과 압축 된 커널 `arch/x86/boot/bzImage`를 볼 수 있습니다. 이제 커널을 컴파일 했으므로 컴퓨터에 커널을 설치하거나 에뮬레이터에서 실행할 수 있습니다.

리눅스 커널 설치
--------------------------------------------------------------------------------

이미 위에서 언급했듯이, 새로운 커널을 시작하는 두 가지 방법을 고려할 것입니다. 첫 번째 경우에서 실제 하드웨어에 새로운 버전의 리눅스 커널을 설치하고 실행할 수 있고 두 번째에서는 가상 머신에서 리눅스 커널을 시작하는 것입니다. 이전 단락에서 우리는 소스 코드에서 리눅스 커널을 빌드하는 방법을 보았으며 결과적으로 압축 된 이미지를 얻었습니다 :

```
...
...
...
Kernel: arch/x86/boot/bzImage is ready  (#73)
```

[bzImage](https://en.wikipedia.org/wiki/Vmlinux#bzImage)를 얻은 후에는 다음을 이요하여 새로운 리눅스 커널의 `headers`, `modules`를 설치해야합니다. :


```
$ sudo make headers_install
$ sudo make modules_install
```

그리고 직접적으로 커널 그 자체를 설치해야합니다. :

```
$ sudo make install
```

이 시점부터 우리는 새로운 버전의 리눅스 커널을 설치했으며 이제는 부트 로더에 대해 다뤄야합니다. 물론 `/boot/grub2/grub.cfg` 설정 파일을 편집하여 수동으로 추가 할 수도 있지만, 이 방법으로 스크립트를 사용하는 것을 선호합니다. 저는 Fedora와 Ubuntu의 두 가지 리눅스 배포판을 사용하고 있습니다. [grub](https://en.wikipedia.org/wiki/GNU_GRUB) 설정 파일을 업데이트하는 방법에는 두 가지가 있습니다. 이 방법으로 다음 스크립트를 사용하고 있습니다. :

```shell
#!/bin/bash

source "term-colors"

DISTRIBUTIVE=$(cat /etc/*-release | grep NAME | head -1 | sed -n -e 's/NAME\=//p')
echo -e "Distributive: ${Green}${DISTRIBUTIVE}${Color_Off}"

if [[ "$DISTRIBUTIVE" == "Fedora" ]] ;
then
    su -c 'grub2-mkconfig -o /boot/grub2/grub.cfg'
else
    sudo update-grub
fi

echo "${Green}Done.${Color_Off}"
```

이것은 새로운 리눅스 커널 설치의 마지막 단계이며, 그 후에 컴퓨터를 재부팅하고 부팅 중에 커널의 새 버전을 선택할 수 있습니다.

두 번째 경우는 가상 머신에서 새 Linux 커널을 시작하는 것입니다. 저는 [qemu](https://en.wikipedia.org/wiki/QEMU)를 선호합니다. 우선 우리는 이를 위해 초기 램 디스크인 [initrd](https://en.wikipedia.org/wiki/Initrd)를 빌드해야합니다. `initrd`는 초기화 과정에서 리눅스 커널이 사용하는 임시 루트 파일 시스템이지만 다른 파일 시스템은 마운트되지 않습니다. 다음 명령으로 `initrd`를 만들 수 있습니다 :

우선 [busybox](https://en.wikipedia.org/wiki/BusyBox)를 다운로드하고 설정을 위해 `menuconfig`를 실행해야합니다 :

```shell
$ mkdir initrd
$ cd initrd
$ curl http://busybox.net/downloads/busybox-1.23.2.tar.bz2 | tar xjf -
$ cd busybox-1.23.2/
$ make menuconfig
$ make -j4
```

`busybox`는 [coreutils](https://en.wikipedia.org/wiki/GNU_Core_Utilities)와 같은 표준 도구 세트를 포함하는 실행 파일인 `/bin/busybox`입니다. `busysbox` 메뉴에서 `Build BusyBox as a static binary (no shared libs)` 옵션을 활성화해야 합니다. :

![busysbox menu](http://i68.tinypic.com/11933bp.png)

메뉴를 다음에서 찾을 수 있습니다. :

```
Busybox Settings
--> Build Options
```

그런 다음 우리는 `busysbox` 설정 메뉴에서 빠져나와 그것을 빌드하고 설치하기 위해 다음 명령을 실행합니다. :

```
$ make -j4
$ sudo make install
```

이제 busybox가 설치되었으므로, 우리는 `initrd`를 빌드할 수 있습니다. 이를 위해 이전 `initrd` 디렉토리로 이동하여 다음을 수행합니다. :

```
$ cd ..
$ mkdir -p initramfs
$ cd initramfs
$ mkdir -pv {bin,sbin,etc,proc,sys,usr/{bin,sbin}}
$ cp -av ../busybox-1.23.2/_install/* .
```

`busybox` 필드를 `bin`, `sbin` 과 기타 디렉토리들에 복사하세요. 이제 시스템에서 첫 번째 프로세스로 실행될 실행 가능한 `init` 파일을 만들어야합니다. 제 `init` 파일은  [procfs](https://en.wikipedia.org/wiki/Procfs) 와 [sysfs](https://en.wikipedia.org/wiki/Sysfs) 파일 시스템을 마운트하고 셸을 실행했습니다. :

```shell
#!/bin/sh

mount -t proc none /proc
mount -t sysfs none /sys

exec /bin/sh
```

이제 우리는 `initrd`가 될 아카이브를 만들 수 있습니다. :

```
$ find . -print0 | cpio --null -ov --format=newc | gzip -9 > ~/dev/initrd_x86_64.gz
```

이제 가상 머신에서 커널을 실행할 수 있습니다. 제가 이미 쓴 것처럼 저는 이에 대해 [qemu](https://en.wikipedia.org/wiki/QEMU)를 선호합니다. 다음 명령으로 커널을 실행할 수 있습니다. :

```
$ qemu-system-x86_64 -snapshot -m 8GB -serial stdio -kernel ~/dev/linux/arch/x86_64/boot/bzImage -initrd ~/dev/initrd_x86_64.gz -append "root=/dev/sda1 ignore_loglevel"
```

![qemu](http://i67.tinypic.com/15i6law.png)

이제 가상 머신에서 리눅스 커널을 실행할 수 있으며 이는 커널 변경 및 테스트를 시작할 수 있음을 의미합니다.

initrd 생성 프로세스를 자동화하려면 [ivandaviov/minimal](https://github.com/ivandavidov/minimal) 또는 [Buildroot](https://buildroot.org/)를 사용하세요.

리눅스 커널 개발 시작하기
---------------------------------------------------------------------------------

이 단락의 주요 요점은 두 가지 질문에 답하는 것입니다. 첫 번째, 패치를 리눅스 커널로 보내기 전에 해야 할 일과 하지 말아야 할 일이 무엇인가. 이 `to do`와 `todo`를 혼동하지 마세요. 저는 리눅스 커널 안에서 여러분이 무엇을 고칠 수 있는지는 모릅니다. 리눅스 커널 소스 코드를 시험하는 동안 워크플로를 알려 드리고자합니다.

우선 다음 명령으로 Linus의 저장소에서 최신 업데이트를 가져옵니다. :

```
$ git checkout master
$ git pull upstream master
```

그런 다음 리눅스 커널 소스 코드가 있는 로컬 저장소가 [mainline](https://github.com/torvalds/linux) 저장소와 동기화됩니다. 이제 소스 코드를 약간 변경할 수 있습니다. 이미 이야기했듯이, 리눅스 커널에서 어디서 시작할 수 있고 리눅스 커널에서의 `TODO`가 무엇인지에 대해서는 조언하지 않습니다. 그러나 초보자를 위한 최고의 장소는 `스테이징` 트리입니다. 다시 말해 [drivers/staging](https://github.com/torvalds/linux/tree/master/drivers/staging)의 드라이버 세트입니다. 스테이징 트리의 관리자는 [Greg Kroah-Hartman](https://en.wikipedia.org/wiki/Greg_Kroah-Hartman)이며 스테이징 트리는 사소한 패치를 수용 할 수있는 곳입니다. 패치를 생성하고 확인한 후 [리눅스 커널 메일 목록](https://lkml.org/)으로 보내는 방법을 설명하는 간단한 예를 살펴 보겠습니다.

[Digi International EPCA PCI](https://github.com/torvalds/linux/tree/master/drivers/staging/dgap) 기반 장치의 드라이버를 살펴보면 295 행에 `dgap_sindex` 함수가 있습니다. :

```C
static char *dgap_sindex(char *string, char *group)
{
	char * ptr;

	if (!string || !group)
		return NULL;

	for (; *string; string++) {
		for (ptr = group; *ptr; ptr++) {
			if (*ptr == *string)
				return string;
		}
	}

	return NULL;
}
```

이 함수는 그룹에서 매치되는 문자를 찾아 해당 위치를 반환합니다. 리눅스 커널의 소스 코드를 살펴보는 동안 [lib/string.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/lib/string.c#L473) 소스 코드 파일에는 `dgap_sinidex`와 같은 기능을 하는 `strpbrk` 함수의 구현이 포함되어 있습니다. 이미 존재하는 함수의 사용자 정의 구현을 사용하는 것은 좋지 않으므로 [drivers/staging/dgap/dgap.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/drivers/staging/dgap/dgap.c) 소스 코드 파일에서 `dgap_sindex` 함수를 제거하고 대신 `strpbrk`를 사용합니다.

우선 리눅스 커널 메인 라인 저장소와 동기화 된 현재 마스터 브랜치를 기반으로 새로운 `git` 브랜치를 만들어 봅시다. :

```
$ git checkout -b "dgap-remove-dgap_sindex"
```

이제 `dgap_sindex`를 `strpbrk`로 바꿀 수 있습니다. 모든 변경을 마친 후에 Linux 커널 또는 [dgap](https://github.com/torvalds/linux/tree/master/drivers/staging/dgap) 디렉토리를 다시 컴파일해야합니다. 커널 설정에서 이 드라이버를 활성화하는 것을 잊지 마세요. 다음에서 찾을 수 있습니다. :

```
Device Drivers
--> Staging drivers
----> Digi EPCA PCI products
```

![dgap menu](http://i65.tinypic.com/v8o5rs.png)

이제 커밋 할 시간입니다. 우리는 이것을 위해 다음 조합을 사용하고 있습니다 :

```
$ git add .
$ git commit -s -v
```

마지막 명령 후에 `$ GIT_EDITOR` 또는 `$ EDITOR` 환경 변수에서 선택되는 편집기가 열립니다. `-s` 명령 행 매개변수는 커밋 로그 메시지의 끝에 커미터에 의해 `Signed-off-by` 라인이 추가됩니다. 각 커밋 메시지의 끝에서 이 행을 찾을 수 있습니다 (예 : [00cc1633](https://github.com/torvalds/linux/commit/00cc1633816de8c95f337608a1ea64e228faf771)). 이 라인의 주요 포인트는 누가 변화를 시켰는지를 찾는 것 입니다. `-v` 옵션은 HEAD 커밋과 커밋 메시지의 맨 아래에 있는 커밋 된 것 사이에서 통합 된 차이를 보여줍니다. 이는 필수적이지는 않지만 때로는 매우 유용합니다. 한 쌍의 단어에 대한 메시지를 커밋합니다. 실제로 커밋 메시지는 두 부분으로 구성 되어 있습니다 :

첫번째 부분은 첫 번째 행에 변화에 대한 간단한 설명을 포함합니다. `[PATCH]` 접두사로 시작하고 그 뒤에 서브 시스템, 드라이버 또는 아키텍처 이름이 있으며 `:` 기호 뒤에 설명이 옵니다. 우리의 경우는 이런 일이 될 것입니다 :

```
[PATCH] staging/dgap: Use strpbrk() instead of dgap_sindex()
```

간단한 설명 후에 일반적으로 우리는 빈 라인과 커밋에 대한 자세한 설명을 갖습니다. 우리의 경우 :

```
The <linux/string.h> provides strpbrk() function that does the same that the
dgap_sindex(). Let's use already defined function instead of writing custom.
```

그리고 커밋 메시지 끝의 `Sign-off-by` 라인. 커밋 메시지의 각 줄은 기호 `80`개를 넘지 않아야 하며 커밋 메시지는 변경 사항을 자세하게 설명해야합니다. `Custom function removed`와 같은 커밋 메시지를 작성하지 말고 수행 한 작업과 이유를 설명해야합니다. 패치 검토자는 자신이 검토 한 내용을 알아야합니다. 이 뷰의 커밋 메시지 외에도 매우 유용합니다. 무언가를 이해할 수 없을 때마다 [git blame](http://git-scm.com/docs/git-blame)을 사용하여 변경 사항에 대한 설명을 읽을 수 있습니다.

패치를 생성하기 위해 변경 시간을 커밋 한 후 `format-patch` 명령으로 이를 할 수 있습니다 :

```
$ git format-patch master
0001-staging-dgap-Use-strpbrk-instead-of-dgap_sindex.patch
```

브랜치의 이름 (이 경우에는 `master`)을 `format-patch` 명령에 전달하여 `master` 브랜치에 없고 `dgap-remove-dgap_sindex` 브랜치에는 있는 마지막 변경 사항에 대한 패치를 생성하는 `format-patch` 명령에 전달했습니다. 알 수 있듯이 `format-patch` 명령은 마지막 변경 사항을 포함하고 커밋 간단한 설명을 기반으로 하는 이름을 가진 파일을 생성합니다. 사용자 정의 이름으로 패치를 생성하려면 `--stdout` 옵션을 사용할 수 있습니다 :

```
$ git format-patch master --stdout > dgap-patch-1.patch
```

패치를 생성 하는 마지막 단계는 패치를 리눅스 커널 메일 리스트로 보내는 것입니다. 물론, 당신은 모든 이메일 클라이언트를 사용할 수 있습니다. `git`은 이것을 위한 특별한 명령을 제공합니다 :`git send-email`. 패치를 보내기 전에 보낼 위치를 알아야합니다. 리눅스 커널 메일 리스트 주소인 `linux-kernel@vger.kernel.org`로 보내면되지만, 많은 메시지 flow 때문에 패치가 무시 될 가능성이 높습니다. 패치를 변경 한 하위 시스템의 관리자에게 패치를 보내는 것이 더 좋습니다. 이 관리자들의 이름을 찾으려면 `get_maintainer.pl` 스크립트를 사용하세요. 코드를 작성한 파일이나 디렉토리를 전달하기만 하면 됩니다.

```
$ ./scripts/get_maintainer.pl -f drivers/staging/dgap/dgap.c
Lidza Louina <lidza.louina@gmail.com> (maintainer:DIGI EPCA PCI PRODUCTS)
Mark Hounschell <markh@compro.net> (maintainer:DIGI EPCA PCI PRODUCTS)
Daeseok Youn <daeseok.youn@gmail.com> (maintainer:DIGI EPCA PCI PRODUCTS)
Greg Kroah-Hartman <gregkh@linuxfoundation.org> (supporter:STAGING SUBSYSTEM)
driverdev-devel@linuxdriverproject.org (open list:DIGI EPCA PCI PRODUCTS)
devel@driverdev.osuosl.org (open list:STAGING SUBSYSTEM)
linux-kernel@vger.kernel.org (open list)
```

이름과 관련 된 이메일 세트가 표시됩니다. 이제 다음과 같이 패치를 보낼 수 있습니다. :

```
$ git send-email --to "Lidza Louina <lidza.louina@gmail.com>" \
  --cc "Mark Hounschell <markh@compro.net>"                   \
  --cc "Daeseok Youn <daeseok.youn@gmail.com>"                \
  --cc "Greg Kroah-Hartman <gregkh@linuxfoundation.org>"      \
  --cc "driverdev-devel@linuxdriverproject.org"               \
  --cc "devel@driverdev.osuosl.org"                           \
  --cc "linux-kernel@vger.kernel.org"
```

이게 끝입니다. 패치가 전송되었으며 이제는 리눅스 커널 개발자의 피드백을 기다려야 하는 것 뿐입니다. 패치를 보내고 관리자가 패치를 수락하면 관리자가 레포지토리의 저장소 (예 : 이 파트에서 본 [patch](https://git.kernel.org/cgit/linux/kernel/git/gregkh/staging.git/commit/?h=staging-testing&id=b9f7f1d0846f15585b8af64435b6b706b25a5c0b))에서 찾은 후 일정 시간 후에 관리자가 Linus에 풀리퀘스트를 보내면 메인 라인 레포지토리에서 여러분의 패치를 볼 수 있습니다.

끝입니다.

약간의 조언
--------------------------------------------------------------------------------

이 파트의 끝에서 저는 리눅스 커널을 개발하는 동안 해야 할 것과 하지 말아야 할 것에 대한 조언을 드리고자 합니다.

* 생각하고 생각하세요. 패치를 보내기 전에 다시 생각하세요.

* 리눅스 커널 소스 코드에서 무언가를 수정할 때마다 컴파일하세요. 어떠한 수정 후에도. 반복해서. 컴파일조차 하지 않는 수정을 좋아하는 사람은 없습니다.

* 리눅스 커널은 코딩 스타일 [guide](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/Documentation/CodingStyle)을 가지며, 이를 준수해야합니다. 변경 사항을 확인하는 데 도움이되는 훌륭한 스크립트가 있습니다. 이 스크립트는 [scripts/checkpatch.pl](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/scripts/checkpatch.pl)입니다. 변경 사항이 있는 소스 코드 파일을 전달하면 다음과 같이 표시됩니다. :


```
$ ./scripts/checkpatch.pl -f drivers/staging/dgap/dgap.c
WARNING: Block comments use * on subsequent lines
#94: FILE: drivers/staging/dgap/dgap.c:94:
+/*
+     SUPPORTED PRODUCTS

CHECK: spaces preferred around that '|' (ctx:VxV)
#143: FILE: drivers/staging/dgap/dgap.c:143:
+	{ PPCM,        PCI_DEV_XEM_NAME,     64, (T_PCXM|T_PCLITE|T_PCIBUS) },

```

또한 `git diff`의 도움으로 문제 가있는 곳을 볼 수 있습니다. :

![git diff](http://oi60.tinypic.com/2u91rgn.jpg)

* [Linus는 github 풀리퀘스트를 수락하지 않습니다](https://github.com/torvalds/linux/pull/17#issuecomment-5654674)

* 변경 사항이 서로 다르고 관련없는 변경 사항으로 구성된 경우 별도의 커밋을 통해 변경 사항을 분할해야합니다. `git format-patch` 명령은 각 커밋에 대한 패치를 생성하고 각 패치의 주제는 `vN` 접두사를 포함하며 여기서 `N`은 패치의 번호입니다. 일련의 패치를 보내려는 경우 `--cover-letter` 옵션을 `git format-patch` 명령에 전달하면 도움이됩니다. 그러면 여러분의 일련의 패치에 대한 변경 사항을 설명하는 데에 사용할 cover letter가 포함 된 추가 파일이 생성됩니다. `git send-email` 명령에서 `--in-reply-to` 옵션을 사용하는 것도 좋습니다. 이 옵션을 사용하면 cover 메세지에 대한 답변으로 일련의 패치를 보낼 수 있습니다. 관리자에게 패치의 구조는 다음과 같습니다. :

```
|--> cover letter
  |----> patch_1
  |----> patch_2
```

`git send-email`의 출력에서 ​​찾을 수있는`--in-reply-to` 옵션의 매개변수로 `message-id`를 전달해야합니다.

귀하의 이메일은 [일반 텍스트](https://en.wikipedia.org/wiki/Plain_text) 형식이어야합니다. 일반적으로 `send-email`과 `format-patch`는 개발 과정에서 매우 유용하므로 명령어에 대한 문서를 보면 [git send-email](http://git-scm.com/docs/git-send-email) 과 [git format-patch](http://git-scm.com/docs/git-format-patch)와 같은 유용한 옵션이 있습니다.

* 패치를 보낸 후 즉각적인 답변을 얻지 못하더라도 놀라지 마십시오. 관리자는 매우 바쁠 수 있습니다.

* [scripts](https://github.com/torvalds/linux/tree/master/scripts) 디렉토리에는 리눅스 커널 개발과 관련된 유용한 스크립트가 많이 있습니다. 우리는 이미 이 디렉토리에서 두 스크립트 `checkpatch.pl` 과 `get_maintainer.pl` 를 보았습니다. 이러한 스크립트 외에 스택의 사용법을 출력하는 [stackusage](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/scripts/stackusage) 스크립트, 압축되지 않은 커널 이미지 추출을 위한 [extract-vmlinux](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/scripts/extract-vmlinux)와 기타 여러 가지를 찾을 수 있습니다. `scripts` 디렉토리 외부에서 커널 개발을 위해 [Lorenzo Stoakes](https://twitter.com/ljsloz)의 매우 유용한 [스크립트](https://github.com/lorenzo-stoakes/kernel-scripts)를 찾을 수 있습니다.

* 리눅스 커널 메일링 리스트에 가입하세요. `lkml`에는 매일 많은 글자가 있지만 리눅스 커널의 현재 상태와 같은 것을 읽고 이해하는 것이 매우 유용합니다. `lkml` 외에도 다른 리눅스 커널 서브 시스템과 관련된 [set](http://vger.kernel.org/vger-lists.html) 메일링 리스트가 있습니다.

* 패치가 처음으로 승인되지 않고 Linux 커널 개발자로부터 피드백을 받는 경우, 수정을 하고 `[PATCH vN]` 접두사 (여기서`N`은 패치 버전 번호)로 패치를 다시 보내십시오. 예를 들면 다음과 같습니다. :

```
[PATCH v2] staging/dgap: Use strpbrk() instead of dgap_sindex()
```

또한 이전 패치 버전의 모든 변경 사항을 설명하는 변경 로그가 포함되어 있어야합니다. 물론, 이것은 리눅스 커널 개발에 대한 철저한 요구 사항 목록은 아니지만 가장 중요한 항목 중 일부는 해결되었습니다.

행복한 해킹되세요!

결론
--------------------------------------------------------------------------------

이것이 다른사람들이 리눅스 커널 커뮤니티에 가입하는데에 도움을 줄 수 있기를 바랍니다!

질문이나 제안 사항이 있으면 [twitter](https://twitter.com/0xAX)에 의견이나 핑을 남겨주세요.

**영어는 제 모국어가 아닙니다. 그리고 여타 불편하셨던 점에 대해서 정말로 사과드립니다. 만약 실수를 찾아내셨다면 부디 [linux-insides 원본](https://github.com/0xAX/linux-internals)으로, 번역에 대해서는 [linux-insides 한글 번역](https://github.com/junsooo/linux-insides-ko)으로 PR을 보내주세요.**

Links
--------------------------------------------------------------------------------

* [blog posts about assembly programming for x86_64](https://0xax.github.io/categories/assembler/)
* [Assembler](https://en.wikipedia.org/wiki/Assembly_language#Assembler)
* [distro](https://en.wikipedia.org/wiki/Linux_distribution)
* [package manager](https://en.wikipedia.org/wiki/Package_manager)
* [grub](https://en.wikipedia.org/wiki/GNU_GRUB)
* [kernel.org](https://kernel.org/)
* [version control system](https://en.wikipedia.org/wiki/Version_control)
* [arm64](https://en.wikipedia.org/wiki/ARM_architecture#AArch64_features)
* [bzImage](https://en.wikipedia.org/wiki/Vmlinux#bzImage)
* [qemu](https://en.wikipedia.org/wiki/QEMU)
* [initrd](https://en.wikipedia.org/wiki/Initrd)
* [busybox](https://en.wikipedia.org/wiki/BusyBox)
* [coreutils](https://en.wikipedia.org/wiki/GNU_Core_Utilities)
* [procfs](https://en.wikipedia.org/wiki/Procfs)
* [sysfs](https://en.wikipedia.org/wiki/Sysfs)
* [Linux kernel mail listing archive](https://lkml.org/)
* [Linux kernel coding style guide](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/Documentation/CodingStyle)
* [How to Get Your Change Into the Linux Kernel](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/Documentation/SubmittingPatches)
* [Linux Kernel Newbies](http://kernelnewbies.org/)
* [plain text](https://en.wikipedia.org/wiki/Plain_text)
