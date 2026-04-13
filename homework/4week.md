# 4주차 - K8S(EKS) Identity and Access Management 심화 기술 가이드

## 목차

1. [필요 지식 — K8S 인증/인가 기초 개념](#1-필요-지식--k8s-인증인가-기초-개념)
1. [K8S 인증/인가 기초 실습 구조 이해](#2-k8s-인증인가-기초-실습-구조-이해)
1. [K8S(EKS) Identity and Access Management 전체 구조](#3-k8seks-identity-and-access-management-전체-구조)
1. [K8S(EKS) Pod(SA) with IAM Role → AWS 리소스 사용 (IRSA)](#4-k8seks-podsa-with-iam-role--aws-리소스-사용-irsa)
1. [심화: EKS Pod Identity — 차세대 IRSA](#5-심화-eks-pod-identity--차세대-irsa)

-----

## 1. 필요 지식 — K8S 인증/인가 기초 개념

### 1.1 인증(Authentication)과 인가(Authorization)의 구분

보안 시스템에서 가장 먼저 명확히 구분해야 하는 두 개념입니다.

|개념    |영어                    |핵심 질문             |K8S에서의 구현                          |
|------|----------------------|------------------|-----------------------------------|
|**인증**|Authentication (AuthN)|“당신은 누구인가?”       |X.509 인증서 / OIDC 토큰 / SA 토큰으로 신원 확인|
|**인가**|Authorization (AuthZ) |“당신은 무엇을 할 수 있는가?”|RBAC으로 리소스 접근 권한 결정                |

```
kubectl get pods 요청의 전체 처리 흐름:

클라이언트 (kubectl)
  │ HTTPS 요청 (인증서 또는 Bearer Token 포함)
  ▼
kube-apiserver
  ├─ [1] Authentication (인증)
  │       → "이 요청자는 누구인가?"
  │       → X.509 인증서 / Bearer Token / OIDC 토큰 검증
  │       → 실패 시: 401 Unauthorized
  │
  ├─ [2] Authorization (인가)
  │       → "이 행위가 허용되는가?"
  │       → RBAC: user/group이 pods GET 권한 보유 여부 확인
  │       → 실패 시: 403 Forbidden
  │
  ├─ [3] Admission Control
  │       → "이 요청이 정책에 부합하는가?"
  │       → Mutating Webhook (변환), Validating Webhook (검증)
  │       → OPA/Gatekeeper, Kyverno, PSA
  │
  └─ [4] 요청 처리 → etcd 조회/수정 후 응답
```

### 1.2 K8S API 서버의 인증 메커니즘 종류

Kubernetes는 다양한 인증 방식을 **플러그인 체인** 형태로 지원합니다. 여러 방식이 동시에 활성화될 수 있으며, **하나라도 성공하면 통과**합니다.

|인증 방식                    |설명                            |EKS 사용 여부                  |
|-------------------------|------------------------------|---------------------------|
|**X.509 클라이언트 인증서**      |CA가 서명한 인증서로 신원 증명            |✅ 노드/컴포넌트 인증 핵심            |
|**Static Bearer Token**  |정적 토큰 파일 기반                   |❌ 보안상 비권장                  |
|**Bootstrap Token**      |노드 초기 등록용 임시 토큰               |✅ 노드 부트스트랩 시               |
|**Service Account Token**|파드 내 자동 마운트 JWT               |✅ 파드 → API 서버 인증 기본        |
|**OIDC Token**           |외부 IdP(Identity Provider)의 JWT|✅ EKS 사용자 인증 핵심            |
|**Webhook Token**        |외부 webhook으로 토큰 검증 위임         |✅ EKS aws-iam-authenticator|
|**Authenticating Proxy** |프록시가 헤더로 신원 전달                |특수 환경                      |

### 1.3 K8S 접근 주체(Subject) 모델

Kubernetes는 자체 사용자 오브젝트가 없습니다. 사용자는 외부 시스템에서 관리됩니다.

```
K8S 접근 주체 (Subject)
  │
  ├─ [1] 일반 사용자 (Normal User)
  │       - K8S 내부 오브젝트 없음 (kubectl get users → 명령 없음)
  │       - 인증서 CN(Common Name) 또는 OIDC 토큰 sub 클레임으로 식별
  │       - kubectl을 사용하는 인간 사용자 / CI·CD 시스템
  │
  └─ [2] 서비스 어카운트 (ServiceAccount)
          - K8S 네임스페이스 오브젝트로 관리
          - 파드 내 프로세스가 API 서버와 통신 시 사용
          - 네임스페이스 생성 시 default SA 자동 생성
          - JWT 토큰 자동 발급 및 파드에 마운트
```

**Service Account 자동 마운트 구조**

```
파드 생성 시 /var/run/secrets/kubernetes.io/serviceaccount/ 자동 마운트
  ├── token      ← JWT 토큰 (K8S 1.21+는 Bound SA Token, 만료 있음)
  ├── ca.crt     ← 클러스터 CA 인증서 (API 서버 TLS 검증용)
  └── namespace  ← 현재 네임스페이스 이름

파드 내 프로세스 → API 서버 요청:
  Authorization: Bearer <token>
  (TLS: ca.crt로 API 서버 인증서 검증)
```

### 1.4 RBAC (Role-Based Access Control) 핵심 개념

K8S의 표준 인가(Authorization) 방식. 4가지 핵심 리소스로 구성됩니다.

```
┌────────────────────────────────────────────────────────────┐
│  [1] Role (네임스페이스 스코프)                              │
│  [2] ClusterRole (클러스터 전체 스코프)                      │
│      → "무엇을 할 수 있는가?" — 권한 집합 정의               │
│                                                            │
│  rules:                                                    │
│  - apiGroups: [""]           # 코어 API (Pod, Service)     │
│    resources: ["pods"]                                     │
│    verbs: ["get", "list", "watch"]                         │
│  - apiGroups: ["apps"]       # apps 그룹 (Deployment)      │
│    resources: ["deployments"]                              │
│    verbs: ["get", "list", "create", "update", "delete"]   │
└─────────────────────────────┬──────────────────────────────┘
                              │ 연결 (Binding)
                              ▼
┌────────────────────────────────────────────────────────────┐
│  [3] RoleBinding (네임스페이스 스코프)                       │
│  [4] ClusterRoleBinding (클러스터 전체 스코프)               │
│      → Subject(주체) ↔ Role/ClusterRole 연결               │
│                                                            │
│  subjects:                                                 │
│  - kind: User           # 일반 사용자                       │
│  - kind: Group          # 사용자 그룹                       │
│  - kind: ServiceAccount # 서비스 어카운트                   │
└────────────────────────────────────────────────────────────┘
```

**Role / ClusterRole / RoleBinding / ClusterRoleBinding 스코프 비교**

|구분                  |스코프     |관리 리소스                         |사용 사례         |
|--------------------|--------|-------------------------------|--------------|
|`Role`              |네임스페이스 내|Pod, Service, Deployment 등     |팀별 NS 권한      |
|`ClusterRole`       |클러스터 전체 |노드, PV, 네임스페이스 등 비네임스페이스 리소스 포함|클러스터 관리자, 모니터링|
|`RoleBinding`       |네임스페이스 내|Role 또는 ClusterRole 바인딩 가능     |특정 NS에만 권한 부여 |
|`ClusterRoleBinding`|클러스터 전체 |ClusterRole 바인딩                |전체 클러스터 권한    |


> **핵심 패턴**: `ClusterRole + RoleBinding` 조합 → ClusterRole(권한 재사용) + RoleBinding(네임스페이스 격리) 동시 달성

**K8S 내장 ClusterRole**

|ClusterRole        |권한 범위                               |사용 대상     |
|-------------------|------------------------------------|----------|
|`cluster-admin`    |모든 리소스 전체 권한 (super-user)           |클러스터 관리자  |
|`admin`            |네임스페이스 내 대부분 (Role 수정 포함)           |팀 관리자     |
|`edit`             |네임스페이스 내 읽기/쓰기 (Role/RoleBinding 제외)|개발자       |
|`view`             |네임스페이스 내 읽기 전용                      |모니터링, 감사  |
|`system:node`      |노드가 필요한 권한                          |kubelet   |
|`system:kube-proxy`|kube-proxy 필요 권한                    |kube-proxy|

**Verb 종류 전체**

|Verb              |HTTP               |설명                 |
|------------------|-------------------|-------------------|
|`get`             |GET                |단일 리소스 조회          |
|`list`            |GET (collection)   |목록 조회              |
|`watch`           |GET (watch)        |변경 이벤트 스트리밍 구독     |
|`create`          |POST               |생성                 |
|`update`          |PUT                |전체 수정              |
|`patch`           |PATCH              |부분 수정              |
|`delete`          |DELETE             |단일 삭제              |
|`deletecollection`|DELETE (collection)|일괄 삭제              |
|`exec`            |POST /exec         |컨테이너 내 명령 실행       |
|`impersonate`     |-                  |다른 사용자로 행동 (매우 위험) |
|`bind`            |-                  |RoleBinding 생성 권한  |
|`escalate`        |-                  |자신보다 높은 권한의 Role 부여|

### 1.5 기타 인가(Authorization) 모드

|인가 모드          |설명                        |EKS 사용      |
|---------------|--------------------------|------------|
|**RBAC**       |Role 기반 인가 (표준)           |✅ 기본 활성화    |
|**ABAC**       |속성 기반 인가 (파일 기반)          |❌ 관리 불편, 비권장|
|**Node**       |kubelet 전용 (자신의 노드/파드만 접근)|✅ 활성화       |
|**Webhook**    |외부 시스템에 인가 위임             |특수 환경       |
|**AlwaysAllow**|모든 요청 허용                  |❌ 테스트 전용    |

### 1.6 Admission Controller

인증/인가 후 요청을 가로채어 검증하거나 변환하는 플러그인 체계입니다.

```
처리 순서:
요청 → Authentication → Authorization
  │
  ▼ Mutating Admission (직렬 처리)
  │  - 요청 내용 변환/보강
  │  - 예: Istio 사이드카 주입, VPA Admission, 기본값 설정
  │
  ▼ Object 직렬화 검증
  │
  ▼ Validating Admission (병렬 처리)
  │  - 요청 허용/거부 결정 (하나라도 거부 시 전체 거부)
  │  - 예: OPA/Gatekeeper, Kyverno, Pod Security Admission (PSA)
  │
  ▼ etcd에 저장 및 응답
```

-----

## 📝 Ch.1 기술 면접 질문

### Q1. Kubernetes에서 Authentication과 Authorization의 차이를 설명하고, API 서버가 요청을 처리하는 전체 순서를 말씀해 주세요.

> **핵심 답변 방향**: AuthN은 “누구인가” (신원 확인), AuthZ는 “무엇을 할 수 있는가” (권한 확인). API 서버의 처리 순서는 Authentication → Authorization → Admission Control → etcd 처리 순으로 설명해야 합니다.

**▶ 1차 꼬리질문**: Kubernetes에서 Authentication이 성공했음에도 403이 반환되는 상황은 어떤 경우인가요?

> 인증(AuthN)은 성공했으나 인가(AuthZ)가 실패한 경우. 즉, 신원은 확인됐지만 요청한 리소스에 대한 RBAC 권한이 없는 상황. 예: 개발자가 인증서로 정상 인증되었으나, 해당 네임스페이스의 Deployment를 삭제하는 ClusterRole이 바인딩되어 있지 않은 경우.

**▶▶ 2차 꼬리질문**: Admission Controller가 Authentication, Authorization과 독립적으로 분리된 이유는 무엇인가요?

> RBAC은 “어떤 리소스를 조작할 수 있는가”를 정의하지만, “어떤 내용으로 조작해야 하는가”는 표현하기 어렵습니다. Admission Controller는 이를 보완합니다. 예를 들어 “Deployment 생성 권한이 있는 사람이라도, 반드시 리소스 Request/Limit이 설정된 Deployment만 생성 가능”과 같은 규칙은 Validating Webhook으로만 표현 가능합니다. 관심사를 분리함으로써 각 레이어가 명확한 책임을 가집니다.

**▶▶▶ 3차 꼬리질문**: `impersonate`, `bind`, `escalate` verb가 특히 위험한 이유와, 이를 이용한 권한 상승 공격 시나리오를 설명해 보세요.

> `impersonate`: 다른 사용자/그룹으로 API 호출 가능 → system:masters 그룹으로 impersonate하면 cluster-admin 탈취. `bind`: RoleBinding 생성 권한 → cluster-admin을 자신의 SA에 바인딩 → 클러스터 전체 권한 획득. `escalate`: 자신보다 높은 권한의 Role 부여 가능 → K8S는 기본적으로 escalation protection이 있지만 bind와 조합 시 우회 가능. 세 verb 모두 절대적으로 최소화하고 정기적으로 감사해야 합니다.

-----

### Q2. RBAC에서 ClusterRole과 Role의 차이, 그리고 ClusterRole + RoleBinding 조합이 필요한 이유를 설명해 주세요.

> **핵심 답변 방향**: Role은 네임스페이스 스코프, ClusterRole은 클러스터 전체 스코프. ClusterRole + RoleBinding 조합은 권한 재사용과 네임스페이스 격리를 동시에 달성하는 핵심 패턴입니다.

**▶ 1차 꼬리질문**: 개발팀이 여러 네임스페이스에서 Deployment 읽기/쓰기 권한이 필요할 때 가장 효율적인 RBAC 설계는 무엇인가요?

> ClusterRole 하나에 Deployment 읽기/쓰기 규칙을 정의합니다. 그리고 각 네임스페이스(dev-ns, staging-ns 등)에 해당 ClusterRole을 RoleBinding으로 연결합니다. ClusterRoleBinding을 사용하면 모든 네임스페이스에 접근 가능해져 과도한 권한이 됩니다. 이 패턴으로 Role 중복 정의를 방지하면서 네임스페이스 격리를 동시에 달성할 수 있습니다.

**▶▶ 2차 꼬리질문**: `resourceNames` 필드로 특정 ConfigMap만 접근을 허용하는 패턴의 한계는 무엇인가요?

> `resourceNames`는 `list`, `watch`와 함께 사용할 수 없습니다. `list` 요청은 리소스명 없이 컬렉션 전체를 요청하는 구조이므로 resourceNames 필터가 적용되지 않습니다. 따라서 `get`에만 resourceNames를 적용하고 `list`는 별도 rule로 허용하는 방식을 써야 하며, 이 경우 list에서는 모든 ConfigMap이 노출됩니다. 진정한 격리가 필요하다면 네임스페이스 자체를 분리하는 것이 더 안전합니다.

**▶▶▶ 3차 꼬리질문**: 수십 개의 팀이 하나의 EKS 클러스터를 공유할 때 RBAC과 Admission Controller를 어떻게 조합하여 팀 간 격리를 구현하겠습니까?

> RBAC 레이어: 팀별 네임스페이스 분리 + 팀별 IAM Role → K8S 그룹 매핑 + ClusterRole(edit/view)을 팀 그룹에 RoleBinding. Admission Controller 레이어(OPA/Kyverno): 팀 레이블 강제, 이미지 레지스트리 화이트리스트, 리소스 Request/Limit 필수화, 팀이 다른 NS의 리소스를 참조하는 행위 차단. Network Policy 레이어: 네임스페이스 간 기본 통신 차단, 명시적 허용만 허용. 세 레이어가 서로 다른 관심사를 담당하도록 설계합니다.

-----

## 2. K8S 인증/인가 기초 실습 구조 이해

### 2.1 kubeconfig 파일 구조 상세

`kubectl`이 API 서버와 통신하는 설정 파일로, 클러스터/사용자/컨텍스트 3가지 섹션으로 구성됩니다.

```yaml
apiVersion: v1
kind: Config

# [1] 클러스터 접속 정보
clusters:
  - name: my-eks-cluster
    cluster:
      server: https://XXXXXXXX.gr7.ap-northeast-2.eks.amazonaws.com
      # API 서버 TLS 인증서 검증용 CA
      certificate-authority-data: <base64 인코딩된 CA 인증서>

# [2] 사용자 자격증명
users:
  # EKS 방식: exec credential plugin으로 동적 토큰 생성
  - name: eks-admin
    user:
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
        # 토큰 만료(15분) 전 kubectl이 자동으로 재실행

  # X.509 인증서 방식 (일반 K8S 사용자)
  - name: alice
    user:
      client-certificate-data: <base64 클라이언트 인증서>
      client-key-data: <base64 클라이언트 개인키>

# [3] 클러스터 + 사용자 + 네임스페이스 조합
contexts:
  - name: eks-admin@my-eks-cluster
    context:
      cluster: my-eks-cluster
      user: eks-admin
      namespace: default

current-context: eks-admin@my-eks-cluster
```

```bash
# 자주 사용하는 kubeconfig 명령어
kubectl config get-contexts            # 전체 컨텍스트 목록
kubectl config current-context         # 현재 컨텍스트
kubectl config use-context <context>   # 컨텍스트 전환
kubectl config view --minify           # 현재 컨텍스트 설정만 출력

# 멀티 클러스터 kubeconfig 병합
KUBECONFIG=~/.kube/config-a:~/.kube/config-b \
  kubectl config view --flatten > ~/.kube/config
```

### 2.2 X.509 인증서 기반 사용자 인증

**인증서의 CN이 K8S 사용자명, O가 그룹명**으로 RBAC Subject에 매핑됩니다.

```bash
# Step 1. 개인키 생성
openssl genrsa -out alice.key 2048

# Step 2. CSR 생성 (CN=사용자명, O=그룹명)
openssl req -new -key alice.key -out alice.csr \
  -subj "/CN=alice/O=dev-team/O=frontend-team"

# Step 3. K8S CertificateSigningRequest 오브젝트 생성
cat <<EOF | kubectl apply -f -
apiVersion: certificates.k8s.io/v1
kind: CertificateSigningRequest
metadata:
  name: alice-csr
spec:
  request: $(cat alice.csr | base64 | tr -d '\n')
  signerName: kubernetes.io/kube-apiserver-client
  expirationSeconds: 86400
  usages:
    - client auth
EOF

# Step 4. 클러스터 관리자가 CSR 승인
kubectl certificate approve alice-csr

# Step 5. 서명된 인증서 추출
kubectl get csr alice-csr \
  -o jsonpath='{.status.certificate}' | base64 -d > alice.crt

# Step 6. kubeconfig에 등록
kubectl config set-credentials alice \
  --client-certificate=alice.crt \
  --client-key=alice.key

# Step 7. RBAC 권한 부여
kubectl create rolebinding alice-dev \
  --clusterrole=edit \
  --user=alice \
  --namespace=development
```

**인증서 CN/O → RBAC Subject 매핑**

```yaml
subjects:
  - kind: User
    name: alice           # CN=alice
    apiGroup: rbac.authorization.k8s.io
  - kind: Group
    name: dev-team        # O=dev-team
    apiGroup: rbac.authorization.k8s.io
  - kind: Group
    name: frontend-team   # O=frontend-team
    apiGroup: rbac.authorization.k8s.io
```

> **EKS에서의 한계**: EKS는 클러스터 CA에 직접 접근할 수 없으므로 X.509 사용자 인증서 방식보다 IAM 기반 인증(aws-auth / Access Entry)을 주로 사용합니다.

### 2.3 ServiceAccount 토큰 동작 원리

**K8S 1.21 이전 Legacy Token vs K8S 1.21+ Bound Token**

|항목          |Legacy Token     |Bound SA Token (1.21+)       |
|------------|-----------------|-----------------------------|
|저장 방식       |Secret 오브젝트 자동 생성|Projected Volume (Secret 미생성)|
|만료 시간       |없음 (영구 유효)       |기본 1시간, kubelet 자동 갱신        |
|audience 바인딩|없음               |특정 audience에만 유효             |
|파드 바인딩      |없음               |파드 삭제 시 토큰 무효화               |
|보안          |취약 (유출 시 무기한 악용) |강화 (단기 토큰, 자동 갱신)            |

**Projected SA Token 마운트 구조**

```yaml
# K8S가 자동으로 구성하는 Projected Volume
volumes:
  - name: kube-api-access
    projected:
      sources:
        - serviceAccountToken:
            path: token
            expirationSeconds: 3607     # 1시간 + 7초 갱신 여유
            audience: "https://kubernetes.default.svc"
        - configMap:
            name: kube-root-ca.crt      # CA 인증서
        - downwardAPI:
            items:
              - path: namespace
                fieldRef:
                  fieldPath: metadata.namespace
```

**JWT 구조 비교 (일반 SA 토큰 vs IRSA용 토큰)**

```
[일반 SA 토큰 페이로드]
{
  "iss": "https://kubernetes.default.svc",
  "sub": "system:serviceaccount:default:my-sa",
  "aud": ["https://kubernetes.default.svc"],
  "exp": 1234567890,
  "kubernetes.io": {
    "pod": { "name": "my-pod", "uid": "..." },
    "serviceaccount": { "name": "my-sa", "uid": "..." }
  }
}

[IRSA용 Projected SA 토큰 페이로드] — audience가 다름
{
  "iss": "https://oidc.eks.ap-northeast-2.amazonaws.com/id/EXAMPLED",
  "sub": "system:serviceaccount:production:my-sa",
  "aud": ["sts.amazonaws.com"],   ← STS만 이 토큰 사용 가능
  "exp": 1234567890
}
```

### 2.4 RBAC 설계 패턴 및 모범 사례

```yaml
# ❌ 절대 금지: cluster-admin 남발
roleRef:
  kind: ClusterRole
  name: cluster-admin   # 모든 권한 → 절대 부여하지 말 것

---
# ✅ 올바른 패턴: 필요한 권한만 명시
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
  # 특정 리소스 이름만 허용 (세밀한 제어)
  - apiGroups: [""]
    resources: ["configmaps"]
    resourceNames: ["app-config"]   # 이 ConfigMap만 접근
    verbs: ["get"]
```

**권한 감사(Audit) 명령어**

```bash
# 특정 사용자/SA의 권한 목록 확인
kubectl auth can-i get pods --as=alice -n production
kubectl auth can-i --list --as=system:serviceaccount:default:my-sa

# rbac-lookup 도구 (별도 설치)
kubectl rbac-lookup alice --output wide
kubectl rbac-lookup --kind ServiceAccount my-sa -n default

# 특정 동작이 가능한 주체 역추적
kubectl who-can create deployments -n production
```

-----

## 📝 Ch.2 기술 면접 질문

### Q1. kubeconfig 파일의 구조와, EKS에서 사용자 인증이 일반 Kubernetes와 다른 점을 설명해 주세요.

> **핵심 답변 방향**: kubeconfig의 clusters/users/contexts 3섹션 구조를 설명하고, EKS는 exec credential plugin으로 동적 AWS 토큰을 생성하는 반면 일반 K8S는 X.509 인증서나 정적 토큰을 사용한다는 차이를 설명해야 합니다.

**▶ 1차 꼬리질문**: exec credential plugin이 만료된 토큰을 어떻게 자동으로 갱신하는지 동작 원리를 설명해 주세요.

> kubectl은 exec credential plugin 실행 결과에 포함된 `expirationTimestamp`를 확인합니다. API 요청 시마다 토큰 만료 여부를 검사하고, 만료 5분 전 또는 이미 만료된 경우 exec 명령(`aws eks get-token`)을 다시 실행하여 새 토큰을 획득합니다. 이 과정은 사용자에게 투명하게 처리됩니다.

**▶▶ 2차 꼬리질문**: X.509 인증서로 K8S 사용자를 발급할 때 인증서를 취소(revoke)하려면 어떻게 하나요?

> Kubernetes는 CRL(Certificate Revocation List)이나 OCSP를 기본 지원하지 않습니다. 인증서를 취소하려면: 1) 클러스터 CA를 교체(CA Rotation)하는 가장 확실한 방법이 있으나 모든 컴포넌트 재인증이 필요합니다. 2) 현실적인 방법은 해당 사용자의 RBAC 권한을 모두 제거하여 인증은 되지만 인가에서 차단되도록 합니다. 3) 이것이 EKS에서 X.509 대신 IAM 기반 인증을 선호하는 이유 중 하나입니다 — IAM 역할을 비활성화하면 즉시 접근 차단이 가능합니다.

**▶▶▶ 3차 꼬리질문**: Kubernetes 1.21+의 Bound Service Account Token이 기존 Legacy Token보다 보안적으로 개선된 점과, 이것이 IRSA 동작에 어떤 영향을 미치는지 설명해 주세요.

> Legacy Token은 만료가 없어 유출 시 무기한 악용 가능하고 Secret으로 etcd에 영구 저장됩니다. Bound Token은 1시간 만료 + audience 바인딩 + 파드 생명주기 바인딩으로 공격 노출 시간을 대폭 줄였습니다. IRSA 관점에서: IRSA는 `sts.amazonaws.com`을 audience로 하는 별도 Projected Token을 사용합니다. Bound Token의 audience 바인딩 덕분에 이 토큰은 STS 이외의 서비스에서 재사용할 수 없어, 탈취되더라도 K8S API 서버 인증에 사용할 수 없습니다.

-----

### Q2. Kubernetes ServiceAccount의 `automountServiceAccountToken: false` 설정은 언제 사용하며, 왜 보안적으로 중요한가요?

**▶ 1차 꼬리질문**: 파드 레벨과 ServiceAccount 레벨에서 모두 automountServiceAccountToken을 설정할 수 있는데, 우선순위는 어떻게 되나요?

> **파드 레벨이 ServiceAccount 레벨보다 우선**합니다. ServiceAccount에 `automountServiceAccountToken: false`가 설정되어 있어도 파드 스펙에 `automountServiceAccountToken: true`가 있으면 마운트됩니다. 반대도 성립합니다. 따라서 SA 레벨에서 기본적으로 비활성화하고 필요한 파드에서만 명시적으로 활성화하는 전략이 보안상 권장됩니다.

**▶▶ 2차 꼬리질문**: API 서버에 접근할 필요가 없는 파드(예: Nginx 웹서버)에 SA 토큰이 마운트되어 있으면 어떤 위험이 있나요?

> 컨테이너가 침해될 경우, 공격자가 마운트된 SA 토큰을 사용하여 K8S API 서버에 직접 접근할 수 있습니다. default SA는 기본적으로 제한된 권한이지만, 명시적으로 RBAC이 추가되었거나 취약한 권한이 있다면 클러스터 내 정보 수집, 다른 파드 조작, 환경 정보 탈취 등이 가능합니다. 최소 권한 원칙: API 서버 접근이 불필요한 파드는 반드시 `automountServiceAccountToken: false` 설정을 권장합니다.

**▶▶▶ 3차 꼬리질문**: Kubernetes에서 ServiceAccount 기반 공격을 탐지하고 방어하는 실무적인 방법들을 설명해 주세요.

> **탐지**: K8S Audit Log에서 SA 토큰으로 발생하는 비정상 API 호출 패턴 모니터링(예: 파드 내부에서 kubectl 실행, 다른 NS 리소스 조회 등). Falco 같은 런타임 보안 도구로 컨테이너 내 의심 프로세스 탐지. **방어**: automountServiceAccountToken 최소화, SA별 최소 권한 RBAC, NetworkPolicy로 파드 → kube-apiserver 직접 접근 차단(필요한 파드만 허용), OPA로 SA 어노테이션 없이는 특정 권한 SA 사용 불가 정책 적용.

-----

## 3. K8S(EKS) Identity and Access Management 전체 구조

### 3.1 EKS 인증의 고유성 — AWS IAM + K8S RBAC 통합

EKS는 K8S 표준 인증 외에 **AWS IAM을 K8S 인증과 통합**한 독자적인 구조를 가집니다.

```
EKS 인증 전체 흐름:

kubectl get pods
  │
  ▼ [1] kubeconfig exec credential plugin 실행
  aws eks get-token --cluster-name my-cluster
  │
  ▼ [2] AWS STS GetCallerIdentity pre-signed URL 생성
  │  현재 IAM 자격증명(IAM User/Role)으로 서명
  │  Bearer Token = "k8s-aws-v1." + base64(pre-signed URL)
  │
  ▼ [3] EKS API 서버로 Bearer Token 전달
  │  Authorization: Bearer k8s-aws-v1.aHR0cHM6...
  │
  ▼ [4] EKS 컨트롤 플레인 — aws-iam-authenticator (Webhook)
  │  pre-signed URL 실제 호출 → STS GetCallerIdentity 검증
  │  응답: { "Arn": "arn:aws:iam::123456789:user/alice" }
  │
  ▼ [5] aws-auth ConfigMap 또는 EKS Access Entry 조회
  │  IAM ARN → K8S Username / Groups 매핑
  │  예: arn:aws:iam::123:user/alice → username: alice, groups: [dev-team]
  │
  ▼ [6] K8S RBAC로 인가 결정
  │  alice 또는 dev-team에 바인딩된 Role/ClusterRole 확인
  │
  ▼ [7] 요청 처리 또는 거부 (200 OK / 403 Forbidden)
```

### 3.2 EKS 토큰 구조 — STS Pre-signed URL

EKS 토큰은 일반 JWT가 아닌 **Base64 인코딩된 STS pre-signed URL**입니다.

```
토큰 구조:
  k8s-aws-v1.aHR0cHM6Ly9zdHMuYW1hem9uYXdzLmNvbS8/...
  └──────────┘└──────────────────────────────────────┘
    고정 prefix  base64(STS GetCallerIdentity pre-signed URL)

pre-signed URL 포함 내용:
  https://sts.amazonaws.com/
    ?Action=GetCallerIdentity
    &X-Amz-Algorithm=AWS4-HMAC-SHA256
    &X-Amz-Credential=...
    &X-Amz-Date=...
    &X-Amz-Expires=60              ← 60초 유효 (STS 제한)
    &X-Amz-Security-Token=...      ← IAM Role AssumeRole 시 포함
    &X-Amz-SignedHeaders=host;x-k8s-aws-id
    &X-k8s-aws-id=my-cluster       ← 클러스터 식별자 (보안 핵심)
    &X-Amz-Signature=...

EKS 토큰 유효 시간: 15분 (kubectl이 만료 전 자동 재발급)
```

> **X-k8s-aws-id 헤더의 중요성**: 이 헤더가 없으면 A 클러스터용 토큰으로 B 클러스터 API 서버 접근이 가능합니다. aws-iam-authenticator는 이 헤더 값이 자신의 클러스터명과 일치하는지 반드시 검증합니다. 이 헤더는 SignedHeaders에 포함되어 AWS Signature V4로 보호되므로 위조가 불가능합니다.

### 3.3 aws-auth ConfigMap 상세

EKS 클러스터의 `kube-system` 네임스페이스에 자동 생성되는 ConfigMap. **IAM → K8S 매핑 테이블** 역할을 합니다.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: aws-auth
  namespace: kube-system
data:
  mapRoles: |
    # ① 필수: 관리형 노드 그룹 (없으면 노드 클러스터 합류 불가)
    - rolearn: arn:aws:iam::123456789:role/EKSNodeInstanceRole
      username: system:node:{{EC2PrivateDNSName}}
      groups:
        - system:bootstrappers
        - system:nodes

    # ② CI/CD 파이프라인 역할
    - rolearn: arn:aws:iam::123456789:role/GitHubActionsRole
      username: github-actions
      groups:
        - deploy-group

    # ③ 크로스 계정 역할 (다른 AWS 계정에서의 접근)
    - rolearn: arn:aws:iam::987654321:role/CrossAccountAdminRole
      username: cross-account-admin
      groups:
        - system:masters

    # ④ Karpenter 노드 역할
    - rolearn: arn:aws:iam::123456789:role/KarpenterNodeRole
      username: system:node:{{EC2PrivateDNSName}}
      groups:
        - system:bootstrappers
        - system:nodes

  mapUsers: |
    - userarn: arn:aws:iam::123456789:user/alice
      username: alice
      groups:
        - dev-team
    - userarn: arn:aws:iam::123456789:user/bob
      username: bob
      groups:
        - system:masters    # cluster-admin 권한
```

**aws-auth ConfigMap의 구조적 문제점**

```
문제 1. 단일 ConfigMap = 단일 장애점 (SPOF)
  → YAML 들여쓰기 실수 한 줄로 클러스터 전체 접근 불가
  → 복구: 클러스터 생성자 IAM Identity로만 가능

문제 2. 감사(Audit) 추적 어려움
  → "누가 어떤 IAM ARN을 언제 추가했는지" 명확한 기록 없음
  → K8S Audit Log에 ConfigMap 수정 기록은 남지만 세분화 어려움

문제 3. GitOps 관리 시 충돌 위험
  → ArgoCD 등으로 ConfigMap 관리 시 의도치 않은 덮어쓰기 위험

문제 4. 대규모 조직 확장성 한계
  → 수백 개 역할 관리 시 단일 ConfigMap으로 관리 복잡
```

### 3.4 EKS Access Entry — 차세대 IAM 접근 관리 (2023년 출시)

aws-auth ConfigMap의 문제를 해결하는 **EKS 네이티브 IAM 접근 관리 방식**입니다.

**비교표**

|항목            |aws-auth ConfigMap         |EKS Access Entry                |
|--------------|---------------------------|--------------------------------|
|관리 방식         |K8S ConfigMap 직접 수정        |AWS API / 콘솔 / CLI / Terraform  |
|잠금 리스크        |수정 실수 시 전체 잠금              |AWS API로 안전하게 수정                |
|IAM 감사        |K8S Audit Log              |CloudTrail 자동 기록                |
|Terraform 관리  |`kubernetes_config_map` 리소스|`aws_eks_access_entry` 리소스      |
|AWS 관리형 Policy|없음 (K8S RBAC만)             |AmazonEKSClusterAdminPolicy 등 제공|
|최소 EKS 버전     |모든 버전                      |EKS 1.23+ (정식 1.28+)            |

```bash
# Access Entry 생성
aws eks create-access-entry \
  --cluster-name my-cluster \
  --principal-arn arn:aws:iam::123456789:role/DeveloperRole \
  --kubernetes-groups dev-team \
  --username developer

# AWS 관리형 Policy 연결
aws eks associate-access-policy \
  --cluster-name my-cluster \
  --principal-arn arn:aws:iam::123456789:role/AdminRole \
  --policy-arn arn:aws:eks::aws:cluster-access-policy/AmazonEKSClusterAdminPolicy \
  --access-scope type=cluster

# 노드 역할 등록 (EC2_LINUX 타입으로 그룹 자동 설정)
aws eks create-access-entry \
  --cluster-name my-cluster \
  --principal-arn arn:aws:iam::123456789:role/NodeInstanceRole \
  --type EC2_LINUX
```

**AWS 관리형 Access Policy**

|정책                           |K8S 동등 권한    |
|-----------------------------|-------------|
|`AmazonEKSClusterAdminPolicy`|cluster-admin|
|`AmazonEKSAdminPolicy`       |admin        |
|`AmazonEKSEditPolicy`        |edit         |
|`AmazonEKSViewPolicy`        |view         |

### 3.5 EKS 클러스터 생성자(Cluster Creator)의 특별한 지위

```
⚠️ EKS 클러스터 생성자 = 영구 cluster-admin

EKS 클러스터를 생성한 IAM Identity는
aws-auth ConfigMap / Access Entry와 무관하게
EKS 내부(컨트롤 플레인)에서 자동으로 cluster-admin 권한 보유

→ ConfigMap에서 삭제해도 권한 유지됨
→ ConfigMap이 완전히 손상되어도 이 Identity로 복구 가능

운영 함의:
  ① 클러스터 생성 IAM 역할을 절대 삭제하지 말 것
  ② Terraform으로 생성 시 실행 역할을 반드시 문서화
  ③ aws-auth 손상 시 복구:
     → 클러스터 생성자 IAM으로 전환 후 ConfigMap 재설정
     → 또는 EKS Access Entry API로 새 관리자 추가 (ConfigMap 우회)
```

### 3.6 노드(EC2)의 K8S 인증 흐름

```
EC2 인스턴스 시작 (Auto Scaling Group / Karpenter)
  │
  ▼ User Data: /etc/eks/bootstrap.sh my-cluster
  │
  ▼ IMDSv2에서 노드 IAM 역할 자격증명 획득
  curl -H "X-aws-ec2-metadata-token: $TOKEN" \
    http://169.254.169.254/latest/meta-data/iam/security-credentials/NodeRole
  │
  ▼ aws eks get-token으로 K8S Bootstrap Token 발급
  │
  ▼ kube-apiserver 인증
  │  aws-auth mapRoles에서 NodeInstanceRole 확인
  │  → username: system:node:ip-192-168-1-10.ap-northeast-2.compute.internal
  │  → groups: [system:bootstrappers, system:nodes]
  │
  ▼ Node Authorizer (특수 인가 모드)
  │  system:nodes 그룹: 자신이 실행하는 파드/노드 정보만 접근 가능
  │  다른 노드 정보 접근 불가 (Node Isolation 보장)
  │
  ▼ kubelet 정상 동작 → 파드 스케줄링 수신
```

### 3.7 EKS 완전한 IAM 아키텍처 다이어그램

```
┌───────────────────────────────────────────────────────────────┐
│                          접근 주체                             │
│  [인간 사용자]      [CI/CD]        [파드]          [노드]      │
│  IAM User/Role      IAM Role       K8S SA Token    IAM Role   │
└──────┬──────────────────┬──────────────┬──────────────┬───────┘
       │                  │              │              │
       ▼                  ▼              │              ▼
┌─────────────────────────────────┐      │    ┌──────────────────┐
│      AWS IAM 인증 계층           │      │    │  K8S 내부 인증   │
│  aws eks get-token               │      │    │  Projected SA    │
│  → STS pre-signed URL            │      │    │  Token           │
│  → aws-iam-authenticator 검증    │      │    │  → API 서버 직접 │
│  → IAM ARN 확정                  │      │    │    인증          │
│  aws-auth / Access Entry         │      │    └──────┬───────────┘
│  → K8S Username/Groups 매핑      │      │           │
└──────────────┬──────────────────┘      │           │
               └─────────────────────────┴───────────┘
                                         │
                                         ▼
                         ┌───────────────────────────────┐
                         │       K8S RBAC 인가 계층       │
                         │  Role / ClusterRole            │
                         │  RoleBinding / CRoleBinding    │
                         │  → 허용 / 거부                  │
                         └───────────────────────────────┘
```

-----

## 📝 Ch.3 기술 면접 질문

### Q1. EKS에서 kubectl을 실행하면 어떤 인증 과정을 거쳐 API 서버에 접근하나요? 단계별로 설명해 주세요.

> **핵심 답변 방향**: kubeconfig exec credential plugin 실행 → aws eks get-token → STS pre-signed URL 생성 → aws-iam-authenticator가 STS 호출로 IAM Identity 확인 → aws-auth 매핑 → K8S RBAC 인가의 전체 흐름을 단계별로 설명해야 합니다.

**▶ 1차 꼬리질문**: EKS 토큰이 일반 JWT와 구조적으로 다른 이유와, 유효 시간이 15분으로 제한되는 이유는 무엇인가요?

> EKS 토큰은 JWT가 아닌 STS GetCallerIdentity pre-signed URL을 base64 인코딩한 값입니다. 이 방식을 선택한 이유: 1) K8S가 별도 토큰 저장소를 유지할 필요 없이 STS가 실시간 검증 서버 역할을 합니다. 2) IAM Identity의 변경(역할 삭제, 권한 회수 등)이 즉시 반영됩니다. 15분 제한은 STS pre-signed URL의 최대 유효 시간(Expires 파라미터)이 AWS 제약상 최대 60초이고, EKS 토큰 자체의 유효 시간이 15분으로 설계된 것입니다.

**▶▶ 2차 꼬리질문**: aws-auth ConfigMap이 완전히 손상되어 클러스터에 아무도 접근할 수 없게 되었습니다. 어떻게 복구하나요?

> 클러스터 생성자(creator) IAM Identity는 aws-auth와 무관하게 EKS 내부에서 cluster-admin 권한을 보유합니다. 복구 순서: 1) 클러스터 생성 시 사용한 IAM 역할/사용자로 AWS 자격증명을 전환합니다. 2) `aws eks update-kubeconfig`로 kubeconfig를 갱신합니다. 3) 손상된 aws-auth ConfigMap을 재작성합니다. 클러스터 생성 역할까지 삭제됐다면: EKS Access Entry API를 통해 ConfigMap 우회하여 새 관리자를 추가하거나, AWS Support에 연락해야 합니다.

**▶▶▶ 3차 꼬리질문**: X-k8s-aws-id 헤더가 없으면 어떤 보안 공격이 가능하고, EKS는 이를 어떻게 방지하나요? 또한 크로스 계정에서 EKS 접근을 설정할 때 보안상 주의해야 할 점은 무엇인가요?

> X-k8s-aws-id 없이는: A 클러스터용 EKS 토큰(STS pre-signed URL)으로 B 클러스터의 API 서버에 접근 가능합니다. STS GetCallerIdentity 자체는 클러스터를 식별하지 않기 때문입니다. 방지: aws-iam-authenticator가 이 헤더 값이 자신의 클러스터명과 일치하는지 검증하며, 이 헤더는 AWS Signature V4 SignedHeaders에 포함되어 위조 불가합니다. 크로스 계정 주의사항: aws-auth에 타 계정 역할 ARN을 추가할 때 해당 역할에 system:masters를 부여하면 전체 클러스터 권한이 주어지므로, 최소 권한 원칙에 따라 필요한 K8S 그룹만 매핑해야 합니다.

-----

### Q2. aws-auth ConfigMap과 EKS Access Entry의 차이와 각각의 장단점, 실무에서 어떤 방식을 선택하겠습니까?

**▶ 1차 꼬리질문**: aws-auth mapRoles의 username 필드에 `{{EC2PrivateDNSName}}` 템플릿 변수를 사용하는 이유는 무엇인가요?

> K8S Node Authorizer는 `system:node:<nodename>` 형식의 username을 기반으로 노드별 접근 제어를 수행합니다. 모든 노드가 동일한 고정 username을 사용하면 A 노드가 B 노드의 파드 정보에 접근할 수 있어 Node Isolation이 깨집니다. `{{EC2PrivateDNSName}}`으로 각 노드의 실제 hostname을 username으로 사용하면, Node Authorizer가 “이 kubelet은 자신의 노드 정보만 조회 가능”하도록 격리를 보장합니다.

**▶▶ 2차 꼬리질문**: EKS Access Entry의 EC2_LINUX 타입을 사용하면 mapRoles 설정과 비교해 어떤 이점이 있나요?

> EC2_LINUX 타입: EKS가 내부적으로 system:bootstrappers, system:nodes 그룹을 자동 할당하고 EC2 DNS 기반 username을 자동 생성합니다. 이점: 1) ConfigMap 직접 수정 없이 AWS API로 선언적 관리. 2) CloudTrail에 노드 등록 이력 자동 기록 (감사 추적 용이). 3) Terraform의 `aws_eks_access_entry` 리소스로 IaC 관리. 4) 노드 역할 변경 시 ConfigMap 수동 업데이트 불필요.

**▶▶▶ 3차 꼬리질문**: 대규모 조직에서 수십 개 팀이 하나의 EKS 클러스터를 공유할 때 IAM Role → K8S 접근 권한 체계를 어떻게 설계하겠습니까?

> 설계 레이어: 1) IAM 역할: 팀별 역할 (TeamA-EKS, TeamB-EKS) + CI/CD 전용 역할 + 읽기 전용 모니터링 역할 분리. 2) Access Entry: 팀 역할 → K8S 그룹 매핑 (teamA-group, teamB-group, read-only-group). 3) K8S RBAC: ClusterRole(edit/view)을 팀 그룹에 팀 NS로 RoleBinding. 시스템 NS(kube-system, monitoring)는 별도 ops-group만 접근. 4) 거버넌스: Kyverno/OPA로 팀이 다른 NS 참조 금지, AWS SCPs로 팀이 클러스터 IAM 역할 변조 불가 설정. 5) Access Entry를 선택하는 이유: 팀별 역할이 수십 개일 때 ConfigMap 관리는 SPOF 위험이 너무 크기 때문.

-----

## 4. K8S(EKS) Pod(SA) with IAM Role → AWS 리소스 사용 (IRSA)

### 4.1 IRSA가 해결하는 문제

파드가 S3, DynamoDB, SQS, Secrets Manager 등 AWS 리소스에 접근할 때 안전한 자격증명이 필요합니다.

**잘못된 접근 방식과 문제점**

```
❌ 방법 1: 환경변수/K8S Secret에 액세스 키 하드코딩
  env:
    - name: AWS_ACCESS_KEY_ID
      value: AKIAIOSFODNN7EXAMPLE
  문제:
    - 장기 자격증명 → 유출 시 무기한 악용 가능
    - 키 교체 시 파드 재배포 필요
    - etcd에 저장 (암호화 없으면 평문 노출)

❌ 방법 2: EC2 노드 IAM 역할에 광범위한 권한 부여
  NodeInstanceRole에 S3FullAccess + DynamoDBFullAccess 부여
  문제:
    - 해당 노드의 모든 파드가 동일 권한 공유
    - 최소 권한 원칙 완전 위반
    - 노드 탈취 시 전체 AWS 리소스 노출

❌ 방법 3: 파드 내 STS AssumeRole 직접 구현
  문제:
    - 초기 자격증명(AssumeRole 권한)을 어디서 획득?
    - 결국 노드 역할에 의존 → 방법 2와 동일한 문제

✅ IRSA (IAM Roles for Service Accounts):
  파드의 K8S ServiceAccount에 IAM Role을 직접 연결
  → 파드 단위로 격리된 최소 권한의 임시 자격증명 자동 제공
  → 노드 IAM 역할 불변, 다른 파드에 영향 없음
  → 1시간 만료 임시 자격증명으로 자동 갱신
```

### 4.2 OIDC 기반 IRSA 동작 원리 — 완전한 흐름

**사전 설정 단계**

```
1. EKS 클러스터 생성 시 OIDC Provider URL 자동 발급
   예: https://oidc.eks.ap-northeast-2.amazonaws.com/id/EXAMPLED539D4633

2. AWS IAM에 OIDC Identity Provider 등록
   → "이 EKS 클러스터가 서명한 JWT를 AWS가 신뢰" 설정
   → Thumbprint = OIDC 엔드포인트 TLS 인증서 지문

3. IAM Role Trust Policy 설정 (핵심):
   {
     "Effect": "Allow",
     "Principal": {
       "Federated": "arn:aws:iam::123:oidc-provider/oidc.eks.../id/EXAMPLED"
     },
     "Action": "sts:AssumeRoleWithWebIdentity",
     "Condition": {
       "StringEquals": {
         "oidc.eks.../id/EXAMPLED:aud": "sts.amazonaws.com",
         "oidc.eks.../id/EXAMPLED:sub":
           "system:serviceaccount:production:my-sa"  ← 반드시 설정
       }
     }
   }

4. ServiceAccount에 IAM Role ARN 어노테이션:
   annotations:
     eks.amazonaws.com/role-arn: arn:aws:iam::123456789:role/MyPodRole
```

**런타임 동작 단계**

```
파드 생성 (serviceAccountName: my-sa)
  │
  ▼ EKS Pod Identity Webhook (Mutating Admission Webhook)
  │  SA의 eks.amazonaws.com/role-arn 어노테이션 감지
  │  파드에 자동 주입:
  │    환경변수:
  │      AWS_ROLE_ARN=arn:aws:iam::123456789:role/MyPodRole
  │      AWS_WEB_IDENTITY_TOKEN_FILE=/.../token
  │      AWS_DEFAULT_REGION=ap-northeast-2
  │    볼륨: IRSA용 Projected SA Token (aud: sts.amazonaws.com)
  │
  ▼ AWS SDK 초기화 (Default Credential Provider Chain)
  │  AWS_WEB_IDENTITY_TOKEN_FILE 감지
  │  → WebIdentityCredentialsProvider 활성화
  │
  ▼ STS AssumeRoleWithWebIdentity 호출
  │  - WebIdentityToken: 파일에서 JWT 토큰 읽기
  │  - RoleArn: AWS_ROLE_ARN 환경변수
  │  - RoleSessionName: 자동 생성
  │
  ▼ AWS STS 검증 프로세스
  │  1. JWT iss 클레임 → OIDC Provider 식별
  │  2. OIDC JWKS 엔드포인트에서 공개키 조회
  │  3. JWT 서명 검증 (EKS 개인키로 서명 확인)
  │  4. Trust Policy Condition 검증:
  │     aud = "sts.amazonaws.com" ✅
  │     sub = "system:serviceaccount:production:my-sa" ✅
  │
  ▼ 임시 AWS 자격증명 반환 (TTL: 1시간)
  │  AccessKeyId + SecretAccessKey + SessionToken
  │
  ▼ AWS API 호출 (S3, DynamoDB, SQS 등)
  │  해당 IAM Role Permission Policy 범위 내에서만 가능
  │
  ▼ SDK가 만료 전 자동 갱신 (kubelet도 SA Token 자동 갱신)
```

### 4.3 OIDC Token 구조 심화

```
IRSA용 Projected SA Token (JWT) 상세 구조:

Header:
  {
    "alg": "RS256",
    "kid": "abc123..."    ← EKS OIDC JWKS에서 공개키 조회 시 사용
  }

Payload:
  {
    "iss": "https://oidc.eks.ap-northeast-2.amazonaws.com/id/EXAMPLED",
            ← EKS OIDC Issuer URL
    "sub": "system:serviceaccount:production:my-sa",
            ← Trust Policy Condition의 sub 클레임 검증 대상
    "aud": ["sts.amazonaws.com"],
            ← STS만 이 토큰 사용 가능 (다른 서비스 재사용 불가)
    "exp": 1234567890,    ← 만료 시간 (기본 24시간)
    "iat": 1234481490,    ← 발급 시간
    "kubernetes.io": {
      "namespace": "production",
      "pod":            { "name": "my-pod",  "uid": "pod-uuid"  },
      "serviceaccount": { "name": "my-sa",   "uid": "sa-uuid"   }
    }
  }

Signature: RS256 (EKS OIDC 개인키로 서명)
  → AWS STS는 OIDC JWKS에서 공개키 조회하여 서명 검증
```

### 4.4 IRSA 전체 설정 실전 가이드

**Step 1: OIDC Provider 확인 및 IAM 등록**

```bash
# OIDC Issuer URL 확인
OIDC_URL=$(aws eks describe-cluster \
  --name my-cluster \
  --query "cluster.identity.oidc.issuer" \
  --output text)

# IAM에 OIDC Provider 등록 (eksctl 권장)
eksctl utils associate-iam-oidc-provider \
  --cluster my-cluster \
  --approve

# 등록 확인
OIDC_ID=$(basename $OIDC_URL)
aws iam list-open-id-connect-providers | grep $OIDC_ID
```

**Step 2: IAM Role Trust Policy 작성**

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
OIDC_ID=$(aws eks describe-cluster --name my-cluster \
  --query "cluster.identity.oidc.issuer" --output text | cut -d'/' -f5)

cat > trust-policy.json << EOF
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {
      "Federated": "arn:aws:iam::${ACCOUNT_ID}:oidc-provider/oidc.eks.ap-northeast-2.amazonaws.com/id/${OIDC_ID}"
    },
    "Action": "sts:AssumeRoleWithWebIdentity",
    "Condition": {
      "StringEquals": {
        "oidc.eks.ap-northeast-2.amazonaws.com/id/${OIDC_ID}:aud":
          "sts.amazonaws.com",
        "oidc.eks.ap-northeast-2.amazonaws.com/id/${OIDC_ID}:sub":
          "system:serviceaccount:production:s3-reader-sa"
      }
    }
  }]
}
EOF

aws iam create-role \
  --role-name EKS-S3ReaderRole \
  --assume-role-policy-document file://trust-policy.json
```

**Step 3: Permission Policy 연결 (최소 권한)**

```bash
cat > s3-policy.json << EOF
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": ["s3:GetObject", "s3:ListBucket"],
    "Resource": [
      "arn:aws:s3:::my-app-bucket",
      "arn:aws:s3:::my-app-bucket/*"
    ]
  }]
}
EOF

aws iam put-role-policy \
  --role-name EKS-S3ReaderRole \
  --policy-name S3ReadPolicy \
  --policy-document file://s3-policy.json
```

**Step 4: ServiceAccount 생성 및 어노테이션**

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: s3-reader-sa
  namespace: production
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::123456789:role/EKS-S3ReaderRole
    eks.amazonaws.com/token-expiration: "3600"          # 토큰 만료 (초)
    eks.amazonaws.com/sts-regional-endpoints: "true"    # 리전 STS 엔드포인트
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
      serviceAccountName: s3-reader-sa   # IRSA 적용 SA
      containers:
        - name: app
          image: my-app:latest
          # SDK가 환경변수 자동 감지 — 별도 자격증명 설정 불필요
```

**eksctl 원스텝 설정**

```bash
eksctl create iamserviceaccount \
  --cluster my-cluster \
  --namespace production \
  --name s3-reader-sa \
  --attach-policy-arn arn:aws:iam::123456789:policy/S3ReadPolicy \
  --approve \
  --override-existing-serviceaccounts
  # OIDC Provider 확인, IAM Role 생성, Trust Policy 작성,
  # SA 어노테이션까지 자동 처리
```

### 4.5 IRSA 검증 및 디버깅

```bash
# 1. 주입된 환경변수 확인
kubectl exec -it my-pod -n production -- env | grep AWS
# 예상 출력:
# AWS_ROLE_ARN=arn:aws:iam::123456789:role/EKS-S3ReaderRole
# AWS_WEB_IDENTITY_TOKEN_FILE=/var/run/secrets/eks.amazonaws.com/serviceaccount/token

# 2. SA 토큰 JWT 디코딩 (iss, sub, aud, exp 확인)
kubectl exec -it my-pod -n production -- \
  cat /var/run/secrets/eks.amazonaws.com/serviceaccount/token | \
  cut -d. -f2 | base64 -d 2>/dev/null | python3 -m json.tool

# 3. STS 호출로 실제 자격증명 확인
kubectl exec -it my-pod -n production -- aws sts get-caller-identity
# 예상 출력:
# "Arn": "arn:aws:sts::123456789:assumed-role/EKS-S3ReaderRole/botocore-session-..."

# 4. 실제 S3 접근 테스트
kubectl exec -it my-pod -n production -- aws s3 ls s3://my-app-bucket
```

**일반적인 IRSA 트러블슈팅**

|증상                  |원인                      |해결 방법                            |
|--------------------|------------------------|---------------------------------|
|AWS_ROLE_ARN 미주입    |pod-identity-webhook 미동작|Webhook 로그 확인, Add-on 재설치        |
|InvalidIdentityToken|OIDC Provider 미등록       |`associate-iam-oidc-provider` 재실행|
|AccessDenied        |Trust Policy sub 불일치    |SA 이름/NS 오타 확인, CloudTrail 조회    |
|주기적 인증 실패           |SA 토큰 갱신 실패             |kubelet 로그, token-expiration 값 증가|

### 4.6 IRSA 보안 고려사항 심화

**Trust Policy Condition 패턴**

```json
// ❌ 위험: sub 조건 없음 → 클러스터 내 모든 SA가 AssumeRole 가능
{
  "Condition": {
    "StringEquals": {
      "oidc.eks.../id/EXAMPLED:aud": "sts.amazonaws.com"
    }
  }
}

// ✅ 안전: 특정 SA만 허용
{
  "Condition": {
    "StringEquals": {
      "oidc.eks.../id/EXAMPLED:aud": "sts.amazonaws.com",
      "oidc.eks.../id/EXAMPLED:sub":
        "system:serviceaccount:production:my-sa"
    }
  }
}

// 다중 SA 허용 시 StringLike 활용
{
  "Condition": {
    "StringLike": {
      "oidc.eks.../id/EXAMPLED:sub":
        "system:serviceaccount:production:*"
    }
  }
}
```

**Default Credential Provider Chain — 우선순위**

```
AWS SDK 자격증명 탐색 순서:

1. 환경변수 (AWS_ACCESS_KEY_ID 등)
2. Web Identity Token File (AWS_WEB_IDENTITY_TOKEN_FILE) ← IRSA
3. ~/.aws/credentials 파일
4. ~/.aws/config 파일
5. EC2 Instance Metadata (IMDSv2) ← 노드 IAM 역할
6. ECS Task 메타데이터

IRSA 있는 파드: 2번에서 자격증명 획득 → 노드 역할(5번) 사용 안 함
IRSA 없는 파드: 5번으로 폴백 → 노드 IAM 역할 사용
→ 노드 역할에는 최소 권한만 부여해야 하는 핵심 이유
```

-----

## 📝 Ch.4 기술 면접 질문

### Q1. IRSA의 동작 원리를 OIDC 토큰 흐름 관점에서 설명해 주세요.

> **핵심 답변 방향**: EKS OIDC Provider가 SA 토큰 발급 → 파드에 마운트 → SDK가 STS AssumeRoleWithWebIdentity 호출 → IAM Trust Policy의 OIDC Condition 검증 → 임시 자격증명 반환의 흐름을 설명해야 합니다.

**▶ 1차 꼬리질문**: IRSA Trust Policy에서 `sub` 조건을 설정하지 않으면 어떤 보안 위험이 발생하나요?

> sub 조건이 없으면 해당 클러스터의 모든 ServiceAccount가 이 역할에 대해 AssumeRoleWithWebIdentity를 수행할 수 있습니다. 공격자가 임의 파드를 생성하여 높은 권한의 IAM 역할을 탈취할 수 있습니다. 예를 들어 S3 전체 버킷 접근 역할에 sub 조건이 없다면, 클러스터 내 어떤 파드에서든 해당 역할로 S3에 접근 가능합니다. 반드시 `system:serviceaccount:<namespace>:<name>` 형식으로 특정 SA만 허용해야 합니다.

**▶▶ 2차 꼬리질문**: AWS SDK의 Default Credential Provider Chain에서 IRSA와 노드 IAM 역할의 우선순위를 설명하고, 이것이 노드 역할 설계에 어떤 영향을 미치나요?

> 탐색 순서: 환경변수 → Web Identity Token File (IRSA) → credentials 파일 → IMDS(노드 역할). IRSA가 설정된 파드(AWS_WEB_IDENTITY_TOKEN_FILE 주입)는 2번에서 자격증명을 획득하므로 노드 역할(IMDS)을 사용하지 않습니다. 이 원리 덕분에 노드 IAM 역할에는 최소한의 권한(EC2 기본 운영, VPC CNI 등)만 부여하고, 각 파드에 필요한 권한은 IRSA로 분리하는 설계가 가능합니다. IRSA 없는 파드는 노드 역할로 폴백하므로, 노드 역할을 강하게 제한하는 것이 Defense in Depth 전략입니다.

**▶▶▶ 3차 꼬리질문**: 컨테이너 익스플로잇으로 파드가 탈취되어 IRSA 토큰 파일이 유출되었습니다. 이 토큰으로 공격자가 할 수 있는 행위의 범위와, 이를 탐지/완화하는 방법을 설명해 주세요.

> 공격 범위: 탈취한 SA JWT 토큰으로 STS AssumeRoleWithWebIdentity 호출 가능 → 해당 IAM Role의 Permission Policy 범위 내에서 AWS 리소스 접근 가능. 단, 토큰은 audience(sts.amazonaws.com)에만 유효하고 기본 1시간 만료. 탐지: CloudTrail에서 비정상 소스 IP의 AssumeRoleWithWebIdentity 이벤트 탐지, GuardDuty `CredentialAccess:Kubernetes/AnomalousBehavior` 탐지. 완화: IRSA 역할에 최소 권한 적용(S3 특정 버킷만), VPC Endpoint로 STS 트래픽 제한, IMDSv2 강제(hop limit=1), NetworkPolicy로 파드 → 인터넷 STS 직접 접근 차단 후 VPC Endpoint 통해서만 허용.

-----

### Q2. IRSA를 사용하지 않고 파드에서 AWS 리소스에 접근해야 하는 레거시 환경이 있습니다. 보안 위험을 최소화하는 방법은 무엇인가요?

**▶ 1차 꼬리질문**: EC2 노드 IAM 역할을 통해 파드가 AWS 리소스에 접근할 때 특정 파드만 접근하도록 제한하는 방법이 있나요?

> 완전한 격리는 어렵지만 완화 방법이 있습니다: 1) IMDSv2 hop limit을 1로 설정하면 컨테이너 내부에서 IMDS 접근이 불가합니다(hop count가 2 필요). 단, 컨테이너가 HostNetwork를 사용하면 우회 가능. 2) NetworkPolicy로 특정 파드만 169.254.169.254(IMDS)로의 트래픽을 허용. 3) 근본적 해결책은 IRSA 또는 Pod Identity로 마이그레이션하는 것입니다.

**▶▶ 2차 꼬리질문**: Kubernetes Secret에 AWS 액세스 키를 저장할 때 etcd 암호화 외에 추가로 고려해야 할 보안 조치는 무엇인가요?

> 1. etcd 저장 시 Encryption at Rest 활성화 (KMS로 봉투 암호화). 2) Secret 접근 RBAC을 엄격히 제한 (list/watch 금지, 특정 SA만 get 허용). 3) AWS Secrets Manager + External Secrets Operator 조합으로 Secret을 K8S Secret이 아닌 AWS에서 동적으로 주입. 4) Secret Store CSI Driver로 마운트 시점에만 값 주입, etcd에 저장 안 함. 5) 가장 좋은 해결책은 IRSA/Pod Identity로 액세스 키 자체를 제거하는 것입니다.

**▶▶▶ 3차 꼬리질문**: 멀티 클러스터 환경에서 동일한 IAM Role을 여러 EKS 클러스터의 ServiceAccount에 부여해야 할 때 IRSA Trust Policy를 어떻게 설계하나요? 그리고 이때 Pod Identity가 더 나은 이유는 무엇인가요?

> IRSA 방식: Trust Policy의 Principal Federated에 각 클러스터의 OIDC URL을 모두 추가해야 합니다. 클러스터가 10개라면 Statement가 10개, Condition도 10쌍이 됩니다. 클러스터 추가/삭제마다 Trust Policy 수정 필요. Pod Identity 방식: Trust Policy는 `pods.eks.amazonaws.com` 서비스 주체 하나만 설정. 클러스터마다 `aws eks create-pod-identity-association`으로 Association만 추가하면 됩니다. 클러스터 추가/삭제 시 Trust Policy 수정 불필요. 10개 클러스터에서 동일 역할 재사용 시 Pod Identity가 훨씬 관리하기 쉽습니다.

-----

## 5. 심화: EKS Pod Identity — 차세대 IRSA

### 5.1 IRSA의 한계

|IRSA 불편함                  |상세 설명                                         |
|--------------------------|----------------------------------------------|
|클러스터마다 OIDC Provider 등록 필요|클러스터 생성마다 IAM OIDC Provider 등록 + Thumbprint 관리|
|클러스터별 Trust Policy 작성     |IAM Role Trust Policy에 클러스터 OIDC URL 하드코딩     |
|멀티 클러스터 역할 재사용 불편         |동일 역할을 N개 클러스터에서 사용 시 Trust Policy N배 증가      |
|Terraform 관리 복잡성          |OIDC Provider ARN을 변수로 관리, 의존성 복잡             |
|Thumbprint 관리             |OIDC 인증서 갱신 시 Thumbprint 업데이트 필요              |

### 5.2 Pod Identity 동작 방식

```
EKS Pod Identity Agent (DaemonSet, kube-system 네임스페이스)
  └─ 각 노드에서 실행
  └─ 로컬 HTTP 서버: 169.254.170.23:80

파드에서 AWS SDK 자격증명 요청 (IMDS 호출)
  http://169.254.170.23/v1/credentials
  │
  ▼ Pod Identity Agent가 가로채어 처리
  │  - 파드 메타데이터 확인 (어느 SA 사용 중인지)
  │  - EKS API에서 해당 SA의 Pod Identity Association 조회
  │  - 연결된 IAM Role로 STS AssumeRole 호출
  │  - 임시 자격증명 파드에 반환
  │
  ▼ AWS SDK가 자격증명 수신 및 사용

IRSA와의 차이:
  - IRSA: 파드가 직접 STS에 JWT 제출 → STS가 OIDC로 검증
  - Pod Identity: 파드가 로컬 Agent에 요청 → Agent가 STS 호출
```

### 5.3 Pod Identity Association 설정

```bash
# Step 1: EKS Pod Identity Agent Add-on 설치
aws eks create-addon \
  --cluster-name my-cluster \
  --addon-name eks-pod-identity-agent

# 설치 확인
kubectl get pods -n kube-system \
  -l app.kubernetes.io/name=eks-pod-identity-agent

# Step 2: Association 생성 (SA ↔ IAM Role 연결)
aws eks create-pod-identity-association \
  --cluster-name my-cluster \
  --namespace production \
  --service-account my-sa \
  --role-arn arn:aws:iam::123456789:role/MyPodRole

# Association 목록 확인
aws eks list-pod-identity-associations --cluster-name my-cluster
```

**IAM Role Trust Policy — IRSA 대비 극단적 단순화**

```json
// IRSA Trust Policy (클러스터별 OIDC URL 포함, 멀티 클러스터 시 복잡)
{
  "Statement": [{
    "Principal": {
      "Federated": "arn:aws:iam::123:oidc-provider/oidc.eks.../id/AAAA"
    },
    "Action": "sts:AssumeRoleWithWebIdentity",
    "Condition": { ... }
  }]
}

// Pod Identity Trust Policy (단순, 범용)
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {
      "Service": "pods.eks.amazonaws.com"
    },
    "Action": [
      "sts:AssumeRole",
      "sts:TagSession"    ← 세션 태그 지원 → ABAC 정책 가능
    ]
  }]
}
// 클러스터 OIDC URL 없음 → 어느 클러스터에서든 재사용
// Association으로 어떤 클러스터/SA에서 사용할지 별도 제어
```

**sts:TagSession을 활용한 ABAC 정책**

```json
// 세션 태그 기반 네임스페이스별 S3 경로 분리
{
  "Effect": "Allow",
  "Action": "s3:GetObject",
  "Resource": "arn:aws:s3:::company-data/${aws:PrincipalTag/kubernetes-namespace}/*"
}
// production SA → s3://company-data/production/* 접근
// staging SA    → s3://company-data/staging/* 접근
// 하나의 IAM Role로 네임스페이스별 리소스 격리 달성
```

### 5.4 IRSA vs Pod Identity 비교 및 선택 기준

|항목              |IRSA              |Pod Identity                   |
|----------------|------------------|-------------------------------|
|도입 시기           |EKS 초기            |2023년 11월 (EKS 1.24+)          |
|OIDC Provider 등록|클러스터마다 필요         |불필요 (EKS 관리)                   |
|Trust Policy 복잡도|클러스터 OIDC URL 포함  |`pods.eks.amazonaws.com` 서비스 주체|
|멀티 클러스터 역할 재사용  |Trust Policy 수정 필요|Association만 추가                |
|세션 태그 (ABAC)    |❌                 |✅ (`sts:TagSession`)           |
|추가 DaemonSet    |불필요               |Pod Identity Agent 필요          |
|EKS 버전          |모든 버전             |EKS 1.24+                      |
|Windows 노드      |✅                 |✅                              |

**선택 기준**

```
신규 클러스터 (EKS 1.24+)
  → Pod Identity 권장
  → 단순한 Trust Policy, 멀티 클러스터 재사용, ABAC 지원

기존 IRSA 운영 중
  → 유지 (마이그레이션 필요 없음)

멀티 클러스터 (동일 역할 여러 클러스터에서 재사용)
  → Pod Identity 강력 권장

네임스페이스별 리소스 격리 필요 (ABAC)
  → Pod Identity (세션 태그 활용)

EKS 1.24 미만 또는 레거시 클러스터
  → IRSA
```

-----

## 📝 Ch.5 기술 면접 질문

### Q1. EKS Pod Identity가 IRSA와 근본적으로 다른 동작 방식을 설명하고, 어떤 상황에서 Pod Identity로 전환하는 것이 유리한지 말씀해 주세요.

> **핵심 답변 방향**: IRSA는 파드가 직접 STS에 OIDC JWT를 제출하는 방식, Pod Identity는 노드에 실행 중인 Agent가 파드를 대신하여 자격증명을 획득하는 방식. Trust Policy 단순화와 멀티 클러스터 재사용이 핵심 차별점입니다.

**▶ 1차 꼬리질문**: Pod Identity Agent가 Down되면 해당 노드의 파드들이 AWS 자격증명을 얻지 못하게 되는데, 이 Single Point of Failure를 어떻게 완화하나요?

> Pod Identity Agent는 DaemonSet으로 배포되어 각 노드에 1개씩 실행됩니다. Agent 장애 시: 1) DaemonSet의 `restartPolicy: Always`와 readinessProbe로 자동 재시작. 2) `priorityClassName: system-node-critical`을 설정하여 리소스 부족 시에도 Agent가 먼저 evict되지 않도록 보호. 3) AWS SDK는 자격증명 캐싱을 수행하므로 Agent 일시 다운 시 캐시된 자격증명으로 단기간 동작 가능. 4) IRSA와 달리 Agent 의존성이 생기는 것이 Pod Identity의 트레이드오프입니다.

**▶▶ 2차 꼬리질문**: `sts:TagSession`으로 세션 태그를 사용한 ABAC 정책을 실제로 어떻게 구현하고, 기존 RBAC 기반 접근과 비교했을 때 장점은 무엇인가요?

> Pod Identity Trust Policy에 `sts:TagSession` 허용 + IAM Permission Policy에 `${aws:PrincipalTag/kubernetes-namespace}` 변수 사용. 예: 각 팀의 파드가 `s3://team-data/<namespace>/` 경로만 접근하도록 하나의 IAM Role로 구현 가능. 장점: IAM Role 수 감소(팀별 역할 대신 단일 역할), 정책 관리 단순화. 기존 RBAC은 K8S API 접근 제어, ABAC은 AWS 리소스 접근 제어 → 두 레이어를 조합하면 K8S 레벨과 AWS 레벨 모두 세밀한 접근 제어 가능.

**▶▶▶ 3차 꼬리질문**: 기존 IRSA를 사용하는 클러스터를 Pod Identity로 마이그레이션할 때 어떤 순서로 진행하고, 마이그레이션 중 서비스 중단 없이 전환하는 방법은 무엇인가요?

> 무중단 마이그레이션 순서: 1) Pod Identity Agent Add-on 설치 (기존 IRSA와 공존 가능). 2) 새 IAM Role 생성 (Pod Identity용 Trust Policy, `pods.eks.amazonaws.com`). 3) Pod Identity Association 생성 (새 역할과 기존 SA 연결). 4) SA의 IRSA 어노테이션(`eks.amazonaws.com/role-arn`)은 **아직 유지**. 5) 카나리 방식: 파드 일부를 새 SA(어노테이션 제거)로 교체하여 Pod Identity 동작 검증. 6) 검증 완료 후 나머지 파드 교체. 7) 기존 IRSA 역할 및 Trust Policy 정리. 핵심: IRSA와 Pod Identity는 동시 운영 가능하므로 점진적 마이그레이션 가능.
