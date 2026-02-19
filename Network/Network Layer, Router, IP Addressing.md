---
date: 2026-02-19
category:
  - 네트워크
tags:
  - NetworkLayer
  - Router
  - IP
Person:
  - 김태현
---
# 1. Network Layer?
## 1. Network 레이어가 왜 필요한가?
- L2(Data Link)만 있으면 안 되는지? 왜 굳이 L3가 필요한지?
- L2는 **같은 네트워크 내부 통신만 가능**
- 다른 네트워크로 가려면?
    → 경로 계산 필요  
    → 중간 장비 필요  
    → 논리적 주소 필요
## 2. 네트워크 계층의 두 가지 핵심 기능

### 1. Forwarding
- 라우터 안에서 패킷을 '입력 포트 → 출력 포트'로 전달
- 로컬(router 내부) 기능
- 매 패킷마다 실행됨
- 매우 빠르게 동작해야 함
- Data Plane 담당
- 비유
	- “교차로에서 어느 방향으로 차를 보낼지 결정하는 행위”
### 2. Routing
- 출발지 → 목적지까지의 전체 경로 결정
- 네트워크 전체 관점
- 경로 계산
- Control Plane 담당
- 비유
	- “여행 출발 전에 전체 경로를 계획하는 행위”
## 3. 네트워크는 어떤 서비스를 보장하는가? (Network Service Model)
- Guaranteed delivery
- Bounded delay
- In-order delivery
- Minimal bandwidth
- Security
### Internet의 실제 모델
> Best Effort Service
- 전달 보장 없음
- 지연 보장 없음
- 순서 보장 없음
- 대역폭 보장 없음
- Why
	- 확장성 → 수십억 장치 지원 필요
	- 단순화
	- 전송계층(TCP)이 보완 → 계층 분리

---
# 2. Network Layer Structure
- 네트워크 계층은 두 개의 Plane으로 구성됨
	- **Data Plane**
	- **Control Plane**
	
## 2.1 Data Plane
> 패킷을 실제로 전달하는 기능
### 특징
- 로컬(router 내부) 기능
- 매 패킷마다 동작
- 매우 빠른 처리 필요
- Forwarding 수행
### 수행 작업
- 목적지 IP 확인
- Forwarding Table 조회
- 출력 포트 결정
- TTL(Time to Live) 감소
- 헤더 수정

## 2.2 Control Plane

> 패킷이 갈 경로를 계산하는 기능
### 특징
- 네트워크 전체 관점
- 라우팅 알고리즘 수행
- 라우팅 테이블 생성
- 경로 변경 시 재계산
### 수행 작업
- 이웃 라우터와 정보 교환
- 최단 경로 계산
- Routing Table (RIB) 생성
- FIB(Foward information Base)로 전달

## 2.3 비교 정리

| 구분    | Data Plane | Control Plane |
| ----- | ---------- | ------------- |
| 역할    | 실행         | 계산            |
| 범위    | 로컬         | 네트워크 전체       |
| 수행 시점 | 매 패킷       | 경로 변화 시       |
| 구현    | 하드웨어 중심    | 소프트웨어 중심      |
| 담당 기능 | Forwarding | Routing       |

## 2.4 Control Plane 구조 두가지
### A. Traditional (Per-router control)
- 모든 라우터가 자체적으로 계산
- 분산 구조
- OSPF, RIP 등 → 각 라우터가 두뇌를 가짐
- ![[Pasted image 20260219170050.png]]
### B. SDN (Software Defined Networking)
- 중앙 컨트롤러가 경로 계산
- 라우터는 단순 실행 장치
- 논리적 중앙 집중 구조 → 두뇌는 중앙에 있음
- ![[Pasted image 20260219170115.png]]

---

# 3. Router 내부 살펴보기
## 3.1 Router Architecture

![[Pasted image 20260219165954.png]]
### 기본 구조

라우터는 크게 두 영역으로 나뉨:
- **Control Plane (상단)**
     - Routing Processor   
- **Data Plane (하단)**
    - Input Port
    - Switching Fabric       
    - Output Port 
### 흐름 요약
- 패킷 흐름:
	```
	Input Port → Switching Fabric → Output Port
	```
- Control Plane은 위에서 테이블을 계산함  
- Data Plane은 아래에서 실행함
## 3.2 Input Port
![[Pasted image 20260219170154.png]]
### 수행 기능
- Physical Layer 처리 (비트 수신)
- Data Link Layer 처리 (Ethernet 등)
- Forwarding Table Lookup
- Queueing (버퍼링)
### 핵심 개념
#### Destination-based forwarding
→ 목적지 IP 기반 결정
![[Pasted image 20260219170232.png]]
#### Longest Prefix Matching
→ 가장 긴 prefix와 매칭되는 항목 선택
![[Pasted image 20260219170225.png]]

### 왜 Input Port에서 Lookup 수행하는가?
- 중앙 CPU에 맡기면 병목 발생
- 각 포트에서 분산 처리    
- Line Speed 유지 목적

## 3.3 Switching Fabric

### 역할
> 입력 포트 → 출력 포트 연결

라우터의 “심장”
### 구현 방식 3가지
![[Pasted image 20260219170320.png]]

| 방식       | 특징        |
| -------- | --------- |
| Memory   | CPU 거쳐 전달 |
| Bus      | 공유 버스 사용  |
| Crossbar | 병렬 연결 가능  |

## 3.4 Input Port Queuing

### 언제 발생?
- 입력 속도 > Switching Fabric 처리 속도
### 문제: HOL Blocking (Head-of-Line Blocking)
- 큐 맨 앞 패킷이 막히면
- 뒤 패킷도 못 나감    
- 성능 저하 발생
- HOL Blocking![[Pasted image 20260219170355.png]]
## 3.5 Output Port
![[Pasted image 20260219170421.png]]
### 수행 기능
- Buffering
- Scheduling
- Link Layer 처리
- Physical 전송    
### 왜 Buffer 필요?
Switch Fabric → Output 속도 초과 시  → 대기 필요
## 3.6 Scheduling Mechanisms
![[Pasted image 20260219170444.png]]

### 패킷 중 누가 먼저 나갈 것인가?
#### A. FIFO (First in first out)
- 먼저 온 패킷 먼저 처리
- 공정함
- QoS 불가능
#### B. Priority Scheduling
- 우선순위 높은 패킷 먼저 전송
- QoS 가능  
- 저우선순위 기아 가능
#### C. Round Robin scheduling
- 여러 큐를 순환하며 하나씩 전송 반복
- 비교적 공정
	- 패킷 크기 차이 고려 X
	- 작은 패킷 큐가 유리
- 우선순위 기아상태 없음
#### D. Weighted Fair Queuing (WFQ)
> 큐에 가중치 부여,
> 가충치 비율만큼 대역폭 분배
- 공정성 + QoS 가능
- 구현 복잡
- 계산 오버헤드 존재
#### 스케줄링 비교 정리
| 방식       | 공정성 | QoS 지원 | 복잡도   | 문제점       |
| -------- | --- | ------ | ----- | --------- |
| FIFO     | 낮음  | X      | 매우 낮음 | 트래픽 구분 불가 |
| Priority | 낮음  | O      | 낮음    | 기아 발생     |
| RR       | 중간  | 제한적    | 중간    | 패킷 크기 무시  |
| WFQ      | 높음  | X      | 높음    | 구현 복잡     |

---

# 4. IP Addressing

## 4.1 왜 IP 주소가 필요한가?
- MAC 주소만으로는 안 되는지?
- 왜 또 다른 주소 체계가 필요한지?
### 이유
- MAC → 로컬 네트워크 내부 식별자
- IP → 전 세계 네트워크 간 식별자
- 논리적 주소 필요
## 4.2 IPv4
![[Pasted image 20260219170726.png]]
![[Pasted image 20260219170701.png]]
- 32 bits (4 octets)
- dotted decimal 표기
- 예: `192.168.1.10`
### 주소 개수
`2^32 ≈ 43억 개` → 이미 고갈됨
### IPv4 Header 주요 필드
#### 기본 구조
- Identifier : 여러조각 쪼개졌을 때 재조합 시 필요
- Flag : 쪼개기 조건 알 수 있음
- Fragment Offset : 쪼개진지 몇번째 패킷인지
- TTL (Time to Live = 수명)
- Protocol (TCP, UDP 등)
- Source IP
- Destination IP
#### TTL의 역할
- 홉 수 제한
- 무한 루프 방지
- 라우터(홉) 통과 시 1 감소
- TTL=0 일 경우 패킷 폐기

## 4.3 IPv6
![[Pasted image 20260219171309.png]]

#### 기본 구조
- 128 bits
- 16진수 표기
- 콜론(:) 구분
- 예: `2001:0db8::1`
### 주소 개수
`2^128 ≈ 3.4 × 10^38` → 사실상 무한

### IPv6 특징
- Header 단순화
- Checksum 제거
- Fragmentation 라우터에서 수행 안 함  
- IPsec 기본 설계 포함
- 확장해더를 추가로 가질 수 있음
	![[Pasted image 20260219171423.png]]

## 4.4 Subnetting
### IP 주소 구조
`[ Network Portion | Host Portion ]`
### Subnet Mask 역할
![[Pasted image 20260219171706.png]]
- 어디까지가 Network?
- 어디까지가 Host?
- 예: `200.23.16.0/23`
	→ 앞 32비트 중 23비트가 네트워크
### CIDR 표기
`IP / prefix length`
#### 예시
`Host 수 = 2^(32 - prefix) - 2`

| Prefix | Host 수 |
| ------ | ------ |
| /24    | 256    |
| /25    | 128    |
| /30    | 4      |

## 4.5 Longest Prefix Matching

> 가장 긴 prefix와 매칭되는 경로 선택

### 예시
- Forwarding Table:
	`192.168.1.0/24 → A 192.168.1.128/25 → B`
- 목적지:
	`192.168.1.130`
	→ /25 선택
## 4.6 Public vs Private IP
![[Pasted image 20260219171847.png]]
### Private IP 범위
![[Pasted image 20260219172221.png]]
![[Pasted image 20260219172230.png]]

| 범위             | 대역      |
| -------------- | ------- |
| 10.0.0.0/8     | Class A |
| 172.16.0.0/12  | Class B |
| 192.168.0.0/16 | Class C |
→ 인터넷에서 라우팅 안 됨
## 4.7 DHCP
### 역할
- IP 자동 할당
- Subnet mask
- Default gateway
- DNS 정보 제공
### 과정 (DORA)
![[Pasted image 20260219171956.png]]
1. Discover : DHCP 서버 찾기
2. Offer : 클라이언트에게 할당해 줄 IP 주소 제안
3. Request : DHCP Offer 잘 받았는데, 이 IP 주소 쓸게
4. Ack : 최종 승인
## 4.8 NAT
![[Pasted image 20260219171855.png]]
- 왜 필요한가? IPv4 주소 부족
- 동작 원리
	Private IP → Public IP 변환
- 장점
	- 주소 절약
	- 내부 네트워크 보호


이후 내용들
- 그래서 어떻게 "잘" 전달하는가?
	→ Routing Protocols
	- link state : Dijkstra
	- distance vector : Dellman-Ford
- OSPF, BGP
	- eBGP, iBGP
	![[Pasted image 20260219172712.png]]