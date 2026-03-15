# 🚀 AEWS 4기 - 1주차: Amazon EKS 소개 및 배포

> **작성일**: 2026년 3월 15일  
> **스터디**: AEWS(AWS EKS Workshop Study) 4기  
> **주제**: Amazon EKS 소개 및 배포, Cluster Endpoint Access, Fully Private Cluster

---

## 목차

1. [Amazon EKS 소개](#1-amazon-eks-소개)
2. [Amazon EKS 배포 및 확인](#2-amazon-eks-배포-및-확인)
3. [EKS Cluster Endpoint Access](#3-eks-cluster-endpoint-access)
4. [EKS Fully Private Cluster](#4-eks-fully-private-cluster)
5. [도전과제 1 - EKS vs Vanilla k8s 비교](#5-도전과제-1---eks-vs-vanilla-k8s-비교)
6. [도전과제 2 - Fully Private Cluster에서 Nginx 배포](#6-도전과제-2---fully-private-cluster에서-nginx-배포)
7. [도전과제 3 - EKS 보안 그룹 정리](#7-도전과제-3---eks-보안-그룹-정리)
8. [정리 및 소감](#8-정리-및-소감)

---

## 1. Amazon EKS 소개

### 1-1. Amazon EKS란?

**Amazon EKS(Elastic Kubernetes Service)** 는 AWS에서 제공하는 완전관리형 Kubernetes 서비스로, Kubernetes 클러스터 운영의 복잡성을 제거해줍니다.

- 공식 문서: https://docs.aws.amazon.com/ko_kr/eks/latest/userguide/what-is-eks.html
- AWS가 **Control Plane(컨트롤 플레인)** 을 직접 관리 → 사용자는 **Data Plane(데이터 플레인)** 에 집중
- 두 개의 VPC 구조:
    - **AWS 관리 VPC**: Control Plane (etcd, kube-apiserver 등)
    - **고객 VPC**: Worker Node (실제 워크로드 실행)

### 1-2. EKS 아키텍처

```
┌──────────────────────────────────────────────────────┐
│                   AWS 관리 영역 (VPC)                  │
│  ┌──────────────────────────────────────────────────┐ │
│  │              Control Plane                       │ │
│  │  kube-apiserver  |  etcd  |  kube-scheduler      │ │
│  │  kube-controller-manager  |  cloud-controller    │ │
│  └──────────────────────────────────────────────────┘ │
└───────────────────────┬──────────────────────────────┘
                        │ EKS Managed ENI
┌───────────────────────▼──────────────────────────────┐
│                   고객 VPC                             │
│  ┌──────────────────────────────────────────────────┐ │
│  │              Data Plane (Worker Nodes)           │ │
│  │   Node1 (kubelet, kube-proxy, containerd)        │ │
│  │   Node2 (kubelet, kube-proxy, containerd)        │ │
│  └──────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────┘
```

### 1-3. Kubernetes 클러스터 구성 요소

#### Control Plane 구성 요소

| 컴포넌트 | 역할 |
|---|---|
| **kube-apiserver** | Kubernetes API 노출, 내/외부 통신 창구 |
| **etcd** | 클러스터 상태 저장 (Key-Value Store) |
| **kube-scheduler** | 포드를 적절한 노드에 예약/배치 |
| **kube-controller-manager** | 클러스터 상태 감시 및 원하는 상태 유지 |
| **cloud-controller-manager** | AWS 리소스(LB, 라우팅 등) 연동 관리 |

#### Data Plane (Worker Node) 구성 요소

| 컴포넌트 | 역할 |
|---|---|
| **kubelet** | API 서버와 통신, 노드/포드 상태 관리 |
| **kube-proxy** | 포드 간 네트워킹(iptables/ipvs) 관리 |
| **Container Runtime** | 컨테이너 실행 (기본: containerd) |

> 💡 Kubernetes 1.24부터 **dockershim이 제거**되어, Docker를 컨테이너 런타임으로 직접 사용 불가. 기본 런타임은 **containerd** 사용.

### 1-4. 워크로드 종류

| 워크로드 유형 | Kubernetes 객체 | 특징 |
|---|---|---|
| 상태 비저장(Stateless) | Deployment + ReplicaSet | 웹 서버 등, 쉽게 대체 가능 |
| 상태 저장(Stateful) | StatefulSet | DB 등, 고유 ID와 안정적 스토리지 필요 |
| 노드별 실행 | DaemonSet | 모니터링 에이전트 등, 모든 노드에 실행 |
| 일회성 작업 | Job / CronJob | 배치 처리, 정기 실행 작업 |

### 1-5. EKS 배포 방법

EKS 클러스터를 생성하는 방법은 다양합니다:

1. **AWS Management Console** - GUI 클릭 방식
2. **eksctl** - EKS 전용 CLI 도구 (`eksctl create cluster`)
3. **IaC 도구** - Terraform, CDK, CloudFormation
4. **EKS Anywhere** - 온프레미스 환경 (VMware vSphere, Bare Metal 등)

---

## 2. Amazon EKS 배포 및 확인

### 2-1. 로컬 환경 준비 (macOS)

#### AWS CLI 설치 및 자격 증명 설정

```bash
# AWS CLI 설치
brew install awscli
aws --version

# IAM 자격 증명 설정
aws configure
# AWS Access Key ID: <액세스 키 입력>
# AWS Secret Access Key: <시크릿 키 입력>
# Default region name: ap-northeast-2

# 확인
aws sts get-caller-identity
```

#### EC2 Key Pair 생성

```bash
# Key Pair 생성 (pem 파일 생성)
aws ec2 create-key-pair \
  --key-name my-keypair \
  --query 'KeyMaterial' \
  --output text > my-keypair.pem

# 권한 설정
chmod 400 my-keypair.pem

# 확인
aws ec2 describe-key-pairs --key-names my-keypair
```

#### kubectl 및 Helm 설치

```bash
# kubectl 설치
brew install kubernetes-cli
kubectl version --client=true

# Helm 설치
brew install helm
helm version
```

#### 유용한 k8s 도구 설치

```bash
# krew 설치
brew install krew

# k9s 설치 (TUI 기반 클러스터 관리)
brew install k9s

# kube-ps1 설치 (프롬프트에 컨텍스트/네임스페이스 표시)
brew install kube-ps1

# kubectx 설치 (컨텍스트 전환 간소화)
brew install kubectx

# kubecolor 설치 (kubectl 출력 하이라이트)
brew install kubecolor
echo "alias k=kubectl" >> ~/.zshrc
echo "alias kubectl=kubecolor" >> ~/.zshrc
echo "compdef kubecolor=kubectl" >> ~/.zshrc

# krew PATH 설정
export PATH="${KREW_ROOT:-$HOME/.krew}/bin:$PATH"
```

#### Terraform 설치 (tfenv 사용)

```bash
# tfenv 설치
brew install tfenv

# 설치 가능 버전 확인
tfenv list-remote

# 특정 버전 설치 및 사용 설정
tfenv install 1.14.6
tfenv use 1.14.6

# 버전 확인
terraform version

# 자동완성 설정
terraform -install-autocomplete
```

### 2-2. EKS 클러스터 배포 (Terraform)

```bash
# 실습 코드 다운로드
git clone https://github.com/gasida/aews.git
cd aews
tree aews

# 1주차 작업 디렉터리로 이동
cd 1w

# VPC 및 EKS 배포 (약 12분 소요)
terraform init
terraform plan
terraform apply -auto-approve
```

### 2-3. EKS 정보 확인

```bash
# kubeconfig 업데이트
aws eks update-kubeconfig --region ap-northeast-2 --name <클러스터명>

# 클러스터 정보 확인
kubectl cluster-info
kubectl get nodes -o wide

# 시스템 파드 확인
kubectl get pods -n kube-system

# 노드 그룹 확인
eksctl get nodegroup --cluster <클러스터명>

# Add-on 확인
eksctl get addons --cluster <클러스터명>
```

### 2-3-1. 인프라 엔드포인트 및 노드 그룹 상태

### [Command] API 서버 DNS 확인
```bash
dig +short $APIDNS
```
**[Output]**
```text
43.203.27.2
3.39.188.105
```
> **[해석]**: EKS Control Plane의 퍼블릭 IP 주소들이 조회되었습니다. 이는 클러스터 엔드포인트가 인터넷을 통해 접근 가능한 상태임을 의미합니다.

### [Command] 노드 그룹 상세 정보 조회
```bash
aws eks describe-nodegroup --cluster-name $CLUSTER_NAME --nodegroup-name $CLUSTER_NAME-node-group | jq
```
**[Output (핵심)]**
```json
{
  "nodegroup": {
    "status": "ACTIVE",
    "capacityType": "ON_DEMAND",
    "instanceTypes": ["t3.medium"],
    "amiType": "AL2023_x86_64_STANDARD",
    "scalingConfig": { "minSize": 1, "maxSize": 4, "desiredSize": 2 }
  }
}
```
> **[해석]**: 노드 그룹이 `ACTIVE` 상태이며, Amazon Linux 2023(`AL2023`) 기반의 `t3.medium` 인스턴스 2대가 온디맨드로 운영 중임을 알 수 있습니다.

---

### 2-3-2. 노드(Node) 자원 상태 분석

### [Command] 노드 요약 정보 (Wide View)
```bash
kubectl get node -owide
```
**[Output]**
```text
NAME             STATUS   ROLES    AGE   INTERNAL-IP     EXTERNAL-IP      OS-IMAGE                        CONTAINER-RUNTIME
ip-192-168-1-140 Ready    <none>   10m   192.168.1.140   13.124.239.138   Amazon Linux 2023.10.20260302   containerd://2.1.5
ip-192-168-2-154 Ready    <none>   10m   192.168.2.154   3.38.194.56      Amazon Linux 2023.10.20260302   containerd://2.1.5
```
> **[해석]**: 두 대의 노드가 서로 다른 서브넷(`192.168.1.x`, `192.168.2.x`)에 배치되어 있으며, 모두 `Ready` 상태로 정상입니다. 외부 접속용 `EXTERNAL-IP`가 할당된 퍼블릭 노드 구성입니다.

---

### 2-3-3. 인증 및 보안 설정

### [Command] Kubeconfig 및 인증 토큰 확인
```bash
aws eks get-token --cluster-name $CLUSTER_NAME --region $AWS_DEFAULT_REGION | jq
```
**[Output]**
```json
{
  "kind": "ExecCredential",
  "status": {
    "expirationTimestamp": "2026-03-15T02:50:45Z",
    "token": "k8s-aws-v1.aHR0cHM6Ly..."
  }
}
```
> **[해석]**: AWS STS를 이용한 임시 보안 토큰이 정상 생성되었습니다. `kubectl`은 이 토큰을 사용하여 API 서버와 통신하며, 토큰은 약 15분 후 만료되어 보안을 유지합니다.

---

### 2-3-4. 시스템 컴포넌트 (`kube-system`) 분석

### [Command] 시스템 포드 상태 확인
```bash
kubectl get pod -n kube-system -o wide
```
**[Output]**
```text
NAME                      READY   STATUS    RESTARTS   IP              NODE
aws-node-cpgvt            2/2     Running   0          192.168.2.154   ip-192-168-2-154...
coredns-d487b6fcb-6lkbt   1/1     Running   0          192.168.2.88    ip-192-168-2-154...
kube-proxy-2vb4w          1/1     Running   0          192.168.2.154   ip-192-168-2-154...
```
> **[해석]**: 핵심 서비스인 네트워킹(`aws-node`), DNS(`coredns`), 통신규칙(`kube-proxy`) 포드들이 가용 영역별 노드에 고르게 분산되어 실행 중입니다.

### [Command] Kube-Proxy 설정(Mode) 확인
```bash
kubectl get cm -n kube-system kube-proxy-config -o yaml | grep mode
```
**[Output]**
```text
mode: "iptables"
```
> **[해석]**: EKS의 기본 서비스 로드밸런싱 모드인 `iptables` 방식을 사용하여 트래픽을 처리하고 있습니다.

### [Command] CoreDNS 가용성 정책(PDB) 확인
```bash
kubectl get pdb -n kube-system coredns -o jsonpath='{.spec}' | jq
```
**[Output]**
```json
{ "maxUnavailable": 1 }
```
> **[해석]**: 클러스터 업데이트나 노드 점검 시에도 최소 1대의 CoreDNS 포드는 항상 가동되도록 보장하는 정책이 적용되어 있습니다.

---

### 2-3-5. EKS 관리형 애드온(Add-ons)

### [Command] 설치된 애드온 리스트
```bash
aws eks list-addons --cluster-name myeks | jq
```
**[Output]**
```json
{ "addons": [ "coredns", "kube-proxy", "vpc-cni" ] }
```
> **[해석]**: AWS가 직접 관리하고 패치를 제공하는 3가지 핵심 애드온이 모두 정상적으로 설치되어 관리되고 있습니다.

---

#### 워커 노드 SSH 접속 후 확인

```bash
##이하 명령어 수행 결과물 생략

ssh ec2-user@$NODE1

# 컨테이너 런타임 확인
systemctl status containerd

# kubelet 상태 확인
systemctl status kubelet

# CNI 설정 확인
ls /etc/cni/net.d/

# cgroup 정보 확인
cat /proc/cgroups
```

## 3. EKS Cluster Endpoint Access

### 3-1. 개요
EKS Cluster Endpoint Access란 사용자가 kubectl 등을 통해 쿠버네티스 API 서버에 접근하는 방식을 제어하는 네트워크 설정입니다. EKS는 AWS VPC(컨트롤 플레인)와 고객 VPC(데이터 플레인) 두 영역으로 분리되어 있으며, 이 사이의 통신 경로를 3가지 모드로 설정할 수 있습니다

EKS Cluster Endpoint Access = 쿠버네티스 컨트롤 플레인의 kube-apiserver에 대한 네트워크 접근 방식을 제어하는 설정

EKS API 서버 엔드포인트에 대한 접근 방식은 3가지 모드로 구성됩니다:

| 모드 | 설명 | 특징 |
|---|---|---|
| **Public** (기본값) | API 서버가 인터넷에 노출 | 어디서든 kubectl 접근 가능, 보안 주의 |
| **Public & Private** | 인터넷 + VPC 내부 모두 접근 | 노드→API 통신은 VPC 내부, 외부 접근도 가능 |
| **Fully Private** | VPC 내부에서만 접근 가능 | 가장 높은 보안 수준, VPN/Bastion 필요 |

### 3-2. 통신 흐름

#### Public 모드
```
kubectl(외부) → 인터넷 → EKS API Endpoint (Public IP) → Control Plane
Worker Node   → 인터넷 → EKS API Endpoint (Public IP) → Control Plane
```

#### Public & Private 모드
```
kubectl(외부) → 인터넷 → EKS API Endpoint (Public IP) → Control Plane
Worker Node   → VPC 내부 → EKS Managed ENI → Control Plane (Private)
```

#### Fully Private 모드
```
kubectl(내부) → VPC → EKS API Endpoint (Private IP) → Control Plane
Worker Node   → VPC → EKS Managed ENI → Control Plane
```

### 3-3. Endpoint Access 변경 (Terraform)

```hcl
# eks 모듈 내 endpoint_public_access, endpoint_private_access 설정
module "eks" {
  source  = "terraform-aws-modules/eks/aws"

  cluster_endpoint_public_access  = true
  cluster_endpoint_private_access = true

  # 퍼블릭 접근 허용 CIDR 설정 (특정 IP만 허용)
  cluster_endpoint_public_access_cidrs = ["203.0.113.0/24"]
}
```

### 3-4. 실습 환경 정리

```bash
# kubeconfig 삭제 및 리소스 정리
rm -rf ~/.kube/config
nohup sh -c "terraform destroy -auto-approve" > delete.log 2>&1 &
tail -f delete.log
```

---

## 4. EKS Fully Private Cluster

### 4-1. 개요

인터넷 접근이 완전히 차단된 Private 환경에서 EKS를 운영하는 구성입니다.

- 참고: https://docs.aws.amazon.com/ko_kr/eks/latest/userguide/private-clusters.html
- 참고 패턴: https://aws-ia.github.io/terraform-aws-eks-blueprints/patterns/fully-private-cluster/

### 4-2. 필수 사전 지식

| 항목 | 내용 |
|---|---|
| **Private Subnet** | 인터넷 게이트웨이 없는 라우팅 테이블 |
| **보안 그룹** | 인바운드/아웃바운드 트래픽 제어 |
| **VPC Endpoint** | AWS 서비스에 Private Link로 접근 |
| **EKS 보안 그룹** | 클러스터/노드 간 통신 제어 |

### 4-3. Fully Private Cluster에 필요한 VPC Endpoint 목록

```
인터넷 없이 EKS를 운영하려면 아래 VPC Endpoint가 필요합니다:

- com.amazonaws.<region>.eks          (EKS API)
- com.amazonaws.<region>.ecr.api      (ECR API)
- com.amazonaws.<region>.ecr.dkr      (ECR Docker)
- com.amazonaws.<region>.s3           (S3 - ECR 레이어 저장소)
- com.amazonaws.<region>.ec2          (EC2 API)
- com.amazonaws.<region>.sts          (STS - IAM 인증)
- com.amazonaws.<region>.elasticloadbalancing  (ELB)
- com.amazonaws.<region>.autoscaling  (Auto Scaling)
- com.amazonaws.<region>.logs         (CloudWatch Logs)
- com.amazonaws.<region>.ssm          (SSM)
- com.amazonaws.<region>.ssmmessages  (SSM Session Manager)
- com.amazonaws.<region>.ec2messages  (SSM EC2 Messages)
```

### 4-4. 배포 (Terraform, 약 16분 소요)

```bash
cd terraform-aws-eks-blueprints/patterns/fully-private-cluster

terraform init
terraform apply -auto-approve

# Bastion EC2 접속 후 확인
# (Bastion은 동일 VPC 내 Public Subnet에 위치)
```

### 4-5. 실습 완료 후 정리

```bash
# 보안 그룹 규칙 제거 (eks-private-cluster의 HTTPS bastion-ec2 sg 규칙)
# AWS 콘솔 또는 CLI로 보안 그룹 인바운드 규칙 수정

# 리소스 삭제
terraform destroy -auto-approve
```

---

## 5. 도전과제 1 - EKS vs Vanilla k8s 비교

### 5-1. Control Plane 비교

| 항목 | Vanilla Kubernetes | Amazon EKS |
|---|---|---|
| **운영 주체** | 사용자 직접 관리 | AWS 완전 관리형 |
| **설치 방법** | kubeadm, kops 등 | eksctl, Terraform, Console |
| **고가용성** | 사용자가 직접 HA 구성 | AWS가 자동으로 Multi-AZ HA 보장 |
| **etcd 관리** | 사용자가 백업/복원 관리 | AWS가 자동 백업/복원 |
| **API Server 접근** | 사용자 설정에 따라 다름 | Public/Private/둘다 3가지 모드 선택 |
| **버전 업그레이드** | 수동으로 각 컴포넌트 업그레이드 | Console/CLI로 버전 선택 업그레이드 |
| **비용** | EC2 비용만 발생 | Control Plane 비용 + EC2 비용 ($0.10/hr) |
| **모니터링** | 별도 구성 필요 | CloudWatch와 통합 가능 |

### 5-2. Data Plane 비교

| 항목 | Vanilla Kubernetes | Amazon EKS |
|---|---|---|
| **노드 관리** | 사용자가 직접 노드 추가/제거 | Managed Node Group, Fargate 사용 가능 |
| **오토 스케일링** | Cluster Autoscaler 직접 구성 | Managed Node Group 내장 스케일링 + Karpenter |
| **노드 OS** | 직접 선택 및 관리 | Amazon Linux 2/2023, Bottlerocket, Windows |
| **컨테이너 런타임** | 직접 설치 (containerd 등) | containerd 기본 탑재 |
| **IAM 통합** | 별도 인증 구성 (OIDC 등) | IRSA(IAM Roles for Service Accounts) 네이티브 지원 |
| **네트워킹(CNI)** | Calico, Flannel 등 직접 선택 | AWS VPC CNI 기본 사용 (Pod = VPC IP) |
| **스토리지** | 직접 CSI 드라이버 설치 | EBS, EFS CSI 드라이버 Add-on으로 쉽게 설치 |
| **보안 그룹** | 노드 레벨 보안 그룹만 | 노드 레벨 + Pod 레벨 보안 그룹 지원 |

### 5-3. 핵심 차이 요약

```
Vanilla k8s:
  - 완전한 자유도, 모든 것을 직접 제어
  - 운영 복잡도 높음
  - 인프라 비용만 발생

Amazon EKS:
  - Control Plane은 AWS가 관리 (HA, 패치, 업그레이드)
  - AWS 서비스와 긴밀한 통합 (IAM, VPC, ELB, ECR 등)
  - 관리형 서비스 비용 추가 발생
  - 빠른 구성 및 운영 편의성
```

---

## 6. 도전과제 2 - Fully Private Cluster에서 Nginx 배포

### 6-1. Bastion EC2 접속

```bash
# Bastion EC2에 SSH 접속
ssh -i my-keypair.pem ec2-user@<BASTION_PUBLIC_IP>

# kubeconfig 설정
aws eks update-kubeconfig --region ap-northeast-2 --name <CLUSTER_NAME>

# 클러스터 연결 확인
kubectl get nodes
```

### 6-2. Nginx Deployment 배포

```yaml
# nginx-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
        resources:
          requests:
            cpu: "100m"
            memory: "128Mi"
          limits:
            cpu: "200m"
            memory: "256Mi"
```

```bash
kubectl apply -f nginx-deployment.yaml
kubectl get pods -o wide
```

### 6-3. 외부 노출 설정 (NLB를 통한 LoadBalancer Service)

> Fully Private Cluster에서는 인터넷 직접 노출 대신 **내부 NLB(Internal Load Balancer)** 를 사용해 VPC 내부에서 접근합니다.

```yaml
# nginx-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
  annotations:
    # 내부(Internal) NLB 사용
    service.beta.kubernetes.io/aws-load-balancer-type: "nlb"
    service.beta.kubernetes.io/aws-load-balancer-scheme: "internal"
    service.beta.kubernetes.io/aws-load-balancer-cross-zone-load-balancing-enabled: "true"
spec:
  type: LoadBalancer
  selector:
    app: nginx
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
```

```bash
kubectl apply -f nginx-service.yaml

# 서비스 확인 (EXTERNAL-IP는 내부 NLB DNS)
kubectl get svc nginx-service

# 내부 NLB DNS로 접근 확인 (Bastion에서 curl)
NLB_DNS=$(kubectl get svc nginx-service -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
curl http://$NLB_DNS
```

### 6-4. 배포 확인

```bash
# 파드 상태 확인
kubectl get pods -l app=nginx

# 서비스 엔드포인트 확인
kubectl get endpoints nginx-service

# 파드 로그 확인
kubectl logs -l app=nginx

# Bastion에서 NLB를 통한 접근 테스트
curl -v http://$NLB_DNS
```

---

## 7. 도전과제 3 - EKS 보안 그룹 정리

### 7-1. EKS 보안 그룹 종류

EKS 운영 시 관련된 보안 그룹은 총 3가지입니다:

```
┌─────────────────────────────────────────────────────┐
│  1. 클러스터 보안 그룹 (Cluster Security Group)        │
│     - EKS가 자동 생성                                 │
│     - Control Plane ↔ Worker Node 통신               │
├─────────────────────────────────────────────────────┤
│  2. 추가 클러스터 보안 그룹 (Additional Cluster SG)    │
│     - 사용자가 추가 지정 가능                          │
│     - Control Plane ENI에 적용                        │
├─────────────────────────────────────────────────────┤
│  3. 노드 보안 그룹 (Node Security Group)              │
│     - Worker Node EC2에 적용                         │
│     - Pod 간 통신, 외부 접근 제어                      │
└─────────────────────────────────────────────────────┘
```

### 7-2. 클러스터 보안 그룹 (Cluster SG)

| 방향 | 포트 | 프로토콜 | 대상 | 용도 |
|---|---|---|---|---|
| **인바운드** | 443 | TCP | Worker Node SG | kubectl API 호출 |
| **인바운드** | 443 | TCP | Bastion SG | 관리자 접근 |
| **아웃바운드** | 10250 | TCP | Worker Node SG | kubelet API 접근 (로그, exec) |
| **아웃바운드** | All | All | 0.0.0.0/0 | 기본 아웃바운드 |

### 7-3. 노드 보안 그룹 (Node SG)

| 방향 | 포트 | 프로토콜 | 대상 | 용도 |
|---|---|---|---|---|
| **인바운드** | All | All | 동일 Node SG | 노드 간 Pod 통신 |
| **인바운드** | 10250 | TCP | Cluster SG | kubelet (Control Plane에서) |
| **인바운드** | 443 | TCP | Cluster SG | API 서버와 통신 |
| **인바운드** | 30000-32767 | TCP | 0.0.0.0/0 | NodePort 서비스 |
| **아웃바운드** | All | All | 0.0.0.0/0 | 기본 아웃바운드 |

### 7-4. Fully Private Cluster 추가 보안 그룹 규칙

```
Bastion EC2 → Cluster SG (HTTPS 443): Bastion에서 kubectl 접근
Worker Node → VPC Endpoint SG (HTTPS 443): AWS 서비스 접근
```

### 7-5. 보안 그룹 적용 흐름 다이어그램

```
[관리자] ──443──► [Bastion EC2]
                      │
                      │ 443
                      ▼
           [클러스터 SG (Control Plane ENI)]
                      │
          ┌───────────┼───────────┐
          │ 10250     │ 443       │
          ▼           ▼           ▼
      [Node SG]   [Node SG]   [Node SG]
      Worker1     Worker2     Worker3
          │
          │ All Traffic (Pod ↔ Pod)
          └───────────────────────┘
```

---

## 8. 정리 및 소감

### 학습 내용 요약

이번 1주차에서는 다음 내용을 학습했습니다:

1. **EKS 아키텍처**: Control Plane(AWS 관리)과 Data Plane(고객 관리)의 분리 구조 이해
2. **EKS 배포**: Terraform을 활용한 VPC + EKS 클러스터 배포 실습
3. **Endpoint Access 모드**: Public / Public+Private / Fully Private 3가지 모드별 동작 차이
4. **Fully Private Cluster**: 인터넷 차단 환경에서 VPC Endpoint를 활용한 완전 격리 EKS 운영
5. **보안 그룹**: EKS 관련 3가지 보안 그룹의 역할과 적용 방식

### EKS 운영 시 권장 Endpoint Access 모드

```
개발 환경  : Public (빠른 접근, 비용 최소화)
스테이징   : Public & Private (외부 접근 허용 + 내부 통신 최적화)
프로덕션   : Fully Private (최고 보안, VPN/Bastion 통해 접근)
```

### 참고 자료

- [Amazon EKS 공식 문서](https://docs.aws.amazon.com/ko_kr/eks/latest/userguide/what-is-eks.html)
- [EKS 아키텍처](https://docs.aws.amazon.com/ko_kr/eks/latest/userguide/eks-architecture.html)
- [EKS Blueprints for Terraform](https://github.com/aws-ia/terraform-aws-eks-blueprints)
- [EKS Fully Private Cluster 패턴](https://aws-ia.github.io/terraform-aws-eks-blueprints/patterns/fully-private-cluster/)
- [AWS re:Invent 2024 - The future of Kubernetes on AWS](https://www.youtube.com/watch?v=_wwu0VKy3w4)
- [AEWS 실습 코드 (GitHub)](https://github.com/gasida/aews)

---

*작성: 심재엽 | AEWS 4기 1주차*
