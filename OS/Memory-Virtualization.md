# 메모리 가상화 (Memory Virtualization)

운영체제는 physical CPU, physical memory, disk처럼 제한된 자원을 여러 프로그램이 함께 쓰도록 관리한다. 메모리에 적용되는 대표적인 추상화가 **memory virtualization**이다.

- **Memory Virtualization**: 프로세스는 자신만의 연속된 메모리를 가진 것처럼 실행된다.

초기 시스템에서는 운영체제와 하나의 실행 프로그램만 메모리에 올려 순차적으로 처리해도 충분했다. 하지만 프로그램은 계산만 하지 않는다. disk, network, terminal 같은 I/O가 끝나기를 기다리는 시간도 있다. 이때 프로세스는 `Blocked` 또는 `Waiting` 상태가 되고, 실행 가능한 다른 프로세스가 없으면 CPU는 `idle` 상태가 되어 자원이 낭비된다.

이를 방지하기 위해 운영체제는 **Multiprogramming**과 **Time Sharing** 방식을 사용한다.

|                      | 개념                                                          | 특징                                                            |
| -------------------- | ----------------------------------------------------------- | ------------------------------------------------------------- |
| **Multiprogramming** | 여러 프로세스를 메모리에 함께 올려 두고, 한 프로세스가 I/O를 기다리는 동안 다른 프로세스를 실행 | I/O 때문에 CPU가 idle 상태가 되는 시간을 줄임<br>CPU Utilization $\uparrow$ |
| **Time Sharing**     | 각 프로세스에 짧은 실행 시간을 주고, 시간이 끝나면 다음 프로세스로 빠르게 전환             | CPU를 짧은 time slice로 나눔<br>Response Time $\downarrow$             |

프로세스 A가 disk나 network I/O를 기다리는 동안 운영체제는 다른 프로세스 B를 실행한다. 여기에 짧은 time slice와 빠른 전환이 더해지면, 사용자는 하나의 시스템을 여러 사람이 동시에 쓰는 것처럼 느낀다.

## 1. 왜 주소 공간이 필요한가

여러 프로세스를 physical address만으로 메모리에 함께 올리면 두 가지 문제가 생긴다.

1. 각 프로그램을 physical memory의 어느 위치에 **배치**할 것인가? $\to$ **Placement**
2. 한 프로세스가 다른 프로세스나 kernel memory를 읽거나 수정하지 못하게 어떻게 **막을** 것인가? $\to$ **Protection**

이 문제를 해결하기 위해 운영체제는 각 프로세스에 독립적인 **address space**를 제공한다.

프로세스는 자신의 메모리가 0번지부터 연속적으로 이어지는 전용 공간이라고 본다. 그 안에는 **code, heap, stack** 등이 배치된다. 하지만 이 주소가 실제 physical memory 위치를 직접 가리키지는 않는다. 하드웨어와 운영체제는 프로세스가 만든 virtual address를 physical address로 변환하면서 허용되지 않은 접근을 차단한다.

| 프로세스가 보는 것                                                               | 실제 시스템이 관리하는 것                                                                                        |
| ------------------------------------------------------------------------ | ----------------------------------------------------------------------------------------------------- |
| - 자기 메모리를 가진 것처럼 보임<br>- 0번지부터 이어지는 독립적이고 연속적인 **virtual address space** | - 여러 프로세스의 주소를 서로 다른 **physical memory** 위치에 배치<br>- address translation과 protection check 수행 |

서로 다른 프로세스는 모두 자신의 메모리가 0번지에서 시작한다고 생각한다. 하지만 실제 physical memory에서는 서로 겹치지 않는 영역이나 서로 다른 physical frame에 배치된다.

### 1.1. 메모리 가상화의 목표

memory virtualization은 한정된 physical memory를 여러 프로세스가 함께 사용하면서도 각 프로세스에는 독립적인 address space를 제공한다. 이 추상화가 제대로 동작하려면 다음 세 가지 목표를 함께 만족해야 한다.

| 목표               | 의미                                                                                               |
| ---------------- | ------------------------------------------------------------------------------------------------ |
| **Transparency** | 프로세스는 자신만의 독립적이고 연속적인 virtual address space만 사용하면 된다.<br>physical memory의 어느 위치에 배치되어 있는지 알 필요가 없다.   |
| **Efficiency**   | 주소 변환과 메모리 관리가 실행 성능과 메모리 사용량에 과도한 부담을 주지 않아야 한다.                                                |
| **Protection**   | 프로세스는 자신에게 허용된 virtual address와 권한 범위 안에서만 접근할 수 있어야 한다.<br>다른 프로세스나 kernel memory에는 접근할 수 없어야 한다. |

이 중 protection의 핵심은 **isolation**이다. 한 프로세스가 잘못된 주소에 접근하거나 비정상적으로 동작해도 그 영향이 다른 프로세스나 kernel까지 퍼져서는 안 된다. 따라서 address space는 프로세스마다 메모리를 따로 쓰는 것처럼 보이게 하는 추상화이자, 실행을 분리하는 보호 경계다.

### 1.2. Example: 가상 주소 공간은 독립적으로 사용한다

다음 프로그램은 heap에 `int` 하나를 할당하고 그 주소와 저장된 값을 출력한다.
```c
// mem_isolation.c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

int main(void)
{
    int *p = malloc(sizeof(*p)); // Heap 영역에 int 할당
    if (p == NULL) {
        perror("malloc");
        return 1;
    }

    *p = 0; // 0으로 초기화
    printf("pid=%ld, p=%p\n", (long)getpid(), (void *)p);

    for (int i = 0; i < 5; i++) {
        sleep(1);
        (*p)++; // p의 값 증가
        printf("pid=%ld, *p=%d\n", (long)getpid(), *p);
    }

    free(p);
    return 0;
}
```

```bash
$ gcc mem_isolation.c -o mem_isolation
$ ./mem_isolation & ./mem_isolation
```

실행 결과:
```sh
$ ./mem_isolation & ./mem_isolation
[1] 5799
pid=5799, p=0x577721c2e2a0
pid=5800, p=0x5d70b39402a0
pid=5799, *p=1
pid=5800, *p=1
...
pid=5799, *p=5
pid=5800, *p=5
[1]+  Done                    ./mem_isolation
```

1. 두 프로세스는 각각 `p`가 가리키는 heap 영역의 값을 `0`에서 `5`까지 증가시킨다. 한 프로세스가 `*p`를 증가시켜도 다른 프로세스의 `*p`에는 영향을 주지 않는다. 각 프로세스가 서로 독립적인 virtual address space를 사용하기 때문이다.
2. 출력된 `p` 값은 실행할 때마다 달라질 수 있다. ASLR(Address Space Layout Randomization)이 heap, stack, shared library, executable code 등의 배치 주소를 실행마다 바꾸기 때문이다.
	- **Address Space Layout Randomization**: 공격자가 code나 data의 위치를 미리 예측하기 어렵게 만들기 위해 프로그램의 메모리 배치를 실행마다 무작위화하는 보안 기법이다.
3. 반대로 두 프로세스에서 같은 virtual address가 관찰되더라도 그것이 같은 physical memory를 가리킨다는 뜻은 아니다. virtual address의 해석과 physical memory mapping은 프로세스마다 독립적으로 관리된다.

## 2. 프로세스가 보는 가상 주소 공간

**virtual address space**는 운영체제가 각 프로세스에 보여 주는 메모리의 모습이다. 프로세스는 실제 physical memory 배치와 관계없이 낮은 주소에서 높은 주소로 이어지는 하나의 연속된 공간을 사용하는 것처럼 동작한다.

아래 그림은 전형적인 사용자 프로세스의 virtual address space를 개념적으로 나타낸 것이다. 실제 주소와 배치는 ELF 형식, loader, shared library, ASLR, architecture, kernel 설정에 따라 달라진다.
![[virtual-address-space.png]]

| 영역                           | 주요 내용                                              | 특징                                                                      |
| ---------------------------- | -------------------------------------------------- | ----------------------------------------------------------------------- |
| **Kernel virtual memory**    | kernel code, kernel data, kernel stack 등                      | 사용자 모드에서는 접근할 수 없다.                                                     |
| **User stack**               | 함수 호출마다 생성되는 stack frame, 지역 변수, 반환 주소, 저장된 레지스터 등 | 실행 중 생성되며, 일반적으로 낮은 주소 방향으로 성장한다. 함수 반환 시 해당 frame이 제거된다.               |
| Unmapped / free region       | stack 확장을 위한 여유 공간                                 | 필요에 따라 stack이 이 영역 쪽으로 확장될 수 있다.                                        |
| **Memory-mapped region**     | shared library, file mapping, anonymous mapping 등  | `printf()` 같은 library function의 코드도 보통 shared library mapping을 통해 사용된다. |
| Unmapped / free region       | heap과 memory-mapped 영역 사이의 여유 공간                   | heap 또는 mapping의 확장을 위한 공간이다.                                           |
| **Run-time heap**            | `malloc()` 계열이 요청한 동적 메모리                          | 일반적으로 높은 주소 방향으로 성장한다.                                                  |
| **Read/write data**          | 초기화된 global/static 변수와 초기화되지 않은 global/static 변수         | `.data`, `.bss` 등이 포함되며 read/write가 가능하다.                                    |
| **Read-only code and data**  | 실행 명령어, read-only 상수, 문자열 리터럴 등                        | 실행 파일에서 load되며 code는 일반적으로 read/execute 권한을 가진다.                              |
| **Low-address guard region** | 사용자 프로그램이 일반적으로 사용하지 않는 낮은 주소 영역                   | `NULL` pointer 접근을 감지하기 위해 보통 mapping하지 않거나 접근을 제한한다.                   |

Heap은 일반적으로 높은 주소 방향으로 성장하고 user stack은 낮은 주소 방향으로 성장한다. 두 영역이 서로 반대 방향으로 자라면, 실행 중 어느 쪽의 메모리 요구가 더 커질지 미리 알 수 없어도 가운데의 여유 address space를 유연하게 쓸 수 있다. 다만 두 영역이 충분히 확장되어 서로 충돌하면 더 이상 address space를 확보할 수 없다.

함수가 호출되면 user stack에는 해당 호출을 위한 **stack frame**이 생성된다. stack frame에는 지역 변수, 반환 주소, 저장된 레지스터, 함수 인자 등이 들어갈 수 있으며, 함수 호출이 중첩될수록 frame도 차례로 쌓인다. 함수가 반환되면 해당 frame은 제거된다. 이 모든 동작은 각 프로세스의 독립적인 user stack 안에서 이루어진다.

가장 낮은 주소인 `0` 부근은 일반적으로 사용자 프로그램에 mapping하지 않는 영역으로 남겨 둔다. C에서 `NULL` pointer는 보통 주소 `0`을 의미하므로 잘못된 **null pointer dereference**가 발생하면 이 영역에 대한 접근이 fault로 처리되어 오류를 빠르게 감지할 수 있다.

### 2.1. Code/Global/Heap/Stack은 서로 다른 주소 범위에 놓인다

프로그램이 포인터 값으로 확인하는 주소는 physical address가 아니라 virtual address다. Code/global/heap/stack 주소를 출력하면 이들이 서로 다른 주소 범위에 놓인다는 점을 볼 수 있다.

```c
// mem_address.c
#include <stdio.h>
#include <stdlib.h>

static int global_data = 10;

int main(void)
{
    int local = 3;
    void *heap = malloc(1);
    if (heap == NULL) {
        perror("malloc");
        return 1;
    }

    printf("location of code  : %p\n", (void *)main);
    printf("location of global: %p\n", (void *)&global_data);
    printf("location of heap  : %p\n", heap);
    printf("location of stack : %p\n", (void *)&local);

    free(heap);
    return 0;
}
```

실행 결과:
```sh
$ ./mem_address
location of code  : 0x5a05c23e31c9
location of global: 0x5a05c23e6010
location of heap  : 0x5a05d22d72a0
location of stack : 0x7fff615e7b3c

$ ./mem_address
location of code  : 0x5e34e4f001c9
location of global: 0x5e34e4f03010
location of heap  : 0x5e3501fcf2a0
location of stack : 0x7ffeedc4ec7c
```

1. 이 실행에서는 code와 global data가 비교적 낮은 주소 범위에 있고 heap은 그 위쪽에 있으며 stack은 높은 주소 범위에 나타난다.
2. 이 주소들은 모두 프로그램이 관찰하는 virtual address다. 출력값만으로 해당 데이터가 physical memory의 어떤 frame에 존재하는지는 알 수 없다.

두 번의 실행 결과에서 절대 주소가 달라지는 것은 ASLR이 실행마다 executable, heap, stack 등의 배치를 바꾸기 때문이다. 또한 주소 공간의 정확한 배치와 각 영역 사이의 간격은 실행 환경, ABI, ASLR, loader 정책에 따라 달라질 수 있다.

#### 참고: `/proc/<pid>/maps`로 가상 주소 공간 확인하기

실행 중인 프로세스의 virtual memory mapping과 접근 권한은 `/proc/<pid>/maps`에서 확인할 수 있다. 이 파일에는 실행 파일, `[heap]`, shared library, `[stack]` 등이 어떤 virtual address 범위에 어떤 권한으로 mapping되어 있는지 나온다.

```
$ grep -E 'mem_address|\[heap\]|\[stack\]|libc\.so' /proc/<pid>/maps
```

```sh
# 시작 주소-끝 주소 권한 파일 offset ... 매핑된 파일 또는 영역
$ grep -E 'mem_address|\[heap\]|\[stack\]|libc\.so' /proc/11806/maps
6156df7ef000-6156df7f0000 r--p 00000000 08:30 76162                      /home/daeun/CSocrates2/OS/examples/mem_address
6156df7f0000-6156df7f1000 r-xp 00001000 08:30 76162                      /home/daeun/CSocrates2/OS/examples/mem_address
6156df7f3000-6156df7f4000 rw-p 00003000 08:30 76162                      /home/daeun/CSocrates2/OS/examples/mem_address
6156e717d000-6156e719e000 rw-p 00000000 00:00 0                          [heap]
...
783c65828000-783c659b0000 r-xp 00028000 08:30 58206                      /usr/lib/x86_64-linux-gnu/libc.so.6
783c65a03000-783c65a05000 rw-p 00202000 08:30 58206                      /usr/lib/x86_64-linux-gnu/libc.so.6
...
7ffd8e502000-7ffd8e524000 rw-p 00000000 00:00 0                          [stack]
```
- `r-xp` 구간에는 실행 가능한 code 영역이 포함된다.
- `r--p` 구간에는 ELF header, read-only data, relocation 이후 read-only로 바뀐 영역 등이 포함될 수 있다.
- `rw-p` 구간에는 초기화된 global/static 변수와 BSS처럼 read/write 가능한 data 영역이 포함된다.

### 2.2. Global/static 데이터의 형태: `.data`, `.bss`, `.rodata`

실행 파일에 포함되는 global/static data도 모두 같은 곳에 놓이지는 않는다. 초기화 여부와 수정 가능 여부에 따라 `.data`, `.bss`, `.rodata`처럼 서로 다른 ELF section에 배치된다.

| Section       | 데이터                                      | 초기화 방식             | 실행 파일에 저장되는 내용         | 접근 권한      |
| ------------- | ---------------------------------------- | ------------------ | ---------------------- | ---------- |
| **`.data`**   | 초기값이 있는 global/static 변수<br>수정 가능한 전역 배열 | 선언한 초기값으로 시작       | 초기값 바이트가 저장됨           | read/write |
| **`.bss`**    | 초기값이 없거나 `0`으로 초기화한 global/static 변수     | 프로그램 시작 시 0으로 초기화  | 필요한 크기만 기록됨 (`NOBITS`) | read/write |
| **`.rodata`** | 문자열 리터럴<br>`const` global 배열과 상수 등 | 선언한 read-only 초기값으로 시작 | 초기값 바이트가 저장됨           | read-only  |

```c
// data_layout.c

#include <stdlib.h>

int        g_init   = 10;         // 초기화된 전역       → .data
int        g_uninit;              // 초기화되지 않은 전역 → .bss
const char read_only[] = "hello"; // 읽기 전용 문자열 → .rodata
char writable[] = "hello";        // 수정 가능한 배열 → .data
static int s_init   = 5;          // 초기화된 정적       → .data

int main(void)
{
    static int s_uninit;          // 초기화되지 않은 정적 → .bss
    int        local = 3;         // 지역 변수            → stack
    int       *heap  = malloc(16); // 동적 할당            → heap
    if (heap == NULL)
        return 1;

    (void)local;
    (void)s_uninit;
    free(heap);

    return 0;
}
```

```bash
$ gcc -Wall -Wextra -O0 -g data_layout.c -o data_layout

# ELF section의 이름과 크기 확인
$ readelf -SW data_layout | grep -E '\.(text|rodata|data|bss)'

# 변수 심볼이 어느 종류의 section에 들어갔는지 확인
$ nm -S --defined-only data_layout | \
	grep -E 'g_init|g_uninit|read_only|writable|s_init|s_uninit'
```

실행 결과:
```sh
$ readelf -SW data_layout | grep -E '\.(text|rodata|data|bss)'
  [16] .text             PROGBITS        0000000000001080 001080 00012b 00  AX  0   0 16
  [18] .rodata           PROGBITS        0000000000002000 002000 00000a 00   A  0   0  4
  [25] .data             PROGBITS        0000000000004000 003000 000020 00  WA  0   0  8
  [26] .bss              NOBITS          0000000000004020 003020 000010 00  WA  0   0  4

$ nm -S --defined-only data_layout | grep -E 'g_init|g_uninit|read_only|writable|s_init|s_uninit'
0000000000004010 0000000000000004 D g_init
0000000000004028 0000000000000004 B g_uninit
0000000000002004 0000000000000006 R read_only
000000000000401c 0000000000000004 d s_init
0000000000004024 0000000000000004 b s_uninit.2509
0000000000004014 0000000000000006 D writable
```

선언한 변수들은 예상한 section에 배치되어 있다.

`readelf` 결과를 보면 각 section의 성격도 드러난다. `.data`와 `.bss`에는 쓰기 가능을 뜻하는 `W` flag가 있고 `.rodata`는 read-only다. 특히 `.bss`는 `NOBITS`로 나타난다. 초기값을 지정하지 않은 변수들은 프로그램 시작 시 0으로 초기화되지만, 실행 파일 안에 0으로 채운 byte가 그대로 저장되지는 않는다. 실행 파일에는 필요한 크기만 기록되고, 실제 영역은 프로그램을 load할 때 준비된다.

| `nm` 표시 | 해당 변수                | 배치된 section | 의미                         |
| ------- | -------------------- | ----------- | -------------------------- |
| `D`     | `g_init`, `writable` | `.data`     | 초기값이 있고 실행 중 수정 가능한 전역 데이터 |
| `d`     | `s_init`             | `.data`     | 초기값이 있는 `static` 데이터       |
| `B`     | `g_uninit`           | `.bss`      | 초기값을 지정하지 않은 전역 데이터        |
| `b`     | `s_uninit.2509`      | `.bss`      | 초기값을 지정하지 않은 `static` 데이터  |
| `R`     | `read_only`          | `.rodata`   | 수정할 수 없는 read-only 데이터         |
`local`과 `heap`은 ELF section에서 직접 확인되지 않는다. `local`은 `main()`이 실행될 때 user stack의 stack frame 안에 만들어지고 `malloc()`이 반환하는 영역은 실행 중 heap 또는 anonymous mapping에서 확보된다.

## 3. Memory API

### 3.1. 동적 메모리 할당 API

사용자 프로그램은 `malloc()`과 `free()`로 동적 메모리를 다룬다. 다만 `malloc(3)` 자체는 system call이 아니라 user-space allocator가 제공하는 **library function**이다.

- Library function: 일반적으로 user space에서 실행되며 필요한 경우에만 system call로 kernel 기능을 요청한다.
- System call: address space 관리, file I/O, process 생성처럼 kernel만 할 수 있는 작업을 요청하는 interface다.

프로그램이 필요한 크기의 메모리를 요청하면 allocator는 이미 확보해 둔 heap 또는 anonymous mapping 안에서 적절한 block을 찾아 돌려준다. 기존 공간이 부족할 때에만 `brk(2)` 또는 `mmap(2)`으로 kernel에 추가 address space를 요청한다.

- Anonymous: 특정 파일과 연결되지 않은 메모리 영역

```c
#include <stdlib.h>

void *malloc(size_t size);                        // size 바이트 할당. 초기화되지 않음
void *calloc(size_t nmemb, size_t size);          // nmemb * size 바이트 할당 후 0으로 초기화
void *realloc(void *_Nullable ptr, size_t size);  // 기존 block을 size 바이트로 조정
void *reallocarray(void *_Nullable ptr, size_t nmemb, size_t size);

// RETURN VALUE
//	On Success, return a pointer to the allocated memory
//	On Error,   return NULL and set errno

void free(void *_Nullable ptr);                     // block 해제

// RETURN VALUE
//	returns no value, and preserves errno
```


`malloc()`이 반환한 영역의 초기 내용은 알 수 없으므로 읽기 전에 직접 값을 저장해야 한다.
반면 `calloc()`은 할당한 바이트를 0으로 초기화한다.

배열 크기를 계산할 때 `malloc(n * size)`는 `n * size`의 정수 overflow를 자동으로 검사하지 않는다.
배열을 할당할 때는 `calloc()` 또는 `reallocarray()`가 더 안전하다.

`realloc()`의 두 번째 인자는 "추가할 크기"가 아니라 **조정 후 전체 크기**다. 실패하면 기존 block은 그대로 유지되므로 반환값을 기존 포인터에 바로 대입하지 말고 임시 포인터를 사용해야 한다.

### 3.2. 할당 실패와 Linux OOM

#### 할당 실패와 `errno`

동적 할당 함수의 실패 여부는 **반환값이 `NULL`인지**로 판단한다. `errno`는 실패 원인을 확인하거나 오류 메시지를 출력하기 위한 보조 정보다.

```c
#include <errno.h>
#include <stdio.h>
#include <stdlib.h>

void *allocate_buffer(size_t size)
{
    void *p = malloc(size);

    if (p == NULL) {
        perror("malloc");
        return NULL;
    }

    return p;
}
```

Linux에서 `malloc()`, `calloc()`, `realloc()` 등이 실패하면 보통 `errno`는 `ENOMEM`으로 설정된다. 이는 단순히 시스템 전체 RAM이 부족하다는 뜻만은 아니다.

| 가능한 원인             | 의미                              |
| ------------------ | ------------------------------- |
| physical memory 또는 swap 부족  | 새 page를 확보하기 어려운 상황              |
| `RLIMIT_AS` 제한     | 프로세스가 사용할 수 있는 virtual address space 제한 초과   |
| `RLIMIT_DATA` 제한   | data segment 및 heap 관련 제한 초과    |
| `max_map_count` 제한 | 프로세스가 만들 수 있는 mapping 수 제한 초과   |
| 너무 큰 요청            | 표현 가능한 객체 크기 또는 allocator 제한 초과 |

#### 할당 성공이 실제 사용 가능성을 보장하지는 않는다

Linux는 기본적으로 optimistic overcommit 정책을 쓸 수 있다. 이 경우 `malloc()`이 `NULL`이 아닌 값을 반환했다고 해서 그 크기만큼의 physical memory가 이미 확보되었다는 뜻은 아니다.

virtual address space 예약에는 성공해 포인터를 받았더라도 그 page에 처음 쓰는 시점에 physical page를 확보하지 못하면 시스템은 OOM(out of memory) 상태에 빠지고 OOM killer가 프로세스를 종료할 수 있다.

```
malloc(큰 크기)
   ↓
virtual address space 예약 성공
   ↓
포인터 반환 (non-NULL)
   ↓
실제 page에 처음 쓰기
   ↓
page fault 발생
   ↓
physical page 확보 실패 가능
   ↓
OOM killer가 프로세스를 종료할 수 있음
```

따라서 `NULL` 검사는 반드시 해야 한다. 하지만 `malloc()`의 성공만으로 프로그램이 이후에도 그 메모리를 끝까지 쓸 수 있다고 보장할 수는 없다. 이는 demand paging과 Linux overcommit 정책이 함께 작동한 결과다.

##### Overcommit 정책과 OOM Killer 동작 확인

|항목|의미|
|---|---|
|`/proc/sys/vm/overcommit_memory`|overcommit 정책 선택 (0: heuristic, 1: always, 2: strict)|
|`/proc/<pid>/oom_score_adj`|OOM killer가 프로세스를 선택할 때의 가중치 조정|

OOM killer는 단순히 가장 큰 프로세스를 종료하지 않는다. 시스템 전체의 메모리 부족인지, 특정 cgroup, cpuset, memory policy 안에서의 부족인지 확인하고 각 프로세스의 메모리 사용량과 `oom_score_adj` 같은 조정값을 반영해 victim을 고른다. 선택된 프로세스에는 `SIGKILL`이 전달되고, 회수 가능한 메모리를 빨리 정리해 다음 allocation이 진행될 여지를 만든다.

### 3.3. Multithreaded Application: allocator 보호와 객체 보호

#### thread는 무엇을 공유하는가

프로세스는 하나의 address space를 가지며 그 안에서 여러 thread가 동시에 실행될 수 있다. 같은 프로세스에 속한 thread들은 heap, global variable, 열린 file descriptor, `mmap` 영역처럼 address space 대부분을 공유한다. 반면 각 thread는 자신의 stack과 register를 따로 가진다.

공유는 곧 race의 원인이 된다. 두 thread가 같은 자원을 동시에 수정하면 자료구조가 일관성을 잃을 수 있으므로 공유 자원은 보호가 필요하다. 동적 메모리에서 보호 대상은 두 층으로 나뉜다.

| 보호 대상                                  | 누가 보호하는가        |
| -------------------------------------- | --------------- |
| allocator 내부 자료구조 (free list, arena 등) | glibc allocator |
| 프로그램이 공유하는 포인터와 객체의 소유권/수명             | 프로그램            |

#### allocator 내부: glibc가 보호한다

`malloc()`, `free()`, `calloc()`, `realloc()` 자체는 멀티스레드 환경에서 호출할 수 있다. 여러 thread가 동시에 할당/해제하면 free list 같은 내부 메타데이터에서 경쟁이 생긴다. glibc allocator는 이를 mutex로 보호한다.

모든 thread가 단일 mutex 하나를 두고 경쟁하면 그 자체가 병목이 된다. 그래서 contention이 감지되면 glibc는 **memory allocation arena**를 추가로 만든다. arena는 allocator가 관리하는 큰 메모리 영역이며 각 arena는 독립된 mutex를 가진다. arena의 backing memory는 kernel로부터 `brk()` 또는 `mmap()`으로 확보한다. 즉 allocator는 kernel에서 받은 address space 위에서 영역을 관리한다.

thread들이 서로 다른 arena를 사용하면 같은 mutex를 두고 경쟁하지 않으므로 contention이 줄어든다. 따라서 다음처럼 `malloc()` 호출 자체를 별도 mutex로 감쌀 필요는 없다.

```c
void *p = malloc(1024);   // allocator 내부에서 동기화 처리
free(p);
```

#### 프로그램이 공유하는 객체: 프로그램이 보호한다

만약 여러 thread가 **같은 포인터 변수 또는 같은 객체를 공유**한다면, 그 객체의 소유권과 접근 범위는 allocator가 보호하지 않으므로 프로그램이 직접 동기화해야 한다.

```c
#include <pthread.h>
#include <stdlib.h>

static pthread_mutex_t buffer_lock = PTHREAD_MUTEX_INITIALIZER;
static int *shared_buffer;

void replace_buffer(size_t count)
{
    int *new_buffer = calloc(count, sizeof(*new_buffer));
    if (new_buffer == NULL)
        return;

    pthread_mutex_lock(&buffer_lock);

    int *old_buffer = shared_buffer;
    shared_buffer = new_buffer;

    pthread_mutex_unlock(&buffer_lock);

    free(old_buffer);
}
```

`shared_buffer`를 읽거나 사용하는 다른 thread도 같은 `buffer_lock`을 사용해야 한다. 잠금 없이 다른 thread가 이전 포인터를 들고 있는 상태에서 `free(old_buffer)`가 실행되면 use-after-free가 발생할 수 있다.

### 3.4. `brk()`와 `mmap()`: allocator가 주소 공간을 확보하는 방식

allocator는 사용자 요청마다 곧바로 kernel에 system call을 보내지 않는다. 이미 확보한 큰 영역을 작은 block으로 나누어 관리하다가 공간이 부족할 때만 추가 address space를 요청한다.

```
malloc(size)
   ↓
allocator 내부 free block 탐색
   ↓
사용 가능한 block이 있으면 분할 후 반환
   ↓
없으면 heap 확장 또는 새 mapping 생성
   ↓
새 영역을 다시 작은 block으로 분할
```

```c
#include <unistd.h>

int   brk(void *addr);            // 힙 끝(break)을 addr로 설정
void *sbrk(intptr_t increment);   // 힙을 increment만큼 증감
```
```
 RETURN VALUE
    On success, brk() returns zero.
    On error, -1 is returned, and errno is set to ENOMEM.

    On success, sbrk() returns the previous program break.
	       (If the break was increased, then this  value  is  a  pointer
	       to  the  start  of  the  newly  allocated memory).
	On error, (void *) -1 is returned, and errno is set to ENOMEM.
```

program break는 프로세스 data segment의 끝을 가리킨다. 이를 증가시키면 heap이 확장되고 감소시키면 heap의 일부를 반환하는 효과가 난다.

다만 일반 애플리케이션이 `brk()`나 `sbrk()`를 직접 호출하는 것은 권장하지 않는다. allocator도 같은 heap 경계를 관리하기 때문에 프로그램이 임의로 break를 움직이면 allocator의 내부 상태와 충돌할 수 있다.

또 다른 방식은 `mmap()`이다.
```c
#include <sys/mman.h>

void *mmap(void addr[.length], size_t length,
			int prot, int flags,
            int fd, off_t offset);

int munmap(void addr[.length], size_t length);
```

```
RETURN VALUE
	On success, mmap() returns a pointer to the mapped area.
	On error, the value MAP_FAILED
       (that is, (void *) -1) is returned, and errno is set to indicate the error.

    On success, munmap() returns 0.
    On failure, it returns -1, and errno is set  to  indicate the error (probably to EINVAL).
```

`mmap()`은 크게 두 가지 용도로 쓰인다.

|방식|주요 flag|의미|
|---|---|---|
|file-backed mapping|`MAP_PRIVATE` 또는 `MAP_SHARED`|파일 내용을 주소 공간에 매핑|
|anonymous mapping|`MAP_ANONYMOUS`|파일 없이 0으로 초기화된 익명 영역 생성|

`MAP_PRIVATE`는 Copy-on-Write 방식의 private mapping이다. 프로세스가 매핑한 데이터를 수정해도 원본 파일과 다른 프로세스의 mapping에는 바로 반영되지 않는다.

`MAP_SHARED`는 수정 내용이 같은 파일을 mapping한 다른 프로세스에 보일 수 있으며 파일에도 반영될 수 있다.

현재 glibc allocator는 일반적으로 heap을 `sbrk()`로 확장하고, 일정 기준보다 큰 요청은 private anonymous `mmap()`으로 처리할 수 있다. 다만 이 기준은 allocator 구현과 설정에 따라 달라지는 내부 정책이므로 프로그램이 특정 크기를 기준으로 동작을 가정하면 안 된다.

`brk()`와 `mmap()`이 성공한 직후에도 모든 physical page가 즉시 준비되는 것은 아니다. 실제 page는 해당 주소에 처음 접근할 때 page fault를 통해 연결될 수 있다.

## 4. 주소 변환의 발전: Base/Bounds → Segmentation → Paging

### 4.1. Base & Bounds: 주소 공간 전체를 재배치

각 프로세스는 주소가 0부터 시작한다고 믿지만 실제 프로그램은 physical memory의 임의 위치에 올라간다. 이 간극은 base register로 메운다. 프로그램이 virtual address `VA`를 만들면 하드웨어는 base register를 더해 physical address를 만든다.

```text
PA = base + VA
```

여기에 bounds(limit) register를 두면 `VA`가 허용 범위 안에 있는지 검사한다. `VA >= bound`이면 예외를 발생시켜 protection을 제공한다. base와 bounds register는 privileged mode에서만 변경할 수 있다.

이 방식의 장점은 단순함이다. 같은 프로그램을 physical memory 어디에나 올릴 수 있고, 프로세스는 0번지부터 시작하는 것처럼 보인다. 하지만 address space **전체를 연속된 physical memory에 통째로** 올려야 한다. Code와 stack/heap 사이에 큰 빈 공간이 있어도 그만큼 physical memory를 잡아 두므로 낭비가 크다.

> 참고: IA-32/x86의 역사적 segmentation은 segment register와 descriptor table을 통해 base, limit, 권한을 표현한다. **현대 x86-64 user space address translation의 중심은 paging**이다.

### 4.2. Segmentation: 실제로 쓰는 구간만 배치하기

Segmentation은 code/heap/stack 같은 **논리적 segment마다** 별도의 base/bounds/권한 정보를 둔다. 실제로 쓰는 영역만 physical memory에 놓을 수 있어 address space 내부의 빈 공간 낭비가 줄어든다.

- virtual address의 상위 bit로 어느 segment인지 구분한다.
- 하위 bit는 그 segment 안에서의 offset으로 사용한다.
- segment마다 권한을 다르게 줄 수 있어 code 영역을 read-only로 공유할 수 있다.
- stack처럼 반대 방향으로 자라는 segment는 확장 방향 정보가 필요하다.

그러나 segment는 가변 크기다. 여러 segment가 생성되고 삭제되기를 반복하면 physical memory 곳곳에 작은 빈틈이 생긴다. 전체 빈 공간의 합은 충분해도 큰 연속 영역이 없어서 새 segment를 배치하지 못하는 **external fragmentation**이 발생한다.

### 4.3. Free-space management: 가변 크기 할당의 비용

가변 크기 block을 다루는 한 external fragmentation은 피하기 어렵다. 그래서 새 요청을 어느 빈칸에 넣을지 결정하는 정책이 필요하다.

|정책|선택 기준|특징|
|---|---|---|
|First Fit|처음 발견한 충분한 빈 block|빠르지만 앞부분 단편화 가능|
|Best Fit|요청 크기에 가장 가까운 빈 block|작은 잔여 조각을 많이 만들 수 있음|
|Worst Fit|가장 큰 빈 block|큰 block을 남기려 하지만 효율이 보장되지는 않음|
|Next Fit|이전 탐색 위치부터 재개|첫 부분만 반복 탐색하지 않음|

- **splitting**: 큰 free block을 요청 크기와 잔여 block으로 나눈다.
- **coalescing**: 해제된 이웃 free block을 합쳐 큰 요청을 받을 수 있게 한다.
- **buddy system**: 2의 거듭제곱 크기로 블록을 관리하여 짝 buddy와의 병합을 단순화한다.

이 문제의 핵심은 "가변 크기 block"이다. 다음 단계인 paging은 **address space와 physical memory를 모두 고정 크기 단위로 나누어** external fragmentation을 없앤다.

## 5. Paging과 Page Table: 현대 가상 메모리

Paging은 virtual address space를 고정 크기 **page**로 나누고 physical memory를 같은 크기의 **page frame**으로 나눈다. 어떤 virtual page도 같은 크기의 임의 frame에 들어갈 수 있으므로 segmentation에서 문제가 됐던 external fragmentation이 사라진다. 다만 마지막 page가 완전히 채워지지 않는 정도의 **internal fragmentation**은 남을 수 있다.

|문제|페이징의 해결 방식|
|---|---|
|세그먼트의 가변 크기|모든 단위를 고정 크기로 통일|
|외부 단편화|어떤 page든 어떤 빈 frame에 배치 가능|
|보호와 공유 단위의 부재|page 단위로 권한과 공유 상태 설정|
|일부만 load하기 어려움|page 단위로 필요한 부분만 load하고 교체 가능|

### 5.1. VPN, Offset, PFN

page 크기가 `2^p` byte라면 virtual address의 하위 `p` bit는 page 안에서의 위치인 **offset**이고 나머지 상위 bit는 **VPN(Virtual Page Number)**이다.

```text
virtual address = [ VPN | offset ]
                      │
                      └─ page table lookup
                              │
physical address = [ PFN | offset ]
```

page 크기를 2의 거듭제곱으로 정하면 VPN과 offset을 나누는 일이 단순한 bit 분리가 된다. 예를 들어 4 KiB page는 `2^12` byte이므로 offset은 12 bit다.

### 5.2. Page Table Entry(PTE)에 들어 있는 정보

프로세스마다 page table이 있고 VPN은 그 table의 index가 된다. PTE에는 PFN뿐 아니라 다음과 같은 상태 정보가 들어간다.

|PTE 정보|역할|
|---|---|
|present / valid|physical memory에 현재 load되어 있는가|
|read / write / execute|접근 권한|
|user / supervisor|사용자 모드 접근 가능 여부|
|accessed / referenced|최근 참조 흔적, 회수 정책의 입력|
|dirty|load 후 수정 여부, writeback 판단의 입력|

여기서 **TLB miss**와 **page fault**를 구분해야 한다.

|상황|뜻|일반적 처리|
|---|---|---|
|TLB miss|TLB에 변환 cache가 없음|page table walk를 통해 변환을 찾고 TLB를 채운다.|
|page fault|변환이 없거나 권한이 맞지 않거나 page가 RAM에 없음|예외 진입 후 kernel이 page-in, COW, 권한 오류를 처리한다.|

TLB miss는 정상적인 cache miss일 수 있다. 반면 page fault는 page 부재, COW, lazy allocation, 잘못된 권한 접근 등 여러 원인으로 발생하는 예외다.

### 5.3. TLB: 페이지 테이블 접근 비용을 줄이는 변환 캐시

Paging의 약점은 address translation 자체에도 메모리 접근이 필요하다는 점이다. page table도 메모리에 있으므로 TLB가 없다면 데이터를 한 번 읽기 전에 page table을 먼저 읽어야 한다. 그래서 MMU 안에는 자주 사용하는 VPN→PFN 변환을 cache하는 **TLB(Translation Lookaside Buffer)**가 있다.

|상황|처리|
|---|---|
|**TLB hit**|TLB에 VPN→PFN 변환이 있어 즉시 physical address를 만들고 메모리 접근으로 진행한다.|
|**TLB miss**|page table을 탐색해 변환을 얻은 뒤 TLB를 갱신한다.|

TLB도 cache이므로 **locality**에 의존한다.

```c
// 배열을 순차적으로 훑으면 같은 page의 원소들이 연달아 접근된다.
long sum = 0;
for (size_t i = 0; i < n; i++) {
    sum += a[i];
}
```

- **공간 지역성(spatial locality)**: `a[i]`를 본 뒤 같은 page 안의 `a[i+1]`을 곧 보므로, page 첫 접근 뒤에는 TLB hit가 이어질 가능성이 높다.
- **시간 지역성(temporal locality)**: 같은 code, loop variable, working set을 가까운 시간 안에 다시 접근하면 기존 변환이 TLB에 남아 있을 수 있다.

TLB miss 처리 방식은 아키텍처마다 다르다.

|방식|처리 주체|예|
|---|---|---|
|**HW-managed**|하드웨어가 page table walk 후 TLB를 채움|x86|
|**SW-managed**|하드웨어는 예외만 발생시키고 OS가 핸들러에서 처리|MIPS 계열 등|

문맥 전환으로 address space가 바뀌면 이전 프로세스의 변환 정보는 새 프로세스에 맞지 않는다. 프로세스 P1의 VPN 10과 P2의 VPN 10은 서로 다른 PFN을 가리킬 수 있으므로, 잘못된 TLB entry가 남으면 다른 프로세스의 메모리를 가리켜 protection이 깨진다. 이를 막는 방법은 두 가지다. 문맥 전환 때 TLB를 flush하거나, TLB entry에 ASID 또는 x86의 PCID 같은 address space identifier를 넣어 같은 VPN이라도 어느 address space의 변환인지 구분한다. flush는 단순하지만 다음 실행 때 TLB miss가 늘고, 식별자 방식은 그 비용을 줄이지만 하드웨어와 kernel의 협력이 필요하다.

### 5.4. 페이지 테이블이 커질 때: 다단계 테이블과 huge page

단일 선형 page table은 넓은 virtual address space 전체에 대해 PTE를 만들어야 하므로 실제로 쓰지 않는 영역이 많아도 공간을 낭비한다. 다단계 page table은 상위 table에서 하위 table을 필요할 때만 만들기 때문에 비어 있는 큰 주소 범위에는 하위 table 자체를 만들지 않는다.

리눅스의 일반화된 계층은 다음과 같다.

```text
PGD → P4D → PUD → PMD → PTE
```

architecture가 모든 단계를 실제로 사용하지 않는 경우에는 일부 단계가 folded 된다. 또한 PMD나 PUD 단계에서 더 큰 연속 physical memory 영역을 직접 mapping하는 huge page를 쓸 수 있다. huge page는 TLB entry 하나가 더 넓은 범위를 덮게 해 TLB pressure와 page table overhead를 줄인다. 대신 더 큰 연속 physical memory 영역이 필요하고 internal fragmentation 가능성도 커진다.

## 6. Demand Paging: 물리 메모리보다 큰 주소 공간

모든 virtual page를 프로세스 시작 시 RAM에 올릴 필요는 없다. 실제로 접근될 때만 page를 준비하면 더 많은 프로세스를 메모리에 남겨 둘 수 있고 physical memory보다 큰 address space도 제공할 수 있다. 예전의 overlay 기법에서는 프로그래머가 code/data 일부를 직접 내리고 올려야 했지만, demand paging에서는 운영체제가 이 일을 투명하게 처리한다.

### 6.1. page fault의 기본 흐름

virtual address space를 만들 때 모든 page에 즉시 physical memory를 붙일 필요는 없다. 운영체제는 우선 사용할 주소 범위와 접근 권한만 등록해 두고 프로세스가 실제로 **해당 주소를 처음 읽거나 쓸 때** 필요한 page를 준비할 수 있다.

```text
PTE present bit = 0
   ↓
page fault 발생
   ↓
OS가 physical page 확보 또는 원본에서 복구
   ↓
PTE 갱신 (present bit = 1)
   ↓
멈췄던 명령 재실행
```

여기서 page fault는 단순히 "잘못된 접근"을 뜻하지 않는다. 유효한 virtual address이지만 아직 physical page가 연결되지 않은 경우에도 page fault가 발생한다. 운영체제는 이 fault를 처리하면서 필요한 내용을 메모리에 준비한다. 반대로 주소 범위가 존재하지 않거나 read-only page에 쓰기를 시도하는 등 접근이 허용되지 않는 경우에는 복구할 수 없는 fault로 처리된다.

### 6.2. 유효한 fault와 잘못된 접근

```text
CPU가 virtual address에 접근
  ↓
PTE 또는 권한 확인 실패 → page fault 예외
  ↓
VMA 존재와 권한 검사
  ↓
anonymous page / file-backed page / COW / 잘못된 접근 분기
  ↓
physical frame 확보 또는 기존 page 복구
  ↓
PTE 갱신, TLB 관련 처리
  ↓
멈췄던 명령 재실행
```

page fault는 반드시 오류가 아니다. 다음은 정상적인 fault의 대표 사례다.

|종류|왜 fault가 발생하는가|이후 처리|
|---|---|---|
|Lazy allocation|주소 범위는 만들었지만 아직 실제 frame을 붙이지 않음|0으로 채운 새 page를 mapping|
|File-backed fault|실행 파일, shared library, `mmap` 파일의 해당 page가 아직 RAM에 없음|파일 page cache에서 읽거나 I/O 요청|
|Copy-on-Write fault|부모와 자식이 read-only로 page를 공유하다가 한쪽이 쓰려 함|새 page 복사 후 writable mapping|
|Swap-in fault|이전에 swap으로 내보낸 anonymous page에 다시 접근|swap에서 읽어 frame에 복구|

COW도 같은 page fault 메커니즘을 사용한다. `fork()` 직후 부모와 자식은 같은 physical page를 read-only로 공유한다. 이후 둘 중 한쪽이 page에 쓰기를 시도하면 protection fault 형태의 page fault가 발생한다. 운영체제는 새 physical page를 만든 뒤 기존 내용을 복사하여 해당 프로세스만 새 page를 쓰도록 연결한다. 즉 COW는 비상주 page를 메모리로 가져오는 경우와 달리 **이미 존재하는 공유 page를 독립된 writable page로 분리**하는 경우다.

반대로 mapping되지 않은 주소를 접근하거나 read-only page에 잘못 쓰면 kernel은 사용자 프로세스에 `SIGSEGV` 같은 signal을 보낸다.

### 6.3. Swap

당장 필요하지 않은 page는 disk의 swap 공간으로 내보냈다가(swap out) 다시 필요해지면 가져올 수 있다(swap in). 이 때문에 시스템은 실제 physical memory보다 큰 address space를 제공할 수 있다.

다만 모든 page가 swap으로만 오가는 것은 아니다. file-backed page는 원본 파일에서 다시 읽을 수 있고 깨끗한 file page라면 별도 기록 없이 버려도 된다. 반면 anonymous page는 대응하는 원본 파일이 없으므로 변경된 내용을 나중에 되살려야 한다면 swap 같은 저장 공간이 필요하다.

### 6.4. Example: Copy-on-Write

```c
// cow_demo.c
#include <stdio.h>
#include <stdlib.h>
#include <sys/types.h>
#include <sys/wait.h>
#include <unistd.h>

int main(void)
{
    int value = 1;
    pid_t pid = fork();
    if (pid < 0) {
        perror("fork");
        return 1;
    }

    if (pid == 0) {
        value = 99; // 이 쓰기가 COW 분리를 유발할 수 있음
        printf("child : value=%d, address=%p\n", value, (void *)&value);
        fflush(stdout);
        _exit(0);
    }

    if (waitpid(pid, NULL, 0) == -1) {
        perror("waitpid");
        return 1;
    }
    printf("parent: value=%d, address=%p\n", value, (void *)&value);
    return 0;
}
```

|함수 시그니처|역할|RETURN VALUE|
|---|---|---|
|`pid_t fork(void);`|현재 프로세스를 복제해 자식 프로세스를 생성.|부모: 자식 PID / 자식: `0` / 오류: `-1`|
|`pid_t waitpid(pid_t pid, int *status, int options);`|지정한 자식의 상태 변화를 대기.|성공: 상태가 변한 자식 PID / 오류: `-1`|

부모와 자식에서 지역 변수의 **virtual address**는 보통 같아 보일 수 있지만 자식이 값을 쓰면 두 프로세스는 서로 독립된 결과를 유지한다. 핵심은 `fork()` 직후 모든 page를 즉시 복사하지 않고 공유한 뒤 첫 쓰기 시점에만 복사한다는 점이다. 대부분의 `fork()` 뒤에는 `exec()`가 이어지므로 이 지연 복사가 불필요한 대량 복사를 막는다.

## 7. 메모리가 부족해지면: 교체 정책과 Thrashing

physical memory는 전체 virtual address space의 일부만 보관하는 빠른 계층이다. 사실상 **cache처럼** 동작한다. 새 page를 올려야 하는데 여유 frame이 없으면 운영체제는 현재 RAM에 있는 page를 골라 회수(reclaim)하거나 필요하다면 swap으로 내보내야 한다.

교체 정책의 목표:

> 앞으로 덜 사용할 page를 가능한 한 잘 골라 내보내기

### 7.1. 왜 교체 정책이 중요한가

page fault의 비용은 일반 메모리 접근보다 매우 크다. 평균 메모리 접근 시간(AMAT)은 다음처럼 생각할 수 있다.

```text
AMAT = P_hit × T_M + P_miss × T_D
```

- `T_M`: 메모리 접근 시간
- `T_D`: disk 또는 매우 느린 저장장치에서 page를 복구하는 시간

메모리 접근이 약 100ns이고 disk 접근이 약 10ms라면 느린 쪽의 비용은 대략 `10^5`배 수준이다. 따라서 작은 miss 비율도 평균 성능을 크게 떨어뜨린다. paging이 실용적이려면 필요한 page가 대부분 RAM에 남아 있어야 하고 앞으로 **가까운 시간 안에 다시 쓸 가능성이 낮은 page**를 내보내는 것이 좋다.

|정책|희생 page|장점과 한계|
|---|---|---|
|OPT|미래에 가장 늦게 다시 참조될 page|이론적 최적이지만 미래를 알 수 없어 구현 불가|
|FIFO|가장 먼저 들어온 page|단순하지만 자주 쓰는 page도 내보낼 수 있음|
|Random|무작위 page|단순한 기준선, 특정 패턴에 덜 민감할 수 있음|
|LRU|가장 오래 참조되지 않은 page|지역성을 잘 이용하지만 정확한 구현 비용이 큼|
|Clock|참조 bit가 0인 page|LRU 근사. 1이면 0으로 내리고 다음에 다시 기회 부여|

FIFO, LRU, Clock은 "어떤 과거 정보를 미래 예측에 쓸 것인가"를 설명하기 위한 기본 model이다. 실제 Linux reclaim은 이 정책들을 그대로 구현하지 않고 anonymous/file page의 상태, 참조 흔적, dirty 상태, memory zone, cgroup, writeback 가능성 등을 함께 고려한다.

### 7.2. Dirty page, writeback, prefetch, clustering

교체에서는 "얼마나 최근에 썼는가"뿐 아니라 "내보내는 비용이 얼마나 큰가"도 중요하다.

- **clean page**: 원본 파일이 있거나 변경된 내용이 없으면 필요할 때 disk에서 다시 읽을 수 있으므로 비교적 싸게 회수할 수 있다.
- **dirty page**: RAM에 올라온 뒤 수정됐다. file-backed page라면 writeback이 필요할 수 있고 anonymous page라면 swap 영역에 저장해야 할 수 있다.

같은 조건이라면 clean page를 먼저 회수하는 편이 비용이 적다. page를 들이고 내보내는 타이밍에도 선택지가 있다.

|개념|의미|
|---|---|
|**demand paging**|실제 접근이 발생한 뒤 가져온다.|
|**prefetch**|곧 필요할 가능성이 높은 page를 미리 가져온다. 순차 접근처럼 성공 가능성이 높을 때만 이득이다.|
|**clustering**|여러 page의 I/O를 묶어 처리해 저장장치 접근 효율을 높인다.|
|**watermark**|메모리가 완전히 바닥난 뒤가 아니라, 여유가 일정 수준 아래로 내려가면 미리 회수한다.|

### 7.3. watermark와 direct reclaim

실제 운영체제는 빈 frame이 완전히 0이 된 뒤에만 회수를 시작하지 않는다.

```text
free memory가 low watermark 아래로 내려가면
  ↓
백그라운드 reclaim 스레드가 회수 시작
  ↓
free memory를 high watermark 근처까지 회복
  ↓
다시 대기
```

여유 메모리가 충분하지 않은데 어떤 프로세스가 즉시 page allocation을 요청하면 그 프로세스 자신이 reclaim에 참여하는 **direct reclaim**이 발생할 수 있다. 이때 allocation latency가 커질 수 있다.

### 7.4. Working Set, WSS, Thrashing

Working Set은 "이 프로세스가 지금 원활히 실행되려면 실제로 필요한 page"를 참조 이력으로 근사하는 개념이다. 시간 `t`에서 최근 window `Δ` 동안 참조한 page의 집합을 `W(t, Δ)`로 쓴다.

|구분|Working Set|Residence Set|
|---|---|---|
|의미|최근 `Δ` 동안 실제로 참조된 page 집합|지금 이 순간 RAM에 실제로 resident한 page 집합|
|기준|참조 이력과 locality|physical residency|
|크기|WSS(Working Set Size)|현재 배정 및 resident frame 수|
|이상적 관계|필요한 page 집합|`Residence Set ⊇ Working Set`|

- `Residence Set`이 `Working Set`을 포함하면 필요한 page가 RAM에 있으므로 page fault가 드물다.
- 반대로 프로세스가 자기 Working Set보다 적은 frame만 받으면 방금 내보낸 page를 곧 다시 필요로 하는 일이 반복된다.
- 모든 프로세스의 WSS 합이 physical memory를 넘으면 시스템이 실제 작업보다 paging에 더 많은 시간을 쓰는 **thrashing**에 빠진다.

이때 해결책은 단순히 page를 더 자주 교체하는 것이 아니다. 일부 프로세스를 잠시 실행 대상에서 빼 남은 프로세스의 Working Set이 RAM에 들어가게 하는 **admission control**이 더 나을 수 있다. Linux는 memory pressure가 끝내 해소되지 않으면 OOM killer로 victim을 선택하는 최후 수단도 사용한다.

## 8. 전체 흐름 정리

프로세스가 사용자 코드에서 메모리 주소 하나를 읽거나 쓰는 순간을 하나의 흐름으로 이어 보면 다음과 같다.

```text
1. 프로그램이 virtual address VA를 생성
   ↓
2. MMU가 TLB에서 VPN→PFN 변환을 찾음
   ├─ hit  → PFN + offset으로 physical memory 접근
   └─ miss → page table walk
                ├─ 유효한 PTE → TLB 갱신 후 접근
                └─ 부재/권한 문제 → page fault
                                      ↓
3. kernel이 VMA와 권한을 확인
   ├─ invalid access → SIGSEGV 등 오류 처리
   ├─ anonymous lazy allocation → 0으로 초기화된 새 page 또는 zero page 준비
   ├─ file-backed → page cache 또는 저장장치에서 읽기
   ├─ COW write → 새 page 복사 후 writable mapping
   └─ swapped out → swap에서 복구
                                      ↓
4. physical memory가 부족하면 reclaim / writeback / swap out
                                      ↓
5. PTE와 TLB 상태를 정리하고 원래 명령 재실행
```

처음의 질문으로 돌아가면, 프로세스가 어떤 주소에 접근할 때 운영체제는 다음을 차례로 보장한다.

1. 그 주소가 해당 프로세스에 허용된 주소인지 **보호**한다.
2. virtual address를 실제 frame으로 **변환**한다.
3. 아직 실제 page가 없다면 필요한 시점에 **채운다**.
4. RAM이 부족하면 덜 필요한 page를 **회수하거나 내보낸다**.

따라서 virtual memory는 단순히 주소를 다른 주소로 바꾸는 기능이 아니다. 독립된 address space라는 추상화에서 출발해, 제한된 physical memory 위에서 여러 프로세스가 안전하고 효율적으로 동작하게 만드는 운영체제의 핵심 체계다.

|단계|운영체제가 보장하는 것|
|---|---|
|address space|프로세스별 독립적 메모리 모델|
|page table + TLB|빠른 변환과 권한 검사|
|page fault|필요한 순간에만 실제 page를 준비|
|reclaim + swap|제한된 RAM을 여러 프로세스 사이에서 재분배|
|교체와 working set 정책|지역성을 이용해 fault 비용을 줄임|

## 9. (참고) CXL 메모리

지금까지는 "고정된 RAM을 운영체제가 어떻게 나누어 쓰고 유지하는가"에 집중했다. CXL(Compute Express Link)은 CPU에 가까운 DIMM slot만으로 확보하던 메모리 용량과 공유 방식의 한계를 넓히려는 interconnect다. PCIe 기반 위에서 CPU, memory device, accelerator 사이의 cache-coherent 통신을 지원하며 크게 세 가지 protocol로 나뉜다.

- **CXL.io**: device 구성과 I/O 성격의 통신
- **CXL.cache**: device가 host memory를 cache하면서 coherence를 유지하는 통신
- **CXL.mem**: host CPU가 CXL device memory를 load/store 방식으로 접근하는 통신

CXL을 사용하면 host에 메모리 용량을 더 붙이거나 여러 device가 메모리 자원을 더 유연하게 나눠 쓸 수 있다.

|방향|의미|
|---|---|
|**용량 확장**|host CPU가 device에 연결된 메모리를 접근해 가용 메모리를 늘림|
|**Pooling**|여러 host와 accelerator가 memory pool을 공유하고 동적으로 할당|
|**Disaggregation**|CPU와 memory를 물리적으로 분리하되 논리적으로 연결|

메모리 용량 확장, memory pooling, CPU와 memory의 물리적 분리는 data center와 대규모 AI/ML workload에서 특히 중요하다. 다만 CXL memory는 보통 local DDR보다 access latency가 크므로 운영체제는 NUMA policy, memory tier, placement policy를 함께 고려해야 한다.

이 관점에서 CXL은 page table과 demand paging을 대체하는 기술이 아니다. 오히려 "어떤 physical memory frame이 더 빠르고 가까운가"라는 계층을 넓히고, 기존 page placement와 reclaim policy에 새 제약을 더하는 확장 사례로 볼 수 있다.

## References

- Remzi H. Arpaci-Dusseau & Andrea C. Arpaci-Dusseau, _Operating Systems: Three Easy Pieces_.
    - Ch. 13: Address Spaces
    - Ch. 14: Memory API
    - Ch. 15-17: Address Translation, Segmentation, Free-Space Management
    - Ch. 18-20: Paging, TLBs, Smaller Tables
    - Ch. 21-22: Beyond Physical Memory: Mechanisms and Policies
- William Stallings, _Operating Systems: Internals and Design Principles_ - Memory Management, Virtual Memory
- 강의자료
    - `25-OS-LN#5-Ch13(주소공간).pdf`
    - `25-OS-LN#5-1(메모리레이아웃-가상주소공간).pdf`
    - `25-OS-LN#6(Chap14,15,16-MemoryAPI&Segmentation).pdf`
    - `25-OS-LN#7(Paging&).pdf`
    - `25-OS-LN#8(Chap18&19&20&21&22).pdf`
    - `25-OS-LN#8-1(WSS).pdf`
- [CXL Consortium](https://computeexpresslink.org/)
