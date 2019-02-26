# 커널 부팅 과정

이 챕터에서는 리눅스 커널 부팅 과정에 대해 다루고 있습니다. 여기에서 커널 로딩 과정의 전체 주기를 설명하는 게시물 시리즈를 확인할 수 있습니다:

* [부트로더에서 커널까지](linux-bootstrap-1.md) - 컴퓨터를 켠 이후부터 커널의 첫번째 인스트럭션을 실행하기까지의 모든 단계들을 설명합니다.
* [First steps in the kernel setup code](linux-bootstrap-2.md) - describes first steps in the kernel setup code. You will see heap initialization, query of different parameters like EDD, IST and etc...
* [Video mode initialization and transition to protected mode](linux-bootstrap-3.md) - describes video mode initialization in the kernel setup code and transition to protected mode.
* [Transition to 64-bit mode](linux-bootstrap-4.md) - describes preparation for transition into 64-bit mode and details of transition.
* [Kernel Decompression](linux-bootstrap-5.md) - describes preparation before kernel decompression and details of direct decompression.
* [Kernel random address randomization](linux-bootstrap-6.md) - describes randomization of the Linux kernel load address.

이 챕터는 `Linux kernel v4.17` 기준으로 작성되었습니다.
