# 3주차 - EKS Scaling 심화 기술 가이드

-----

## 목차

1. [EKS 관리형 노드 그룹](#1-eks-관리형-노드-그룹)
1. [Scaling Up 소개](#2-scaling-up-소개)
1. [HPA — Horizontal Pod Autoscaler](#3-hpa--horizontal-pod-autoscaler)
1. [VPA — Vertical Pod Autoscaler](#4-vpa--vertical-pod-autoscaler)
1. [KEDA — Kubernetes Event-driven Autoscaling](#5-keda--kubernetes-event-driven-autoscaling)
1. [CPA — Cluster Proportional Autoscaler](#6-cpa--cluster-proportional-autoscaler)
1. [CA/CAS — Cluster Autoscaler](#7-cacas--cluster-autoscaler)
1. [Karpenter — Just-in-time Nodes](#8-karpenter--just-in-time-nodes)
1. [Fargate — Nodeless Serverless Compute](#9-fargate--nodeless-serverless-compute)
1. [스케일링 전략 통합 설계](#10-스케일링-전략-통합-설계)

-----

## 1. EKS 관리형 노드 그룹

### 1.1 개념 및 구조

EKS 관리형 노드 그룹(Managed Node Group, MNG)은 AWS가 EC2 인스턴스의 프로비저닝, 등록, 수명주기를 자동으로 관리하는 노드 그룹입니다. 내부적으로 **EC2 Auto Scaling Group(ASG)** 을 기반으로 동작하며, EKS 컨트롤 플레인이 ASG를 직접 제어합니다.

```
EKS Control Plane
  │
  ├─ Managed Node Group (MNG)
  │   └─ EC2 Auto Scaling Group (ASG)
  │       ├─ EC2 Instance 1 (kubelet, kube-proxy, VPC CNI)
  │       ├─ EC2 Instance 2
  │       └─ EC2 Instance N
  │
  └─ Fargate Profile (서버리스, 별도 챕터)
```

### 1.2 관리형 vs 자체 관리형 비교

|항목                   |관리형 노드 그룹 (MNG)       |자체 관리형 노드 그룹|
|---------------------|----------------------|------------|
|노드 프로비저닝             |AWS 자동 관리             |직접 구성       |
|AMI 업데이트             |롤링 업데이트 자동 지원         |직접 교체       |
|Drain 자동화            |✅ (업데이트/삭제 시 자동 drain)|❌ 직접 처리     |
|인스턴스 타입 유연성          |다중 타입 지원              |다중 타입 지원    |
|Spot 통합              |✅                     |✅           |
|커스텀 AMI              |✅ (제한적)               |✅ (완전)      |
|Cluster Autoscaler 연동|✅                     |✅           |
|Karpenter 연동         |❌ (Karpenter는 MNG 우회) |❌           |

### 1.3 노드 그룹 업데이트 전략

MNG의 노드 업데이트(AMI 교체, 인스턴스 타입 변경)는 두 가지 전략으로 수행됩니다.

**Rolling Update (기본)**

```
1. 새 노드 프로비저닝 (신규 AMI)
2. 기존 노드 cordon (신규 스케줄링 차단)
3. 기존 노드 drain (파드 이동)
4. 기존 노드 종료
→ maxUnavailable 설정으로 동시 교체 노드 수 제어
```

**Force Update**

- 즉시 기존 노드 종료 → 파드 재스케줄링
- PDB(Pod Disruption Budget) 무시 가능
- 장애 허용 워크로드 또는 긴급 패치 상황에서 사용

### 1.4 Launch Template과 커스터마이징

MNG는 EC2 Launch Template과 연동하여 다음을 커스터마이징합니다.

- **커스텀 AMI**: Bottlerocket, Amazon Linux 2023, Ubuntu 등
- **User Data**: kubelet 추가 인수, 노드 레이블/테인트 설정
- **인스턴스 스토어**: NVMe SSD 마운트 자동화
- **네트워크 설정**: 추가 보안 그룹, 다중 ENI

```bash
# 노드 그룹 생성 시 kubelet extra args 예시 (User Data)
/etc/eks/bootstrap.sh my-cluster \
  --kubelet-extra-args \
  '--node-labels=role=worker,env=prod \
   --register-with-taints=dedicated=gpu:NoSchedule \
   --max-pods=110'
```

### 1.5 노드 그룹 설계 원칙

**목적별 노드 그룹 분리**

```
┌─────────────────────────────────────────────────────┐
│ 시스템 노드 그룹                                      │
│   - CoreDNS, aws-load-balancer-controller 등         │
│   - On-Demand, m5.large, min=2                      │
├─────────────────────────────────────────────────────┤
│ 워커 노드 그룹 (On-Demand)                           │
│   - 일반 서비스 파드                                 │
│   - m5.xlarge, min=2, max=10                        │
├─────────────────────────────────────────────────────┤
│ 워커 노드 그룹 (Spot)                                │
│   - 배치 작업, 내결함성 워크로드                      │
│   - m5.xlarge+m5.2xlarge+c5.xlarge (다중 타입)      │
├─────────────────────────────────────────────────────┤
│ GPU 노드 그룹 (필요 시)                              │
│   - ML 추론/학습 전용                                │
│   - p3.2xlarge, NoSchedule taint                    │
└─────────────────────────────────────────────────────┘
```

-----

## 2. Scaling Up 소개

### 2.1 스케일링의 두 축

EKS 스케일링은 **파드 레벨**과 **노드 레벨** 두 축으로 나뉩니다.

```
부하 증가
  │
  ▼
[파드 레벨 스케일링]              [노드 레벨 스케일링]
  │                                │
  ├─ HPA: 파드 수 증가             ├─ CA: 노드 수 증가 (ASG)
  ├─ VPA: 파드 리소스 증가         └─ Karpenter: 노드 즉시 프로비저닝
  ├─ KEDA: 이벤트 기반 파드 증가
  └─ CPA: 클러스터 비율 기반 증가
```

### 2.2 스케일링 계층 구조

```
Level 3: 클러스터 자동화
  Cluster Autoscaler / Karpenter
  (노드가 부족할 때 노드를 추가/제거)
        ↑ 트리거
Level 2: 파드 자동화
  HPA / VPA / KEDA / CPA
  (부하에 따라 파드를 늘리거나 리소스 조정)
        ↑ 트리거
Level 1: 부하 발생
  사용자 트래픽, 이벤트, 배치 작업
```

### 2.3 스케일링 결정 요소

|결정 요소                |관련 스케일러      |설명         |
|---------------------|-------------|-----------|
|CPU/메모리 사용률          |HPA, VPA     |리소스 기반 스케일링|
|외부 메트릭 (SQS, Kafka 등)|KEDA         |이벤트 기반 스케일링|
|클러스터 노드 수 비례         |CPA          |인프라 비율 스케일링|
|Pending 파드 감지        |CA, Karpenter|노드 레벨 스케일링 |

### 2.4 스케일링 지연(Latency) 이해

효과적인 스케일링 설계를 위해 각 단계의 지연 시간을 이해해야 합니다.

```
부하 증가 이벤트
  │
  ├─ [메트릭 수집 지연]     Metrics Server: ~15~60초 집계 주기
  ├─ [HPA 반응 지연]        HPA: 15초 루프 + stabilization 300초
  ├─ [스케줄링 지연]         kube-scheduler: 수 ms ~ 수 초
  ├─ [노드 프로비저닝 지연]  CA+ASG: 1~3분 / Karpenter: 30~60초
  ├─ [노드 준비 지연]        kubelet 등록 + CNI 초기화: 1~2분
  └─ [파드 준비 지연]        컨테이너 pull + 시작: 10초~수 분

총 지연: 최악의 경우 5~10분
```

**지연 최소화 전략**

- 메트릭 수집 주기 단축: `--metric-resolution=15s`
- HPA stabilizationWindowSeconds 단축
- Karpenter 사용 (CA 대비 노드 프로비저닝 2~3배 빠름)
- 컨테이너 이미지 사전 캐싱 (DaemonSet으로 pull)
- Pending 파드 발생 전 **예측적 스케일링** (KEDA + 외부 메트릭)

-----

## 3. HPA — Horizontal Pod Autoscaler

### 3.1 개념 및 동작 원리

HPA는 Deployment, ReplicaSet, StatefulSet의 **파드 수(replica)를 자동으로 조절**하는 Kubernetes 네이티브 컨트롤러입니다. 컨트롤 루프 방식으로 동작하며 `kube-controller-manager`에 내장되어 있습니다.

```
HPA 컨트롤 루프 (기본 15초 간격):

1. Metrics Server (또는 Custom Metrics API)에서 메트릭 수집
2. 목표 메트릭 값 대비 현재 값으로 원하는 replica 수 계산
3. Deployment의 replicas 필드 업데이트
4. kube-scheduler가 신규 파드를 노드에 배치
```

### 3.2 스케일링 계산 공식

```
원하는 replica 수 = ceil(현재 replica 수 × (현재 메트릭 값 / 목표 메트릭 값))

예시:
  현재 replica: 3
  현재 CPU 사용률: 90%
  목표 CPU 사용률: 50%
  → 원하는 replica = ceil(3 × (90/50)) = ceil(5.4) = 6
```

**다중 메트릭 사용 시**: 각 메트릭에 대해 계산 후 **최댓값**을 선택합니다.

### 3.3 HPA v2 메트릭 종류

|메트릭 종류             |소스                  |설명           |예시                             |
|-------------------|--------------------|-------------|-------------------------------|
|`Resource`         |Metrics Server      |CPU/메모리 사용률  |`cpu: 50%`                     |
|`Pods`             |Custom Metrics API  |파드별 커스텀 메트릭  |`packets_per_second: 1000`     |
|`Object`           |Custom Metrics API  |클러스터 오브젝트 메트릭|`requests_per_second` (Ingress)|
|`External`         |External Metrics API|외부 시스템 메트릭   |SQS 큐 길이, Datadog 메트릭          |
|`ContainerResource`|Metrics Server      |특정 컨테이너 리소스  |사이드카 제외 메인 컨테이너 CPU            |

### 3.4 HPA 동작 상세 설정

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: my-app-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-app
  minReplicas: 2
  maxReplicas: 20
  metrics:
    # CPU 기반
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 60
    # 커스텀 메트릭 (Prometheus Adapter 필요)
    - type: Pods
      pods:
        metric:
          name: http_requests_per_second
        target:
          type: AverageValue
          averageValue: "500"
  behavior:
    # 스케일 아웃 설정
    scaleUp:
      stabilizationWindowSeconds: 30    # 스케일 아웃 안정화 대기 (짧게)
      policies:
        - type: Percent
          value: 100                    # 현재 replica의 100%만큼 한 번에 증가 가능
          periodSeconds: 60
        - type: Pods
          value: 5                      # 또는 최대 5개씩 증가
          periodSeconds: 60
      selectPolicy: Max                 # 더 큰 값 선택
    # 스케일 인 설정
    scaleDown:
      stabilizationWindowSeconds: 300   # 스케일 인 안정화 대기 (길게 - 플래핑 방지)
      policies:
        - type: Percent
          value: 20                     # 한 번에 20%씩만 감소
          periodSeconds: 60
```

### 3.5 Metrics Server

HPA의 기본 메트릭 소스. 각 노드의 `kubelet /metrics/resource` 엔드포인트에서 CPU/메모리 사용량을 수집합니다.

```
Metrics Server
  │ pull (매 15초)
  ├─ kubelet (Node 1) → 파드 CPU/메모리 사용량
  ├─ kubelet (Node 2)
  └─ kubelet (Node N)
         ↓
  aggregation → HPA가 조회하는 metrics.k8s.io API
```

**Metrics Server 한계**

- 히스토리 없음 (현재 값만 제공, 저장 없음)
- CPU/메모리만 제공 (커스텀 메트릭 미지원)
- 커스텀/외부 메트릭은 **Prometheus Adapter** 또는 **KEDA** 필요

### 3.6 HPA 설계 시 주의사항

**요청(Request) 설정 필수**

HPA CPU 스케일링은 `resources.requests.cpu` 기준으로 사용률을 계산합니다. Request가 없으면 HPA가 동작하지 않습니다.

```yaml
resources:
  requests:
    cpu: "200m"      # 반드시 설정
    memory: "256Mi"
  limits:
    cpu: "1000m"
    memory: "512Mi"
```

**플래핑(Flapping) 방지**

메트릭이 임계값 근처에서 진동할 때 스케일 인/아웃이 반복되는 현상. `stabilizationWindowSeconds`를 충분히 설정하고 `scaleDown.policies`로 감소 속도를 제한합니다.

**Readiness와 연동**

새 파드가 Ready 상태가 되기 전까지 HPA 계산에서 제외됩니다. `readinessProbe`를 정확히 설정하여 준비되지 않은 파드가 트래픽을 받지 않도록 합니다.

-----

## 4. VPA — Vertical Pod Autoscaler

### 4.1 개념 및 필요성

VPA(Vertical Pod Autoscaler)는 파드의 **CPU/메모리 Request와 Limit을 자동으로 조정**합니다. 파드 수를 늘리는 HPA와 달리, 개별 파드의 리소스 크기를 최적화합니다.

**VPA가 필요한 이유**

- 개발자가 임의로 설정한 Request/Limit이 실제 사용량과 괴리 → 과다 예약으로 노드 낭비
- HPA는 수평 확장이 불가능한 워크로드(단일 인스턴스 DB, StatefulSet)에 적용 불가
- 실제 사용 패턴을 학습하여 최적 리소스 권고

### 4.2 VPA 구성 요소

```
┌─────────────────────────────────────────────────────┐
│ VPA 컴포넌트                                         │
│                                                     │
│  VPA Recommender  ← 메트릭 히스토리 분석 → 권고값 산출│
│       │                                             │
│  VPA Updater      ← 권고값으로 파드 재시작           │
│       │                                             │
│  VPA Admission    ← 신규 파드 생성 시 리소스 주입    │
│  Controller                                         │
└─────────────────────────────────────────────────────┘
```

**VPA Recommender**

Prometheus 또는 Metrics Server에서 과거 리소스 사용량 히스토리를 분석하여 권고값을 산출합니다. 백분위수(percentile) 기반으로 계산: CPU는 p95, 메모리는 p90 사용량을 기준으로 권고.

**VPA Updater**

VPA 권고값과 현재 파드 Request 간 차이가 임계값을 초과하면, 해당 파드를 **evict(퇴거)** 합니다. 파드가 재생성될 때 Admission Controller가 새 Request 값을 주입합니다.

**VPA Admission Controller**

파드 생성/재시작 시 VPA 권고값을 Request에 자동 주입하는 Webhook.

### 4.3 VPA 운영 모드

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: my-app-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-app
  updatePolicy:
    updateMode: "Auto"     # Off / Initial / Recreate / Auto
  resourcePolicy:
    containerPolicies:
      - containerName: "my-container"
        minAllowed:
          cpu: "100m"
          memory: "128Mi"
        maxAllowed:
          cpu: "4"
          memory: "8Gi"
        controlledResources: ["cpu", "memory"]
        controlledValues: RequestsAndLimits   # RequestsOnly / RequestsAndLimits
```

|모드        |동작                            |사용 상황           |
|----------|------------------------------|----------------|
|`Off`     |권고값만 계산, 적용 안 함               |모니터링·분석 전용      |
|`Initial` |파드 최초 생성 시에만 주입               |재시작 없이 신규 파드만 조정|
|`Recreate`|범위 벗어나면 파드 재시작                |재시작 허용 워크로드     |
|`Auto`    |Recreate + 향후 In-place 업데이트 지원|완전 자동화          |

### 4.4 HPA + VPA 병행 사용 시 충돌 문제

HPA(CPU 기반)와 VPA를 동시에 사용하면 **충돌**이 발생합니다.

- VPA가 CPU Request를 올림 → HPA 사용률이 낮아짐 → 스케일 인
- HPA가 파드 수를 줄임 → 부하 집중 → VPA가 다시 Request 올림 → 반복

**권장 해결책**

```
방법 1: VPA를 Off 모드로 권고값만 수집, 사람이 직접 Request 설정
방법 2: HPA를 CPU 외 메트릭(RPS, 큐 길이)으로 설정 + VPA로 CPU/메모리 조정
방법 3: Goldilocks (VPA Off 모드 자동화 도구) 사용
```

### 4.5 In-place Pod Vertical Scaling (Kubernetes 1.27+)

기존 VPA는 리소스 변경 시 파드를 재시작해야 했습니다. Kubernetes 1.27(Alpha)부터 파드 재시작 없이 CPU Request/Limit을 변경하는 **In-place 업데이트**를 지원합니다.

- 메모리는 여전히 재시작 필요 (OS 레벨 제약)
- EKS에서 안정화되면 VPA의 가장 큰 단점(재시작)이 해소

-----

## 5. KEDA — Kubernetes Event-driven Autoscaling

### 5.1 개념 및 HPA와의 차이

KEDA(Kubernetes Event-driven Autoscaling)는 **외부 이벤트 소스(SQS, Kafka, Redis 등)의 메트릭을 기반으로 파드를 스케일링**하는 오픈소스 프로젝트입니다. CNCF Graduated 프로젝트.

|항목     |HPA                           |KEDA                    |
|-------|------------------------------|------------------------|
|메트릭 소스 |Metrics Server, Custom Metrics|50+ 외부 스케일러             |
|0으로 스케일|❌ (최소 1개 유지)                  |✅ (0 → N, N → 0)        |
|설정 방식  |HPA 리소스                       |ScaledObject / ScaledJob|
|이벤트 기반 |제한적                           |완전 지원                   |
|Pull 간격|15초 고정                        |스케일러별 설정 가능             |

**0으로 스케일(Scale to Zero)** 이 KEDA의 핵심 차별점입니다. 이벤트가 없을 때 파드를 0개로 줄여 비용을 절감하고, 이벤트 발생 시 즉시 스케일 아웃합니다.

### 5.2 KEDA 아키텍처

```
외부 이벤트 소스
  (SQS, Kafka, Redis, Prometheus, MySQL...)
         │ 메트릭 폴링
         ▼
  KEDA Metrics Adapter
  (External Metrics API 구현체)
         │
         ▼
  KEDA Operator (ScaledObject 감시)
  ├─ HPA 자동 생성/관리
  └─ 0 ↔ 1 전환 (HPA 우회, 직접 제어)
         │
         ▼
  Deployment / Job replica 조정
```

KEDA는 내부적으로 **HPA를 자동 생성**합니다. 외부 메트릭 값을 Kubernetes External Metrics API로 노출하여 HPA가 이를 기반으로 스케일링하도록 합니다. 단, Scale to Zero는 HPA를 우회하여 KEDA Operator가 직접 처리합니다.

### 5.3 주요 스케일러 종류

|카테고리     |스케일러              |메트릭                 |
|---------|------------------|--------------------|
|**메시지 큐**|AWS SQS           |큐 메시지 수             |
|         |Apache Kafka      |Consumer Lag        |
|         |RabbitMQ          |큐 메시지 수             |
|         |Azure Service Bus |메시지 수               |
|**캐시/DB**|Redis             |List 길이, Streams Lag|
|         |MySQL / PostgreSQL|쿼리 결과값              |
|**모니터링** |Prometheus        |PromQL 쿼리 결과        |
|         |Datadog           |Datadog 메트릭         |
|         |CloudWatch        |AWS CloudWatch 메트릭  |
|**스토리지** |AWS S3            |오브젝트 수              |
|**HTTP** |HTTP Add-on       |초당 요청 수             |
|**기타**   |Cron              |시간 기반 스케줄           |

### 5.4 ScaledObject 설정 예시

**SQS 큐 기반 스케일링**

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: sqs-scaledobject
spec:
  scaleTargetRef:
    name: message-processor
  minReplicaCount: 0            # Scale to Zero
  maxReplicaCount: 50
  pollingInterval: 15           # 스케일러 폴링 주기 (초)
  cooldownPeriod: 300           # Scale to Zero 전 대기 시간 (초)
  triggers:
    - type: aws-sqs-queue
      authenticationRef:
        name: keda-aws-credentials
      metadata:
        queueURL: https://sqs.ap-northeast-2.amazonaws.com/123456789/my-queue
        queueLength: "10"       # 파드 1개당 처리할 메시지 수
        awsRegion: ap-northeast-2
        identityOwner: operator # IRSA 사용
```

**Kafka Consumer Lag 기반**

```yaml
triggers:
  - type: kafka
    metadata:
      bootstrapServers: kafka-broker:9092
      consumerGroup: my-consumer-group
      topic: my-topic
      lagThreshold: "100"       # 파드 1개당 허용 Lag
      offsetResetPolicy: latest
```

**Prometheus 메트릭 기반**

```yaml
triggers:
  - type: prometheus
    metadata:
      serverAddress: http://prometheus-server.monitoring:9090
      metricName: http_requests_total
      query: sum(rate(http_requests_total{service="my-app"}[2m]))
      threshold: "500"          # 파드 1개당 초당 500 RPS
```

### 5.5 ScaledJob — 배치 워크로드

일반 Deployment가 아닌 **Job** 단위로 스케일링할 때 사용합니다. 메시지 하나당 Job 하나를 생성하는 패턴.

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledJob
metadata:
  name: sqs-batch-job
spec:
  jobTargetRef:
    template:
      spec:
        containers:
          - name: batch-processor
            image: my-batch:latest
  triggers:
    - type: aws-sqs-queue
      metadata:
        queueURL: https://sqs.../batch-queue
        queueLength: "1"        # 메시지 1개당 Job 1개
```

### 5.6 TriggerAuthentication — AWS IRSA 연동

KEDA가 SQS, CloudWatch 등 AWS 서비스에 접근할 때 IRSA(IAM Roles for Service Accounts)를 사용합니다.

```yaml
apiVersion: keda.sh/v1alpha1
kind: TriggerAuthentication
metadata:
  name: keda-aws-credentials
spec:
  podIdentity:
    provider: aws-eks    # IRSA 사용
```

-----

## 6. CPA — Cluster Proportional Autoscaler

### 6.1 개념 및 필요성

CPA(Cluster Proportional Autoscaler)는 **클러스터의 노드 수 또는 코어 수에 비례하여 특정 파드의 replica 수를 자동 조정**합니다. CPU/메모리 사용률이 아닌 인프라 규모 자체를 기준으로 합니다.

**주요 사용 사례**

- **CoreDNS 스케일링**: 노드가 늘어날수록 DNS 쿼리량도 증가 → CoreDNS 파드 수 비례 증가
- **kube-proxy**: 노드별로 반드시 실행해야 하는 컴포넌트 (DaemonSet 대신 Proportional 방식)
- **Ingress Controller**: 노드 수에 따라 적절한 replica 수 유지
- **Prometheus Node Exporter 집계기**: 노드 수 비례

```
클러스터 노드: 10개  → CoreDNS: 2개
클러스터 노드: 30개  → CoreDNS: 3개
클러스터 노드: 100개 → CoreDNS: 6개
```

### 6.2 CPA 동작 원리

CPA는 Kubernetes API를 통해 노드 수와 CPU 코어 수를 주기적으로 조회하고, 설정된 공식에 따라 replica 수를 계산하여 Deployment를 업데이트합니다.

```
CPA Controller (30초 간격):
  1. 현재 노드 수 및 총 CPU 코어 수 조회
  2. ConfigMap의 ladder/linear 설정으로 replica 수 계산
  3. 대상 Deployment replica 업데이트
```

### 6.3 스케일링 모드

**Ladder 모드 (계단식)**

노드 수 또는 코어 수 구간에 따라 replica를 명시적으로 지정합니다.

```json
{
  "coresToReplicas": [
    [1, 1],
    [64, 3],
    [512, 5],
    [1024, 7]
  ],
  "nodesToReplicas": [
    [1, 1],
    [16, 2],
    [32, 3],
    [64, 4]
  ]
}
```

**Linear 모드 (선형 비례)**

```json
{
  "coresPerReplica": 256,    # 코어 256개당 replica 1개
  "nodesPerReplica": 16,     # 노드 16개당 replica 1개
  "min": 1,
  "max": 20,
  "preventSinglePointOfFailure": true  # 최소 2개 유지
}
```

### 6.4 CoreDNS CPA 실전 설정

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cluster-proportional-autoscaler-coredns
  namespace: kube-system
spec:
  replicas: 1
  template:
    spec:
      containers:
        - name: autoscaler
          image: registry.k8s.io/cpa/cluster-proportional-autoscaler:1.8.6
          command:
            - /cluster-proportional-autoscaler
            - --namespace=kube-system
            - --configmap=coredns-autoscaler
            - --target=deployment/coredns
            - --logtostderr=true
```

-----

## 7. CA/CAS — Cluster Autoscaler

### 7.1 개념 및 동작 원리

CA(Cluster Autoscaler)는 **Pending 상태의 파드가 발생하면 노드를 추가**하고, **노드 리소스 사용률이 낮으면 노드를 제거**하는 클러스터 레벨 자동 스케일러입니다.

EKS 환경에서는 EC2 Auto Scaling Group(ASG)을 통해 노드를 추가/제거합니다.

```
파드 스케줄링 실패 (Pending)
  │
  ▼
CA가 Pending 파드 감지 (10초 간격 체크)
  │
  ▼
어느 노드 그룹을 확장할지 시뮬레이션
  ├─ expander 알고리즘으로 노드 그룹 선택
  └─ ASG DesiredCapacity 증가 → EC2 인스턴스 시작
  │
  ▼
노드 준비 완료 (약 1~3분)
  │
  ▼
파드 스케줄링 완료
```

**Scale-Down 조건**

- 노드의 모든 파드가 다른 노드로 이동 가능 (스케줄링 시뮬레이션)
- 노드 리소스 사용률이 임계값(기본 50%) 이하로 10분 이상 유지
- 시스템 파드(kube-system)나 로컬 스토리지 파드가 없음
- PodDisruptionBudget 위반 없음

### 7.2 CA 핵심 설정 파라미터

```yaml
# CA Deployment args
- --scale-down-utilization-threshold=0.5    # 이 이하이면 scale down 후보
- --scale-down-delay-after-add=10m          # 스케일 아웃 후 scale down 금지 시간
- --scale-down-unneeded-time=10m            # 불필요 노드 유지 시간
- --scan-interval=10s                       # Pending 파드 체크 주기
- --max-node-provision-time=15m             # 노드 프로비저닝 타임아웃
- --expander=least-waste                    # 노드 그룹 선택 알고리즘
- --balance-similar-node-groups=true        # 유사 노드 그룹 간 균등 분배
- --skip-nodes-with-local-storage=false     # 로컬 스토리지 파드 있어도 scale down
- --skip-nodes-with-system-pods=true        # 시스템 파드 있으면 scale down 제외
```

### 7.3 Expander 알고리즘

CA가 여러 노드 그룹 중 어느 것을 확장할지 결정하는 알고리즘입니다.

|알고리즘         |설명                       |적합 사례   |
|-------------|-------------------------|--------|
|`random`     |무작위 선택                   |단순 환경   |
|`most-pods`  |가장 많은 Pending 파드를 수용하는 그룹|파드 수 최대화|
|`least-waste`|리소스 낭비가 가장 적은 그룹         |비용 최적화  |
|`price`      |가장 저렴한 노드 그룹             |비용 최소화  |
|`priority`   |우선순위 기반                  |세밀한 제어  |

### 7.4 CA와 ASG 연동 설정

CA가 EKS 노드 그룹(ASG)을 제어하려면 **ASG에 특정 태그**가 필요합니다.

```
태그 설정:
  k8s.io/cluster-autoscaler/enabled = true
  k8s.io/cluster-autoscaler/<cluster-name> = owned
```

또한 CA 파드의 서비스 어카운트에 **IRSA**로 ASG 관련 IAM 권한이 필요합니다.

```json
{
  "Action": [
    "autoscaling:DescribeAutoScalingGroups",
    "autoscaling:DescribeAutoScalingInstances",
    "autoscaling:SetDesiredCapacity",
    "autoscaling:TerminateInstanceInAutoScalingGroup",
    "ec2:DescribeInstanceTypes",
    "ec2:DescribeLaunchTemplateVersions"
  ]
}
```

### 7.5 CA의 구조적 한계

CA는 수년간 검증된 솔루션이지만 다음 한계가 있습니다.

|한계         |설명                            |
|-----------|------------------------------|
|느린 프로비저닝   |ASG를 통한 EC2 시작: 1~3분 소요       |
|노드 그룹 사전 정의|사용할 인스턴스 타입을 미리 노드 그룹으로 생성해야 함|
|ASG 의존성    |ASG의 모든 제약을 그대로 받음            |
|인스턴스 다양성 제한|노드 그룹당 인스턴스 타입 제한             |
|시뮬레이션 오버헤드 |노드/파드 수 증가 시 계산 비용 증가         |

이러한 한계를 극복하기 위해 **Karpenter**가 등장했습니다.

-----

## 8. Karpenter — Just-in-time Nodes

### 8.1 개념 및 CA와의 차이

Karpenter는 AWS가 주도하여 개발한 오픈소스 노드 프로비저너로, **Pending 파드의 요구사항을 직접 분석하여 최적의 EC2 인스턴스를 즉시 프로비저닝**합니다. CNCF Sandbox → Incubating 프로젝트.

**CA vs Karpenter 핵심 차이**

|항목         |Cluster Autoscaler|Karpenter                 |
|-----------|------------------|--------------------------|
|노드 프로비저닝   |ASG를 통한 간접 제어     |EC2 Fleet API 직접 호출       |
|프로비저닝 속도   |1~3분              |30~60초                    |
|인스턴스 선택    |노드 그룹 내 미리 정의된 타입 |수백 가지 EC2 타입 중 실시간 최적 선택  |
|노드 그룹 필요 여부|✅ 필수              |❌ 불필요 (NodePool만 정의)      |
|빈 패킹 최적화   |제한적               |✅ 파드 요구사항 기반 정밀 선택        |
|Spot 통합    |ASG Spot 활용       |✅ 자동 Spot 최적화 + 중단 처리     |
|비용 최적화     |제한적               |Disruption(통합) 기능으로 지속 최적화|

### 8.2 Karpenter 아키텍처

```
Pending 파드 발생
  │
  ▼
Karpenter Controller
  │ 파드 요구사항 분석
  │ (CPU, Memory, GPU, nodeSelector, affinity, taint 등)
  │
  ▼
EC2 인스턴스 타입 선택 (NodePool의 requirements 기반)
  │ 수백 가지 EC2 타입 중 최적 선택
  │ (비용, 가용성, 요구사항 충족)
  │
  ▼
EC2 Fleet API / RunInstances 직접 호출
  │
  ▼
인스턴스 시작 → kubelet 등록 → 파드 스케줄링
(약 30~60초)
```

### 8.3 NodePool과 EC2NodeClass

Karpenter v0.32 이후 CRD 구조가 변경되었습니다.

**NodePool** — 노드 프로비저닝 정책 정의

```yaml
apiVersion: karpenter.sh/v1beta1
kind: NodePool
metadata:
  name: default
spec:
  template:
    metadata:
      labels:
        role: worker
    spec:
      nodeClassRef:
        apiVersion: karpenter.k8s.aws/v1beta1
        kind: EC2NodeClass
        name: default
      requirements:
        # 인스턴스 카테고리
        - key: karpenter.k8s.aws/instance-category
          operator: In
          values: ["c", "m", "r"]
        # 인스턴스 세대 (최신 세대만)
        - key: karpenter.k8s.aws/instance-generation
          operator: Gt
          values: ["2"]
        # CPU 범위
        - key: karpenter.k8s.aws/instance-cpu
          operator: In
          values: ["4", "8", "16", "32"]
        # Spot + On-Demand 혼합
        - key: karpenter.sh/capacity-type
          operator: In
          values: ["spot", "on-demand"]
        # x86_64 아키텍처
        - key: kubernetes.io/arch
          operator: In
          values: ["amd64"]
      # 노드 최대 수명 (비용 최적화, 오래된 노드 교체)
      expireAfter: 720h
  # 동시 프로비저닝 제한
  limits:
    cpu: "1000"
    memory: 1000Gi
  # Disruption 정책 (비용 최적화)
  disruption:
    consolidationPolicy: WhenUnderutilized  # 노드 통합
    consolidateAfter: 30s
```

**EC2NodeClass** — AWS 특화 설정 정의

```yaml
apiVersion: karpenter.k8s.aws/v1beta1
kind: EC2NodeClass
metadata:
  name: default
spec:
  # AMI 설정
  amiFamily: AL2
  amiSelectorTerms:
    - alias: al2@latest    # 최신 EKS 최적화 AMI 자동 선택
  # 서브넷 선택 (태그 기반)
  subnetSelectorTerms:
    - tags:
        karpenter.sh/discovery: my-cluster
  # 보안 그룹 선택
  securityGroupSelectorTerms:
    - tags:
        karpenter.sh/discovery: my-cluster
  # IAM 인스턴스 프로파일
  instanceProfile: KarpenterNodeInstanceProfile
  # 인스턴스 스토어 정책
  instanceStorePolicy: RAID0
  # 추가 User Data
  userData: |
    #!/bin/bash
    /etc/eks/bootstrap.sh my-cluster \
      --kubelet-extra-args '--max-pods=110'
  # EBS 설정
  blockDeviceMappings:
    - deviceName: /dev/xvda
      ebs:
        volumeSize: 50Gi
        volumeType: gp3
        encrypted: true
```

### 8.4 Disruption — 지속적 비용 최적화

Karpenter의 Disruption 기능은 **이미 프로비저닝된 노드를 지속적으로 최적화**합니다.

```
Disruption 유형:

1. Consolidation (통합)
   낮은 사용률의 노드 여러 개 → 하나의 더 큰 노드로 통합
   또는 빈 노드를 제거
   
   [Node A: 20% 사용] [Node B: 15% 사용]
          ↓ Consolidation
   [Node C: 35% 사용]   (Node A, B 종료)

2. Drift (드리프트 교체)
   NodePool 설정 변경 시 기존 노드를 새 설정으로 교체
   (AMI 업데이트, 태그 변경 등)

3. Expiration (만료)
   expireAfter 설정으로 오래된 노드 강제 교체
   (패치 적용, 보안 갱신)
```

### 8.5 Spot 인스턴스와 중단 처리

Karpenter는 Spot 인스턴스 중단(Interruption) 2분 전 알림을 수신하여 자동으로 파드를 드레인하고 대체 노드를 프로비저닝합니다.

```
AWS EC2 Spot Interruption Notice (2분 전)
  │
  ▼
Karpenter Interruption Handler
  │ 해당 노드 Cordon + Drain
  ├─ 중요 파드 → On-Demand 노드로 이동
  └─ 새 Spot 노드 프로비저닝 시작
```

**Spot 내결함성 설계**

```yaml
# NodePool에서 On-Demand를 fallback으로 설정
requirements:
  - key: karpenter.sh/capacity-type
    operator: In
    values: ["spot", "on-demand"]   # spot 우선, 없으면 on-demand

# 파드 레벨에서 Spot 허용 설정
tolerations:
  - key: karpenter.sh/capacity-type
    operator: Equal
    value: spot
    effect: NoSchedule
```

### 8.6 CA에서 Karpenter로 마이그레이션

```
마이그레이션 전략:

1. Karpenter 설치 (CA와 공존)
2. 신규 NodePool 생성, 일부 워크로드 테스트
3. CA의 노드 그룹 min=max로 고정 (CA 비활성화)
4. Karpenter가 기존 노드를 점진적으로 교체
5. CA 제거

주의사항:
- CA와 Karpenter 동시 운영 시 경합 가능 → 충분한 테스트 필요
- PDB(Pod Disruption Budget) 사전 설정 필수
- Karpenter가 관리하는 노드에 CA 태그 금지
```

-----

## 9. Fargate — Nodeless Serverless Compute

### 9.1 개념 및 특징

AWS Fargate는 EC2 인스턴스(노드)를 직접 관리하지 않고 **파드를 서버리스로 실행**하는 컴퓨팅 엔진입니다. 파드마다 격리된 마이크로 VM에서 실행되며, AWS가 인프라 전체를 관리합니다.

```
일반 EKS (EC2):                  EKS + Fargate:
┌─────────────────┐               ┌─────────────────┐
│ EC2 Node        │               │ Fargate Pod     │ ← 파드마다 격리 VM
│ ┌─────┐ ┌─────┐ │               │ (자체 커널)      │
│ │Pod A│ │Pod B│ │               └─────────────────┘
│ └─────┘ └─────┘ │               ┌─────────────────┐
│ (공유 OS 커널)   │               │ Fargate Pod     │
└─────────────────┘               └─────────────────┘
                                  (노드 관리 불필요)
```

### 9.2 Fargate Profile

EKS에서 어떤 파드를 Fargate로 실행할지 **Fargate Profile**로 정의합니다. 네임스페이스와 레이블 기반으로 매칭합니다.

```yaml
# eksctl Fargate Profile 예시
fargateProfiles:
  - name: fp-default
    selectors:
      - namespace: default
        labels:
          compute: fargate
      - namespace: batch-jobs
  - name: fp-kube-system
    selectors:
      - namespace: kube-system
        labels:
          k8s-app: kube-dns  # CoreDNS만 Fargate로
```

### 9.3 Fargate 과금 모델

Fargate는 **파드가 실행되는 시간 × 예약된 vCPU/메모리**로 과금됩니다.

```
과금 기준:
  - vCPU: 0.04048 USD/vCPU/시간
  - 메모리: 0.004445 USD/GB/시간

예시:
  파드 Request: 1 vCPU, 2GB memory
  실행 시간: 1시간
  비용: (1 × 0.04048) + (2 × 0.004445) = 0.0494 USD/시간

최소 리소스: 0.25 vCPU, 0.5GB
지원 조합: vCPU × 메모리 조합이 제한됨 (AWS 문서 참조)
```

### 9.4 Fargate의 제약 및 한계

|항목         |제약                        |
|-----------|--------------------------|
|DaemonSet  |❌ 미지원 (노드가 없으므로)          |
|HostNetwork|❌ 미지원                     |
|HostPort   |❌ 미지원                     |
|영구 스토리지    |EFS만 지원 (EBS 미지원)         |
|특권 컨테이너    |❌ 미지원                     |
|GPU        |❌ 미지원                     |
|커스텀 AMI    |❌ 미지원                     |
|VPC CNI    |별도 동작 (Secondary IP 방식 동일)|
|최대 Pod 크기  |4 vCPU, 30GB 메모리          |
|스케일링 속도    |EC2 대비 느림 (콜드 스타트)        |

### 9.5 Fargate가 적합한 워크로드

```
적합:
  ✅ 간헐적 배치 작업 (야간 배치, 이벤트 트리거 작업)
  ✅ 개발/테스트 환경 (노드 관리 오버헤드 제거)
  ✅ 보안이 중요한 멀티테넌트 격리 (파드 레벨 VM 격리)
  ✅ 예측 불가능한 소규모 트래픽

부적합:
  ❌ 고성능 컴퓨팅 (GPU, 고속 스토리지)
  ❌ DaemonSet이 필수인 워크로드 (모니터링 에이전트 등)
  ❌ 지속적인 대규모 트래픽 (EC2 대비 비용 높음)
  ❌ 낮은 레이턴시 요구 워크로드 (콜드 스타트)
```

### 9.6 Fargate + KEDA 조합

Fargate 파드에도 KEDA를 적용하여 이벤트 기반 스케일링이 가능합니다. 단, Fargate의 콜드 스타트 지연(수십 초)을 고려해야 합니다.

-----

## 10. 스케일링 전략 통합 설계

### 10.1 스케일러 선택 매트릭스

|워크로드 특성       |권장 파드 스케일러          |권장 노드 스케일러     |
|--------------|--------------------|---------------|
|웹 API (트래픽 기반)|HPA (RPS 메트릭)       |Karpenter      |
|배치 처리 (큐 기반)  |KEDA (SQS/Kafka)    |Karpenter      |
|이벤트 드리븐 (간헐적) |KEDA (Scale to Zero)|Fargate        |
|단일 인스턴스 DB    |VPA                 |CA (노드 안정성)    |
|인프라 컴포넌트      |CPA                 |—              |
|ML 추론         |HPA + KEDA          |Karpenter (GPU)|
|개발/테스트        |—                   |Fargate        |

### 10.2 멀티 스케일러 통합 아키텍처

```
[외부 트래픽]
    │
    ▼
[KEDA] ← Prometheus RPS 메트릭
    │ Scale API Deployment (0 ~ N)
    ▼
[API Deployment Pods]
    │ 파드 부족 시
    ▼
[Karpenter] ← Pending 파드 감지
    │ EC2 인스턴스 직접 프로비저닝
    ▼
[New EC2 Node] → 파드 스케줄링

[CoreDNS]
    │
[CPA] ← 노드 수 비례
    │ CoreDNS replica 자동 조정
    ▼
[CoreDNS Pods]

[특정 컨테이너 리소스 최적화]
[VPA Off 모드] → 권고값 수집 → 수동 Request 설정
```

### 10.3 비용 최적화 설계

**Spot + On-Demand 혼합 전략**

```
[On-Demand 노드]                [Spot 노드]
  - 중요 서비스                   - 배치 처리
  - Stateful 워크로드             - 내결함성 API
  - 시스템 컴포넌트               - 개발 환경
  min=2 (항상 유지)               min=0 (Spot 가용 시만)
```

**Karpenter Consolidation 주기 설계**

```yaml
disruption:
  consolidationPolicy: WhenUnderutilized
  consolidateAfter: 30s      # 빠른 통합 (비용 절감)
  # vs
  consolidateAfter: 5m       # 느린 통합 (안정성 우선)
```

**Savings Plans + Karpenter**

Karpenter가 On-Demand를 선택할 때 Savings Plans가 자동 적용됩니다. 베이스라인 트래픽은 Savings Plans로 커버, 스파이크는 Spot으로 처리하는 하이브리드 전략이 최적입니다.

### 10.4 면접 핵심 질문 — 스케일링

-----

#### Q1. HPA와 VPA를 동시에 사용할 때 발생하는 충돌 문제를 설명하고 해결 방법을 제시하세요.

> **핵심 답변 방향**: HPA가 CPU 사용률 기반으로 스케일 아웃/인 하는 동안 VPA가 CPU Request를 변경하면 HPA의 계산 기준이 바뀌어 스케일링 루프가 발생합니다.

**▶ 1차 꼬리질문**: CPU 외 메트릭(RPS, 큐 길이)으로 HPA를 구성하면 VPA와의 충돌을 피할 수 있나요?

> CPU 기반 HPA와 VPA가 충돌하는 이유는 VPA가 CPU Request를 변경하기 때문. RPS나 외부 메트릭 기반 HPA는 CPU Request에 의존하지 않으므로 충돌 없음. 단, VPA가 메모리를 변경하면 파드 재시작이 발생하여 HPA 스케일 아웃에 영향.

**▶▶ 2차 꼬리질문**: Goldilocks는 무엇이며 VPA와 어떻게 연동되나요?

> Goldilocks는 VPA를 Off 모드로 배포하여 권고값을 수집하고, 이를 시각화하는 도구. 개발자가 UI에서 권고값을 확인하고 수동으로 Request를 설정. 자동 적용 없이 권고값만 활용하는 안전한 방식.

**▶▶▶ 3차 꼬리질문**: Kubernetes 1.27+의 In-place VPA가 안정화되면 HPA+VPA 병행 설계가 어떻게 바뀔 수 있나요?

> 파드 재시작 없이 CPU Request 변경 가능 → VPA의 가장 큰 단점 제거. CPU 기반 HPA와의 충돌은 여전히 존재하지만, 메모리 재시작 문제 해소로 혼합 전략의 실용성 증가. 단, CPU In-place 변경이 HPA 계산에 미치는 영향 검증 필요.

-----

#### Q2. KEDA의 Scale to Zero는 어떻게 동작하며, 이때 첫 요청 처리 지연(Cold Start)을 어떻게 완화하나요?

> **핵심 답변 방향**: KEDA가 HPA를 우회하여 0→1 전환을 직접 처리하는 방식, 그리고 Cold Start 동안의 트래픽 처리 방법(큐잉, 타임아웃 설정)을 설명해야 합니다.

**▶ 1차 꼬리질문**: Scale to Zero에서 Scale to One으로 전환 시 첫 번째 요청이 드롭될 수 있나요?

> 큐 기반(SQS, Kafka) 워크로드: 메시지가 큐에 대기하므로 드롭 없음. HTTP 요청 기반: KEDA HTTP Add-on의 Interceptor가 요청을 버퍼링하고, 파드 준비 후 전달. 단, 파드 시작 시간 동안 응답 지연 발생(수 초~수십 초).

**▶▶ 2차 꼬리질문**: Fargate에서 KEDA Scale to Zero를 사용하면 Cold Start가 더 길어지는 이유와 완화 방법은?

> Fargate는 EC2 기반보다 콜드 스타트가 길음(수십 초). 완화: minReplicaCount=1로 최소 1개 유지(비용 발생), 또는 예측 가능한 패턴이면 Cron 스케일러로 사전 스케일 아웃.

**▶▶▶ 3차 꼬리질문**: KEDA에서 여러 트리거(SQS + Prometheus)를 동시에 설정하면 어떤 replica 수가 선택되나요?

> 각 트리거에서 계산된 replica 수 중 **최댓값**이 선택됩니다. HPA의 다중 메트릭 처리 방식과 동일. SQS가 5개 replica를 요구하고 Prometheus가 3개를 요구하면 5개로 스케일링.

-----

#### Q3. Cluster Autoscaler(CA)와 Karpenter의 근본적인 차이를 설명하고, 언제 CA를 유지하고 언제 Karpenter로 전환하나요?

> **핵심 답변 방향**: CA는 ASG를 간접 제어(프로비저닝 느림, 사전 정의된 노드 그룹 필요)하고, Karpenter는 EC2 API를 직접 호출(빠름, 동적 인스턴스 선택)한다는 차이를 구조적으로 설명해야 합니다.

**▶ 1차 꼬리질문**: Karpenter의 Disruption(Consolidation)이 운영 중인 서비스에 미치는 영향과 안전하게 설정하는 방법은?

> Consolidation이 실행되면 파드가 evict되어 다른 노드로 이동. PDB 설정이 없으면 가용성 문제 발생 가능. 안전 설정: PDB로 최소 가용 파드 수 보장, consolidateAfter를 충분히 길게, budgets로 동시 Disruption 파드 수 제한.

**▶▶ 2차 꼬리질문**: CA와 Karpenter를 동시에 운영할 때 발생하는 경합 문제와 해결 방법은?

> 둘 다 Pending 파드를 감지하여 각자 노드 추가 시도 → 중복 노드 프로비저닝. 해결: CA는 특정 노드 그룹 label로 격리, Karpenter는 별도 NodePool로 분리. 또는 마이그레이션 기간에 CA max=min으로 고정하여 사실상 비활성화.

**▶▶▶ 3차 꼬리질문**: Karpenter의 EC2NodeClass에서 amiSelectorTerms를 `alias: al2@latest`로 설정하면 어떤 운영 리스크가 있고, 어떻게 관리하나요?

> 최신 AMI 자동 채택 → AMI 업데이트 시 Drift가 발생하여 모든 노드가 롤링 교체됨. 리스크: 새 AMI의 예상치 못한 변경사항. 관리 방법: AMI alias 대신 특정 AMI ID로 고정하고, 테스트 환경에서 검증 후 프로덕션 적용. 또는 Disruption budgets로 동시 교체 노드 수를 제한하여 점진적 롤아웃.

