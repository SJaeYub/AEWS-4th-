# 4주차 - K8S(EKS) Identity and Access Management 심화 기술 가이드

## 목차

1. [필요 지식 — K8S 인증/인가 기초](#1-필요-지식--k8s-인증인가-기초)
1. [K8S 인증/인가 기초 실습 구조 이해](#2-k8s-인증인가-기초-실습-구조-이해)
1. [K8S(EKS) Identity and Access Management 전체 구조](#3-k8seks-identity-and-access-management-전체-구조)
1. [K8S(EKS) Pod(SA) with IAM Role → AWS 리소스 사용 (IRSA)](#4-k8seks-podsa-with-iam-role--aws-리소스-사용-irsa)
1. [심화: EKS Pod Identity (차세대 IRSA)](#5-심화-eks-pod-identity-차세대-irsa)
1. [면접 질문 — 3-Depth 구조](#6-면접-질문--3-depth-구조)

-----

## 1. 필요 지식 — K8S 인증/인가 기초

### 1.1 인증(Authentication)과 인가(Authorization)의 구분

보안 시스템의 두 핵심 개념을 명확히 구분해야 합니다.

|개념    |영어                    |질문                |K8S에서의 답           |
|------|----------------------|------------------|-------------------|
|**인증**|Authentication (AuthN)|“당신은 누구인가?”       |사용자/서비스 어카운트 신원 확인 |
|**인가**|Authorization (AuthZ) |“당신은 무엇을 할 수 있는가?”|RBAC으로 리소스 접근 권한 결정|

```
kubectl get pods 요청 흐름:

클라이언트
  │ HTTPS 요청 (인증서/토큰 포함)
  ▼
kube-apiserver
  ├─ [1] Authentication (인증) → "이 요청자는 누구인가?"
  │       └─ X.509 인증서 / Bearer Token / OIDC 토큰 검증
  │
  ├─ [2] Authorization (인가) → "이 행위가 허용되는가?"
  │       └─ RBAC: user/group이 pods/get 권한 보유 여부 확인
  │
  ├─ [3] Admission Control → "이 요청이 정책에 부합하는가?"
  │       └─ ValidatingWebhook, MutatingWebhook, OPA/Kyverno
  │
  └─ [4] 요청 처리 → etcd 조회 후 응답
```

### 1.2 K8S API 서버의 인증 메커니즘 종류

Kubernetes는 다양한 인증 방식을 플러그인 형태로 지원합니다. 여러 방식이 동시에 활성화될 수 있으며, **하나라도 인증에 성공하면 통과**합니다.

|인증 방식                    |설명                            |EKS 사용 여부                  |
|-------------------------|------------------------------|---------------------------|
|**X.509 클라이언트 인증서**      |CA가 서명한 인증서로 신원 증명            |✅ 노드 인증에 사용                |
|**Bearer Token (Static)**|정적 토큰 파일 기반                   |❌ 보안상 비권장                  |
|**Bootstrap Token**      |노드 초기 등록용 임시 토큰               |✅ 노드 부트스트랩                 |
|**Service Account Token**|파드 내 자동 마운트 JWT               |✅ 파드 인증 기본 방식              |
|**OIDC Token**           |외부 IdP(Identity Provider)의 JWT|✅ EKS 사용자 인증 핵심            |
|**Webhook Token**        |외부 webhook으로 토큰 검증 위임         |✅ EKS aws-iam-authenticator|
|**Authenticating Proxy** |프록시가 헤더로 신원 전달                |특수 환경                      |

### 1.3 K8S 사용자 모델의 두 종류

Kubernetes는 자체 사용자 오브젝트가 없습니다. 사용자는 외부 시스템에서 관리됩니다.

```
K8S 접근 주체 (Subject)
  │
  ├─ [1] 일반 사용자 (Normal User)
  │       - K8S 내부 오브젝트 없음
  │       - 인증서 CN(Common Name) 또는 OIDC 토큰의 sub 클레임으로 식별
  │       - kubectl을 사용하는 인간 사용자
  │
  └─ [2] 서비스 어카운트 (ServiceAccount)
          - K8S 네임스페이스 오브젝트로 관리
          - 파드 내 프로세스가 API 서버와 통신 시 사용
          - 자동으로 JWT 토큰 생성 및 파드에 마운트
          - 기본: default ServiceAccount (네임스페이스별 자동 생성)
```

### 1.4 RBAC (Role-Based Access Control) 핵심 개념

K8S의 기본 인가 방식. 4가지 핵심 리소스로 구성됩니다.

```
RBAC 구성 요소:

┌─────────────────────────────────────────────────────────┐
│ Role / ClusterRole                                      │
│ "무엇을 할 수 있는가?" — 권한 정의                        │
│                                                         │
│  rules:                                                 │
│  - apiGroups: [""]                                      │
│    resources: ["pods"]                                  │
│    verbs: ["get", "list", "watch"]                      │
└─────────────────────┬───────────────────────────────────┘
                      │ RoleBinding / ClusterRoleBinding
                      │ "누구에게 이 권한을 부여하는가?"
                      │
                      ▼
┌─────────────────────────────────────────────────────────┐
│ Subject (접근 주체)                                      │
│                                                         │
│  subjects:                                              │
│  - kind: User          # 일반 사용자                     │
│  - kind: Group         # 사용자 그룹                     │
│  - kind: ServiceAccount # 서비스 어카운트                │
└─────────────────────────────────────────────────────────┘
```

**Role vs ClusterRole**

|구분                  |스코프                               |사용 사례                   |
|--------------------|----------------------------------|------------------------|
|`Role`              |특정 네임스페이스                         |개발팀이 dev 네임스페이스 파드 조회   |
|`ClusterRole`       |클러스터 전체                           |클러스터 노드 조회, 네임스페이스 목록 조회|
|`RoleBinding`       |특정 네임스페이스에 Role 또는 ClusterRole 바인딩|-                       |
|`ClusterRoleBinding`|클러스터 전체에 ClusterRole 바인딩          |cluster-admin 부여        |

### 1.5 K8S 내장 ClusterRole

자주 사용되는 기본 ClusterRole입니다.

|ClusterRole        |권한 범위                   |사용 대상     |
|-------------------|------------------------|----------|
|`cluster-admin`    |모든 리소스 전체 권한            |클러스터 관리자  |
|`admin`            |네임스페이스 내 대부분 권한         |팀 관리자     |
|`edit`             |네임스페이스 내 읽기/쓰기 (Role 제외)|개발자       |
|`view`             |네임스페이스 내 읽기 전용          |모니터링, 감사  |
|`system:node`      |노드가 필요한 권한              |kubelet   |
|`system:kube-proxy`|kube-proxy 필요 권한        |kube-proxy|

### 1.6 Admission Controller

인증/인가 후 요청을 가로채어 검증 또는 변환하는 플러그인 체계입니다.

```
Admission Controller 유형:

Mutating Admission Webhook
  └─ 요청 변환 (주입, 기본값 설정)
  └─ 예: Istio sidecar 주입, VPA Admission Controller

Validating Admission Webhook
  └─ 요청 검증 (정책 위반 거부)
  └─ 예: OPA/Gatekeeper, Kyverno, PSA(Pod Security Admission)

처리 순서: Mutating 먼저 → 직렬 처리 → Validating 나중 → 병렬 처리
```

-----

## 2. K8S 인증/인가 기초 실습 구조 이해

### 2.1 kubeconfig 파일 구조

`kubectl`이 API 서버와 통신하는 설정 파일. 클러스터, 사용자, 컨텍스트 정보를 담습니다.

```yaml
apiVersion: v1
kind: Config

# 클러스터 목록 (여러 클러스터 관리 가능)
clusters:
  - name: my-eks-cluster
    cluster:
      server: https://XXXXXXXX.gr7.ap-northeast-2.eks.amazonaws.com
      certificate-authority-data: <base64 CA 인증서>

# 사용자 자격증명 목록
users:
  - name: eks-admin
    user:
      # EKS 방식: exec으로 aws CLI를 호출하여 토큰 동적 생성
      exec:
        apiVersion: client.authentication.k8s.io/v1beta1
        command: aws
        args:
          - eks
          - get-token
          - --cluster-name
          - my-eks-cluster
          - --region
          - ap-northeast-2
        env:
          - name: AWS_PROFILE
            value: admin-profile

# 컨텍스트: 클러스터 + 사용자 + 네임스페이스 조합
contexts:
  - name: eks-admin@my-eks-cluster
    context:
      cluster: my-eks-cluster
      user: eks-admin
      namespace: default

current-context: eks-admin@my-eks-cluster
```

### 2.2 X.509 인증서 기반 사용자 인증 흐름

클러스터 CA가 서명한 인증서로 K8S 사용자를 인증하는 방식입니다.

```
1. 개인키 생성
   openssl genrsa -out user.key 2048

2. CSR(Certificate Signing Request) 생성
   openssl req -new -key user.key -out user.csr \
     -subj "/CN=alice/O=dev-team"
   # CN = K8S 사용자명, O = K8S 그룹명

3. K8S CertificateSigningRequest 오브젝트 생성
   kubectl apply -f csr.yaml

4. 클러스터 관리자가 CSR 승인
   kubectl certificate approve alice-csr

5. 서명된 인증서 추출
   kubectl get csr alice-csr -o jsonpath='{.status.certificate}' | base64 -d > user.crt

6. kubeconfig에 등록
   kubectl config set-credentials alice \
     --client-certificate=user.crt \
     --client-key=user.key
```

**인증서의 CN과 O가 K8S RBAC의 User/Group과 매핑됩니다.**

```yaml
# RoleBinding에서 인증서 CN/O로 Subject 지정
subjects:
  - kind: User
    name: alice          # CN=alice
    apiGroup: rbac.authorization.k8s.io
  - kind: Group
    name: dev-team       # O=dev-team
    apiGroup: rbac.authorization.k8s.io
```

### 2.3 ServiceAccount 토큰 동작 원리

파드가 API 서버와 통신할 때 사용하는 자동 생성 토큰입니다.

```
ServiceAccount 생성
  │ K8S가 자동으로 JWT 토큰 생성
  ▼
파드 생성 시 자동 마운트
  /var/run/secrets/kubernetes.io/serviceaccount/
  ├── token    ← JWT 토큰 (API 서버 CA가 서명)
  ├── ca.crt   ← 클러스터 CA 인증서
  └── namespace ← 현재 네임스페이스

파드 내 프로세스 → API 서버 요청:
  Authorization: Bearer <token>
```

**K8S 1.21+ Projected Service Account Token**

기존 토큰은 만료 없이 영구 유효했습니다. 1.21부터 **Bound Service Account Token**이 기본화되어:

- **만료 시간 설정** (기본 1시간, kubelet이 자동 갱신)
- **특정 대상(audience) 바인딩** → 다른 서비스에서 재사용 불가
- **파드 생명주기에 바인딩** → 파드 삭제 시 토큰 무효화

```yaml
# Projected Volume으로 토큰 마운트 (K8S 자동 구성)
volumes:
  - name: kube-api-access
    projected:
      sources:
        - serviceAccountToken:
            path: token
            expirationSeconds: 3607  # 1시간 + 7초 갱신 여유
            audience: "https://kubernetes.default.svc"
        - configMap:
            name: kube-root-ca.crt
        - downwardAPI: ...
```

### 2.4 RBAC 설계 패턴

**최소 권한 원칙 (Least Privilege)**

```yaml
# ❌ 나쁜 예: 광범위한 권한
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: all-access
subjects:
  - kind: ServiceAccount
    name: my-app
    namespace: default
roleRef:
  kind: ClusterRole
  name: cluster-admin   # 모든 권한 부여 - 절대 금지
  apiGroup: rbac.authorization.k8s.io

---
# ✅ 좋은 예: 필요한 권한만 명시
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: production
rules:
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["pods/log"]
    verbs: ["get"]
```

**Verb 종류**

|Verb              |설명       |HTTP               |
|------------------|---------|-------------------|
|`get`             |단일 리소스 조회|GET                |
|`list`            |목록 조회    |GET (collection)   |
|`watch`           |변경 이벤트 구독|GET (watch)        |
|`create`          |생성       |POST               |
|`update`          |전체 수정    |PUT                |
|`patch`           |부분 수정    |PATCH              |
|`delete`          |삭제       |DELETE             |
|`deletecollection`|일괄 삭제    |DELETE (collection)|
|`exec`            |컨테이너 exec|POST /exec         |

-----

## 3. K8S(EKS) Identity and Access Management 전체 구조

### 3.1 EKS 인증의 고유성

EKS는 Kubernetes 표준 인증 외에 **AWS IAM을 K8S 인증과 통합**한 독자적인 구조를 가집니다. 이것이 일반 K8S와 EKS의 가장 큰 차이점입니다.

```
EKS 인증 전체 흐름:

kubectl 명령
  │
  ▼ kubeconfig의 exec credential plugin 실행
aws eks get-token --cluster-name my-cluster
  │
  ▼ AWS STS에 GetCallerIdentity 사전 서명 URL 생성
  │  (AWS IAM 자격증명으로 서명)
  │
  ▼ K8S API 서버로 Bearer Token 전달
  │  (token = base64(pre-signed STS URL))
  │
  ▼ EKS 컨트롤 플레인의 aws-iam-authenticator (Webhook)
  │  STS URL을 호출하여 IAM Identity 확인
  │  → ARN 반환: arn:aws:iam::123456789:user/alice
  │
  ▼ aws-auth ConfigMap (또는 EKS Access Entry) 조회
  │  IAM ARN → K8S Username/Groups 매핑
  │
  ▼ K8S RBAC로 인가 결정
  │  Username/Groups에 바인딩된 Role/ClusterRole 확인
  │
  ▼ 요청 처리 또는 거부
```

### 3.2 aws-iam-authenticator와 토큰 구조

EKS 토큰은 일반 JWT가 아닌 **Base64 인코딩된 STS pre-signed URL**입니다.

```
EKS 토큰 디코딩:
  k8s-aws-v1.aHR0cHM6Ly9zdHMuYW1hem9uYXdzLmNvbS8/...

  prefix: k8s-aws-v1.
  body:   base64(https://sts.amazonaws.com/?Action=GetCallerIdentity
                 &X-Amz-Algorithm=AWS4-HMAC-SHA256
                 &X-Amz-Credential=...
                 &X-Amz-Date=...
                 &X-Amz-Expires=60  ← 60초 유효
                 &X-Amz-Security-Token=...
                 &X-Amz-SignedHeaders=host;x-k8s-aws-id
                 &X-Amz-Signature=...)
```

**토큰 만료**: STS pre-signed URL은 **15분** 후 만료됩니다. kubectl은 만료 전 자동으로 토큰을 재발급합니다.

### 3.3 aws-auth ConfigMap (구 방식)

EKS 클러스터 생성 시 `kube-system` 네임스페이스에 자동 생성되는 ConfigMap. IAM → K8S 매핑 테이블입니다.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: aws-auth
  namespace: kube-system
data:
  # IAM 역할 → K8S 매핑 (노드 그룹, CI/CD 시스템 등)
  mapRoles: |
    # 관리형 노드 그룹 (필수 - 없으면 노드가 클러스터에 합류 불가)
    - rolearn: arn:aws:iam::123456789:role/EKSNodeInstanceRole
      username: system:node:{{EC2PrivateDNSName}}
      groups:
        - system:bootstrappers
        - system:nodes

    # CI/CD 파이프라인 역할
    - rolearn: arn:aws:iam::123456789:role/GitHubActionsRole
      username: github-actions
      groups:
        - deploy-group

    # 다른 AWS 계정의 역할 (크로스 계정 접근)
    - rolearn: arn:aws:iam::987654321:role/CrossAccountRole
      username: cross-account-admin
      groups:
        - system:masters

  # IAM 사용자 → K8S 매핑 (직접 IAM 사용자 매핑)
  mapUsers: |
    - userarn: arn:aws:iam::123456789:user/alice
      username: alice
      groups:
        - dev-team
    - userarn: arn:aws:iam::123456789:user/bob
      username: bob
      groups:
        - system:masters   # cluster-admin 권한

  # 계정 수준 매핑 (잘 사용 안 함)
  mapAccounts: |
    - "123456789012"
```

**aws-auth ConfigMap의 구조적 문제점**

```
1. 단일 ConfigMap = 단일 장애점
   - 잘못 수정하면 클러스터 전체 접근 불가
   - 클러스터 생성자(cluster creator)만 복구 가능

2. YAML 형식 오류에 취약
   - 들여쓰기 실수 → 전체 매핑 무효화

3. GitOps 관리 어려움
   - ConfigMap 직접 수정 → 감사 추적 어려움

4. 확장성 제한
   - 대규모 조직에서 수백 개의 역할 관리 복잡
```

### 3.4 EKS Access Entry (신 방식, 2023년 출시)

aws-auth ConfigMap의 구조적 문제를 해결하기 위해 EKS가 도입한 **EKS 네이티브 IAM 접근 관리 방식**입니다.

```
Access Entry 구조:

AWS EKS API
  └─ Access Entry (IAM Principal → K8S 매핑)
      ├─ IAM Principal: arn:aws:iam::123456789:role/DeveloperRole
      ├─ K8S Username: developer-{{SessionName}}
      ├─ K8S Groups: ["dev-team", "view-only"]
      └─ Access Policy (선택): AmazonEKSClusterAdminPolicy 등
```

**Access Entry vs aws-auth ConfigMap**

|항목           |aws-auth ConfigMap|EKS Access Entry|
|-------------|------------------|----------------|
|관리 방식        |K8S ConfigMap 수정  |AWS API/콘솔/CLI  |
|장애 리스크       |수정 실수 시 잠금        |API 기반으로 안전     |
|IAM 감사       |어려움               |CloudTrail 자동 기록|
|Terraform/IaC|ConfigMap 관리 복잡   |AWS 리소스로 관리     |
|기본 활성화       |EKS 구버전 기본        |EKS 1.28+ 선택 가능 |
|Access Policy|없음 (K8S RBAC만)    |AWS 관리형 정책 제공   |

**AWS 관리형 Access Policy 종류**

|정책                           |권한 수준                |
|-----------------------------|---------------------|
|`AmazonEKSClusterAdminPolicy`|cluster-admin (전체 권한)|
|`AmazonEKSAdminPolicy`       |admin (네임스페이스 관리)    |
|`AmazonEKSEditPolicy`        |edit (읽기/쓰기)         |
|`AmazonEKSViewPolicy`        |view (읽기 전용)         |

### 3.5 EKS 클러스터 생성자(Cluster Creator)의 특별한 지위

EKS 클러스터를 생성한 IAM Identity는 **aws-auth ConfigMap과 무관하게 자동으로 cluster-admin 권한**을 가집니다. 이는 EKS 내부적으로 처리되며, ConfigMap에서 삭제해도 권한이 유지됩니다.

```
⚠️ 중요한 운영 함의:
  - 클러스터 생성 IAM 역할을 삭제하면 복구 수단 없음
  - Terraform으로 클러스터 생성 시 Terraform 실행 역할을 기록해둘 것
  - 조직 계정에서는 전용 클러스터 생성 역할을 지정하고 보존
```

### 3.6 EKS의 완전한 인증/인가 아키텍처

```
┌─────────────────────────────────────────────────────────────┐
│                     접근 주체                               │
│                                                             │
│  [인간 사용자]          [CI/CD 시스템]       [AWS 서비스]   │
│  개발자, 운영자         GitHub Actions       EC2, Lambda    │
│  IAM User/Role         IAM Role             IAM Role        │
└──────────────┬──────────────────┬──────────────┬───────────┘
               │                  │              │
               ▼                  ▼              ▼
┌─────────────────────────────────────────────────────────────┐
│              AWS IAM 인증 계층                              │
│                                                             │
│  aws eks get-token → STS pre-signed URL                    │
│           ↓                                                 │
│  aws-iam-authenticator (EKS Webhook) → IAM ARN 확인        │
│           ↓                                                 │
│  aws-auth ConfigMap / EKS Access Entry → K8S Identity 매핑 │
└──────────────────────────────┬──────────────────────────────┘
                               │ K8S Username / Groups
                               ▼
┌─────────────────────────────────────────────────────────────┐
│               K8S RBAC 인가 계층                            │
│                                                             │
│  Role / ClusterRole (권한 정의)                             │
│  RoleBinding / ClusterRoleBinding (주체 ↔ 권한 연결)       │
└──────────────────────────────┬──────────────────────────────┘
                               │ 허용/거부
                               ▼
                     K8S API 서버 처리
```

### 3.7 노드(EC2)의 K8S 인증

EKS 노드(EC2 인스턴스)는 kubelet이 부트스트랩 시 **IAM 역할로 EKS 토큰을 발급받아** K8S API 서버에 인증합니다.

```
EC2 인스턴스 시작
  │
  ▼ User Data 스크립트 실행
/etc/eks/bootstrap.sh my-cluster
  │
  ▼ IMDSv2에서 IAM 역할 자격증명 획득
http://169.254.169.254/latest/api/token
  │
  ▼ aws eks get-token으로 K8S 토큰 발급
  │  (노드 IAM 역할로 STS 서명)
  │
  ▼ api-server 인증
  │  aws-auth의 mapRoles에서 노드 역할 확인
  │  → username: system:node:{{EC2PrivateDNSName}}
  │  → groups: [system:bootstrappers, system:nodes]
  │
  ▼ kubelet이 system:nodes 그룹 권한으로 동작
  (Node Authorizer: 자신의 노드/파드 정보만 접근 가능)
```

-----

## 4. K8S(EKS) Pod(SA) with IAM Role → AWS 리소스 사용 (IRSA)

### 4.1 IRSA가 해결하는 문제

파드가 AWS 리소스(S3, DynamoDB, SQS 등)에 접근해야 할 때 안전한 자격증명을 제공하는 방법이 필요합니다.

**문제가 있는 접근 방법들**

```
❌ 방법 1: 환경 변수에 액세스 키 하드코딩
   env:
     - name: AWS_ACCESS_KEY_ID
       value: AKIAIOSFODNN7EXAMPLE
   → 비밀 노출, 키 관리 부담, 키 유출 시 광범위한 피해

❌ 방법 2: EC2 노드 IAM 역할에 직접 권한 부여
   노드 IAM Role에 S3FullAccess 부여
   → 해당 노드의 모든 파드가 모든 S3 버킷 접근 가능
   → 최소 권한 원칙 위반, 노드 탈취 시 전체 노출

❌ 방법 3: Kubernetes Secret으로 AWS 자격증명 저장
   kubectl create secret generic aws-creds \
     --from-literal=AWS_ACCESS_KEY_ID=...
   → 장기 자격증명, 갱신 부담, etcd 암호화 필요
```

**✅ IRSA: 올바른 방법**

```
파드의 ServiceAccount에 IAM Role을 연결하여
파드 단위로 최소 권한의 AWS 자격증명을 임시로 제공
```

### 4.2 IRSA 동작 원리 — 단계별 상세

IRSA는 **OpenID Connect(OIDC)** 프로토콜을 기반으로 동작합니다.

```
IRSA 전체 흐름:

[사전 설정]
EKS 클러스터 → OIDC Provider URL 자동 생성
  예: https://oidc.eks.ap-northeast-2.amazonaws.com/id/EXAMPLED539D4633E53DE1B716D3041E

AWS IAM에 OIDC Provider 등록
  → EKS가 발급한 토큰을 AWS가 신뢰하도록 설정

IAM Role 생성 (Trust Policy 핵심)
  {
    "Effect": "Allow",
    "Principal": {
      "Federated": "arn:aws:iam::123456789:oidc-provider/oidc.eks.ap-northeast-2.amazonaws.com/id/EXAMPLED"
    },
    "Action": "sts:AssumeRoleWithWebIdentity",
    "Condition": {
      "StringEquals": {
        "oidc.eks.../id/EXAMPLED:aud": "sts.amazonaws.com",
        "oidc.eks.../id/EXAMPLED:sub": "system:serviceaccount:default:my-service-account"
      }
    }
  }

ServiceAccount에 IAM Role ARN 어노테이션:
  eks.amazonaws.com/role-arn: arn:aws:iam::123456789:role/MyPodRole
```

```
[런타임 동작]

파드 생성
  │
  ▼ EKS Pod Identity Webhook (Mutating Webhook)
  │  ServiceAccount의 role-arn 어노테이션 감지
  │  → 파드에 환경변수 및 볼륨 자동 주입:
  │     AWS_ROLE_ARN=arn:aws:iam::123456789:role/MyPodRole
  │     AWS_WEB_IDENTITY_TOKEN_FILE=/var/run/secrets/eks.amazonaws.com/serviceaccount/token
  │     볼륨: Projected Service Account Token (audience: sts.amazonaws.com)
  │
  ▼ 파드 내 AWS SDK 초기화
  │  AWS_WEB_IDENTITY_TOKEN_FILE 환경변수 감지
  │  → WebIdentityCredentialsProvider 활성화
  │
  ▼ STS AssumeRoleWithWebIdentity 호출
  │  - WebIdentityToken: /var/run/secrets/eks.amazonaws.com/.../token 파일 내용
  │  - RoleArn: AWS_ROLE_ARN 환경변수
  │  - RoleSessionName: 자동 생성
  │
  ▼ IAM Trust Policy 검증
  │  - OIDC 토큰 서명 검증 (EKS OIDC Provider)
  │  - Condition 검증:
  │    aud = sts.amazonaws.com ✅
  │    sub = system:serviceaccount:default:my-service-account ✅
  │
  ▼ 임시 AWS 자격증명 반환 (TTL: 1시간)
  │  Access Key ID + Secret Access Key + Session Token
  │
  ▼ AWS API 호출 (S3, DynamoDB, SQS 등)
  │  해당 IAM Role의 Permission Policy 범위 내에서만 가능
  │
  ▼ 자격증명 만료 전 SDK가 자동 갱신
     (K8S가 Service Account Token도 자동 갱신)
```

### 4.3 OIDC Provider와 토큰 구조

EKS는 각 클러스터마다 고유한 **OIDC Issuer URL**을 제공합니다.

```
OIDC Discovery Endpoint:
  https://oidc.eks.ap-northeast-2.amazonaws.com/id/EXAMPLED/.well-known/openid-configuration

OIDC Public Key (JWKS):
  https://oidc.eks.ap-northeast-2.amazonaws.com/id/EXAMPLED/keys

EKS가 발급하는 ServiceAccount Token (JWT) 구조:
  Header:
    alg: RS256
    kid: <key-id>    ← OIDC JWKS에서 공개키 조회 가능

  Payload:
    iss: "https://oidc.eks.ap-northeast-2.amazonaws.com/id/EXAMPLED"
    sub: "system:serviceaccount:default:my-sa"   ← 네임스페이스:SA명
    aud: ["sts.amazonaws.com"]                   ← 대상 (IRSA용)
    exp: 1234567890                              ← 만료 시간
    iat: 1234564290

  Signature: RS256으로 EKS가 서명 (개인키 소유)
```

**AWS STS가 토큰 검증하는 방식:**

1. `iss` 클레임으로 OIDC Provider 식별
1. OIDC JWKS 엔드포인트에서 공개키 조회
1. JWT 서명 검증 (EKS의 개인키로 서명되었는지 확인)
1. `aud`, `sub` 클레임이 Trust Policy Condition과 일치하는지 확인

### 4.4 IRSA 설정 실전 가이드

**Step 1: EKS OIDC Provider 확인/등록**

```bash
# 클러스터 OIDC URL 확인
aws eks describe-cluster --name my-cluster \
  --query "cluster.identity.oidc.issuer" --output text

# IAM에 OIDC Provider 등록 (eksctl 사용 시 자동)
eksctl utils associate-iam-oidc-provider \
  --cluster my-cluster \
  --approve

# 수동 등록 시
OIDC_URL=$(aws eks describe-cluster --name my-cluster \
  --query "cluster.identity.oidc.issuer" --output text)
OIDC_ID=$(echo $OIDC_URL | cut -d '/' -f 5)

aws iam create-open-id-connect-provider \
  --url $OIDC_URL \
  --client-id-list sts.amazonaws.com \
  --thumbprint-list <thumbprint>
```

**Step 2: IAM Role 생성 (Trust Policy 포함)**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::123456789012:oidc-provider/oidc.eks.ap-northeast-2.amazonaws.com/id/EXAMPLED539D4633E53DE1B716D3041E"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "oidc.eks.ap-northeast-2.amazonaws.com/id/EXAMPLED539D4633E53DE1B716D3041E:aud": "sts.amazonaws.com",
          "oidc.eks.ap-northeast-2.amazonaws.com/id/EXAMPLED539D4633E53DE1B716D3041E:sub": "system:serviceaccount:production:s3-reader-sa"
        }
      }
    }
  ]
}
```

**Step 3: Permission Policy 연결**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::my-app-bucket",
        "arn:aws:s3:::my-app-bucket/*"
      ]
    }
  ]
}
```

**Step 4: ServiceAccount에 어노테이션 추가**

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: s3-reader-sa
  namespace: production
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::123456789012:role/S3ReaderRole
    # 선택 옵션: 토큰 만료 시간 조정 (기본 86400초 = 24시간)
    eks.amazonaws.com/token-expiration: "3600"
```

**Step 5: Deployment에서 ServiceAccount 참조**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
  namespace: production
spec:
  template:
    spec:
      serviceAccountName: s3-reader-sa    # IRSA 적용된 SA 참조
      containers:
        - name: app
          image: my-app:latest
          # AWS SDK가 환경변수를 자동으로 감지하므로 별도 설정 불필요
          # AWS_ROLE_ARN, AWS_WEB_IDENTITY_TOKEN_FILE이 자동 주입됨
```

### 4.5 eksctl을 사용한 IRSA 간소화

```bash
# eksctl로 IRSA 설정 한 번에 처리
eksctl create iamserviceaccount \
  --cluster my-cluster \
  --namespace production \
  --name s3-reader-sa \
  --attach-policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess \
  --approve \
  --override-existing-serviceaccounts

# 커스텀 정책과 함께 생성
eksctl create iamserviceaccount \
  --cluster my-cluster \
  --namespace production \
  --name app-sa \
  --attach-policy-arn arn:aws:iam::123456789:policy/MyCustomPolicy \
  --approve
```

### 4.6 IRSA 검증 방법

```bash
# 파드 내에서 주입된 환경변수 확인
kubectl exec -it my-pod -- env | grep AWS

# 예상 출력:
# AWS_ROLE_ARN=arn:aws:iam::123456789:role/S3ReaderRole
# AWS_WEB_IDENTITY_TOKEN_FILE=/var/run/secrets/eks.amazonaws.com/serviceaccount/token
# AWS_DEFAULT_REGION=ap-northeast-2

# 토큰 파일 확인 (JWT 디코딩)
kubectl exec -it my-pod -- cat /var/run/secrets/eks.amazonaws.com/serviceaccount/token | \
  cut -d. -f2 | base64 -d 2>/dev/null | python3 -m json.tool

# STS를 통해 실제 자격증명 확인
kubectl exec -it my-pod -- aws sts get-caller-identity

# 예상 출력:
# {
#   "UserId": "AROAIOSFODNN7EXAMPLE:botocore-session-...",
#   "Account": "123456789012",
#   "Arn": "arn:aws:sts::123456789012:assumed-role/S3ReaderRole/botocore-session-..."
# }
```

### 4.7 IRSA 보안 고려사항

**Condition 없는 Trust Policy의 위험성**

```json
// ❌ 위험: sub 조건 없이 전체 OIDC 허용
"Condition": {
  "StringEquals": {
    "oidc.eks.../id/EXAMPLED:aud": "sts.amazonaws.com"
  }
}
// → 클러스터의 모든 ServiceAccount가 이 역할 AssumeRole 가능

// ✅ 안전: 특정 SA만 허용
"Condition": {
  "StringEquals": {
    "oidc.eks.../id/EXAMPLED:aud": "sts.amazonaws.com",
    "oidc.eks.../id/EXAMPLED:sub": "system:serviceaccount:production:my-sa"
  }
}
```

**다중 네임스페이스 허용 시 StringLike 활용**

```json
// 특정 네임스페이스의 모든 SA 허용
"Condition": {
  "StringLike": {
    "oidc.eks.../id/EXAMPLED:sub": "system:serviceaccount:production:*"
  }
}
```

**자격증명 체인 이해**

```
파드 내 AWS SDK 자격증명 탐색 순서:
1. 환경변수 (AWS_ACCESS_KEY_ID 등) → 없으면
2. AWS_WEB_IDENTITY_TOKEN_FILE (IRSA) → 없으면
3. ~/.aws/credentials 파일 → 없으면
4. EC2 Instance Metadata (IMDS) → 없으면
5. ECS Task 메타데이터

IRSA가 있으면 EC2 노드 IAM 역할보다 우선 적용됨
```

-----

## 5. 심화: EKS Pod Identity (차세대 IRSA)

### 5.1 IRSA의 한계와 Pod Identity의 등장

IRSA는 강력하지만 다음 불편함이 있습니다.

|IRSA의 불편함                               |Pod Identity의 해결                |
|----------------------------------------|--------------------------------|
|클러스터마다 OIDC Provider 등록 필요              |EKS 관리형 OIDC, 별도 등록 불필요         |
|IAM Role Trust Policy에 클러스터별 OIDC URL 포함|Role을 클러스터 무관하게 재사용 가능          |
|동일 역할을 여러 클러스터에서 사용 시 Trust Policy 복잡   |Pod Identity Association으로 단순 매핑|
|Terraform으로 관리 시 OIDC ARN 하드코딩          |EKS API로 선언적 관리                 |

### 5.2 Pod Identity 동작 방식

```
EKS Pod Identity Agent (DaemonSet, kube-system)
  └─ 각 노드에서 실행
  └─ 파드의 SA → IAM Role 매핑 정보 보유

파드에서 IMDS 호출 (169.254.170.23)
  → Pod Identity Agent가 가로채어 처리
  → EKS API로 해당 SA의 연결된 IAM Role 조회
  → STS를 통해 임시 자격증명 발급
  → 파드에 자격증명 반환
```

### 5.3 Pod Identity Association 설정

```bash
# EKS Pod Identity Agent 설치 (Add-on)
aws eks create-addon \
  --cluster-name my-cluster \
  --addon-name eks-pod-identity-agent

# Pod Identity Association 생성
aws eks create-pod-identity-association \
  --cluster-name my-cluster \
  --namespace production \
  --service-account my-sa \
  --role-arn arn:aws:iam::123456789:role/MyPodRole
```

**IAM Role Trust Policy (IRSA보다 단순)**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "pods.eks.amazonaws.com"
      },
      "Action": [
        "sts:AssumeRole",
        "sts:TagSession"
      ]
    }
  ]
}
```

IRSA와 달리 클러스터별 OIDC URL 없이 `pods.eks.amazonaws.com` 서비스 주체만 사용합니다.

### 5.4 IRSA vs Pod Identity 선택 기준

|상황                     |권장                     |
|-----------------------|-----------------------|
|EKS 1.24 이하 클러스터       |IRSA (Pod Identity 미지원)|
|멀티 클러스터 환경 (동일 역할 재사용) |Pod Identity           |
|기존 IRSA 이미 사용 중        |유지 (마이그레이션 필요 없음)      |
|신규 클러스터 구성 (EKS 1.24+) |Pod Identity 권장        |
|Terraform으로 관리 (단순성 중요)|Pod Identity           |

-----

## 6. 면접 질문 — 3-Depth 구조

-----

### Q1. EKS에서 kubectl을 실행하면 어떤 인증 과정을 거쳐 API 서버에 접근하나요?

> **핵심 답변 방향**: kubeconfig의 exec credential plugin → aws eks get-token → STS pre-signed URL 생성 → aws-iam-authenticator가 STS 호출로 IAM Identity 확인 → aws-auth 매핑 → K8S RBAC 인가의 전체 흐름을 단계별로 설명해야 합니다.

**▶ 1차 꼬리질문**: EKS 토큰이 일반 JWT와 다른 이유는 무엇이며, 토큰의 유효 시간은 얼마나 되나요?

> EKS 토큰은 JWT가 아닌 STS pre-signed URL을 base64 인코딩한 값. GetCallerIdentity 요청에 AWS Signature V4가 포함된 URL. 유효 시간은 15분이며 kubectl이 만료 전 자동 재발급. STS를 통해 IAM Identity를 직접 검증하므로 별도 토큰 저장소 불필요.

**▶▶ 2차 꼬리질문**: aws-auth ConfigMap이 손상되어 클러스터에 아무도 접근할 수 없게 되었습니다. 어떻게 복구하나요?

> 클러스터 생성자(creator) IAM Identity는 aws-auth와 무관하게 cluster-admin 권한 보유. 클러스터 생성 시 사용한 IAM 역할/사용자로 전환 후 접근. 그 역할도 삭제됐다면 EKS API(콘솔/CLI)를 통해 Access Entry 추가 (aws-auth 우회). 방지: Access Entry 방식으로 전환하거나, 복수의 cluster-admin 계정 유지.

**▶▶▶ 3차 꼬리질문**: 다른 AWS 계정의 IAM 역할이 EKS 클러스터에 접근하도록 설정하는 방법과, 이때 발생하는 STS 호출 체인을 설명해 보세요.

> 계정 B의 역할이 계정 A의 EKS 접근: 계정 A의 aws-auth에 계정 B의 역할 ARN 추가. STS 호출 체인: 계정 B 사용자 → AssumeRole(계정 B 역할) → AssumeRole(계정 A 역할) 또는 계정 B 역할 직접 사용 → aws eks get-token → STS GetCallerIdentity 검증(계정 B ARN 반환) → aws-auth에서 계정 B ARN 매핑 조회.

-----

### Q2. IRSA(IAM Roles for Service Accounts)의 동작 원리를 OIDC 토큰 흐름 관점에서 설명하세요.

> **핵심 답변 방향**: EKS OIDC Provider가 SA 토큰을 발급 → 파드에 마운트 → SDK가 STS AssumeRoleWithWebIdentity 호출 → IAM Trust Policy의 OIDC Condition 검증 → 임시 자격증명 반환의 흐름을 설명해야 합니다.

**▶ 1차 꼬리질문**: IRSA Trust Policy에서 `sub` 조건을 설정하지 않으면 어떤 보안 위험이 있나요?

> sub 조건 없으면 해당 클러스터의 모든 SA가 역할 AssumeRole 가능. 공격자가 임의 파드에 악성 SA를 마운트하여 높은 권한의 IAM 역할 탈취 가능. 반드시 `system:serviceaccount:<namespace>:<sa-name>` 형식으로 특정 SA만 허용해야 함.

**▶▶ 2차 꼬리질문**: 파드 내 AWS SDK가 IRSA 자격증명을 자동으로 사용하는 원리와, EC2 노드 IAM 역할과의 우선순위는 어떻게 되나요?

> SDK의 Default Credential Chain: 환경변수 → Web Identity Token File(IRSA) → ~/.aws/credentials → IMDS(EC2 Role) 순. AWS_WEB_IDENTITY_TOKEN_FILE 환경변수가 있으면 IRSA가 IMDS(EC2 노드 역할)보다 우선. IRSA 설정된 파드는 노드 IAM 역할이 아닌 지정된 IAM 역할만 사용.

**▶▶▶ 3차 꼬리질문**: IRSA 자격증명 갱신이 실패할 경우 어떤 증상이 나타나며, 근본 원인을 어떻게 진단하나요?

> 증상: AWS SDK 호출 실패, `ExpiredTokenException` 또는 `InvalidIdentityToken`. 원인: SA 토큰 갱신 실패(kubelet 문제), OIDC Provider 공개키 만료, STS 엔드포인트 접근 불가(네트워크 정책). 진단: 파드의 토큰 파일 만료시간 확인(`jwt.io`), kubelet 로그 확인, STS 엔드포인트 연결 테스트(`curl https://sts.amazonaws.com`), CloudTrail에서 AssumeRoleWithWebIdentity 실패 이벤트 확인.

-----

### Q3. K8S RBAC에서 ClusterRole과 Role의 차이, 그리고 ClusterRoleBinding과 RoleBinding의 조합에 따른 권한 범위를 설명하세요.

> **핵심 답변 방향**: Role은 네임스페이스 스코프, ClusterRole은 클러스터 전체 스코프. RoleBinding은 특정 네임스페이스에만 바인딩, ClusterRoleBinding은 전체 클러스터에 바인딩. ClusterRole + RoleBinding 조합이 특히 중요합니다.

**▶ 1차 꼬리질문**: ClusterRole을 RoleBinding으로 바인딩하면 어떻게 되나요? 왜 이 조합을 쓰나요?

> ClusterRole은 클러스터 전체 리소스를 정의할 수 있지만, RoleBinding으로 바인딩하면 해당 네임스페이스 내에서만 ClusterRole의 권한이 적용됨. 이 조합의 이점: ClusterRole 하나를 여러 네임스페이스에 재사용 가능 → Role 중복 정의 방지. 예: `edit` ClusterRole을 팀별 네임스페이스에 RoleBinding으로 부여.

**▶▶ 2차 꼬리질문**: 서비스 어카운트에 과도한 RBAC 권한이 부여된 것을 어떻게 감사(audit)하나요?

> `kubectl auth can-i --list --as=system:serviceaccount:ns:sa-name`으로 SA 권한 목록 확인. `rbac-lookup` 도구로 특정 주체의 전체 권한 조회. K8S Audit Log 분석으로 실제 사용된 API 확인. Polaris, Fairwinds Insights로 과도한 권한 자동 탐지. 정기적 권한 검토 프로세스 수립.

**▶▶▶ 3차 꼬리질문**: Pod Security Admission(PSA)과 RBAC의 역할 차이를 설명하고, OPA/Gatekeeper 또는 Kyverno를 추가로 도입하는 이유는 무엇인가요?

> RBAC: K8S API 작업 수준 인가 (파드 생성 가능 여부). PSA: 파드 보안 기준 검증 (privileged 컨테이너 허용 여부, hostNetwork 등). OPA/Gatekeeper·Kyverno: RBAC·PSA로 표현할 수 없는 복잡한 조직 정책 구현. 예: 특정 이미지 레지스트리만 허용, 리소스 Request/Limit 필수, 특정 레이블 강제, 네이밍 규칙 적용. RBAC가 “누가” 무엇을 할 수 있는지를 정의한다면, PSA/OPA는 “무엇이” 허용되는지를 정의.

-----

### Q4. EKS에서 Audit Log를 통해 보안 이벤트를 탐지하는 방법을 설명하세요.

**▶ 1차 꼬리질문**: K8S Audit Log의 4가지 레벨(None, Metadata, Request, RequestResponse)을 언제 구분하여 사용하나요?

> None: 로그 기록 안 함 (헬스체크 등). Metadata: 요청 메타데이터만 (who, what, when) - 대부분 API에 적용. Request: 요청 본문 포함 - 민감 리소스 생성/수정 시. RequestResponse: 요청+응답 모두 - 보안 감사 필요 리소스 (Secret, ConfigMap). 응답까지 기록하면 로그 크기 급증하므로 꼭 필요한 리소스만 적용.

**▶▶ 2차 꼬리질문**: EKS에서 Audit Log를 CloudWatch Logs로 전송하고 이상 탐지하는 구체적인 방법은?

> EKS 컨트롤 플레인 로그(audit) 활성화 → CloudWatch Log Group 자동 생성. CloudWatch Metric Filter로 이상 패턴 탐지: `{ $.verb = "delete" && $.objectRef.resource = "secrets" }`. Metric Alarm으로 알림. 또는 CloudWatch Log Insights로 쿼리: `fields @timestamp, user.username, verb, objectRef.resource | filter verb in ["create","delete"] and objectRef.resource = "clusterrolebindings"`. Amazon GuardDuty for EKS를 활성화하면 머신러닝 기반 이상 탐지 자동화.

**▶▶▶ 3차 꼬리질문**: 파드 탈취(컨테이너 익스플로잇)가 발생했을 때 IRSA 자격증명이 노출될 수 있는 시나리오와 이를 탐지/완화하는 방법을 설명하세요.

> 시나리오: 공격자가 컨테이너 RCE를 통해 `/var/run/secrets/eks.amazonaws.com/serviceaccount/token` 파일 탈취 → 다른 환경에서 STS AssumeRoleWithWebIdentity 호출 시도. 완화: 토큰은 특정 audience(sts.amazonaws.com)에만 유효하고 1시간 후 만료됨. 하지만 1시간 내 악용 가능. 탐지: CloudTrail에서 비정상 Source IP의 AssumeRoleWithWebIdentity 이벤트 탐지. GuardDuty `CredentialAccess:Kubernetes/AnomalousBehavior` 탐지. 완화 심화: VPC Endpoint로 STS 트래픽 제한, IMDSv2 강제(hop limit=1), NetworkPolicy로 컨테이너 외부 통신 제한.
