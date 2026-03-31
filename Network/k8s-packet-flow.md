# 쿠버네티스에서 패킷은 어떻게 흐르는가

하나의 패킷이 외부 클라이언트에서 출발해 목적지 Pod에 도달하기까지의 과정
```
외부 클라이언트
    ↓
클러스터 진입: NodePort / LoadBalancer / Ingress
    ↓
Service: iptables DNAT로 Pod IP로 변환
    ↓
노드 내부: veth → bridge → Pod (같은 노드)
    또는
노드 간: CNI 라우팅 → 목적지 노드 → Pod (다른 노드)
    ↓
( CNI 플러그인 비교: Flannel → Calico → Cilium )
```
## 목차

1. 쿠버네티스 아키텍처
2. **클러스터** 진입 (Exposing Service)
3. **Service**와 iptables DNAT
4. **노드 내부** (veth와 bridge)
5. **노드 간** (CNI Routing)
6. CNI 플러그인 비교 (Flannel → Calico → Cilium)

---

## 1. 쿠버네티스 아키텍처
[Kubernetes - Cluster Architecture](https://kubernetes.io/docs/concepts/architecture/)

![](img/k8s-cluster-architecture.png)
- Pod
	- 쿠버네티스에서 배포/실행의 최소 단위.
	- 하나 이상의 컨테이너 묶음.
	- 고유한 IP를 할당받는다.
#### Control Plane
- 클러스터 전체의 상태를 관리하는 두뇌. 
- Pod 스케줄링, 상태 감시, Endpoints 갱신 등을 담당한다. 
- 직접 패킷을 처리하지는 않는다.
#### Worker Node
- 실제로 Pod가 실행되는 노드. 
- 각 노드에는 다음 컴포넌트가 항상 실행된다.
	- kubelet: Control Plane의 지시를 받아 Pod를 생성/삭제하고 CNI 플러그인을 호출
	- **kube-proxy**: iptables 룰을 관리해 Service → Pod 트래픽을 처리
	- **CNI Plugin**: Pod 생성 시 네트워크 인터페이스(veth)를 구성하고 라우팅을 설정
#### Storage Node 
- 퍼시스턴트 볼륨을 제공하는 노드. 
- 네트워크 흐름과는 직접 관련이 없으므로 다루지 않는다.
## 2. 클러스터 진입 — Exposing Service

- source: [Kubernetes NodePort vs LoadBalancer vs Ingress](https://medium.com/google-cloud/kubernetes-nodeport-vs-loadbalancer-vs-ingress-when-should-i-use-what-922f010849e0)

![](img/k8s-exposing-service.png)

- 외부 클라이언트가 클러스터 내 Service에 접근하는 방법:
	1. **NodePort**: Service is accessed via `NodeIP:port`
	2. **LoadBalancer**: Service is accessed via LoadBalncer
	3. **Ingress**

### 2.1. NodePort

모든 노드의 특정 포트를 열어서 외부에 노출한다. 외부 클라이언트는 `NodeIP:port`로 접근한다.
![](img/k8s-service-nodeport.png)

- 가장 단순한 방법
- 모든 노드에 동일한 포트가 열리므로, 어느 노드로 접근해도 동작
- 단점: 포트 번호가 외부에 노출되고, 30000~32767 범위만 사용 가능

### 2.2. LoadBalancer

클라우드 환경에서 외부 로드 밸런서를 생성하고 고정 IP(VIP)를 부여한다. 외부 클라이언트는 이 VIP로 접근한다.
![](img/k8s-service-loadbalancer.png)

- 퍼블릭 클라우드(AWS, GCP, Azure)에서 주로 사용
- Service마다 외부 IP가 하나씩 생성됨

### 2.3. Ingress

여러 Service 앞에 하나의 진입점을 두고, 도메인/경로 기반으로 트래픽을 라우팅한다.
![](img/k8s-service-ingress.png)

- 여러 Service를 하나의 외부 IP로 노출 가능 → LoadBalancer 대비 비용 절감
- L7(HTTP/HTTPS) 수준의 라우팅 제공
- Ingress는 쿠버네티스 오브젝트이고, 실제 구현은 Ingress Controller가 담당

## 3. Service와 iptables DNAT

외부에서 들어온 패킷이든, 클러스터 내부에서 출발한 패킷이든
Service를 거치면 동일하게 iptables DNAT가 동작한다.

source: [Demystifying Kubernetes Service Packet Path](https://medium.com/@abhishek.amjeet/demystifying-kubernetes-services-packet-path-98297874e5f5)
![](img/k8s-iptable-chains.png)

### 3.1. ClusterIP

Service에는 **ClusterIP**라는 고정 IP가 부여된다. 
하지만 이 IP는 어떤 인터페이스에도 할당되어 있지 않은 **가상 IP**다.

```bash
ip addr  # 어떤 노드에서 실행해도 ClusterIP는 보이지 않음
```
- **DNAT(Destination NAT)**
	- ClusterIP로 향하는 패킷은 커널의 **iptables**가 중간에서 가로채 목적지 IP를 실제 Pod IP로 교체한다.

### 3.2. kube-proxy

각 노드에서 실행되는 **kube-proxy**가 iptables DNAT 룰을 관리한다.

```
패킷: dst=10.96.0.1:80 (ClusterIP)
         ↓
iptables PREROUTING (DNAT)
         ↓
패킷: dst=10.244.1.5:8080 (실제 Pod IP)
```

iptables rule 구조:
```bash
# ClusterIP로 오는 패킷을 서비스 체인으로
-A KUBE-SERVICES -d 10.96.0.1/32 -p tcp --dport 80 -j KUBE-SVC-XXXX

# 50% 확률로 Pod A, 나머지는 Pod B (로드밸런싱)
-A KUBE-SVC-XXXX -m statistic --mode random --probability 0.5 -j KUBE-SEP-AAAA
-A KUBE-SVC-XXXX -j KUBE-SEP-BBBB

# 실제 DNAT
-A KUBE-SEP-AAAA -p tcp -j DNAT --to-destination 10.244.1.5:8080
-A KUBE-SEP-BBBB -p tcp -j DNAT --to-destination 10.244.1.6:8080
```
예시: [Demystifying Kubernetes Services Packet Path](https://medium.com/@abhishek.amjeet/demystifying-kubernetes-services-packet-path-98297874e5f5)
![](img/k8s-nat-table-ex.png)
### 3.3. Pod가 교체되어도 연결이 유지되는 이유

**Endpoints Controller**가 Service selector와 일치하는 살아있는 Pod 목록을 **Endpoints** 오브젝트로 관리한다. Pod가 죽거나 새로 뜨면 자동으로 갱신된다.

```
Pod A (10.244.1.5) 종료
    ↓
Endpoints Controller: 목록에서 제거, 새 Pod C (10.244.1.9) 추가
    ↓
kube-proxy: Endpoints 변경 감지
    ↓
iptables 룰 업데이트 (Pod A 제거 → Pod C 추가)
    ↓
이후 패킷은 자동으로 Pod C로 DNAT
```

클라이언트는 ClusterIP만 알면 된다. 
Pod가 몇 번 교체되든 iptables 룰이 자동으로 갱신되므로 연결이 유지된다.

## 4. 노드 내부 — veth와 bridge

DNAT 이후 패킷의 목적지는 실제 Pod IP다. 
이제 패킷이 Pod까지 어떻게 도달하는지를 따라간다.

### 4.1. Pod는 격리된 Network Namespace 안에 있다

각 Pod는 독립된 **Linux Network Namespace**를 가진다. Pod마다 완전히 별개의 네트워크 스택(인터페이스, 라우팅 테이블)이 존재한다.

```
Node
├── Host Network Namespace
│     eth0 (Node IP: 192.168.1.10)
│
├── Pod A Network Namespace
│     eth0 (Pod IP: 10.244.1.5)
│
└── Pod B Network Namespace
      eth0 (Pod IP: 10.244.1.6)
```

격리되어 있으므로 외부와 통신하려면 호스트 네임스페이스와의 연결 지점이 필요하다.

### 4.2. veth pair: 네임스페이스를 잇는 가상 케이블

Linux 커널이 제공하는 **veth pair(가상 이더넷 쌍)** 가 그 연결 지점이다. 파이프처럼 한쪽에 들어간 패킷이 반드시 반대쪽으로 나온다.

- 한쪽 끝 → Pod Namespace 안 (`eth0`)
- 반대쪽 끝 → Host Namespace 안 (`vethXXX`)

```
Pod Namespace              Host Namespace
  eth0 (10.244.1.5)  ←──→  vethXXX
                               │
                            bridge (cni0)
```

### 4.3. 같은 노드 안에서: bridge가 중계한다

CNI 플러그인이 생성하는 `cni0` bridge에 모든 veth의 호스트 쪽 끝이 연결된다. 같은 노드 안의 Pod 간 통신은 이 bridge를 통해 L2 switching으로 처리된다.

```
Pod A (10.244.1.5)                    Pod B (10.244.1.6)
   eth0                                   eth0
    │                                      │
 vethAAA ──→ bridge (cni0) ──→ vethBBB
```

흐름:

1. Pod A `eth0` → vethAAA → bridge
2. bridge가 목적지 MAC 주소를 보고 vethBBB로 전달
3. vethBBB → Pod B `eth0`

## 5. 노드 간 — CNI 라우팅

목적지 Pod가 다른 노드에 있다면, bridge를 벗어나 노드 간 라우팅이 필요하다.

### 5.1. 문제: Pod IP는 물리 네트워크에서 라우팅되지 않는다

Pod IP는 클러스터 내부에서만 유효한 가상 주소다.
물리 네트워크 장비는 이 주소로 패킷을 어디로 보내야 하는지 모른다.

```
Node 1 (192.168.1.10)                Node 2 (192.168.1.11)
  Pod A → veth → bridge               bridge → veth → Pod C
                 │                        ↑
                 └──→ ??? ───────────────┘
                   물리 네트워크는 Pod IP를 모름
```

### 5.2. CNI 플러그인
CNI 플러그인이 이 문제를 해결한다.
##### 오버레이(Overlay)
Pod 패킷을 물리 네트워크가 이해할 수 있는 패킷으로 한 번 더 감싸서(encapsulation) 전달한다.

```
[Pod IP 패킷]
    ↓ encapsulation (VXLAN 등)
[물리 IP 헤더 | Pod IP 패킷]  ← 물리 네트워크로 전송
    ↓ decapsulation
[Pod IP 패킷]
```

##### 언더레이(Underlay)
물리 네트워크 장비에 Pod 라우트를 직접 알려준다. encapsulation 없이 패킷이 직접 라우팅된다.

```
물리 라우터가 Pod IP 대역 라우트를 BGP로 학습
    ↓
Pod 패킷이 encapsulation 없이 직접 라우팅됨
```

## 전체 흐름 정리

```
외부 클라이언트
    │
    │ ① NodePort / LoadBalancer / Ingress로 클러스터 진입
    ↓
Node (192.168.1.10)
    │
    │ ② iptables DNAT: ClusterIP → Pod IP 변환 (kube-proxy가 룰 관리)
    │    Pod 교체 시 Endpoints Controller → kube-proxy → 룰 자동 갱신
    ↓
목적지가 같은 노드라면:
    │ ③-A veth → bridge (cni0) → veth → Pod
    ↓
목적지가 다른 노드라면:
    │ ③-B veth → bridge → CNI 라우팅 (오버레이 또는 언더레이)
    │            → 목적지 Node → bridge → veth → Pod
    ↓
Pod (목적지)
```

---
## 6. CNI 플러그인 비교 (Flannel → Calico → Cilium)

### 6.1. Flannel: 단순한 오버레이

**핵심**: VXLAN으로 Pod 패킷을 감싸서 노드 간 전달

```
pod1 → veth → cni0 (bridge) → flannel.1 (VXLAN encap)
  → 물리 네트워크 → flannel.1 (VXLAN decap) → cni0 → veth → pod2
```

- **언제 쓰는가**: 네트워크 정책이 필요 없는 소규모 클러스터, 빠르게 구성해야 하는 개발/테스트 환경
- **한계**:
	- 모든 노드 간 패킷에 VXLAN encap/decap 오버헤드
	- 네트워크 정책 기능 없음

### 6.2. Calico: 라우팅 기반으로의 전환

**핵심**: bridge 없이 Pod마다 `/32` host route를 라우팅 테이블에 직접 등록. Felix가 iptables와 라우팅 테이블을 관리하고, Bird가 BGP로 라우트를 배포

```
# IP-IP 모드 (기본값): 노드 간 패킷을 IP-IP 터널로 전달
pod1 → cali-x → tunl0 (IP-IP encap) → 물리 네트워크
  → tunl0 (IP-IP decap) → cali-y → pod2

# BGP 모드: encapsulation 없이 직접 라우팅
pod1 → cali-x → 물리 네트워크 (no encap) → cali-y → pod2
```

- **언제 쓰는가**: 네트워크 정책이 필요한 환경, 물리 네트워크 장비가 BGP를 지원하는 경우(BGP 모드로 오버헤드 제거 가능)
- **한계**: 여전히 iptables에 의존 → Pod 수 증가 시 성능 저하

### 6.3. Cilium: eBPF로의 전환

**핵심**: iptables 대신 eBPF를 커널에 직접 삽입. 패킷이 커널에 들어오는 가장 이른 시점(XDP/TC hook)에 처리

```
Flannel/Calico:  NIC → iptables 체인 순회 (O(N)) → Pod
Cilium:          NIC → eBPF 해시맵 조회 (O(1)) → Pod
```

- **언제 쓰는가**: 대규모 클러스터(Pod 수백~수천), 고성능이 필요한 환경, L7 네트워크 정책이나 트래픽 가시성이 필요한 경우, kube-proxy까지 대체하고 싶은 경우
- **한계**: 구성이 복잡하고, eBPF를 지원하는 커널 버전이 필요

### 6.4. 세 플러그인 비교

| |Flannel|Calico|Cilium|
|---|---|---|---|
|노드 간 방식|VXLAN 오버레이|IP-IP 또는 BGP|VXLAN 또는 BGP|
|핵심 기술|VXLAN|iptables + BGP|eBPF|
|네트워크 정책|없음|있음 (Felix)|있음 (eBPF, L7까지)|
|성능 (Pod 증가 시)|오버레이 오버헤드|iptables O(N)|eBPF O(1)|
|kube-proxy 대체|불가|불가|가능|
|선택 기준|소규모/단순|정책 필요|대규모/고성능|

