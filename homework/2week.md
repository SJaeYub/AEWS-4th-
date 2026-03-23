## 목차

1. [AWS VPC CNI 소개](#1-aws-vpc-cni-소개)
1. [노드의 네트워크 구조](#2-노드의-네트워크-구조)
1. [노드 간 파드 통신](#3-노드-간-파드-통신)
1. [파드에서 외부 통신](#4-파드에서-외부-통신)
1. [AWS VPC CNI 주요 설정](#5-aws-vpc-cni-주요-설정)
1. [노드의 파드 수 제한 — Secondary IP · Prefix Delegation](#6-노드의-파드-수-제한)
1. [Kubernetes Service와 EKS 네트워킹](#7-kubernetes-service와-eks-네트워킹)
1. [AWS LoadBalancer Controller와 NLB](#8-aws-loadbalancer-controller와-nlb)
1. [Ingress와 ALB (L7)](#9-ingress와-alb-l7)
1. [ExternalDNS](#10-externaldns)
1. [Gateway API](#11-gateway-api)
1. [CoreDNS](#12-coredns)

-----

## 1. AWS VPC CNI 소개

### 1.1 CNI(Container Network Interface)란?

CNI는 CNCF에서 관리하는 **컨테이너 네트워크 설정 표준 인터페이스 스펙**입니다. kubelet이 파드를 생성·삭제할 때 `/etc/cni/net.d/` 설정 파일을 기반으로 CNI 바이너리(`/opt/cni/bin/`)를 호출하며, 플러그인은 다음 책임을 집니다.

- 파드 네트워크 네임스페이스 생성 및 인터페이스 부착
- IP 주소 할당 (IPAM 서브플러그인 또는 자체 구현)
- 라우팅 설정 및 네트워크 정책 적용

CNI 플러그인은 단일 바이너리이거나 체인(chained) 형태로 여러 플러그인을 순차 실행할 수 있습니다. AWS VPC CNI는 `aws-cni` 메인 플러그인과 `egress-cni`(IPv4 egress 지원) 두 바이너리를 함께 사용합니다.

### 1.2 주요 CNI 플러그인 종류 및 비교

|CNI            |네트워크 방식                          |데이터 플레인            |네트워크 정책           |특징                               |
|---------------|---------------------------------|-------------------|------------------|---------------------------------|
|**AWS VPC CNI**|Native VPC                       |iptables / eBPF(선택)|별도 필요             |ENI Secondary IP 직접 할당, AWS 완전 통합|
|**Calico**     |Overlay(VXLAN/IPIP) 또는 BGP Native|iptables / eBPF    |L3/L4 정책 풍부       |BGP 피어링으로 오버레이 없는 라우팅 가능         |
|**Cilium**     |eBPF Native                      |eBPF 전용            |L3~L7 (HTTP, gRPC)|가장 풍부한 정책, Hubble 관찰가능성          |
|**Flannel**    |Overlay(VXLAN)                   |iptables           |미지원               |단순, 학습용, 프로덕션 비권장                |
|**Weave Net**  |Overlay(VXLAN)                   |iptables           |기본 지원             |암호화 옵션, 소규모 클러스터                 |
|**Multus**     |메타 CNI                           |다중 플러그인 위임         |위임                |파드에 NIC 여러 개 부착 (NFV/텔코 환경)      |


> **EKS 선택 시 판단 기준**: VPC IP 소비가 문제라면 Calico(VXLAN)나 Cilium, 성능·AWS 통합이 우선이면 AWS VPC CNI. EKS Add-on으로 Cilium ENI 모드(IP 할당은 VPC CNI, 데이터 플레인은 eBPF)를 조합하는 방식도 증가 추세.

### 1.3 Native VPC Networking의 핵심 원리

**오버레이 방식의 문제**

오버레이 CNI는 VPC 위에 별도의 가상 네트워크 평면을 만듭니다. 파드 IP(`10.244.x.x`)는 호스트 OS와 VPC 라우터에게는 불투명하므로, 패킷을 호스트 IP로 캡슐화(encapsulation)해서 전달하고 목적지에서 역캡슐화(decapsulation)합니다. 이 과정에서 CPU 오버헤드, MTU 감소(VXLAN은 50바이트 헤더 추가), 네트워크 트러블슈팅 복잡도가 증가합니다.

**AWS VPC CNI의 접근법**

파드에 VPC의 실제 Secondary IP를 직접 할당합니다. 파드가 VPC의 일급 시민(First-class citizen)이 되므로:

- 캡슐화 오버헤드 없음 → 지연시간 최소화
- VPC Flow Logs, Security Group, Network ACL이 파드 수준에서 동작
- AWS 서비스(RDS, ElastiCache, MSK 등)와 직접 연결 가능
- VPC Reachability Analyzer로 파드 경로 분석 가능

```
[ 오버레이 방식 ]                        [ AWS VPC CNI ]
파드: 10.244.1.5/24 (가상 CIDR)          파드: 192.168.1.15/32 (VPC 실제 IP)
     ↕ VXLAN (UDP 8472)                      ↕ veth pair (직접)
노드: 192.168.1.10                       노드: 192.168.1.10
     ↕ 캡슐화 후 IP 전달                      ↕ VPC 라우팅 (캡슐화 없음)
노드: 192.168.2.10                       노드: 192.168.2.10
     ↕ 역캡슐화                               ↕
파드: 10.244.2.7/24 (가상 CIDR)          파드: 192.168.2.22/32 (VPC 실제 IP)
```

### 1.4 핵심 컴포넌트 상세

**ipamd (IP Address Manager Daemon)**

`aws-node` DaemonSet의 핵심 프로세스. `/var/run/aws-node/ipam.sock` Unix 소켓을 통해 gRPC 서버를 실행하며, CNI 플러그인의 IP 요청을 처리합니다.

주요 책임:

- ENI 생성 및 서브넷에서 Secondary IP 할당 (EC2 API 호출)
- Warm Pool 유지: 파드 생성 전 IP를 미리 확보해 할당 지연 방지
- 파드 IP 반납 시 Warm Pool로 회수
- Node Label(`vpc.amazonaws.com/eniLimitedPodDensity`)에 max-pod 정보 업데이트

**CNI Plugin Binary (`aws-cni`)**

kubelet이 호출하는 실행 바이너리. stdin으로 CNI 설정 JSON을 받고 환경변수로 네임스페이스/컨테이너 ID를 전달받습니다.

수행 작업:

1. ipamd gRPC 호출 → IP 할당 요청
1. `veth pair` 생성 (파드 네임스페이스 ↔ 호스트 네임스페이스)
1. 파드 네임스페이스 veth에 IP 할당, default route 설정 (`169.254.1.1` link-local)
1. 호스트 네임스페이스 veth에 `/32` 라우팅 엔트리 추가
1. Proxy ARP 활성화 (`net.ipv4.conf.eniXXX.proxy_arp = 1`)

**aws-k8s-agent (Node Agent)**

Node 레벨에서 ENI 트렁킹, Security Group for Pods, 네트워크 정책 기능을 담당하는 보조 에이전트. `aws-node` 파드 내부에서 별도 프로세스로 실행됩니다.

-----

### Ch.1 면접 질문 — AWS VPC CNI 소개

-----

#### Q1. AWS EKS에서 기본 CNI 플러그인으로 AWS VPC CNI를 사용하는 이유가 무엇인가요?

> **핵심 답변 방향**: Native VPC Networking의 이점(오버레이 제거, AWS 서비스 통합, 보안 그룹 파드 적용)과 트레이드오프(VPC IP 소비)를 함께 설명해야 합니다.

**▶ 1차 꼬리질문**: 오버레이 방식과 비교했을 때 성능상 구체적으로 어떤 차이가 있나요?

> 캡슐화/역캡슐화 오버헤드 제거, MTU 감소 없음, 커널 네트워크 스택 단축 경로 활용을 설명해야 합니다.

**▶▶ 2차 꼬리질문**: 그렇다면 실제로 레이턴시나 처리량에서 수치적으로 어느 정도 차이가 발생하나요? Calico VXLAN 대비 측정해 본 경험이 있나요?

> VXLAN 오버헤드는 일반적으로 5~15%의 CPU 추가 소비, RTT 기준 0.1~0.5ms 차이. iPerf3 등 벤치마크 경험을 언급하면 가산점.

**▶▶▶ 3차 꼬리질문**: 만약 AWS VPC CNI 대신 Cilium ENI 모드를 사용하면 성능 특성이 어떻게 달라지나요? 어떤 상황에서 Cilium ENI 모드를 선택하겠습니까?

> Cilium ENI 모드 = IP 할당은 VPC CNI, 데이터 플레인은 eBPF. kube-proxy 대체로 iptables 규칙 제거 → 대규모 서비스 클러스터에서 유리. L7 네트워크 정책, Hubble 관찰가능성 필요 시 선택.

-----

#### Q2. ipamd의 Warm Pool이란 무엇이고 왜 필요한가요?

> **핵심 답변 방향**: 파드 생성 시 AWS API 호출 지연(수백 ms ~ 수 초)을 미리 IP를 확보해 두는 방식으로 해소한다는 설명이 핵심입니다.

**▶ 1차 꼬리질문**: Warm Pool 크기를 너무 크게 설정하면 어떤 문제가 발생하나요?

> 서브넷 IP를 과점유하여 다른 노드/서비스의 IP 부족 유발. 특히 /24 서브넷에서 노드가 많을 때 IP 소진 문제 발생.

**▶▶ 2차 꼬리질문**: 실제 운영 중 서브넷 IP가 소진되어 파드 생성이 실패한 상황을 만난 적 있나요? 어떻게 해결했나요?

> 트러블슈팅 경험: `kubectl describe pod`에서 “no available IPs” 오류 확인 → `aws ec2 describe-subnets`로 가용 IP 확인 → WARM_IP_TARGET 낮추기, 서브넷 추가, Prefix Delegation 전환 등.

**▶▶▶ 3차 꼬리질문**: WARM_IP_TARGET, WARM_ENI_TARGET, MINIMUM_IP_TARGET을 동시에 설정하면 어떤 우선순위로 동작하나요? 계산 공식을 설명해 보세요.

> `목표 IP 수 = MAX(현재파드수 + WARM_IP_TARGET, MINIMUM_IP_TARGET)`, ENI 수 = `CEIL(목표IP / ENI당IP) + WARM_ENI_TARGET`. 세 파라미터 조합 전략을 설명할 수 있어야 함.

-----

#### Q3. CNI 플러그인을 선택할 때 어떤 기준으로 결정하시나요?

**▶ 1차 꼬리질문**: 네트워크 정책(NetworkPolicy)이 반드시 필요한 환경이라면 어떤 CNI를 선택하겠습니까?

> AWS VPC CNI 단독으로는 불가. Calico(iptables/eBPF), Cilium(eBPF, L7까지), AWS Network Policy Controller(EKS 네이티브, eBPF) 중 선택 기준 설명.

**▶▶ 2차 꼬리질문**: Calico와 Cilium 네트워크 정책의 차이점은 무엇인가요?

> Calico: L3/L4 정책, iptables 또는 eBPF 백엔드, 성숙도 높음. Cilium: L3/L4/L7까지 지원(HTTP 경로, gRPC 메서드 수준), eBPF 전용, Hubble로 가시성 제공.

**▶▶▶ 3차 꼬리질문**: Multus CNI는 어떤 상황에서 사용하며, EKS에서의 적용 제약은 무엇인가요?

> NFV/텔코 환경에서 파드에 NIC 다중 부착 시 사용. EKS에서는 보조 ENI를 Multus secondary interface로 활용 가능하지만, AWS 관리형 기능과 충돌 가능성 및 복잡도 증가 주의.

-----

## 2. 노드의 네트워크 구조

### 2.1 ENI(Elastic Network Interface) 할당 체계

EC2 인스턴스는 인스턴스 타입에 따라 최대 N개의 ENI를 가질 수 있으며, 각 ENI는 서브넷의 IP를 Primary IP 1개 + Secondary IP 최대 M개 가질 수 있습니다. AWS VPC CNI는 이 Secondary IP를 파드에 1:1 매핑합니다.

```
EC2 Node (m5.xlarge: 최대 ENI 4개, ENI당 최대 IP 15개)
│
├── Primary ENI (eth0) — 노드 자체 통신용
│   ├── Primary IP: 192.168.1.10  → 노드 OS, kubelet, kube-proxy
│   ├── Secondary IP: 192.168.1.11 → Pod A (할당됨)
│   ├── Secondary IP: 192.168.1.12 → Pod B (할당됨)
│   └── Secondary IP: 192.168.1.13 → [Warm Pool 대기]
│
├── Secondary ENI (eth1) — 파드 IP 공급용
│   ├── Primary IP: 192.168.1.20  → 사용 안 함
│   ├── Secondary IP: 192.168.1.21 → Pod C
│   ├── Secondary IP: 192.168.1.22 → Pod D
│   └── Secondary IP: 192.168.1.23 → [Warm Pool 대기]
│
└── Secondary ENI (eth2) — Warm Pool 전용 (파드 미생성 대기)
    ├── Primary IP: 192.168.1.30
    ├── Secondary IP: 192.168.1.31 → [대기]
    └── Secondary IP: 192.168.1.32 → [대기]
```

**왜 Primary ENI의 Primary IP는 파드에 안 쓰이는가?**
Primary IP는 노드 OS의 eth0에 바인딩되어 있으며, kubelet·kube-proxy·SSH 등 노드 시스템 통신에 사용됩니다. 파드에 할당하면 노드 자체 통신이 끊어지므로 예외 처리됩니다. Secondary ENI의 Primary IP도 마찬가지로 ENI 운영에 필요하여 파드에 미할당.

### 2.2 veth pair와 파드 네트워크 네임스페이스

Linux 네트워크 네임스페이스(netns)는 완전히 격리된 네트워크 스택(인터페이스, 라우팅, iptables 등)을 제공합니다. 파드는 고유한 netns를 가지며, 컨테이너들은 pause 컨테이너의 netns를 공유합니다.

`veth(Virtual Ethernet) pair`는 반드시 쌍으로 존재하는 가상 인터페이스로, 한쪽으로 들어온 패킷이 반대쪽으로 나오는 직통 가상 케이블입니다.

```
[ Host Network Namespace ]                 [ Pod Network Namespace ]
  eniXXXXXXX (veth host end)        ←→     eth0 (veth pod end)
  │                                         │
  │ IP 없음 (링크만 존재)                    │ IP: 192.168.1.11/32
  │ proxy_arp: enabled                      │ Default GW: 169.254.1.1
  │                                         │
  └── 호스트 라우팅 테이블:                  └── 파드 라우팅 테이블:
      192.168.1.11 dev eniXXX                   default via 169.254.1.1
      scope link                                 192.168.1.11 dev eth0 src 192.168.1.11
```

**Proxy ARP의 역할**

파드에서 `169.254.1.1`(게이트웨이)으로 ARP 요청을 보내면, 호스트 측 veth가 자신의 MAC 주소로 응답합니다(Proxy ARP). 이 덕분에 파드는 실제 게이트웨이가 없어도 호스트 네트워크 스택으로 패킷을 전달할 수 있습니다. `169.254.1.1`은 link-local 주소이므로 실제 할당된 IP 없이도 동작합니다.

**pause 컨테이너(infra 컨테이너)의 역할**

파드 내 첫 번째로 생성되는 특수 컨테이너. 파드의 네트워크 네임스페이스 생명주기를 관리합니다. 애플리케이션 컨테이너가 재시작되어도 netns는 유지되므로, IP 주소와 네트워크 상태가 보존됩니다.

### 2.3 멀티 ENI 환경의 라우팅 테이블 분리

ENI가 여러 개일 때 Linux에서 비대칭 라우팅 문제가 발생할 수 있습니다. 예를 들어 eth1로 들어온 패킷의 응답이 eth0으로 나가면 클라우드 네트워크에서 패킷이 드롭됩니다.

AWS VPC CNI는 이 문제를 **Policy-Based Routing(정책 기반 라우팅)** 으로 해결합니다.

```
# ENI별 독립 라우팅 테이블 생성
ip rule add from 192.168.1.20/32 lookup 101   # eth1의 IP → table 101 사용
ip route add default via 192.168.1.1 dev eth1 table 101

# 동작 원리: 특정 출발지 IP를 가진 패킷은 해당 ENI 전용 라우팅 테이블 사용
# 응답 패킷이 반드시 들어온 인터페이스로 나가도록 보장 (Symmetric Routing)
```

-----

### Ch.2 면접 질문 — 노드의 네트워크 구조

-----

#### Q1. EKS 노드에서 파드가 생성될 때 네트워크 설정이 어떻게 이루어지는지 단계별로 설명해 보세요.

> **핵심 답변 방향**: kubelet → CNI 바이너리 호출 → ipamd IP 요청 → veth pair 생성 → 파드 네임스페이스 설정 → 호스트 라우팅 추가의 순서를 명확히 서술해야 합니다.

**▶ 1차 꼬리질문**: veth pair에서 호스트 측 인터페이스에 IP가 없는데, 파드가 어떻게 게이트웨이를 찾아 패킷을 전달하나요?

> Proxy ARP 메커니즘. 파드에서 169.254.1.1(link-local GW)로 ARP 요청 → 호스트 측 veth가 자신의 MAC으로 응답(proxy_arp=1) → 파드는 호스트 스택으로 패킷 전달.

**▶▶ 2차 꼬리질문**: pause 컨테이너(infra 컨테이너)가 없다면 어떤 문제가 발생하나요?

> 애플리케이션 컨테이너가 재시작될 때 netns가 함께 삭제되어 IP와 포트가 바뀜. pause 컨테이너가 netns 생명주기 보유자 역할을 해야 파드 IP 안정성 보장.

**▶▶▶ 3차 꼬리질문**: 멀티 ENI 환경에서 Policy-Based Routing이 없으면 어떤 비대칭 라우팅 문제가 발생하고, AWS VPC CNI는 이를 어떻게 해결하나요?

> eth1으로 들어온 패킷 응답이 eth0으로 나가면 AWS VPC 보안 그룹이 출발지 IP 불일치로 드롭. VPC CNI는 ENI별 독립 라우팅 테이블(`ip rule`, `ip route table N`)로 응답이 반드시 수신 인터페이스로 나가도록 보장(Symmetric Routing).

-----

#### Q2. Security Group for Pods는 어떤 원리로 동작하나요?

**▶ 1차 꼬리질문**: 일반 Secondary IP 방식과 SGP의 차이를 ENI 구조 관점에서 설명해 보세요.

> 일반 방식: 파드 IP가 ENI Secondary IP → 노드 ENI의 보안 그룹 상속. SGP: 브랜치 ENI를 트렁크 ENI에 VLAN 태깅으로 연결 → 파드마다 독립 ENI → 보안 그룹 파드 단위 적용.

**▶▶ 2차 꼬리질문**: SGP 사용 시 파드 시작이 느려질 수 있다고 하는데, 그 이유와 완화 방법은 무엇인가요?

> 브랜치 ENI 생성/연결에 AWS API 호출 시간 필요(수 초). WARM_PREFIX_TARGET이나 ENI Trunking 사전 준비로 완화. 중요도 낮은 파드는 SGP 미적용하여 영향 최소화.

**▶▶▶ 3차 꼬리질문**: SGP와 Calico 네트워크 정책을 동시에 사용하면 어떤 충돌이나 주의사항이 있나요?

> SGP는 AWS 인프라 레벨(보안 그룹), Calico는 커널 iptables 레벨에서 동작. 둘 다 활성화 시 두 계층 모두 통과해야 통신 가능. 디버깅 시 두 레이어를 순서대로 점검해야 하며, iptables 규칙 폭발 주의.

-----

## 3. 노드 간 파드 통신

### 3.1 VPC Native 라우팅의 동작 원리

AWS VPC CNI에서 노드 간 파드 통신은 오버레이 없이 **VPC 라우팅 인프라를 직접 활용**합니다. 이것이 가능한 이유는 AWS가 ENI에 할당된 Secondary IP 정보를 VPC 라우팅 인프라에 자동으로 반영하기 때문입니다.

```
Pod A (192.168.1.11) → Pod B (192.168.2.22) 통신 흐름

① Pod A → veth → Node1 호스트 스택 도달
② Node1 라우팅 테이블: 192.168.2.22는 로컬 엔트리 없음 → default GW(VPC 라우터)로 전달
③ VPC 라우터: 192.168.2.22는 Node2의 eth1(Secondary ENI)에 할당된 IP임을 인식
   └─ AWS 내부에서 ENI Secondary IP → ENI 매핑 정보를 관리하기 때문
④ Node2 eth1(192.168.1.20)으로 패킷 전달
⑤ Node2 호스트 라우팅: 192.168.2.22 dev eniYYY → veth → Pod B
```

**핵심 포인트**: AWS는 EC2 인스턴스에 ENI Secondary IP를 할당할 때 해당 IP를 VPC 라우팅 레이어에 등록합니다. 별도의 라우팅 테이블 항목 추가 없이 VPC 내에서 파드 IP가 직접 라우팅됩니다.

### 3.2 Security Group for Pods (SGP)

일반 모드에서는 파드의 패킷이 노드의 ENI를 통과하므로 보안 그룹이 노드 단위로 적용됩니다. SGP를 활성화하면 파드마다 독립적인 보안 그룹을 적용할 수 있습니다.

**구현 방식: 트렁크 ENI + 브랜치 ENI**

```
Node
├── Trunk ENI (eth0) — 기존 노드 통신 및 브랜치 ENI 수용
│   └── Branch ENI 1 → Pod A (SG: backend-sg)
│   └── Branch ENI 2 → Pod B (SG: frontend-sg)
└── 일반 Secondary IP ENI (eth1, eth2...) — SGP 미적용 파드
```

브랜치 ENI는 VLAN 태깅으로 트렁크 ENI에 멀티플렉싱됩니다. 각 파드가 물리적으로 독립된 ENI를 가지므로 보안 그룹이 파드 수준에서 적용됩니다. 단, **Nitro 인스턴스**에서만 지원되며, 브랜치 ENI 수 제한이 별도로 있습니다.

### 3.3 노드 간 통신 시 주의사항

**같은 서브넷 vs 다른 서브넷**

같은 AZ, 같은 서브넷 내 노드 간 통신은 동일 서브넷 로컬 라우팅으로 처리됩니다. 다른 AZ의 노드와 통신할 때는 **AZ 간 데이터 전송 비용**이 발생합니다(약 $0.01/GB). 고트래픽 서비스는 Topology Aware Routing으로 AZ 내부 통신을 우선하도록 설계해야 합니다.

**MTU 고려**

VPC의 기본 MTU는 9001바이트(Jumbo Frame)이나, 인터페이스 설정에 따라 1500일 수 있습니다. AWS VPC CNI는 기본적으로 점보 프레임을 지원하므로 오버레이 CNI의 MTU 감소 문제가 없습니다. 단, VPN/Direct Connect 경유 시 MTU 불일치를 점검해야 합니다.

-----

### Ch.3 면접 질문 — 노드 간 파드 통신

-----

#### Q1. AWS VPC CNI에서 노드 간 파드 통신 시 VXLAN 같은 캡슐화 없이 어떻게 직접 라우팅이 가능한가요?

> **핵심 답변 방향**: AWS가 ENI Secondary IP 정보를 VPC 라우팅 인프라에 자동 반영 → 파드 IP가 VPC 라우터에게 직접 라우팅 가능한 주소로 인식됨.

**▶ 1차 꼬리질문**: 다른 AZ에 있는 노드의 파드와 통신할 때 추가로 고려해야 할 사항은 무엇인가요?

> AZ 간 데이터 전송 비용($0.01/GB), 레이턴시 증가. Topology Aware Routing으로 AZ 내부 파드 우선 라우팅 설정 권장.

**▶▶ 2차 꼬리질문**: Topology Aware Routing은 어떤 원리로 동작하며, EndpointSlice와 어떻게 연동되나요?

> kube-proxy가 EndpointSlice의 `hints.forZones` 정보를 읽어 같은 AZ의 엔드포인트를 우선 선택. 자동 힌트 생성은 `service.kubernetes.io/topology-mode: auto` 설정으로 활성화.

**▶▶▶ 3차 꼬리질문**: Topology Aware Routing이 완벽하지 않은 경우(파드 분포 불균형)를 설명하고, 이를 보완하는 방법은 무엇인가요?

> 특정 AZ 파드가 모두 다운되면 해당 AZ 요청이 실패. `externalTrafficPolicy: Local`과 조합 시 더 위험. Pod Topology Spread Constraints로 AZ 균등 분배를 강제하여 보완.

-----

#### Q2. 파드 간 통신 중 패킷 드롭이 발생했을 때 어떻게 트러블슈팅하나요?

**▶ 1차 꼬리질문**: 네트워크 정책, 보안 그룹, NACL 중 어느 레이어에서 문제가 발생했는지 어떻게 판별하나요?

> 순서: 파드 내 tcpdump → 호스트 iptables 로그 → 보안 그룹 규칙 확인 → NACL 확인 → VPC Flow Logs 분석. 레이어별 순차 배제법.

**▶▶ 2차 꼬리질문**: VPC Flow Logs에서 ACCEPT인데도 애플리케이션에서 연결이 안 될 수 있는 원인은 무엇인가요?

> conntrack 상태 불일치(비대칭 라우팅), MTU 불일치(대용량 패킷 드롭), 애플리케이션 레벨 오류(TLS 인증서 불일치), 파드 내 iptables 충돌.

**▶▶▶ 3차 꼬리질문**: MTU 불일치로 인한 패킷 드롭을 어떻게 진단하고 해결하나요?

> `ping -M do -s 1400 <대상IP>`로 PMTUD 테스트. ICMP “fragmentation needed” 메시지 차단 여부 확인. VPN/Direct Connect 경유 시 MSS clamping(`iptables -t mangle -A FORWARD -p tcp --tcp-flags SYN,RST SYN -j TCPMSS --clamp-mss-to-pmtu`) 설정으로 해결.

-----

## 4. 파드에서 외부 통신

### 4.1 SNAT(Source NAT) 동작 원리

파드의 Secondary IP는 VPC 내에서 라우팅 가능하지만, **인터넷 게이트웨이(IGW)는 Public IP가 연결된 인터페이스(노드의 Primary ENI)만 인터넷으로 내보낼 수 있습니다.** 따라서 파드가 인터넷으로 패킷을 보낼 때 iptables가 출발지 IP를 노드 Primary IP로 변환합니다.

```
iptables 처리 흐름 (PREROUTING/POSTROUTING):

파드 패킷: src=192.168.1.11, dst=8.8.8.8
           ↓
  POSTROUTING → AWS-SNAT-CHAIN-0
  └─ 목적지가 VPC CIDR(192.168.0.0/16)이면 → RETURN (SNAT 미적용)
  └─ 목적지가 VPC 외부이면 → AWS-SNAT-CHAIN-1
     └─ MASQUERADE (노드 Primary IP로 변환)

변환 후: src=192.168.1.10(노드 IP), dst=8.8.8.8
          ↓ NAT Gateway
변환 후: src=<EIP>, dst=8.8.8.8
```

### 4.2 SNAT 관련 설정과 트레이드오프

|설정                                  |동작                     |장점           |단점                    |
|------------------------------------|-----------------------|-------------|----------------------|
|`EXTERNALSNAT=false` (기본)           |인터넷 통신 시 노드 IP로 SNAT   |간단, 별도 설정 불필요|파드 원본 IP 손실, 로그 추적 어려움|
|`EXTERNALSNAT=true`                 |SNAT 비활성화, 파드 IP 그대로 전달|파드 IP 보존     |NAT GW가 파드 서브넷을 알아야 함 |
|`AWS_VPC_K8S_CNI_EXCLUDE_SNAT_CIDRS`|특정 CIDR만 SNAT 제외       |세밀한 제어       |설정 복잡도 증가             |

**SNAT 비활성화가 적합한 경우**

- On-premise와 Direct Connect/VPN 연결: 파드 IP 그대로 전달하여 방화벽 정책 적용 가능
- 감사 로그에서 파드 IP 추적 필요 시
- 파드가 Elastic IP를 직접 보유해야 하는 경우 (Security Group for Pods + SNAT 비활성화)

### 4.3 IPv6 지원과 Dual-Stack

EKS는 IPv6 전용 클러스터를 지원합니다. IPv6 모드에서는 파드가 IPv6 주소를 받으며, VPC IPv6 CIDR에서 할당됩니다. IPv6는 SNAT 없이 인터넷과 직접 통신 가능하므로(글로벌 유니캐스트 주소) SNAT 복잡도가 사라집니다.

```
IPv4 파드 통신:
  파드 IPv4 → SNAT → 노드 IP → NAT GW → IGW → 인터넷

IPv6 파드 통신:
  파드 IPv6 → IGW → 인터넷 (SNAT 없음)
```

단, IPv6 전용 클러스터는 인터넷으로의 IPv4 트래픽을 NAT64/DNS64로 처리해야 합니다.

-----

### Ch.4 면접 질문 — 파드에서 외부 통신

-----

#### Q1. EKS 파드가 인터넷으로 통신할 때 SNAT가 적용되는 과정을 설명해 보세요.

> **핵심 답변 방향**: iptables `AWS-SNAT-CHAIN`이 목적지가 VPC 외부인 패킷의 출발지 IP를 노드 Primary IP로 변환한다는 흐름을 설명해야 합니다.

**▶ 1차 꼬리질문**: SNAT를 비활성화(`EXTERNALSNAT=true`)하면 어떤 추가 설정이 필요한가요?

> 파드 서브넷에서 NAT Gateway로 가는 라우팅을 VPC 라우팅 테이블에 명시적으로 추가 필요. 또는 파드 IP에 EIP 직접 연결(SGP 필요).

**▶▶ 2차 꼬리질문**: Direct Connect로 연결된 온프레미스 환경에서 파드의 원본 IP를 보존해야 한다면 어떤 설정을 하겠습니까?

> `EXTERNALSNAT=true` + `AWS_VPC_K8S_CNI_EXCLUDE_SNAT_CIDRS`로 온프레미스 CIDR 지정 → 해당 대역으로의 통신만 SNAT 제외. 온프레미스 방화벽에서 파드 CIDR 허용 규칙 추가.

**▶▶▶ 3차 꼬리질문**: IPv6 EKS 클러스터에서 SNAT가 필요 없는 이유를 설명하고, 그럼에도 IPv4 대상과 통신해야 할 때 어떻게 처리하나요?

> IPv6 글로벌 유니캐스트 주소 → IGW가 직접 라우팅, SNAT 불필요. IPv4 대상 통신: DNS64(AAAA 레코드 합성) + NAT64(IPv6 → IPv4 변환) 게이트웨이 조합. Route 53 Resolver에서 DNS64 활성화 가능.

-----

## 5. AWS VPC CNI 주요 설정

### 5.1 Warm Pool 파라미터 심화

ipamd의 Warm Pool은 3가지 독립 파라미터로 제어되며, 이들이 조합되어 최종 목표 IP 수를 결정합니다.

```
목표 IP 수 = MAX(
  현재 파드 수 + WARM_IP_TARGET,
  MINIMUM_IP_TARGET
)

목표 ENI 수 = CEIL(목표 IP 수 / ENI당 IP 수) + WARM_ENI_TARGET
```

**파라미터 조합 전략**

|상황          |권장 설정                                   |이유               |
|------------|----------------------------------------|-----------------|
|예측 불가능한 스파이크|`WARM_ENI_TARGET=1`                     |ENI 단위로 대량 IP 선확보|
|파드 수 안정적    |`WARM_IP_TARGET=2, MINIMUM_IP_TARGET=10`|IP 낭비 최소화        |
|서브넷 IP 부족   |`WARM_IP_TARGET=1, MINIMUM_IP_TARGET=5` |최소한의 Warm Pool 유지|
|대규모 배치 작업   |`WARM_IP_TARGET=10`                     |빠른 파드 프로비저닝      |

**IP 소진 문제와 대응**

/24 서브넷은 256개 IP를 보유하지만, Warm Pool로 인해 실제 파드 수보다 더 많은 IP가 선점됩니다. 서브넷 IP 소진은 “no available IPs” 오류로 파드 스케줄링 실패를 유발합니다.

대응 방법:

1. 서브넷 CIDR을 넉넉하게 설계 (`/19` 이상 권장)
1. Prefix Delegation 활성화로 IP 효율 향상 (6장 참고)
1. MINIMUM_IP_TARGET / WARM_IP_TARGET 값 최소화
1. 다중 서브넷으로 노드 분산

### 5.2 Security Group for Pods 설계 고려사항

SGP는 강력한 기능이지만 다음 제약을 이해하고 설계해야 합니다.

- **Branch ENI 한계**: 인스턴스 타입별 최대 Branch ENI 수 제한 있음. 초과 시 파드는 일반 Secondary IP 모드로 동작
- **노드 타입**: Nitro 기반 인스턴스만 지원 (t2 계열 미지원)
- **준비 지연**: Branch ENI 연결 시간이 Secondary IP 방식보다 길어 파드 시작 지연 발생 가능
- **HostNetwork 파드 미지원**: HostNetwork 사용 파드는 SGP 미적용

### 5.3 Network Policy 지원

기본 AWS VPC CNI는 네트워크 정책을 지원하지 않습니다. 네트워크 정책이 필요한 경우:

|방법                               |구현                      |특징                 |
|---------------------------------|------------------------|-------------------|
|**Calico 병행 설치**                 |VPC CNI + Calico (정책 전용)|검증된 조합, 별도 관리 필요   |
|**AWS Network Policy Controller**|VPC CNI에 내장 (eBPF)      |EKS 네이티브, AWS 관리형  |
|**Cilium (ENI 모드)**              |Cilium이 IP 관리도 담당       |L7 정책, Hubble 관찰가능성|

AWS Network Policy Controller는 EKS 1.25부터 지원되며, eBPF를 사용하여 iptables 규칙 없이 네트워크 정책을 적용합니다.

-----

### Ch.5 면접 질문 — AWS VPC CNI 주요 설정

-----

#### Q1. 운영 중인 클러스터에서 서브넷 IP가 부족한 상황이 발생했습니다. 즉각적인 대응 방법과 근본적인 해결 방법을 설명해 보세요.

> **핵심 답변 방향**: 즉각 대응(Warm Pool 파라미터 최소화, 불필요 파드 제거)과 근본 해결(Prefix Delegation 전환, 서브넷 추가, CIDR 재설계)을 구분해 설명해야 합니다.

**▶ 1차 꼬리질문**: 운영 중 서브넷 CIDR을 변경하는 것은 가능한가요? 불가능하다면 어떻게 IP를 추가로 확보하나요?

> VPC 서브넷 CIDR 자체는 변경 불가. 해결책: VPC에 Secondary CIDR 블록 추가(`100.64.0.0/16` 등) → 새 서브넷 생성 → 노드 그룹 서브넷에 추가. 단, 기존 파드는 이동 없이 신규 파드부터 새 서브넷 사용.

**▶▶ 2차 꼬리질문**: Secondary CIDR로 `100.64.0.0/10`을 사용하는 이유는 무엇인가요?

> RFC 6598에서 정의한 공유 주소 공간(Shared Address Space). VPC의 주요 CIDR(10.x, 172.16-31.x, 192.168.x)와 겹치지 않으면서 인터넷에서도 라우팅되지 않는 대역. EKS 파드 전용 IP 확장에 최적.

**▶▶▶ 3차 꼬리질문**: Secondary CIDR 추가 후 새 서브넷의 파드와 기존 서브넷의 파드 간 통신은 어떻게 보장되나요? 라우팅과 보안 그룹 관점에서 설명해 보세요.

> 같은 VPC 내이므로 VPC 로컬 라우팅으로 자동 처리. 보안 그룹: 기존 SG의 소스에 새 서브넷 CIDR 추가 필요. 또는 파드 간 공통 보안 그룹으로 통일 관리.

-----

## 6. 노드의 파드 수 제한

### 6.1 제한의 근본 원인

파드 수 제한은 Kubernetes의 정책이 아니라 **AWS EC2 인스턴스의 ENI·IP 할당 한계**에서 비롯됩니다. kubelet의 `--max-pods` 플래그도 이 물리적 한계를 초과하지 않도록 계산된 값을 사용합니다.

### 6.2 Secondary IP 방식

#### 계산 공식

```
최대 파드 수 = (최대 ENI 수 × (ENI당 최대 Secondary IP 수)) + 2

+2: 노드 자체 운영 파드(kube-proxy, aws-node 등)는 HostNetwork를 사용하므로
    IP를 소비하지 않지만, Amazon 공식 max-pods 계산에서 +2를 포함
```

#### 인스턴스 타입별 Secondary IP 방식 한계

|인스턴스       |최대 ENI|ENI당 IP|최대 파드|
|-----------|------|-------|-----|
|t3.small   |3     |4      |11   |
|t3.medium  |3     |6      |17   |
|m5.large   |3     |10     |29   |
|m5.xlarge  |4     |15     |58   |
|m5.2xlarge |4     |15     |58   |
|m5.4xlarge |8     |30     |234  |
|m5.12xlarge|8     |30     |234  |
|c5.large   |3     |10     |29   |
|c5.xlarge  |4     |15     |58   |
|c5.4xlarge |8     |30     |234  |
|r5.xlarge  |4     |15     |58   |


> **주의**: t3 계열은 버스트 가능 인스턴스이며 ENI 수가 적어 파드 밀도가 낮습니다. 파드 밀도 중요 시 m5/c5 계열 권장.

#### Secondary IP 방식의 구조적 한계

Secondary IP 방식은 **ENI 당 IP 수가 제한**되어 있어, 고밀도 파드 배포에는 병목이 됩니다. 예를 들어 m5.xlarge는 아무리 메모리/CPU가 남아있어도 파드를 58개 이상 배포할 수 없습니다.

### 6.3 Prefix Delegation 방식 (심화)

#### 개념과 등장 배경

Prefix Delegation(PD)은 ENI에 **개별 IP 대신 `/28` CIDR 블록(16개 IP)을 통째로 할당**하는 방식입니다. EKS 1.21+, Nitro 인스턴스에서 지원되며, `ENABLE_PREFIX_DELEGATION=true` 환경변수로 활성화합니다.

#### 구조 비교

```
[ Secondary IP 방식 ]
ENI 1개
  ├── Secondary IP: 192.168.1.11 → Pod 1
  ├── Secondary IP: 192.168.1.12 → Pod 2
  ├── Secondary IP: 192.168.1.13 → Pod 3
  └── ... (최대 14개 Secondary IP = 14 파드)

[ Prefix Delegation 방식 ]
ENI 1개
  ├── Prefix: 192.168.1.0/28  → 15개 IP (16 - 1 네트워크 주소) 사용 가능
  │   ├── 192.168.1.1 → Pod 1
  │   ├── 192.168.1.2 → Pod 2
  │   └── ... (15 파드)
  ├── Prefix: 192.168.2.0/28  → 15개 IP
  └── ... (ENI당 최대 Prefix 수만큼 블록 추가)
```

#### Prefix Delegation 최대 파드 수 계산

```
최대 파드 수 = (최대 ENI 수) × (ENI당 최대 Prefix 수) × 16 + 2

m5.xlarge 예시:
  4 (ENI) × 4 (Prefix/ENI) × 16 (IP/Prefix) = 256개 → EKS 제한 110개로 캡
```

EKS는 Kubernetes 공식 권장 최대치(110개)를 상한선으로 제한합니다. Prefix Delegation 활성화 시 대부분의 인스턴스 타입이 이 상한에 도달하므로, 인스턴스 타입 선택의 자유도가 높아집니다.

#### Prefix Delegation 적용 시 ENI 수 감소

Secondary IP 방식에서 많은 ENI가 필요했던 이유는 IP 수 확보 때문이었습니다. PD를 사용하면 **동일한 파드 수를 더 적은 ENI로 수용**할 수 있어, ENI 소비 감소 → 다른 목적의 ENI 여유 확보 → Network Policy, SGP 등에 활용 가능해집니다.

|조건                  |Secondary IP  |Prefix Delegation     |
|--------------------|--------------|----------------------|
|m5.xlarge에서 50 파드 수용|ENI 4개 필요     |ENI 1~2개로 수용 가능       |
|서브넷 IP 소비           |파드 수 ≒ 소비 IP 수|/28 블록 단위 소비 (낭비 가능)  |
|IP 할당 효율            |1 IP = 1 파드   |최대 16 IP 중 일부만 사용 시 낭비|

#### Prefix Delegation의 단점과 주의사항

1. **IP 낭비 가능성**: /28 블록 전체를 서브넷에서 예약하므로, 파드가 적을 때 IP가 낭비됩니다. 서브넷 설계 시 이를 반영해야 합니다.
1. **기존 노드 미적용**: PD 활성화 후 새로 생성된 노드에만 적용됩니다. 기존 노드는 재생성(node drain → terminate) 필요.
1. **서브넷 연속 블록 필요**: /28 블록이 서브넷에서 연속으로 할당되어야 합니다. IP가 단편화된 서브넷에서는 할당 실패 가능.
1. **Windows 노드 미지원**: Linux 노드만 PD 지원.

#### Prefix Delegation 파드 IP 할당 흐름

```
1. ipamd가 ENI에 /28 Prefix 요청 (EC2 AssignIpv4PrefixesToNetworkInterface)
2. AWS가 서브넷에서 연속된 /28 블록 할당 (예: 192.168.1.16/28)
3. ipamd가 /28 내 IP를 Warm Pool에 등록
4. 파드 생성 시 Warm Pool에서 개별 IP 할당
5. 호스트 라우팅 테이블에 파드 IP(/32) 엔트리 추가 (Secondary IP 방식과 동일)
```

파드에서의 동작은 Secondary IP 방식과 동일합니다. 차이는 ipamd가 IP를 AWS API에서 할당받는 단위(개별 IP vs CIDR 블록)뿐입니다.

### 6.4 파드 밀도 설계 체크리스트

```
✅ 인스턴스 타입 선정
   - 예상 파드 수 > 58개 → m5.xlarge 이상 또는 PD 활성화
   - 메모리/CPU 여유 있지만 파드 수 부족 → PD 활성화

✅ 서브넷 설계
   - Secondary IP: 서브넷 IP 수 ≥ (노드 수 × 예상 파드/노드 × 1.3) [여유 30%]
   - Prefix Delegation: 서브넷을 /19 이상으로 설계, /28 단편화 고려

✅ Warm Pool 튜닝
   - 서브넷 IP 여유 있음: WARM_ENI_TARGET=1로 빠른 스케일업
   - 서브넷 IP 부족: WARM_IP_TARGET=2, MINIMUM_IP_TARGET=5로 최소화

✅ Nitro 인스턴스 확인 (PD, SGP 사용 시)
   - t2, t3: Nitro 아님 → PD 미지원
   - t3a, t4g, m5, c5, r5 이상: Nitro 지원
```

-----

### Ch.6 면접 질문 — 노드의 파드 수 제한

-----

#### Q1. m5.xlarge 노드에서 파드를 58개 이상 배포하려고 할 때 어떻게 해결하나요?

> **핵심 답변 방향**: Secondary IP 방식의 ENI 한계(4 ENI × 14 IP = 56 + 2 = 58)를 설명하고, Prefix Delegation으로 `/28` 블록 단위 할당 → 최대 110개 가능을 설명해야 합니다.

**▶ 1차 꼬리질문**: Prefix Delegation 활성화 시 서브넷 IP 사용량이 증가하는 이유는 무엇인가요?

> /28 블록(16 IP)을 통째로 서브넷에서 예약. 예) 파드 3개만 있어도 /28 블록의 16 IP 전체가 서브넷에서 할당됨. 파드 밀도 낮을 때는 Secondary IP보다 IP 낭비 발생.

**▶▶ 2차 꼬리질문**: Prefix Delegation으로 전환하기 위해 기존 노드를 어떻게 교체하나요? Blue/Green 노드 교체 절차를 설명해 보세요.

> 1. 신규 노드 그룹 생성(PD 활성화 상태). 2) 기존 노드 그룹 cordon(스케줄링 차단). 3) drain(파드 이동). 4) 기존 노드 그룹 삭제. 이 과정에서 PodDisruptionBudget이 드레인 속도를 제어하므로 사전 PDB 설정 확인 필요.

**▶▶▶ 3차 꼬리질문**: PD 활성화 후 서브넷에서 /28 블록이 단편화된 경우 IP 할당이 실패할 수 있는데, 이를 사전에 방지하는 방법은 무엇인가요?

> 서브넷을 신규 생성하여 단편화 없는 상태로 시작. 또는 `aws ec2 describe-subnets --filters Name=available-ip-address-count` 등으로 사전 확인. 장기적으로 서브넷 CIDR을 /19 이상으로 설계하여 연속 블록 확보 여유 유지.

-----

#### Q2. t3.medium과 m5.xlarge 중 파드 밀도가 중요한 워크로드에서 어떤 인스턴스를 선택하고, 그 이유는 무엇인가요?

**▶ 1차 꼬리질문**: 인스턴스 선택 시 파드 수 외에 어떤 추가 고려 사항이 있나요?

> CPU/메모리 자원, ENI 수(SGP/Network Policy용 여유), Nitro 여부(PD/SGP 지원), 네트워크 대역폭, 비용 대비 파드 밀도.

**▶▶ 2차 꼬리질문**: 같은 t3.medium 클러스터에서 일부 노드만 파드 수 한계에 도달했습니다. 어떻게 파드를 분산하겠습니까?

> Pod Topology Spread Constraints로 노드 간 균등 분산 강제. PodAffinity/Anti-affinity로 특정 노드 회피. 또는 VPA(Vertical Pod Autoscaler)로 과도한 리소스 요청 최적화.

**▶▶▶ 3차 꼬리질문**: 파드 스케줄링이 안 되는 원인을 진단하는 방법과, “no available IPs” vs “Insufficient cpu” 오류를 구분하는 방법을 설명해 보세요.

> `kubectl describe pod <POD>` → Events 섹션 확인. “no available IPs” → CNI IP 부족. “Insufficient cpu/memory” → 자원 부족. “0/3 nodes are available” → 자원 또는 taint/toleration 불일치. `kubectl get events --sort-by=.lastTimestamp`로 전체 흐름 파악.

-----

## 7. Kubernetes Service와 EKS 네트워킹

### 7.1 Service의 본질: 가상 IP 기반 안정적 엔드포인트

파드는 언제든 재생성될 수 있으며 IP가 변경됩니다. Kubernetes Service는 **변하지 않는 ClusterIP(가상 IP)** 를 제공하고, 이 IP로 들어오는 트래픽을 실제 파드로 분산합니다. ClusterIP는 어떤 네트워크 인터페이스에도 바인딩되지 않는 순수 가상 주소로, 트래픽 처리는 각 노드의 데이터 플레인(kube-proxy 또는 대체재)이 담당합니다.

### 7.2 kube-proxy 4가지 모드 상세 비교

#### (1) iptables 모드 (EKS 기본값)

**동작 원리**

kube-proxy가 Service/Endpoints 변경을 감지할 때마다 각 노드의 iptables 규칙을 갱신합니다. 커널의 netfilter 프레임워크를 사용하며, 실제 트래픽 처리는 커널 내에서 이루어집니다.

```
패킷 처리 체인:

패킷 도달: dst=10.96.80.80:80 (ClusterIP)
  │
  ▼ PREROUTING (nat 테이블)
  │
  ▼ KUBE-SERVICES 체인
  │   └─ dst IP 매칭: -d 10.96.80.80/32 --dport 80 → KUBE-SVC-XXXXX 점프
  │
  ▼ KUBE-SVC-XXXXX 체인 (서비스 체인 = 로드밸런싱)
  │   ├─ -m statistic --mode random --probability 0.33 → KUBE-SEP-AAA (Pod A)
  │   ├─ -m statistic --mode random --probability 0.50 → KUBE-SEP-BBB (Pod B)
  │   └─ → KUBE-SEP-CCC (Pod C) [나머지 확률]
  │
  ▼ KUBE-SEP-XXX 체인 (엔드포인트 체인 = DNAT)
      └─ DNAT --to-destination 192.168.1.11:8080

최종: dst 변경 → 192.168.1.11:8080 으로 파드에 직접 전달
```

**iptables 모드의 한계**

- 규칙 수가 서비스 수에 비례하여 증가: 서비스 1000개 × 파드 10개 = iptables 규칙 수만 개
- 규칙 갱신 시 전체 체인을 재작성(flush & reload): O(n²) 복잡도 → 대규모 클러스터에서 갱신 지연
- 순차 규칙 매칭: 패킷마다 규칙을 순서대로 검사 → 서비스 수 증가 시 지연
- random probability 기반 로드밸런싱: 완벽한 균등 분산 미보장

#### (2) IPVS 모드

**동작 원리**

IPVS(IP Virtual Server)는 Linux 커널의 L4 로드밸런싱 모듈입니다. iptables와 달리 해시 테이블 기반으로 서비스를 관리하여 O(1) 룩업이 가능합니다.

```
IPVS 구조:

Virtual Server (ClusterIP:Port)
  └─ Real Server 1: 192.168.1.11:8080 [weight: 1, conn: 45]
  └─ Real Server 2: 192.168.1.12:8080 [weight: 1, conn: 43]
  └─ Real Server 3: 192.168.2.11:8080 [weight: 1, conn: 47]

ipvsadm -L -n 출력 예시:
TCP  10.96.80.80:80 rr         ← Round Robin 알고리즘
  -> 192.168.1.11:8080         Local   1      45     0
  -> 192.168.1.12:8080         Local   1      43     0
  -> 192.168.2.11:8080         Remote  1      47     0
```

**IPVS 로드밸런싱 알고리즘**

|알고리즘                           |설명        |적합 사례     |
|-------------------------------|----------|----------|
|`rr` (Round Robin)             |순서대로 분산   |동질 서버     |
|`lc` (Least Connection)        |연결 수 최소 서버|처리 시간 불균일 |
|`dh` (Destination Hash)        |목적지 IP 해시 |캐시 서버     |
|`sh` (Source Hash)             |출발지 IP 해시 |세션 고정 필요 시|
|`sed` (Shortest Expected Delay)|예상 지연 최소  |응답 시간 민감  |
|`nq` (Never Queue)             |유휴 서버 우선  |-         |

**IPVS 모드의 장점과 단점**

장점:

- O(1) 해시 기반 룩업 → 서비스 수 증가에도 선형 성능 유지
- 다양한 LB 알고리즘 선택 가능
- 연결 테이블 관리 → 장기 연결 추적 용이

단점:

- 커널 모듈(`ip_vs`, `ip_vs_rr` 등) 사전 로드 필요
- `ipset`에 의존 → 추가 시스템 의존성
- iptables 완전 대체 아님 (NodePort, Masquerade 등 일부는 여전히 iptables 사용)
- EKS에서 IPVS 모드 사용 시 별도 노드 그룹 launch template 설정 필요

#### (3) nftables 모드 (Kubernetes 1.31+ Beta)

**개념**

nftables는 iptables의 후속 기술로, Linux 커널 3.13+부터 포함된 패킷 필터링 프레임워크입니다. iptables의 복잡한 체인 구조를 단순화하고, 원자적 규칙 갱신을 지원합니다.

```
nftables 구조 특징:

# iptables 방식: 규칙이 여러 테이블/체인에 분산
iptables -t nat -A PREROUTING -j KUBE-SERVICES
iptables -t nat -A KUBE-SERVICES -d 10.96.80.80 ...

# nftables 방식: 단일 테이블 내 맵/집합으로 관리
table inet kube-proxy {
  map services {
    typeof ip daddr . tcp dport : verdict
    elements = { 10.96.80.80 . 80 : goto svc-backend }
  }
  chain prerouting {
    ip daddr . tcp dport vmap @services
  }
}
```

**nftables의 핵심 개선점**

- **원자적 규칙 교체**: 규칙 세트 전체를 한 번의 커널 호출로 교체 → 갱신 중 불일치 상태 없음
- **맵/집합 기반 룩업**: 서비스 수 증가해도 단일 맵 룩업으로 처리 → O(1)에 가까운 성능
- **규칙 수 감소**: iptables 대비 규칙 수 수십 배 감소 (동일 서비스 수 기준)

> **EKS 실무 참고**: Kubernetes 1.31에서 Beta로 승격되었으나 EKS에서 기본 활성화까지는 시간이 필요합니다. 현재는 실험적 성격.

#### (4) eBPF 모드 (Cilium / AWS Network Policy Controller)

**eBPF의 원리**

eBPF(extended Berkeley Packet Filter)는 커널 코드를 수정하지 않고 커널 내에서 프로그램을 안전하게 실행하는 기술입니다. 네트워크 처리를 위해 XDP(eXpress Data Path)나 TC(Traffic Control) 훅에 프로그램을 부착합니다.

```
패킷 처리 레이어 비교:

[ iptables/IPVS 방식 ]
NIC → 드라이버 → netfilter hooks → iptables 규칙 매칭 → 전달
                 (소프트웨어 스택 전체 통과)

[ eBPF 방식 (XDP) ]
NIC → 드라이버 → XDP hook → eBPF 프로그램 (즉시 처리/드롭/전달)
                 (netfilter 완전 우회, 가장 빠른 지점에서 처리)

[ eBPF 방식 (TC) ]
NIC → 드라이버 → TC hook → eBPF 프로그램
                 (netfilter 우회, XDP보다 후단이지만 더 많은 컨텍스트 접근)
```

**eBPF 모드의 장점**

- **kube-proxy 완전 대체 가능** (Cilium 사용 시): iptables 규칙 0개로 Service 처리
- **L7 가시성**: HTTP 요청/응답, gRPC 메서드, DNS 쿼리 단위 모니터링
- **네트워크 정책 L7 지원**: “경로 `/admin`으로의 HTTP 요청 차단” 수준의 정책
- **서비스 레이턴시 감소**: 커널 네트워크 스택 단축 경로(fast path) 활용

**EKS에서의 eBPF 선택지**

|방법                           |설명                              |
|-----------------------------|--------------------------------|
|Cilium (standalone)          |kube-proxy 대체, 완전한 eBPF 네트워킹    |
|Cilium ENI 모드                |IP 할당은 VPC CNI, 데이터 플레인만 eBPF   |
|AWS Network Policy Controller|eBPF로 네트워크 정책만 처리, kube-proxy 유지|

#### kube-proxy 모드 비교 요약

|항목     |iptables   |IPVS      |nftables   |eBPF (Cilium)   |
|-------|-----------|----------|-----------|----------------|
|성능(대규모)|낮음         |중-높음      |높음         |최고              |
|룩업 복잡도 |O(n)       |O(1)      |O(1)       |O(1)            |
|LB 알고리즘|Random only|6가지       |Random/Hash|다양              |
|L7 가시성 |없음         |없음        |없음         |있음              |
|커널 요구사항|모든 버전      |`ip_vs` 모듈|3.13+      |4.19+ (5.10+ 권장)|
|EKS 기본 |✅          |설정 필요     |실험적        |별도 설치           |
|운영 복잡도 |낮음         |중간        |낮음         |높음              |

-----

### 7.3 iptables 정책 적용 순서 (심화)

#### iptables 테이블과 체인 구조

iptables는 5개 테이블(raw, mangle, nat, filter, security)과 5개 기본 체인(PREROUTING, INPUT, FORWARD, OUTPUT, POSTROUTING)으로 구성됩니다. 패킷은 **반드시 정해진 순서**로 테이블과 체인을 통과합니다.

#### 인바운드 패킷 처리 순서 (외부 → 파드)

```
패킷 수신
  │
  ▼ [1] raw 테이블 / PREROUTING 체인
  │   └─ conntrack 추적 결정 (NOTRACK 설정 가능)
  │
  ▼ [2] 커넥션 트래킹 (conntrack)
  │   └─ 기존 연결이면 상태 복원 (ESTABLISHED, RELATED)
  │
  ▼ [3] mangle 테이블 / PREROUTING 체인
  │   └─ 패킷 헤더 수정 (ToS, TTL 등) — Kubernetes에서 거의 미사용
  │
  ▼ [4] nat 테이블 / PREROUTING 체인  ★★★ 가장 중요 ★★★
  │   └─ KUBE-SERVICES 체인 진입
  │       ├─ ClusterIP DNAT → 파드 IP로 목적지 변환
  │       ├─ NodePort DNAT → 파드 IP로 목적지 변환
  │       └─ ExternalIP DNAT
  │
  ▼ [5] 라우팅 결정
  │   └─ DNAT 후 새 목적지 IP 기준으로 라우팅
  │       ├─ 로컬 파드 → INPUT 체인으로
  │       └─ 다른 노드 파드 → FORWARD 체인으로
  │
  ┌─── 로컬 전달 ──────────────────────────────────────┐
  ▼ [6a] mangle 테이블 / INPUT                        │
  ▼ [7a] filter 테이블 / INPUT                        │
  │   └─ KUBE-FIREWALL, KUBE-NODEPORTS 등             │
  ▼ [8a] 로컬 프로세스(파드)로 전달                     │
  └─────────────────────────────────────────────────┘
  │
  ┌─── 포워딩 ─────────────────────────────────────────┐
  ▼ [6b] mangle 테이블 / FORWARD                      │
  ▼ [7b] filter 테이블 / FORWARD                      │
  │   └─ KUBE-FORWARD 체인 (conntrack RELATED/ESTAB)  │
  │   └─ 네트워크 정책 적용 (Calico/Cilium 훅)          │
  ▼ [8b] mangle 테이블 / POSTROUTING                  │
  ▼ [9b] nat 테이블 / POSTROUTING                     │
  │   └─ KUBE-POSTROUTING (MASQUERADE)               │
  ▼ [10b] 목적지 파드로 전달                           │
  └─────────────────────────────────────────────────┘
```

#### 아웃바운드 패킷 처리 순서 (파드 → 외부)

```
파드에서 패킷 생성
  │
  ▼ [1] 라우팅 결정
  │   └─ 목적지 IP 기준 어느 인터페이스로 나갈지 결정
  │
  ▼ [2] raw 테이블 / OUTPUT 체인
  │
  ▼ [3] 커넥션 트래킹
  │
  ▼ [4] mangle 테이블 / OUTPUT 체인
  │
  ▼ [5] nat 테이블 / OUTPUT 체인
  │   └─ 로컬에서 발생한 패킷의 DNAT (ClusterIP → 파드 IP)
  │
  ▼ [6] filter 테이블 / OUTPUT 체인
  │   └─ KUBE-FIREWALL
  │
  ▼ [7] 다시 라우팅 결정 (DNAT 후 재평가)
  │
  ▼ [8] mangle 테이블 / POSTROUTING 체인
  │
  ▼ [9] nat 테이블 / POSTROUTING 체인  ★★★
  │   └─ KUBE-POSTROUTING: 클러스터 외부로 나가는 패킷 MASQUERADE
  │   └─ AWS-SNAT-CHAIN: VPC 외부로 나가는 파드 패킷 SNAT
  │
  ▼ [10] NIC를 통해 전송
```

#### Kubernetes가 생성하는 주요 iptables 체인

```
nat 테이블:
  KUBE-SERVICES        → 모든 ClusterIP/NodePort 서비스 진입점
  KUBE-SVC-{hash}      → 특정 서비스의 엔드포인트 로드밸런싱
  KUBE-SEP-{hash}      → 특정 파드(엔드포인트)로 DNAT
  KUBE-NODEPORTS       → NodePort 트래픽 처리
  KUBE-POSTROUTING     → 출력 패킷 MASQUERADE
  AWS-SNAT-CHAIN-{0,1} → VPC 외부 통신 SNAT (AWS VPC CNI)

filter 테이블:
  KUBE-FIREWALL        → 방화벽 규칙 (externalTrafficPolicy 등)
  KUBE-FORWARD         → 포워딩 허용/차단
  KUBE-EXTERNAL-SERVICES → 외부 서비스 접근 제어
```

#### iptables 규칙 확인 방법

```bash
# ClusterIP 10.96.80.80으로의 DNAT 규칙 추적
sudo iptables -t nat -L KUBE-SERVICES -n --line-numbers | grep 10.96.80.80

# 특정 서비스의 로드밸런싱 규칙 확인
sudo iptables -t nat -L KUBE-SVC-XXXXXX -n -v

# 전체 nat 테이블 규칙 수 확인 (규모 파악)
sudo iptables -t nat -L | wc -l

# conntrack 테이블에서 특정 IP 연결 상태 확인
sudo conntrack -L | grep 10.96.80.80
```

-----

### 7.4 ClusterIP 통신 상세 분석 (심화)

#### ClusterIP의 실체

ClusterIP는 실제로 **어디에도 존재하지 않는 가상 IP**입니다. 어떤 네트워크 인터페이스에도 바인딩되지 않으며, ping에 응답하지 않습니다. 오직 iptables/IPVS/eBPF 규칙만이 이 IP로 향하는 패킷을 실제 파드로 리다이렉트합니다.

#### 같은 노드 내 파드 → ClusterIP 통신

```
Pod A (192.168.1.11) → ClusterIP (10.96.80.80:80)

1. Pod A의 패킷이 veth를 통해 호스트 네트워크 스택 진입
2. OUTPUT 체인: KUBE-SERVICES → KUBE-SVC-XXX → KUBE-SEP-YYY
   └─ DNAT: dst 10.96.80.80:80 → 192.168.1.12:8080 (Pod B로 결정)
3. 라우팅 재결정: 192.168.1.12 → 로컬 (같은 노드의 Pod B)
4. FORWARD 체인 통과 → conntrack 상태 기록
5. Pod B의 veth로 전달

# 중요: 같은 노드 내 파드 간 통신도 iptables DNAT 처리됨
# 커널이 직접 처리하므로 호스트 네임스페이스에서 패킷이 보임
```

#### 다른 노드 파드 → ClusterIP 통신

```
Pod A (Node1, 192.168.1.11) → ClusterIP (10.96.80.80:80)
                               └─ DNAT → Pod C (Node2, 192.168.2.11:8080)

1. Node1에서 DNAT 처리: dst → 192.168.2.11:8080
2. Node1 라우팅: 192.168.2.11 → Node2의 ENI로 전달 (VPC 라우팅)
3. Node2 수신: 패킷 dst가 이미 파드 IP이므로 추가 DNAT 없음
4. Node2 라우팅 테이블: 192.168.2.11 dev eniXXX → Pod C veth로 전달

# 핵심: DNAT는 출발지 노드에서 1회만 처리
# 목적지 노드는 이미 실제 파드 IP로 변환된 패킷을 받음
```

#### 응답 패킷 처리 (conntrack의 역할)

```
Pod C (192.168.2.11) → Pod A (응답)

응답 패킷: src=192.168.2.11:8080, dst=192.168.1.11:XXXXX

Node2 POSTROUTING: conntrack에 의해 자동 역변환
  └─ 원본 DNAT의 역방향: src 10.96.80.80:80 으로 변경
     (Pod A는 자신이 ClusterIP와 통신했다고 인식)

최종 도달: src=10.96.80.80:80, dst=192.168.1.11:XXXXX
```

conntrack(connection tracking)이 DNAT 변환을 세션 단위로 추적하여 응답 패킷에 자동으로 역변환을 적용합니다. 이 덕분에 클라이언트 파드는 항상 ClusterIP와 통신하는 것처럼 인식합니다.

#### 세션 어피니티(Session Affinity)

기본 random 분산은 동일 클라이언트의 요청이 매번 다른 파드로 가도록 합니다. `sessionAffinity: ClientIP`를 설정하면 클라이언트 IP를 키로 항상 같은 파드로 연결됩니다.

구현 방식: `KUBE-SVC-XXX` 체인 앞에 `recent` 모듈 기반 IP 기억 규칙 추가. 단, L7 로드밸런서(ALB)를 거치는 경우 클라이언트 IP가 ALB IP로 보여 세션 어피니티가 무력화됩니다.

-----

### 7.5 Service 타입별 동작 원리

#### ClusterIP

클러스터 내부 전용 가상 IP. DNS는 `<service>.<namespace>.svc.cluster.local`로 해석됩니다.

#### NodePort

모든 노드의 지정 포트(30000-32767)로 들어오는 트래픽을 ClusterIP Service로 전달합니다. kube-proxy가 `iptables -t nat -A KUBE-NODEPORTS` 규칙을 생성합니다.

```
외부 트래픽: dst=<AnyNodeIP>:30080
  ↓ PREROUTING → KUBE-NODEPORTS → KUBE-SVC-XXX → DNAT
  ↓
파드 IP:Port 로 전달

externalTrafficPolicy 옵션:
  - Cluster(기본): 어느 노드든 받아서 다른 노드 파드로도 전달 가능
                  → 추가 홉 발생, 클라이언트 IP SNAT됨
  - Local: 해당 노드에 있는 파드에게만 전달
           → 추가 홉 없음, 클라이언트 IP 보존, 파드 없는 노드는 헬스체크 실패
```

#### LoadBalancer (AWS 환경)

AWS LBC 설치 전: `in-tree` 클라우드 프로바이더가 CLB(Classic LB) 생성
AWS LBC 설치 후: 어노테이션에 따라 NLB 또는 ALB 생성 (8장 참고)

-----

### 7.6 AWS 환경에서 외부 노출 방안

#### 방안 비교 전체 구조

```
외부 사용자
   │
   ├─── [방안 1] NodePort + 외부 LB (비권장)
   │    └─ 직접 EC2 노드의 NodePort 접근 → 노드 IP 변경 시 취약
   │
   ├─── [방안 2] Service type=LoadBalancer (CLB/NLB)
   │    └─ AWS LBC: NLB → 파드 직접 (L4)
   │
   ├─── [방안 3] Ingress (ALB)
   │    └─ AWS LBC: ALB → 파드 직접 (L7, HTTP/HTTPS)
   │
   ├─── [방안 4] Gateway API
   │    └─ AWS LBC: GatewayClass → Gateway → HTTPRoute → 파드
   │
   └─── [방안 5] AWS API Gateway + VPC Link
        └─ API Gateway → VPC Link → NLB → 파드 (서버리스 게이트웨이)
```

#### 방안별 상세 비교

|방안                    |L4/L7|고정 IP |경로 라우팅|SSL 종료  |WAF|비용          |
|----------------------|-----|------|------|--------|---|------------|
|NodePort              |L4   |❌     |❌     |❌       |❌  |무료          |
|NLB (Service LB)      |L4   |✅(EIP)|❌     |TLS 패스스루|❌  |NLB 비용      |
|ALB (Ingress)         |L7   |❌     |✅     |✅(ACM)  |✅  |ALB 비용      |
|Gateway API           |L7   |❌     |✅(고급) |✅       |✅  |ALB 비용      |
|API Gateway + VPC Link|L7   |✅     |✅     |✅       |✅  |API GW + NLB|

#### 실무 선택 가이드

```
서비스 노출 방안 결정 트리:

HTTP/HTTPS인가?
  ├─ YES → 호스트/경로 기반 라우팅 필요?
  │          ├─ YES → 서비스 수가 많은가? (IngressGroup 고려)
  │          │         ├─ YES → ALB Ingress + IngressGroup (ALB 공유)
  │          │         └─ NO  → ALB Ingress (서비스당 ALB)
  │          └─ NO  → 단순 HTTP 노출 → NLB + NodePort (비권장) 또는 ALB
  │
  └─ NO (TCP/UDP) → 고정 IP 필요?
                    ├─ YES → NLB (Elastic IP 연결 가능)
                    └─ NO  → NLB (기본) 또는 NodePort (내부 LB 있는 경우)
```

#### Private Cluster에서의 외부 노출

EKS 프라이빗 클러스터에서는 파드가 Private 서브넷에 있어 인터넷 직접 접근이 불가합니다.

```
인터넷 → [Public 서브넷] ALB/NLB → [Private 서브넷] 파드

ALB 설정:
  - scheme: internet-facing (Public 서브넷에 ALB 생성)
  - target-type: ip (Private 서브넷의 파드 IP로 직접 전달)
  - 보안 그룹: ALB SG → 파드 SG로 트래픽 허용
```

#### PrivateLink를 이용한 크로스 계정 서비스 노출

다른 AWS 계정에서 EKS 서비스를 안전하게 접근해야 할 때:

```
계정 A (서비스 제공)                    계정 B (서비스 소비)
  파드 → NLB → VPC Endpoint Service   →  VPC Endpoint (Interface) → 소비자 파드
                (PrivateLink)
  └─ 인터넷 거치지 않고 AWS 내부망으로 연결
  └─ 소비자 계정에서는 ENI IP로 접근
```

-----

### Ch.7 면접 질문 — Kubernetes Service와 EKS 네트워킹

-----

#### Q1. kube-proxy의 iptables 모드와 IPVS 모드의 차이를 설명하고, 대규모 클러스터에서 IPVS를 권장하는 이유는 무엇인가요?

> **핵심 답변 방향**: iptables는 O(n) 순차 규칙 매칭 + 전체 재작성, IPVS는 O(1) 해시 테이블 + 다양한 LB 알고리즘. 서비스 수 증가 시 iptables 갱신 지연 문제를 언급해야 합니다.

**▶ 1차 꼬리질문**: IPVS 모드에서 사용 가능한 로드밸런싱 알고리즘 중 실무에서 언제 rr(Round Robin) 대신 lc(Least Connection)를 사용하나요?

> 요청 처리 시간이 균일하면 rr, 요청마다 처리 시간 편차가 크면 lc. 예) DB 쿼리 처리 서비스(쿼리 복잡도 차이 큼) → lc 적합. 단순 API 서버 → rr으로 충분.

**▶▶ 2차 꼬리질문**: IPVS 모드로 전환 시 기존 iptables 규칙과 충돌하거나 누락되는 기능이 있나요?

> NodePort MASQUERADE, externalIP 처리 일부는 여전히 iptables 사용. ipset 패키지 의존성 추가 필요. 기존 클러스터 전환 시 kube-proxy DaemonSet 설정 변경 후 모든 노드 재시작 권장.

**▶▶▶ 3차 꼬리질문**: eBPF 기반 Cilium으로 kube-proxy를 완전히 대체하면 IPVS 대비 어떤 이점이 있나요? 이미 운영 중인 클러스터에서의 마이그레이션 리스크는 무엇인가요?

> 이점: iptables/IPVS 완전 제거, XDP fast path, L7 가시성. 리스크: Cilium 설치 중 네트워크 단절 가능성, 커널 버전 요구(5.10+), 운영팀 숙련도 필요, 기존 NetworkPolicy 재검증 필요.

-----

#### Q2. ClusterIP로 향하는 패킷이 실제 파드 IP로 변환되는 iptables 처리 순서를 단계별로 설명해 보세요.

> **핵심 답변 방향**: PREROUTING → KUBE-SERVICES → KUBE-SVC-{hash}(확률 기반 분산) → KUBE-SEP-{hash}(DNAT) → 라우팅 재결정의 흐름을 설명해야 합니다.

**▶ 1차 꼬리질문**: conntrack이 없다면 DNAT 응답 패킷 처리에서 어떤 문제가 발생하나요?

> 응답 패킷의 출발지가 파드 IP인데 클라이언트는 ClusterIP로 요청했으므로, conntrack 없이는 역변환이 불가능 → 클라이언트가 응답 패킷을 거부(출발지 IP 불일치).

**▶▶ 2차 꼬리질문**: 같은 노드 내 파드 끼리 ClusterIP로 통신할 때도 iptables를 거치나요? 이를 우회할 수 있는 방법이 있나요?

> 예, 동일 노드라도 OUTPUT → KUBE-SERVICES → DNAT 처리됨. 우회: kube-proxy 없이 Cilium eBPF 사용 시 커널 소켓 레벨에서 직접 파드로 연결(socket-based LB). 또는 파드 IP 직접 참조(DNS로 파드 IP 직접 획득).

**▶▶▶ 3차 꼬리질문**: 서비스 엔드포인트가 0개일 때(파드가 모두 다운) ClusterIP로 요청하면 iptables에서 어떻게 처리되나요? 이를 클라이언트에게 명확히 알리는 방법은 무엇인가요?

> 엔드포인트 없으면 `KUBE-SVC-XXX` 체인에 DNAT 규칙 없음 → 패킷이 `KUBE-REJECT` 또는 드롭 → 클라이언트에 connection refused 또는 timeout. 명확한 알림: readinessProbe 설정 → 준비 안 된 파드 엔드포인트 제외. 상위 레벨에서는 circuit breaker(Hystrix, resilience4j) 적용.

-----

#### Q3. externalTrafficPolicy: Local과 Cluster의 차이를 설명하고, 각각 어떤 상황에서 사용하나요?

**▶ 1차 꼬리질문**: `externalTrafficPolicy: Local`에서 특정 노드에 파드가 없을 때 LB 헬스체크가 실패하면 어떻게 되나요?

> NLB/ALB의 헬스체크가 실패한 노드는 Target Group에서 제외 → 해당 노드로 트래픽 미전달. 파드 분포가 불균형하면 일부 노드에만 트래픽 집중. Pod Topology Spread Constraints로 균등 분포 강제 필요.

**▶▶ 2차 꼬리질문**: `externalTrafficPolicy: Local`에서 클라이언트 IP가 보존되는 원리를 iptables 관점에서 설명해 보세요.

> `Cluster` 모드: 다른 노드 파드로 전달 시 SNAT(kube-proxy MASQUERADE) 적용 → 출발지가 노드 IP로 변경. `Local` 모드: 같은 노드의 파드로만 전달, SNAT 없음 → 원본 클라이언트 IP 보존.

**▶▶▶ 3차 꼬리질문**: NLB IP Target 모드와 externalTrafficPolicy: Local을 함께 사용하면 어떤 시너지가 있고, 어떤 주의사항이 있나요?

> 시너지: NLB → 파드 직접(원본 IP 보존) + Local 정책(SNAT 없음) = 완벽한 클라이언트 IP 보존 + 최소 홉. 주의: IP Target은 파드를 직접 타겟으로 하므로 externalTrafficPolicy가 사실상 무의미(kube-proxy NodePort 경로를 거치지 않음). ALB/NLB IP Target에서 원본 IP는 X-Forwarded-For 또는 Proxy Protocol로 전달.

-----

## 8. AWS LoadBalancer Controller와 NLB

### 8.1 AWS LBC의 아키텍처

AWS LBC는 `controller-runtime` 프레임워크 기반의 Kubernetes 컨트롤러로, 다음 리소스를 감시(Watch)하고 AWS API를 호출하여 리소스를 관리합니다.

```
Kubernetes API Server
  │ Watch: Service(LoadBalancer), Ingress, TargetGroupBinding, IngressClassParams
  ▼
AWS Load Balancer Controller (Deployment)
  │
  ├─ Service Reconciler   → NLB 생성/수정/삭제
  ├─ Ingress Reconciler   → ALB + Listener + Rule + TargetGroup 생성/수정/삭제
  ├─ TargetGroup Binder   → 파드 IP를 TargetGroup에 등록/해제
  └─ Webhook Server       → 리소스 검증 (ValidatingWebhookConfiguration)
```

**IRSA(IAM Roles for Service Accounts) 동작 원리**

```
EKS OIDC Provider (발급자: oidc.eks.ap-northeast-2.amazonaws.com/...)
  │
  ├─ 서비스 어카운트 토큰 (JWT) 발급
  │   └─ aud: sts.amazonaws.com
  │   └─ sub: system:serviceaccount:kube-system:aws-load-balancer-controller
  │
  ▼
AWS STS AssumeRoleWithWebIdentity
  │ 조건: sub == "system:serviceaccount:kube-system:aws-load-balancer-controller"
  ▼
임시 자격증명 (Access Key + Secret Key + Session Token, TTL: 1시간)
  │
  ▼
EC2/ELB/IAM API 호출 (NLB/ALB 관리)
```

이 방식은 EC2 인스턴스 프로파일에 광범위한 권한을 부여하는 것보다 최소 권한 원칙에 부합합니다. 컨트롤러 파드만 LB 관련 권한을 가집니다.

### 8.2 NLB 심화 개념

#### Connection Tracking과 Client IP 보존

NLB는 L4에서 동작하므로 TCP 연결을 종료하지 않고 그대로 전달합니다.

**IP Target 모드에서 클라이언트 IP 보존 메커니즘**

```
Client (1.2.3.4:54321)
  │ TCP SYN
  ▼
NLB (수신, 전달만 함)
  │ 원본 IP 보존하여 파드로 전달 (IP Target 모드)
  │ src=1.2.3.4:54321, dst=192.168.1.11:8080
  ▼
Pod
  └─ request.RemoteAddr == "1.2.3.4:54321" ✅
```

**Instance Target 모드에서의 SNAT**

```
Client (1.2.3.4:54321)
  │
  ▼ NLB → NodePort (NodeIP:30080)
  │ kube-proxy DNAT → 파드 IP
  │ SNAT: src → NodeIP (externalTrafficPolicy: Cluster 시)
  ▼
Pod
  └─ request.RemoteAddr == "NodeIP:XXXXX" ❌ (원본 IP 손실)
  
해결: externalTrafficPolicy: Local 설정
  → SNAT 미적용, 해당 노드의 파드에게만 전달, 원본 IP 보존
  → 단, 해당 노드에 파드 없으면 LB 헬스체크 실패로 트래픽 미전달
```

#### NLB의 연결 유지와 타임아웃

NLB는 idle 연결을 350초(기본값) 후 종료합니다. WebSocket이나 gRPC 장기 연결 서비스에서는 이 타임아웃을 늘리거나, 애플리케이션 레벨에서 keep-alive를 설정해야 합니다.

```
어노테이션으로 타임아웃 제어:
service.beta.kubernetes.io/aws-load-balancer-connection-idle-timeout: "3600"
```

#### 멀티 AZ와 교차 가용영역 로드밸런싱

NLB는 기본적으로 AZ 내에서만 트래픽을 분산합니다. 특정 AZ의 파드가 모두 내려가면 해당 AZ로 전달된 트래픽이 실패합니다. `cross-zone-load-balancing-enabled: "true"` 설정으로 모든 AZ의 파드에 균등 분산합니다.

단, 크로스 AZ 트래픽은 데이터 전송 비용이 발생하므로 트래픽 패턴을 고려해야 합니다.

-----

### Ch.8 면접 질문 — AWS LoadBalancer Controller와 NLB

-----

#### Q1. AWS LBC를 사용하여 NLB를 생성할 때 IP Target과 Instance Target 모드를 어떤 기준으로 선택하나요?

> **핵심 답변 방향**: 성능(직접 전달 vs 추가 홉), 클라이언트 IP 보존 여부, CNI 요구사항(IP Mode는 VPC CNI 필요), 보안 그룹 적용 범위를 종합 판단해야 합니다.

**▶ 1차 꼬리질문**: IP Target 모드에서 파드가 롤링 업데이트 중일 때 트래픽 드롭이 발생할 수 있나요? 어떻게 방지하나요?

> 파드 종료 시 NLB Target Group에서 등록 해제까지 시간 차이로 트래픽이 종료 중인 파드로 전달될 수 있음. 방지: `preStop` 훅에 sleep 추가(deregistration_delay 동안 대기), `terminationGracePeriodSeconds`를 충분히 설정, minReadySeconds 활용.

**▶▶ 2차 꼬리질문**: NLB의 Target Group 헬스체크 실패 임계값을 너무 낮게 설정하면 어떤 문제가 발생하나요?

> 일시적 부하로 인한 헬스체크 응답 지연 → 정상 파드 제외 → 트래픽 집중 → 연쇄 헬스체크 실패(Thundering Herd). 반대로 너무 높으면 비정상 파드가 오래 유지됨. 일반적으로 `healthy_threshold=2, unhealthy_threshold=2, interval=10s` 균형 권장.

**▶▶▶ 3차 꼬리질문**: NLB + IP Target + 파드 자동 스케일링(HPA) 환경에서 스케일 아웃/인 시 Target Group 동기화는 어떻게 이루어지나요?

> AWS LBC가 Endpoints/EndpointSlice 변경을 Watch → Target Group의 파드 IP 등록/해제 API 호출. 스케일 아웃: 새 파드 Ready 상태 → Endpoints 추가 → Target Group 등록 → 헬스체크 통과 후 트래픽 수신. 지연 시간 = 파드 시작 시간 + 헬스체크 통과 시간(보통 20~30초). HPA가 이 지연을 고려하여 선제적 스케일 아웃 필요.

-----

#### Q2. IRSA(IAM Roles for Service Accounts)가 필요한 이유를 설명하고, EC2 인스턴스 프로파일과의 차이를 설명해 보세요.

**▶ 1차 꼬리질문**: IRSA 없이 EC2 인스턴스 프로파일로 LB 권한을 부여하면 어떤 보안 위험이 있나요?

> 노드의 모든 파드가 EC2 인스턴스 자격증명을 공유 → 악성 파드나 탈취된 파드가 LB 권한 남용 가능. IRSA는 서비스 어카운트 단위로 권한 격리.

**▶▶ 2차 꼬리질문**: OIDC Provider와 STS의 관계를 설명하고, 토큰 교환 과정에서 보안이 어떻게 보장되나요?

> EKS가 서명한 JWT(서비스 어카운트 토큰) → STS `AssumeRoleWithWebIdentity` → IAM Role Trust Policy에서 OIDC Issuer + Sub 조건 검증 → 임시 자격증명 발급. JWT 위조 불가(EKS 서명), 역할 가정 조건 명시적 지정으로 보안 보장.

**▶▶▶ 3차 꼬리질문**: IRSA 토큰의 TTL은 기본 86400초(24시간)인데, 이를 단축하면 어떤 영향이 있고, EKS Pod Identity와 IRSA의 차이는 무엇인가요?

> TTL 단축 → 자격증명 만료 더 자주 발생 → 파드가 주기적으로 갱신(자동 처리). 매우 짧게(1시간) 설정하면 갱신 빈도 증가로 STS API 호출량 증가. EKS Pod Identity(EKS 1.29+): IRSA보다 단순한 설정, OIDC Provider 관리 불필요, 서비스 어카운트 ↔ IAM Role 직접 연결.

-----

## 9. Ingress와 ALB (L7)

### 9.1 ALB Ingress의 리소스 매핑

AWS LBC가 Ingress 리소스를 받아 생성하는 AWS 리소스 계층:

```
Kubernetes Ingress
  │
  ▼ AWS LBC 처리
  │
  ├─ ALB (Application Load Balancer)
  │   └─ Listener (80/443)
  │       └─ Listener Rules (조건 + 액션)
  │           ├─ Rule 1: host=api.example.com, path=/v1/* → Target Group A
  │           ├─ Rule 2: host=api.example.com, path=/v2/* → Target Group B
  │           └─ Default Rule: → Target Group C
  │
  └─ Target Groups (서비스별)
      ├─ Target Group A: [192.168.1.11:8080, 192.168.2.11:8080] (IP mode)
      ├─ Target Group B: [192.168.1.12:8080]
      └─ Target Group C: [...]
```

### 9.2 ALB의 SSL/TLS 처리 상세

#### ACM(AWS Certificate Manager) 통합

```
TLS 처리 흐름:
  클라이언트 → HTTPS(443) → ALB [TLS 종료] → HTTP(80) → 파드

ALB가 TLS 세션을 종료하므로:
  - 파드는 HTTP만 처리하면 됨
  - ACM 인증서 갱신이 자동화됨 (ALB가 최신 인증서 자동 사용)
  - 파드에 인증서 배포 불필요
```

#### 인증서 자동 발견

```yaml
# 어노테이션 없이도 도메인 매칭으로 자동 인증서 선택
alb.ingress.kubernetes.io/certificate-arn: |
  arn:aws:acm:ap-northeast-2:123456789:certificate/abc-123,
  arn:aws:acm:ap-northeast-2:123456789:certificate/def-456
# 여러 도메인의 인증서를 동시에 지정 가능 (SNI)
```

### 9.3 IngressGroup의 동작 원리

같은 `group.name`을 가진 Ingress들은 **하나의 ALB를 공유**합니다. AWS LBC는 그룹 내 모든 Ingress의 규칙을 하나의 ALB 리스너 규칙으로 병합합니다.

```
Ingress A (group: shared, order: 10)
  규칙: host=api.example.com → Service A

Ingress B (group: shared, order: 20)
  규칙: host=web.example.com → Service B

                ↓ AWS LBC 처리

ALB (1개)
  Listener :443
    Rule 1 (order:10): host=api.example.com → TG-A
    Rule 2 (order:20): host=web.example.com → TG-B
    Default: 404
```

**비용 효과**: 서비스 10개 × ALB 단가 $0.023/hr = $165/월 → IngressGroup으로 1개 ALB 공유 시 $16.5/월

**IngressGroup 설계 시 주의사항**:

- 다른 네임스페이스의 Ingress가 같은 그룹에 속하면 보안 경계 주의
- 규칙 충돌(같은 host+path) 시 order가 낮은 규칙이 우선
- 그룹 내 한 Ingress 삭제 시 해당 규칙만 ALB에서 제거

### 9.4 ALB 인증(Authentication) 통합

ALB는 Cognito 또는 OIDC Provider와 통합하여 **ALB 레벨에서 인증**을 처리할 수 있습니다.

```
클라이언트 → ALB
  └─ 인증 안 된 요청 → Cognito/IdP 로그인 페이지 리다이렉트
  └─ 인증된 요청 → 파드로 전달 + x-amzn-oidc-identity 헤더 추가

파드: JWT 검증 없이도 ALB가 인증 보장
      request.headers['x-amzn-oidc-data'] 에서 사용자 정보 추출
```

-----

### Ch.9 면접 질문 — Ingress와 ALB (L7)

-----

#### Q1. 여러 팀이 각자의 서비스를 ALB로 외부에 노출하고 싶어합니다. ALB 비용을 최적화하면서도 팀 간 독립성을 유지하는 방법을 설명해 보세요.

> **핵심 답변 방향**: IngressGroup으로 ALB 공유하되, group.name을 팀/환경 단위로 분리하고 group.order로 규칙 우선순위 제어. 팀별 네임스페이스 분리로 보안 경계 유지.

**▶ 1차 꼬리질문**: IngressGroup 사용 시 한 팀의 설정 오류가 다른 팀 서비스에 영향을 줄 수 있는 시나리오를 설명하고, 이를 방지하는 방법은 무엇인가요?

> 시나리오: 같은 host+path 규칙 충돌(낮은 order 규칙이 다른 팀 트래픽 가로챔), 잘못된 인증서 ARN으로 전체 그룹 HTTPS 실패. 방지: OPA/Kyverno로 Ingress 어노테이션 검증, host 도메인 네임스페이스별 할당 정책, CI 단계에서 충돌 감지.

**▶▶ 2차 꼬리질문**: ALB의 Listener Rule은 최대 100개 제한이 있습니다. 서비스가 100개 이상이라면 어떻게 처리하나요?

> 여러 IngressGroup으로 분산(여러 ALB). 또는 wildcard host 규칙으로 묶고 애플리케이션 레벨 라우팅. 또는 API Gateway + VPC Link + NLB 구조로 전환.

**▶▶▶ 3차 꼬리질문**: ALB에서 WAF를 활성화했을 때 false positive(정상 요청 차단)가 발생하면 어떻게 분석하고 튜닝하나요?

> AWS WAF 로그(S3/CloudWatch)에서 차단 요청의 RuleGroupId + RuleId 확인 → 해당 규칙의 패턴 분석 → Count 모드로 전환하여 영향 범위 파악 → 예외 규칙(Exception) 추가 또는 Custom Rule로 우선순위 조정.

-----

## 10. ExternalDNS

### 10.1 동작 원리 상세

ExternalDNS는 Kubernetes Informer를 통해 Service/Ingress 변경을 실시간으로 감지합니다.

```
변경 감지 → 원하는 DNS 레코드 집합 계산 → 현재 Route 53 상태 조회
  → 차이(diff) 계산 → Route 53 ChangeResourceRecordSets API 호출

레코드 유형 결정:
  - Service(LoadBalancer) + NLB → ALIAS 레코드 (NLB DNS명)
  - Service(LoadBalancer) + NLB with EIP → A 레코드 (Elastic IP)
  - Ingress + ALB → ALIAS 레코드 (ALB DNS명)
```

### 10.2 TXT 레코드 소유권 관리

ExternalDNS는 자신이 생성한 DNS 레코드를 식별하기 위해 TXT 레코드를 함께 생성합니다.

```
생성되는 레코드 쌍:
  api.example.com   A/ALIAS → alb-xxx.elb.amazonaws.com
  externaldns-api.example.com TXT → "heritage=external-dns,owner=cluster-prod,resource=ingress/default/my-ingress"

# owner 필드로 클러스터 식별 → 다른 클러스터의 레코드를 삭제하지 않음
```

### 10.3 멀티 클러스터 환경에서의 ExternalDNS

같은 Hosted Zone을 여러 클러스터가 공유할 때:

```
cluster-a (txtOwnerId: cluster-a)   api.example.com → NLB-A
cluster-b (txtOwnerId: cluster-b)   api.example.com → NLB-B

문제: 두 클러스터가 같은 도메인 소유권 주장 → 서로의 레코드를 덮어씀

해결 방법:
  1. 서브도메인 분리: cluster-a.api.example.com, cluster-b.api.example.com
  2. domainFilters로 클러스터별 관리 도메인 분리
  3. policy: upsert-only 설정 (삭제 방지)
```

-----

### Ch.10 면접 질문 — ExternalDNS

-----

#### Q1. ExternalDNS를 사용하지 않으면 서비스 DNS 관리가 어떻게 되고, ExternalDNS 도입으로 어떤 문제가 해결되나요?

**▶ 1차 꼬리질문**: ExternalDNS의 `sync` 정책과 `upsert-only` 정책 중 프로덕션에서 어떤 것을 권장하고 그 이유는 무엇인가요?

> 프로덕션에서 초기에는 `upsert-only` 권장. 실수로 Service 삭제 시 DNS 레코드도 함께 사라지는 위험 방지. 신뢰도 높아지면 `sync`로 전환하여 완전 자동화.

**▶▶ 2차 꼬리질문**: 멀티 클러스터 환경에서 두 클러스터의 ExternalDNS가 같은 Route 53 호스팅 존을 관리하면 어떤 문제가 발생하나요?

> 동일 도메인에 대해 두 클러스터가 서로 다른 LB를 등록 → 마지막 쓰기가 이김(last-write-wins) → 레코드 덮어쓰기. txtOwnerId를 클러스터별로 다르게 설정하면 각자 관리 도메인을 구분할 수 있지만, 같은 도메인이면 여전히 충돌 가능.

**▶▶▶ 3차 꼬리질문**: Active-Active 멀티 클러스터에서 같은 도메인이 두 클러스터의 LB를 모두 가리키도록 DNS를 설정하려면 어떤 구조가 필요한가요?

> ExternalDNS 단독으로는 불가(last-write-wins). Route 53 Health Check + Failover 레코드 정책으로 Active-Active 구성. 또는 AWS Global Accelerator로 여러 LB를 단일 진입점으로 묶음. 또는 두 클러스터가 각각 다른 도메인을 관리하고 상위 레벨 LB(Route 53 weighted)로 분산.

-----

## 11. Gateway API

### 11.1 Ingress의 구조적 한계

Ingress가 등장한 2015년 이후 Kubernetes 네트워킹의 요구사항이 크게 발전했지만, Ingress API는 제한적인 스펙 탓에 벤더별 어노테이션으로 기능을 확장해야 했습니다.

|한계           |설명                               |
|-------------|---------------------------------|
|HTTP/HTTPS 전용|TCP/UDP/gRPC/WebSocket은 표준 스펙 없음 |
|단일 역할        |인프라 설정과 라우팅 규칙이 한 오브젝트에 혼재       |
|제한적 매칭       |host + path만 지원, 헤더/메서드/쿼리파라미터 불가|
|가중치 라우팅 불가   |카나리/블루그린 배포에 어노테이션 의존            |
|벤더 종속        |ALB 고급 기능은 모두 어노테이션으로만 설정        |

### 11.2 Gateway API 리소스 모델 상세

#### GatewayClass

클러스터 전체에서 사용 가능한 LB 구현체를 정의합니다. 인프라팀이 “이 클러스터에서는 ALB를 사용한다”는 표준을 선언합니다.

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
  name: aws-alb
spec:
  controllerName: "eks.amazonaws.com/alb"
  parametersRef:  # 구현체별 고급 설정 참조
    group: eks.amazonaws.com
    kind: IngressClassParams
    name: alb-params
```

#### Gateway

특정 네임스페이스에 LB 인스턴스를 생성합니다. 어느 네임스페이스의 HTTPRoute를 허용할지 정책으로 제어합니다.

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: prod-gateway
  namespace: infra
spec:
  gatewayClassName: aws-alb
  listeners:
    - name: https
      protocol: HTTPS
      port: 443
      tls:
        mode: Terminate
        certificateRefs:
          - kind: Secret
            name: prod-tls
      allowedRoutes:
        namespaces:
          from: Selector          # 특정 네임스페이스만 허용
          selector:
            matchLabels:
              gateway-access: allowed
```

#### HTTPRoute 고급 기능

```yaml
# 카나리 배포: 가중치 기반 트래픽 분산
backendRefs:
  - name: app-stable
    port: 80
    weight: 95
  - name: app-canary
    port: 80
    weight: 5      # 5% 트래픽만 신버전으로

# 헤더 기반 A/B 테스트
matches:
  - headers:
      - name: X-Test-Group
        value: beta
        type: Exact

# 요청 미러링 (트래픽 복제 - 새 버전 섀도 테스트)
filters:
  - type: RequestMirror
    requestMirror:
      backendRef:
        name: app-v2-shadow
        port: 80

# URL 재작성
filters:
  - type: URLRewrite
    urlRewrite:
      path:
        type: ReplacePrefixMatch
        replacePrefixMatch: /api/v2
```

### 11.3 Gateway API와 Ingress 비교

|항목     |Ingress   |Gateway API                     |
|-------|----------|--------------------------------|
|역할 분리  |❌ 단일 오브젝트 |✅ GatewayClass/Gateway/Route 분리 |
|프로토콜   |HTTP/HTTPS|HTTP, HTTPS, TCP, UDP, TLS, gRPC|
|가중치 라우팅|어노테이션     |표준 스펙 (`weight` 필드)             |
|헤더 매칭  |어노테이션     |표준 스펙                           |
|요청 변환  |어노테이션     |표준 필터 API                       |
|트래픽 미러링|❌         |✅                               |
|표준화 수준 |성숙 (v1)   |성숙 (v1, 일부 v1beta1)             |
|벤더 확장  |어노테이션     |parametersRef, policy 어태치먼트     |

-----

### Ch.11 면접 질문 — Gateway API

-----

#### Q1. Ingress 대신 Gateway API를 도입해야 하는 시점과 그 판단 기준을 설명해 보세요.

> **핵심 답변 방향**: 팀 규모 확대로 역할 분리 필요, 가중치 기반 카나리 배포 표준화 필요, TCP/UDP 라우팅 필요, L7 트래픽 정책 표준화 필요 시 도입 고려.

**▶ 1차 꼬리질문**: Gateway API의 GatewayClass, Gateway, HTTPRoute의 역할 분리가 실제 조직 구조와 어떻게 매핑되나요?

> GatewayClass(플랫폼 인프라팀: 어떤 LB 기술 사용), Gateway(SRE/플랫폼팀: 어떤 포트/프로토콜/인증서 사용), HTTPRoute(개발팀: 내 서비스 라우팅 규칙). 관심사 분리로 각 팀이 자율적으로 관리 가능.

**▶▶ 2차 꼬리질문**: HTTPRoute에서 가중치 기반 카나리 배포를 진행하다가 신버전에서 에러율이 급증했습니다. 빠르게 롤백하는 방법은 무엇인가요?

> HTTPRoute의 `weight` 필드를 0으로 변경 → 즉시 기존 버전에 100% 트래픽. Argo Rollouts나 Flagger 같은 Progressive Delivery 도구와 Gateway API 연동 시 자동 롤백 가능(에러율 임계값 기반).

**▶▶▶ 3차 꼬리질문**: Gateway API의 ReferenceGrant는 무엇이며, 왜 필요한가요? 없다면 어떤 보안 위험이 있나요?

> ReferenceGrant: 다른 네임스페이스의 리소스를 참조할 때 명시적 허가를 선언하는 리소스. 예) infra 네임스페이스의 Gateway가 app 네임스페이스의 Service를 백엔드로 사용 시, app 네임스페이스에서 ReferenceGrant로 허가 필요. 없으면: 악의적 HTTPRoute가 다른 네임스페이스의 Service로 트래픽 유도(네임스페이스 탈출) 가능.

-----

## 12. CoreDNS

### 12.1 Kubernetes DNS 아키텍처 전체

```
파드 내 DNS 해석 흐름:

애플리케이션 → glibc/musl DNS 리졸버 → /etc/resolv.conf 참조
  │
  ├─ nameserver: 10.96.0.10 (CoreDNS ClusterIP)
  ├─ search: default.svc.cluster.local svc.cluster.local cluster.local
  └─ options ndots:5

  ↓ DNS 쿼리 (UDP 53)
  
CoreDNS (ClusterIP: 10.96.0.10)
  ├─ 클러스터 내부 도메인 처리 (kubernetes 플러그인)
  │   └─ kube-apiserver에서 Service/Endpoint 정보 실시간 조회
  │
  └─ 외부 도메인 포워딩 (forward 플러그인)
      └─ VPC DNS (169.254.169.253 = Route 53 Resolver 엔드포인트)
          └─ Route 53 Resolver → 외부 DNS
```

### 12.2 ndots와 검색 도메인의 DNS 오버헤드 분석

`ndots:5`는 점의 개수가 5개 미만이면 먼저 검색 도메인을 붙여 시도하는 설정입니다. 이 과정에서 외부 도메인 하나를 해석하는 데 여러 번의 DNS 쿼리가 발생합니다.

```
"google.com" 조회 시 발생하는 쿼리 순서 (ndots:5):

1. google.com.default.svc.cluster.local → NXDOMAIN
2. google.com.svc.cluster.local         → NXDOMAIN
3. google.com.cluster.local             → NXDOMAIN
4. google.com.                          → 성공 (외부 DNS에서 해석)

총 4번의 DNS 쿼리 → CoreDNS에 불필요한 부하

"api.example.com" 조회 (점이 2개, 5 미만):
  동일하게 3번 NXDOMAIN 후 성공 → 총 4 쿼리

해결 방법:
  1. FQDN 사용: "google.com." (끝에 점 추가) → 검색 도메인 건너뜀, 즉시 해석
  2. ndots 낮추기: 파드 spec.dnsConfig.options[ndots=2] → 점이 2개 이상이면 FQDN으로 인식
  3. NodeLocal DNSCache (하단 설명)
```

**ndots 낮추기의 트레이드오프**

ndots을 낮추면 짧은 서비스명(`my-service`)으로 클러스터 내부 서비스를 찾을 때 FQDN이 아닌 것으로 인식해 외부 DNS로 나가려 시도합니다. 짧은 서비스명과 외부 도메인을 모두 자주 사용한다면 신중히 결정해야 합니다.

### 12.3 CoreDNS 플러그인 체계 심화

CoreDNS의 모든 기능은 플러그인으로 구현됩니다. `Corefile`의 순서가 플러그인 실행 순서입니다.

```
.:53 {
    errors          # 오류 로깅 (가장 먼저)
    health {
       lameduck 5s  # 종료 전 5초 유예 (롤링 업데이트 시 연결 안전 종료)
    }
    ready           # /ready 엔드포인트 (readinessProbe 용)
    
    kubernetes cluster.local in-addr.arpa ip6.arpa {
       pods insecure           # 파드 DNS 레코드 활성화
       fallthrough in-addr.arpa ip6.arpa  # 역방향 DNS 폴백 허용
       ttl 30                  # 응답 TTL
    }
    
    prometheus :9153  # 메트릭 노출
    
    forward . /etc/resolv.conf {  # 업스트림 DNS 포워딩
       max_concurrent 1000        # 최대 동시 쿼리 수
       prefer_udp                 # UDP 우선 (TCP 폴백)
    }
    
    cache 30   # 긍정/부정 응답 캐싱 (30초)
    loop       # DNS 루프 감지
    reload     # Corefile 변경 시 자동 재로드
    loadbalance  # 다중 A 레코드 응답 랜덤화 (DNS 기반 LB)
}

# 사내 도메인 전용 포워딩
corp.internal:53 {
    errors
    cache 30
    forward . 10.0.0.2 10.0.0.3 {
       policy round_robin
       health_check 5s
    }
}
```

### 12.4 CoreDNS 응답 유형과 캐싱

|응답 유형          |설명           |cache 플러그인 처리     |
|---------------|-------------|------------------|
|긍정 응답          |도메인 존재, IP 반환|TTL 기준 캐싱 (기본 30s)|
|부정 응답(NXDOMAIN)|도메인 없음       |부정 캐싱 가능 (기본 30s) |
|SERVFAIL       |서버 오류        |캐싱 안 함            |

CoreDNS의 `cache` 플러그인은 부정 응답(NXDOMAIN)도 캐싱합니다. 서비스를 방금 생성했는데 DNS가 안 되는 경우, 이전의 NXDOMAIN 캐시가 원인일 수 있습니다.

### 12.5 NodeLocal DNSCache 아키텍처

대규모 클러스터에서 수천 개의 파드가 모든 DNS 쿼리를 CoreDNS(2~3개 파드)로 보내면 병목이 발생합니다. **NodeLocal DNSCache**는 각 노드에 DNS 캐시 에이전트(link-local IP `169.254.20.10`)를 배포하여 이 문제를 해결합니다.

```
[ 기존 구조 ]
파드 → UDP 53 → kube-proxy → CoreDNS 파드
                (네트워크 홉 + conntrack 부하)

[ NodeLocal DNSCache 구조 ]
파드 → 169.254.20.10:53 (로컬 노드 캐시)
  ├─ 캐시 히트: 즉시 응답 (서브 밀리초)
  └─ 캐시 미스: CoreDNS로 TCP 쿼리 (기존 UDP 대신 TCP 사용)
                └─ TCP: conntrack 상태 없음 → conntrack 테이블 부하 감소

이점:
  - DNS 레이턴시 수 ms → 수십 μs
  - CoreDNS 파드 부하 ~70% 감소
  - conntrack 테이블 UDP 항목 제거 → 커널 메모리 절약
  - 파드 재시작 시 CoreDNS 연결 재수립 비용 감소
```

**NodeLocal DNSCache 구현 방식**

`iptables` NOTRACK 규칙으로 `169.254.20.10:53`으로의 패킷이 conntrack 테이블을 거치지 않도록 설정합니다. DaemonSet으로 배포되는 `node-local-dns` 파드가 link-local IP에 바인딩됩니다.

### 12.6 CoreDNS 성능 튜닝과 장애 대응

**HPA vs 수동 스케일링**

CoreDNS는 Deployment로 배포되므로 HPA로 자동 스케일링 가능합니다. 단, DNS는 상태가 없는 서비스이므로 수평 확장이 용이합니다. 일반적으로 노드 수의 10%를 CoreDNS 파드 수로 설정하는 것이 권장됩니다.

**CoreDNS 장애 패턴**

|증상          |원인                            |해결                                |
|------------|------------------------------|----------------------------------|
|외부 도메인 해석 느림|upstream DNS 응답 지연            |forward health_check + 다중 upstream|
|NXDOMAIN 지속 |캐시에 부정 응답 저장                  |cache TTL 단축 또는 캐시 flush          |
|CoreDNS OOM |대규모 클러스터의 쿼리 폭증               |NodeLocal DNSCache 적용, 리소스 증가     |
|서비스 DNS 불가  |kubernetes 플러그인 - API 서버 연결 끊김|CoreDNS → API 서버 네트워크 확인          |

-----

## 아키텍처 통합 관점: EKS 네트워킹 전체 흐름

클라이언트 요청이 인터넷에서 클러스터 내부 파드까지 도달하는 전체 경로와 각 레이어의 책임을 종합합니다.

```
[인터넷 클라이언트] api.example.com:443
  │
  │ DNS 쿼리
  ▼
[Route 53] ← ExternalDNS가 자동 등록한 ALIAS 레코드
  │ ALIAS → alb-xxxx.ap-northeast-2.elb.amazonaws.com
  ▼
[ALB (L7)] ← AWS LBC가 Ingress 리소스 기반으로 관리
  ├─ TLS 종료 (ACM 인증서)
  ├─ WAF 검사
  ├─ 리스너 규칙: host+path 매칭
  ├─ Target Group (IP Mode): 파드 IP 직접
  ▼
[VPC 라우팅] ← 파드 IP가 VPC Native IP이므로 직접 라우팅
  │ 파드 IP(192.168.x.x) → 해당 노드 ENI로 전달
  ▼
[EC2 Node - 호스트 네트워크 스택]
  ├─ 호스트 라우팅 테이블: 192.168.1.11 dev eniXXX
  ├─ [iptables PREROUTING/FORWARD - 네트워크 정책 적용]
  ▼
[veth pair] ← AWS VPC CNI가 파드 생성 시 설정
  ▼
[Pod eth0: 192.168.1.11] ← ipamd가 ENI Secondary IP 또는 Prefix에서 할당
  │
  ▼
[Application (Port 8080)]
  │ 내부 서비스 호출 필요 시
  ▼
[CoreDNS] ← 서비스명 → ClusterIP 해석
  ▼
[ClusterIP → iptables DNAT] ← kube-proxy가 Service/Endpoints 기반 규칙 관리
  ▼
[Backend Pod] ← 실제 서비스 파드
```

**각 컴포넌트의 책임 요약**

|컴포넌트               |책임                          |
|-------------------|----------------------------|
|AWS VPC CNI (ipamd)|파드 IP 할당, ENI Warm Pool 관리  |
|kube-proxy         |ClusterIP → 파드 IP DNAT 규칙 관리|
|CoreDNS            |클러스터 내부 DNS 해석, 외부 DNS 포워딩  |
|AWS LBC            |ALB/NLB 프로비저닝 및 파드 등록 관리    |
|ExternalDNS        |Route 53 레코드 자동 동기화         |
|Gateway API        |고급 L7 트래픽 제어 (가중치, 헤더 매칭 등) |

-----

## 참고 자료

- [AWS VPC CNI GitHub & 공식 문서](https://github.com/aws/amazon-vpc-cni-k8s)
- [AWS EKS Best Practices Guide - Networking](https://aws.github.io/aws-eks-best-practices/networking/index/)
- [AWS Load Balancer Controller 공식 문서](https://kubernetes-sigs.github.io/aws-load-balancer-controller/)
- [Kubernetes Network Policies 공식 문서](https://kubernetes.io/docs/concepts/services-networking/network-policies/)
- [Gateway API 공식 문서](https://gateway-api.sigs.k8s.io/)
- [CoreDNS 플러그인 공식 문서](https://coredns.io/plugins/)
- [eBPF 공식 문서](https://ebpf.io/)
- [Cilium 공식 문서](https://docs.cilium.io/)
- [NodeLocal DNSCache 공식 문서](https://kubernetes.io/docs/tasks/administer-cluster/nodelocaldns/)
- [Linux Advanced Routing & Traffic Control (iproute2)](https://lartc.org/)

### Ch.12 면접 질문 — CoreDNS

-----

#### Q1. EKS 클러스터에서 DNS 해석이 느리다는 장애가 발생했습니다. 어떻게 트러블슈팅하겠습니까?

> **핵심 답변 방향**: CoreDNS 파드 자원 현황 → ndots:5로 인한 불필요한 쿼리 → 업스트림 DNS 응답 지연 → CoreDNS 파드 수 부족 순으로 단계적으로 확인해야 합니다.

**▶ 1차 꼬리질문**: `ndots:5` 설정으로 인해 DNS 쿼리가 몇 배 증가하는지 계산해 보고, 이를 해결하는 방법을 제시해 보세요.

> 외부 도메인 1개 조회 시 최대 5번 쿼리(검색 도메인 3개 + 원본 1번 + 결과). 해결: 파드 spec에 `ndots: 2` 설정, 또는 외부 도메인은 FQDN(끝에 `.`) 사용, 또는 NodeLocal DNSCache로 NXDOMAIN 응답 로컬 캐싱.

**▶▶ 2차 꼬리질문**: NodeLocal DNSCache를 도입했는데도 DNS 지연이 계속됩니다. 추가적으로 확인해야 할 사항은 무엇인가요?

> 캐시 미스율 확인(Prometheus `coredns_cache_hits_total` vs `coredns_cache_misses_total`). 업스트림 CoreDNS → VPC DNS(169.254.169.253) 구간 지연 확인. CoreDNS의 forward 플러그인 `max_concurrent` 설정 확인. DNS 쿼리 패턴 분석(외부 도메인 vs 내부 도메인 비율).

**▶▶▶ 3차 꼬리질문**: 특정 파드에서만 DNS가 안 되고 다른 파드는 정상인 상황입니다. 원인과 확인 방법을 설명해 보세요.

> 원인 후보: 해당 파드의 dnsPolicy 설정 오류(`None`으로 설정되어 있고 dnsConfig 누락), 네트워크 정책이 파드 → CoreDNS(53포트) 차단, 파드가 CoreDNS와 다른 네임스페이스 CIDR에 있어 라우팅 문제. 확인: `kubectl exec -it <파드> -- cat /etc/resolv.conf` → nameserver 확인. `kubectl exec -it <파드> -- nslookup kubernetes.default` → 응답 확인.

-----

#### Q2. CoreDNS에서 온프레미스 내부 도메인을 포워딩하도록 Corefile을 수정할 때 주의사항은 무엇인가요?

**▶ 1차 꼬리질문**: Corefile을 잘못 수정했을 때 클러스터 전체 DNS가 다운될 수 있나요? 어떻게 안전하게 변경하나요?

> 예, Corefile 문법 오류 시 CoreDNS 파드 재시작 루프 → 클러스터 DNS 전체 불가. 안전한 변경: 1) ConfigMap 수정 전 문법 검증(`coredns -conf <파일> -verify`). 2) 스테이징 클러스터에서 먼저 테스트. 3) `reload` 플러그인 활성화 시 파드 재시작 없이 자동 반영.

**▶▶ 2차 꼬리질문**: 포워딩할 온프레미스 DNS 서버가 이중화되어 있지 않을 때 단일 장애점이 됩니다. CoreDNS 설정에서 이를 보완하는 방법은 무엇인가요?

> `forward . 10.0.0.2 10.0.0.3 { policy round_robin; health_check 5s }` 설정으로 다중 업스트림 + 헬스체크. 한 서버 다운 시 자동 failover. 추가로 EKS VPC에서 사용하는 Route 53 Resolver Endpoint에 온프레미스 포워딩 규칙 추가하는 방법도 대안.

**▶▶▶ 3차 꼬리질문**: CoreDNS와 Route 53 Resolver Outbound Endpoint를 함께 사용하는 구조의 장단점을 비교하고, 어떤 경우에 각각을 선택하겠습니까?

> CoreDNS 포워딩: 파드 단에서 바로 처리, 설정 간단, CoreDNS가 SPOF. Route 53 Resolver Outbound: VPC DNS → AWS 관리형 포워딩, 고가용성, 비용 발생(쿼리당 과금). 대규모 프로덕션 환경에서 고가용성이 중요하면 Resolver Outbound. 소규모/비용 민감 환경에서는 CoreDNS 포워딩으로 충분.
