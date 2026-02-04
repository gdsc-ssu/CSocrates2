# L4: Transport Layer
## 0. Host-to-Host vs. Process-to-Process
> Whereas a **transport-layer protocol** provides logical communication between **processes** running on different hosts, a **network-layer protocol** provides logical communication between **hosts.**
> - *Computer Networking, A Top Down Approach, 7th ed.*
#### L3(Network Layer)의 역할
- **Host-to-Host**: 패킷을 목적지 **호스트**까지 전달하는 계층 
	- $\rightarrow$ "어디까지 보낼 것인가"
- Best-effort 방식으로 패킷 전달하며, 순서나 도착 여부를 보장하지 않음.
#### L3의 한계
- 하나의 호스트에는 여러 네트워크 프로세스가 동시에 존재하므로
- IP 주소만으로는 어떤 프로세스가 수신자인지 알 수 없다.
## 1. L4(Transport Layer)의 역할과 기능
- Transport Layer는 **Process-to-Process** 논리적 통신을 제공하는 계층
- 서로 다른 호스트에서 실행 중인 **프로세스**를 연결
### 1.1. L3와 L4의 역할 분리

|        | Network Layer (L3) | Transport Layer (L4)     |
| ------ | ------------------ | ------------------------ |
| 통신 단위  | Host-to-Host       | Process-to-Process       |
| 식별     | `Destination IP`   | `(dest IP, dest Port)`   |
| 데이터 단위 | 패킷                 | 세그먼트 / 데이터 스트림           |
| 순서 보장  | 보장하지 않음            | 순서 관리 및 재정렬              |
| 전달 신뢰성 | Best-effort        | 전달 결과 확인 및 복구 시도         |
| 흐름 제어  | 없음                 | 수신 측 처리 능력에 맞춤           |
| 연결 관리  | 없음                 | 연결 생성, 유지, 종료 관리         |
| 상태 관리  | Stateless          | Stateful (Connection 유지) |
- Network Layer(L3)는 best-effort 방식으로 호스트 간 패킷 전달을 수행하는 계층
- Transport Layer(L4)는 논리적인 통신이 성립되도록 조건을 관리하는 계층

### 1.2. Transport Layer(L4)의 계층적 위치
#### 1.2.1. 개념적 분류
- 네트워크 계층의 일부
 - OSI/TCP-IP 모델에서 L4로 정의
- Network Layer 위에서 동작하는 상위 계층
- 논리적인 통신 단위 형성
	- 패킷을 실제로 전달하지 않음.
	- 전달된 패킷을 기반으로 애플리케이션 간 통신이 성립되도록 추상화 제공
#### 1.2.2. 구현 위치
- 네트워크 장비가 아닌 **각 호스트의 운영체제 커널**에서 구현
	- 포트, 연결 상태, 순서, 재전송, 흐름 제어를 다루기 위해서
	- 메모리, 타이머, 큐, 상태 머신과 같은 운영체제 자원을 지속적으로 관리
	- 모든 송수신은 시스템 콜을 통해 커널로 전달되며, 커널 내부의 로직을 거쳐 네트워크로 전달
		- $\rightarrow$ 애플리케이션은 네트워크에 직접 접근하지 않음.
## 2. Port와 Connection
### 2.1. Port as OS resource
-  Port는 네트워크 주소가 아닌 OS 자원
#### 2.1.1. Port가 필요한 이유
- 하나의 호스트에는 여러 네트워크 프로그램이 동시에 실행됨.
- IP는 호스트까지만 식별하므로, IP 주소만으로는 패킷을 올바른 프로세스로 전달할 수 없음.
- 커널은 수신 데이터를 적절한 **소켓**으로 분배해야 함 (Demultiplex)
#### 2.1.2. Port의 의미
- Port는 커널이 관리하는 식별자 네임스페이스
	- 여러 네트워크 수신 지점이 동일한 네트워크 네임스페이스 내의 IP 공간에서 충돌하지 않도록 함.
- 커널은 Port를 이용해 수신 데이터를 적절한 소켓으로 demultiplex하며, 해당 소켓은 특정 프로세스에 귀속됨.

### 2.2. Connection
- 논리적 개념이 아닌 커널 내부에 존재하는 **상태(state)** 의 묶음
- 통신을 지속하기 위해 커널이 생성, 유지, 정리하는 상태 자원
#### 2.2.1. Connection이 포함하는 커널 상태 
- Connection은 데이터 + 버퍼 + 번호 + 타이머 + 상태 머신이 결합된 구조
##### `socket` 객체
- 애플리케이션이 참조하는 커널 객체
###### `struct socket`: User space
- User Space의 `fd`와 커널 네트워크 스택을 이어주는 인터페이스
- 시스템콜의 진입점
- 파일 디스크립터를 통해 `socket` 객체에 접근
```c
// include/linux/net.h:116
struct socket { // the kernel representation of a BSD socket
	socket_state	state;
	short			type;
	unsigned long	flags;
	struct file		*file; // File Descriptor와 직접 연결
	struct sock		*sk; // Transport-level state - 실제 네트워킹 상태
	...
};
```
```c
// include/linux/fs.h:1258
// fd → struct file → struct socket → struct sock
struct file { // Represent a file
	spinlock_t			f_lock;
	fmode_t				f_mode;
	const struct file_operations *f_op; // read/write 함수 호출이 소켓 구현으로 디스패치
	
	...
	struct inode			*f_inode; // Socket File로 연결
	unsigned int			f_flags;
	...
};
```
###### `struct sock`: Kernel Space
- Transport Layer의 핵심 상태 객체
- 커널이 소켓을 찾아오기 위한 최소 정보
```c
// include/net/sock.h:151
/**
 *	struct sock_common - minimal network layer representation of sockets
 *	@skc_daddr: Foreign IPv4 addr
 *	@skc_rcv_saddr: Bound local IPv4 addr
 *	@skc_addrpair: 8-byte-aligned __u64 union of @skc_daddr & @skc_rcv_saddr
 *	@skc_hash: hash value used with various protocol lookup tables
 *	@skc_u16hashes: two u16 hash values used by UDP lookup tables
 *	@skc_dport: placeholder for inet_dport/tw_dport
 *	@skc_num: placeholder for inet_num/tw_num
 *	@skc_portpair: __u32 union of @skc_dport & @skc_num
 *	@skc_family: network address family
 *	@skc_state: Connection state
 *	@skc_reuse: %SO_REUSEADDR setting
 *	@skc_reuseport: %SO_REUSEPORT setting
 */
struct sock_common {
	union {
		__addrpair	skc_addrpair;
		struct {
			__be32	skc_daddr; // remote(연결된 상대) IPv4 addr
			__be32	skc_rcv_saddr; // local IPv4 addr
		};
	};
	union  { // 커널이 수신 패킷 들어왔을 때 해시 테이블로 소켓을 빠르게 찾음 (demultiplex)
		unsigned int	skc_hash; 
		__u16		skc_u16hashes[2];
	};
	/* skc_dport && skc_num must be grouped as well */
	union {
		__portpair	skc_portpair;
		struct {
			__be16	skc_dport; // remote port
			__u16	skc_num;   // local port
		};
	};
	unsigned short		skc_family;
	volatile unsigned char	skc_state; // connection state (ESTABLISHED, TIME_WAIT)
	unsigned char		skc_reuse:4;
	unsigned char		skc_reuseport:1;
	...
};
```

```c
/**
  *	struct sock - network layer representation of sockets
  *	@__sk_common: shared layout with inet_timewait_sock
  ...
  */
 struct sock {
	/*
	 * Now struct inet_timewait_sock also uses sock_common, so please just
	 * don't add nothing before this first member (__sk_common) --acme
	 */
	struct sock_common	__sk_common; // timewait 소켓 등과 메모리 레이아웃 공유
	...
};
```
##### 송신/수신 버퍼(queue)
```c
// include/net/sock.h:405
struct sock {
	struct sk_buff_head	sk_receive_queue; // recv buffer
	struct sk_buff_head	sk_write_queue; // send buffer
	int			sk_rcvbuf; // size of receive buffer
	int			sk_sndbuf; // size of send buffer size
};
```
##### 시퀀스 번호 및 전송 진행 상태
- 순서 보장을 위한 기준
```c

```
##### 재전송/유지 관리를 위한 타이머
- ACK 타임아웃
- 재전송 타이머
- 연결 유지 관련 타이머
```c

```
##### 상태 머신
- `LISTEN`, `ESTABLISHED`, `FIN_WAIT`, `TIME_WAIT` 등의 현재 연결 단계를 나타냄.

#### 2.2.2. TCP state machine
- TCP의 상태 머신은 `LISTEN → ESTABLISHED → FIN_WAIT/TIME_WAIT`와 같이 전이
- 각 상태는 커널이 어떤 자원을 얼마나 보유하고 있는지를 의미
	- `LISTEN`: 연결을 수락하기 위한 준비 상태
	- `ESTABLISHED`: 송수신 자원이 모두 활성화된 상태
	- `FIN_WAIT`, `TIME_WAIT`: 연결 종료 후 안정성을 위한 자원 유지 상태
- 상태 전이는 Connection 자원의 생명주기 변화를 의미
### 2.3. Linux kernel: Port의 자원 관리와 Connection의 상태 관리
`bind()`와 `connect()`의 흐름을 통해 Port 네임스페이스 관리와 Connection 상태 관리가 커널 내부에 어떻게 결합되어 있는지 알아보자.
- Linux Kernel Source: [Github - torvalds/linux:Linux kernel source tree](https://linux-kernel-labs.github.io/refs/heads/master/labs/networking.html)
- 참고자료: [Networking - The Linux Kernel Documentation](https://linux-kernel-labs.github.io/refs/heads/master/labs/networking.html)
#### 2.3.1. `bind()`와 Port 네임스페이스
- `bind()`는 커널 내부에서 Port 네임스페이스에 소켓을 등록하는 작업
- 등록 과정에서 **권한 검사, 충돌 검사, 네임스페이스 분리** 발생
##### A. Privileged Port
- 포트는 커널이 통제하는 자원이며, 특정 범위는 권한으로 보호됨.
- 포트는 단순 번호가 아니라 커널 정책의 적용 대상
```c

```
##### B. `bind` 실패 / 포트 충돌
- `bind()`는 주소 설정이 아니라, 커널 내부의 포트 관리 테이블에 소켓을 등록하는 행위
- 이미 점유된 포트에 대해 다시 `bind()`를 시도하면 `EADDRINUSE` 등으로 거부됨.
```c

```
#### 2.3.2. `connect()`와 Connection 상태 생성
- 클라이언트가 `connect()` 호출 시, 클라이언트는 명시적으로 포트를 지정하지 않더라도 커널에 의해 ephemeral port가 자동 할당됨.
- 이 시점부터 커널은 하나의 Connection 상태를 생성하고 관리하기 시작함.
```c

```
#### 2.3.3. `TIME_WAIT`: Port와 Connection 상태 결합
- 연결이 종료되더라도, 커널은 Connection을 즉시 제거하지 않고 일정 시간 동안 `TIME_WAIT` 상태를 유지함.
```c
close(fd);
// -> connection enters TIME_WAIT
```
- 이 동안 Connection 상태가 유지되고, Port는 OS 자원으로 묶여 있어 일정 시간 동안 재사용이 제한됨.
#### 2.3.4. 포트 고갈: Port 자원과 Connection 상태의 결과
- 클라이언트는 연결마다 ephemeral port를 소모함.
- 다수의 짧은 연결이 생성, 종료되면
	- `TIME_WAIT` 상태 누적
	- 재사용 가능한 포트가 줄어 포트 고갈 발생
```c

```
- Port + Connection 상태를 커널이 보유하고 있기 때문에 발생하는 자원 문제
#### 2.3.5. 컨테이너 네트워크와 네임스페이스 분리
- 동일한 머신에서도 `netns`(컨테이너)가 다르면 같은 Port 번호를 사용할 수 있음.
- Port는 전역이 아니라 네트워크 네임스페이스 단위로 분리된 OS 자원
```c

```

## 3. 송신 경로에서의 Transport Layer (`send()`)
Transport Layer에서 "전송"의 의미를 알아보자.
```c
send(fd, buf, len, 0);
````
- `send()`는 데이터를 즉시 네트워크로 내보내는 호출이 아니라
- 커널에게 이 데이터를 **전송 대상으로 등록**해 달라고 **요청**하는 시스템콜
### 3.1. User Space → Kernel Space
- 애플리케이션이 `send()`를 호출한다.
- 파일 디스크립터(`fd`)를 통해 커널 내부의 소켓 객체에 접근한다.
- 시스템 콜을 통해 커널 모드로 진입한다.
```c

```
- 소켓은 파일 디스크리버로 표현되지만
- 실제 동작은 커널 내부의 Transport Layer 로직에 의해 수행됨.
### 3.2. Kernel Space 내부
- 커널은 `send()` 호출이 들어왔다고 해서 즉시 패킷을 보내지 않고
- 다음을 판단함.
	- 현재 연결 상태가 전송 가능한 상태인가? (e.g., TCP state가 `ESTABLISHED`)
	- 수신 측 윈도우 및 로컬 송신 윈도우에 여유가 있는가?
	- 아직 ACK되지 않은 재전송 대기 데이터가 존재하는가?
	- 지금 전송해도 혼잡 제어 정책상 문제가 없는가?
```c

```
- Transport Layer는 데이터를 보낼지 말지, 언제 보낼지, 어떤 크기로 보낼지 결정하는 계층층
### 3.3. Transport Layer의 책임 경계
> Transport Layer는 전송이 성립되기 위한 조건을 관리
##### L4(Transport Layer)
- 전송 가능 여부 판단
- 데이터의 분할 및 세그먼트 구성
- 순서, 재전송, 흐름 제어를 위한 상태 관리
##### L3(Network Layer, IP)
- 실제 패킷 전달
- 경로 선택
- 패킷 전달 성공/실패
## 4. 수신 경로에서의 Transport Layer (`recv()`)
### 4.1. Transport Layer의 역할
- 네트워크로부터 패킷이 도착하면 다음과 같은 작업을 수행함.
- 이 패킷을 어떤 소켓으로 전달할지 결정 (5-tuple 기반 demultiplex)
- 해당 소켓의 receive buffer에 데이터 적재
#### 4.1.1. 5-tuple demultiplex
```c
// 5-tuple
(src IP, src port, dst IP, dst port, protocol)
```
#### 4.1.2. Receive buffer 적재
```c

```
### 4.2. 데이터 추상화: Packet → Segment → Data Stream
Transport Layer는 패킷을 전달하는 계층이 아니라, 패킷을 애플리케이션이 다룰 수 있는 데이터 형태로 변환하는 계층
- L3는 데이터를 패킷 단위로 전달
- L4는 이 패킷들을 애플리케이션이 소비 가능한 형태로 재구성
	- `read()`/`recv()` 시스템 콜을 호출하여 소비함.
#### 4.2.1. TCP: Data Stream 추상화
- 수신한 여러 패킷을 내부적으로 처리하여 연속된 바이트 스트림을 제공
##### TCP 수신 시 내부 처리 흐름
- IP 계층으로부터 순서가 보장되지 않은 세그먼트 수신
- 시퀀스 번호를 기준으로
	- 순서 재정렬
	- 누락 데이터 감지
	- 중복 데이터 제거
```c

```
- 재정렬된 데이터를 receive buffer에 적재
##### TCP 스트림 추상화의 의미
- `read()` 호출은 지금 읽을 수 있는 만큼의 연속된 바이트를 의미
- 애플리케이션은 패킷의 경계, 분할/병합, 재전송을 전혀 인식하지 않음.
#### 4.2.2. UDP: Message 추상화
- 메시지 단위(datagram) 전달을 그대로 유지
##### UDP 수신 시 특징
- 하나의 UDP 패킷 = 하나의 메시지
- 수신한 패킷을 그대로 소켓 receive buffer에 적재
```c

```
##### UDP 메시지 추상화의 의미
- `recvfrom()` 호출은 항상 하나의 메시지 단위를 반환하며 버퍼가 작으면 메시지가 잘릴 수 있음.
- 애플리케이션은 메시지 경계 유지, 순서/손실 처리를 직접 책임짐.
#### 4.2.3. TCP vs. UDP: 추상화의 차이
| 구분           | TCP                | UDP               |
| ------------ | ------------------ | ----------------- |
| 데이터 단위       | 바이트 스트림            | 메시지(datagram)     |
| 패킷 경계        | 숨김                 | 유지                |
| 순서 보장        | O                  | X                 |
| 재전송          | O                  | X                 |
| read/recv 의미 | “연속된 데이터 일부”       | “하나의 메시지”         |
|              | `recv/send(write)` | `recvfrom/sendto` |
- L4에서 TCP와 UDP의 차이는 신뢰성 여부가 아니라, 어떤 추상화와 상태를 구현하느냐의 차이
### 4.3. 수신 경로에서의 책임 경계
##### L4가 관리하는 것
- 데이터 전달을 보장하지 않으나 **전달이 성립되기 위한 조건**을 관리함.
	- 순서 관리 (시퀀스 번호 기반)
	- 손실 감지 (ACK, timeout)
	- 재조립 (out-of-order 처리)
	- 상태 관리 (connection state)
##### L4가 책임지지 않는 것
- 실제 패킷 전달 성공 여부
- 네트워크 경로의 신뢰성
- 물리적 손실