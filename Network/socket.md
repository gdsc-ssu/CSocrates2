# Socket 이 무엇인지 알아보자
- socket과 socket의 종류인 TCP와 UDP의 기본 구조를 알아본다

## 1. what is socket?
- 소켓은 네트워크를 통한 입출력을 하기 위해 사용자에게 필요한 수단을 제공하는 응용 프로토콜 인터페이스이다
### 인터페이스 == socket
- 인터페이스란 서로 다른 두 개의 시스템, 장치 사이에서 정보나 신호를 주고받는 경우의 접점이나 경계면을 의미함
- 네트워크에서 인터페이스 == socket
- Transport인 4계층과 Application인 5계층 사이의 연결이라고 생각하자
- 소켓을 활용한 네트워크 응용 프로그램을 통해 네트워크상에서 데이터를 송/수신

- socket 인터페이스의 위치
![alt text](image.png)
![alt text](image-1.png)

### 네트워크 입/출력을 위한 요소
1. 프로토콜(Protocol)
- 4계층에서 어떤 프로토콜을 사용하는지?
2. 소스 IP 주소(Source IP Address)
3. 소스 포트 번호(Source Port Address)
- 소스를 통해서 누가 보내는지?
4. 목적지 IP 주소(Target IP Address)
5. 목적지 포트 번호(Target Port Address)
- 목적지로 받는 사람이 누군지?
- 참고로 포트번호는 네트워크에서 process들의 식별자라고 생각하면 된다

## 2. 포트(Port)
- 호스트 내에 실행되고 있는 프로세스(Process)를 구분 짓기 위한 16비트의 논리적 할당
- 값의 범위: 0~65535
- 포트에서 통신의 주체는 프로세스와 프로세스 사이에서 이루어지는 것이다
- 만약에 포트 번호가 겹친다면?
    1. 나중에 실행한 프로세스가 죽을 수 있음
    2. 보통은 프로세스 둘 다 죽는다

![alt text](image-2.png)
- well known port
    - tcp 또는 UDP에서 쓰이는 0번부터 65535번까지의 포트 중에서 IANA에 의해서 할당된 0번부터 1023번까지의 포트

## 3. 데이터의 전송 순서
### Big Endian과 Little Endian
- 빅 엔디안(Big Endian)
    - 상위 바이트의 값을 작은 번지수에 저장
- 리틀 엔디안(Little Endian)
    - 상위 바이트의 값을 큰 번지수에 저장
![alt text](image-3.png)

- 호스트 바이트 순서
    - CPU별 데이터 저장 방식을 의미함
    - 다수의 시스템이 리틀 엔디안 사용(x86 AMD, Intel CPU 계열)
- 네트워크 바이트 순서
    - 통일된 데이터 송수신 기준
    - 빅 엔디안 기준으로 함

## 4. 연결 지향형 소켓(SOCKET_STREAM, TCP 소켓)
![alt text](image-4.png)
### SOCKET_STREAM 소켓 유형
- 스트림 방식의 소켓 생성
- UNIX 파이프 개념과 동일
- 연결형(스트림) 서비스 선택 시 사용

### SOCKET_STREAM 소켓 특성
- 메시지 경계가 유지되지 않는다(3,4 계층의 특징)
    - 메시지가 일정한 규격을 가지는 것이 아니라 속도조절, 네트워크 흐름 조절에 의해서 서로 다른 메시지의 크기를 가질 수 있다는 것을 의미
- 전달된 순서대로 수신됨
- 전송된 모든 데이터는 에러 없이 도달
- 세부 기능 조절 자체는 TCP 4계층에서 이루어지게 된다

### Socket의 할 일 !!!!
- 5계층인 application에서 TCP로 써서 보내줘 라고 4계층에게 요구하기

## 5. TCP 소켓 프로그래밍 기본 구조
![alt text](image-5.png)

- TCP client n개가 들어오면 Socket은 n+1개가 존재한다
    - listen이 1개 있기 때문임

#### TCP client의 연결 준비 단계
- adress: socket을 만들고 address에는 통신을 요청하는 상대방의 주소를 넣어준다
#### TCP server의 연결 준비 단계
- adress: socket을 만들어두고 address에는 server가 동작할 IP주소와 port번호를 넣게 된다(누가 들어올지 모르니 내꺼만 넣는 절차이다)
- bind: IP와 port번호를 OS한테도 알려주는 절차가 bind(매우 중요)
- listen: client가 접속을 요청하는걸 기다리는 상태
#### 서비스 처리 단계
- connect: listen하고 있는 server한테 연결요청하기
- accept: client가 접근했으면 새로운 socket을 만들어서 상대방과 데이터를 주고 받을 수 있게 만들어주기
- read, write: client와 server 둘 다 read와 write를 통해서 자료를 송수신한다
    - read == recieve
    - write == send
- close: 데이터를 다 받았으면 client가 client socket을 close한다
- close: 그러면 server도 통신하는 socket을 close한다
#### 서버 종료 단계
- close: 서버 자체가 끝나는 단계에서 close를 진행한다

### JAVA와 C socket 비교
![alt text](image-6.png)
![alt text](image-7.png)

#### JAVA Client
![alt text](image-12.png)
- client -> server 로 연결, server는 항상 읽을 준비를 하고 있음
- client <- server 읽고 UpperCase해서 client한테 보내준다
- client -> server, client는 읽을 준비하고 있다가 받아서 처리하고 끝

- inFromUser라는 buffer만들고
- newSocket("hostname",6789) 소켓에서 서버 IP, 접속할 포트번호

![alt text](image-13.png)
- user한테 sentence가 읽어져 오는 것을 기다리고
- writeBytes로 Data를 상대방 -> server한테 보낸다
- inFromServer.readLine(); 해서 보낸 것을 읽고
- client 소켓 close

#### JAVA Server
![alt text](image-14.png)
- ServerSocket 이라는 class 만들고 객체 만들어서 binding 했다
- server는 while(true) 무한정 client 들어올때까지 기다리게 된다
- listen 상태 돌입하고 client가 접속하면 welcomeSocket.accept() 하면서 sever는 socket을 하나 더 생성하게 된다
    - 이전 가지고 있던 socket은 통신을 위한 socket일 뿐이다
- BufferedReader 랑 InputReader로 소통을 진행

![alt text](image-15.png)
- outToClient 로 client가 접속하자마자 data를 보내고 Buffer를 읽는다
- UpperCase 대문자 변형을 진행하고
- outToClient 에서 output stream에 있는거를 송신하겠다 를 볼 수 있다


- 3 way handshaking
    - listen한 server에게 client가 접속하고 connectioin 하고 accept가 이루어지는 구간
![alt text](image-8.png)

## 5. 비연결 지향형 소켓(SOCKET_DGRAM, UDP 소켓)
![alt text](image-9.png)
### SOCKET_DGRAM 소켓 유형
- 데이터그램 방식의 소켓 생성
- 개별적으로 주소가 쓰여진 패킷 전송 시 사용
- 비연결형(데이터그램) 서비스 선택 시 사용

### SOCKET_DGRAM 소켓 특성
- 패킷은 전달된 순서대로 수신되지 않음
- 에러복구를 하지 않음(즉, 신뢰성이 없음)
- 데이터그램 패킷의 크기 제한이 있음
    - 무조건 같은 크기, UDP 소켓에서는 지정되어 있다
- TCP와 다르게 pipeline이 불가능하다

### JAVA와 C socket 비교
![alt text](image-10.png)
- C 소켓
    - server는 여전히 bind를 거쳐야 한다
    - connection이랑 accept가 없다는 것을 알 수 있다
    - 일단 데이터를 보내는 작업만을 진행한다
![alt text](image-11.png)
- JAVA 소켓
    - DatagramSocket()을 만들고 보내기만 하고 끝이 나는 것을 볼 수 있다

#### JAVA Client
![alt text](image-16.png)
![alt text](image-17.png)
- clientSocket 보면 DatagramSocket 객체를 만들었다
- IPAdress에서 ("hostname") IP 들어가는 것도 볼 수 있다

![alt text](image-18.png)
- new DatagramPacket에서 보내는 네 가지 요소를 확인 가능
- receivePacket 으로 recieve 받을 공간 만들어주고
- clientSocket.recieve 해서 받는다

#### JAVA Server
![alt text](image-19.png)
- new DatagramSocket 생성한 것을 볼 수 있다
- new byte[1024] 로 TCP와 다르게 경계단위를 딱 설정한 상태
- receievePacket 으로 받을 공간을 만들고
- TCP와 다르게 connection이 없기 때문에 기다리다가 receive 한다

![alt text](image-20.png)
- IP Address로 상대방의 주소로 보낸다
- new DatagramPacket의 네 가지 요소 역시 확인 가능