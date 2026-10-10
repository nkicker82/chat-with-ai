# CKS 도메인 1 — Cluster Setup 학습 가이드

> 작성일: 2026-10-09 · 작성자: dw2
> 기준: CKS 커리큘럼(2024-10-15 개편), 시험 환경 Kubernetes v1.35, LFS260 Labs V2026-07-30

## 목차

- [개요](#개요)
- [1. NetworkPolicy로 클러스터 수준 접근 제한](#1-networkpolicy로-클러스터-수준-접근-제한)
- [2. CIS Benchmark와 kube-bench](#2-cis-benchmark와-kube-bench)
- [3. Ingress에 TLS 적용](#3-ingress에-tls-적용)
- [4. 노드 메타데이터와 엔드포인트 보호](#4-노드-메타데이터와-엔드포인트-보호)
- [참고: GUI(Dashboard) 접근 최소화](#참고-guidashboard-접근-최소화)
- [5. 플랫폼 바이너리 검증](#5-플랫폼-바이너리-검증)
- [6. LFS260 랩 매핑](#6-lfs260-랩-매핑)
- [7. 시험 실전 팁과 치트시트](#7-시험-실전-팁과-치트시트)
- [8. 자가 점검 문제](#8-자가-점검-문제)
- [출처](#출처)

## 개요

Cluster Setup은 시험 비중 15%이며, 2024-10-15 개편 이후 출제 역량은 아래 5개입니다 ([LF 공식 커리큘럼 변경 공지](https://training.linuxfoundation.org/cks-program-changes/)). 문항 수는 적지만 명령어·YAML 문법을 정확히 알면 빠르게 점수를 확보할 수 있는 영역입니다.

| # | 공식 역량 (원문) | 이 문서 섹션 | 연결 LFS260 랩 |
| --- | --- | --- | --- |
| 1 | Use Network security policies to restrict cluster level access | NetworkPolicy | 6.1 |
| 2 | Use CIS benchmark to review the security configuration of Kubernetes components (etcd, kubelet, kubedns, kubeapi) | CIS Benchmark와 kube-bench | 2.4, 3.7, 4.4, 9.2-1 |
| 3 | Properly set up Ingress with TLS | Ingress TLS | 6.2, 6.3 |
| 4 | Protect node metadata and endpoints | 노드 메타데이터·엔드포인트 보호 | 랩 없음 (자습) |
| 5 | Verify platform binaries before deploying | 플랫폼 바이너리 검증 | 랩 없음 (자습) |

권장 학습 순서는 NetworkPolicy → CIS/kube-bench → Ingress TLS → 메타데이터·kubelet → 바이너리 검증입니다. 앞의 두 개가 출제 빈도와 난이도 모두 높고, 뒤의 세 개는 패턴이 고정돼 있어 반복 연습으로 끝낼 수 있습니다.

이 문서 사용법: 각 섹션은 개념 → 시험형 YAML/명령 → 함정 순서로 구성했습니다. 섹션을 읽은 뒤 LFS260 랩 클러스터(v1.35.2)에서 직접 타이핑해 봅니다.

## 1. NetworkPolicy로 클러스터 수준 접근 제한

NetworkPolicy는 허용 목록(allow-list)만 표현하는 리소스입니다. 정책에 선택된 Pod는 해당 방향(Ingress/Egress)이 격리되고, 이후에는 어떤 정책이든 허용한 트래픽만 통과합니다. 공식 문서: [Network Policies](https://kubernetes.io/docs/concepts/services-networking/network-policies/).

### 1.1 동작 원리 다섯 가지

1. 정책이 하나도 없으면 모든 트래픽이 허용됩니다 (non-isolated).
2. `podSelector`가 Pod를 고르고 `policyTypes`에 Ingress가 들어가면 그 Pod의 Ingress가 격리됩니다. Egress도 독립적으로 동일합니다.
3. 여러 정책은 합집합(OR)입니다. deny 규칙은 없으므로 "A만 막기"는 "A를 제외한 나머지 허용"으로 바꿔 써야 합니다.
4. 연결 단위로 판단합니다. 허용된 연결의 응답 패킷은 반대 방향 정책과 무관하게 통과합니다.
5. 정책 강제는 CNI가 담당합니다. Flannel 단독처럼 미지원 CNI에서는 리소스가 생성돼도 아무 효과가 없습니다.

`policyTypes`를 생략하면 Ingress는 항상 포함되고, Egress는 `egress` 필드가 있을 때만 포함됩니다. 시험에서는 의도를 분명히 하기 위해 항상 명시하는 습관이 안전합니다.

### 1.2 기본 템플릿 (암기 대상)

Namespace 전체 default-deny (Ingress + Egress):

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: prod-a
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
```

Egress를 막으면 DNS도 막힙니다. 서비스 이름으로 통신해야 하는 문제라면 DNS 허용을 함께 넣습니다:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns
  namespace: prod-a
spec:
  podSelector: {}
  policyTypes:
  - Egress
  egress:
  - to:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: kube-system
      podSelector:
        matchLabels:
          k8s-app: kube-dns
    ports:
    - protocol: UDP
      port: 53
    - protocol: TCP
      port: 53
```

모든 Ingress 허용 (빈 규칙 `- {}`)과 모든 Ingress 차단 (`ingress` 필드 없음 또는 `ingress: []`)의 차이를 정확히 구분해야 합니다.

### 1.3 가장 많이 틀리는 함정: from/to 항목의 AND와 OR

`from` 리스트의 각 항목(`-`)은 OR, 한 항목 안의 `namespaceSelector`와 `podSelector`는 AND입니다. 들여쓰기 한 칸 차이로 의미가 완전히 바뀝니다.

```yaml
# AND: team=a Namespace 안의 app=db Pod만
ingress:
- from:
  - namespaceSelector:
      matchLabels:
        team: a
    podSelector:
      matchLabels:
        app: db

# OR: team=a Namespace의 모든 Pod, 또는 정책과 같은 Namespace의 app=db Pod
ingress:
- from:
  - namespaceSelector:
      matchLabels:
        team: a
  - podSelector:
      matchLabels:
        app: db
```

| 작성 형태 | 의미 |
| --- | --- |
| `namespaceSelector`와 `podSelector`가 같은 항목 | 해당 Namespace 안의, 해당 라벨 Pod만 (AND) |
| `namespaceSelector`와 `podSelector`가 별도 항목 | 해당 Namespace의 모든 Pod 또는 정책과 같은 Namespace의 해당 라벨 Pod (OR) |
| `- podSelector: {...}` 단독 | 정책이 있는 Namespace 안의 Pod만 |
| `- namespaceSelector: {}` 단독 | 모든 Namespace의 모든 Pod (외부 IP는 제외) |

Namespace 지정에는 자동으로 붙는 `kubernetes.io/metadata.name: <ns이름>` 라벨을 쓰면 라벨을 따로 달 필요가 없습니다. 문제에서 특정 라벨(예: `env=prod`)로 지정하라고 하면 그 라벨을 써야 하니 `kubectl get ns --show-labels`로 먼저 확인합니다.

### 1.4 ipBlock과 포트

- `ipBlock`은 클러스터 외부 CIDR용입니다. Pod IP는 바뀌고 SNAT 여부가 CNI마다 달라 Pod 간 제어에는 selector를 씁니다.
- `except`로 일부 대역을 빼고 허용할 수 있습니다. 예: `cidr: 0.0.0.0/0`, `except: [169.254.169.254/32]` (4장 메타데이터 차단에서 다시 사용).
- 포트 범위는 `port: 32000`과 `endPort: 32768`로 지정합니다.
- NetworkPolicy에는 FQDN이 없습니다. "example.com에서 오는 트래픽 차단" 같은 문제는 IP를 조회해 `ipBlock`으로 바꿔야 합니다.

### 1.5 Cilium 관점

Cilium은 표준 NetworkPolicy를 그대로 지원하고, 추가로 `CiliumNetworkPolicy`/`CiliumClusterwideNetworkPolicy`를 제공합니다. 표준 리소스에 없는 기능은 L7(HTTP 메서드·경로) 규칙, `toFQDNs` egress, `ingressDeny`/`egressDeny` 명시적 거부, `toEntities`(world, host, kube-apiserver 등)입니다. 시험은 문제에서 명시하지 않는 한 표준 `networking.k8s.io/v1` NetworkPolicy로 답하는 것이 기본이며, Cilium 고유 리소스는 문제에 CRD 이름이 나올 때만 사용합니다.

### 1.6 검증 명령

```bash
kubectl get netpol -A
kubectl describe netpol <name> -n <ns>     # 해석된 규칙을 사람이 읽기 좋게 출력
kubectl -n dev-ns exec devapp -- curl -s -m 3 frontend.prod-a.svc.cluster.local
kubectl -n dev-ns exec devapp -- nc -zv -w 3 <svc> 80
```

`-m 3`/`-w 3`으로 타임아웃을 짧게 걸어야 시험 시간을 아낍니다. 차단된 연결은 refused가 아니라 timeout으로 나타납니다.

### 1.7 시험형 예제: LFS260 9.2-5 풀이

문제: `workgroup: dev` 라벨 Pod는 prod-a에서 prod-b로 통신할 수 있어야 하고, 두 Namespace에서 dev-ns로 가는 트래픽은 모두 차단해야 합니다.

```yaml
# prod-b: prod-a의 workgroup=dev Pod에서 오는 Ingress만 허용
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dev-from-prod-a
  namespace: prod-b
spec:
  podSelector:
    matchLabels:
      workgroup: dev
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: prod-a
      podSelector:
        matchLabels:
          workgroup: dev
---
# dev-ns: prod-a, prod-b에서 오는 Ingress 차단 = 그 둘을 제외한 출처만 허용
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-from-prod
  namespace: dev-ns
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchExpressions:
        - key: kubernetes.io/metadata.name
          operator: NotIn
          values: ["prod-a", "prod-b"]
```

두 번째 정책은 "deny 규칙이 없다"는 원리 3을 `NotIn`으로 우회한 형태입니다. 다만 이 규칙은 클러스터 외부 IP를 허용하지 않으므로, 외부 접근도 유지해야 한다면 `ipBlock` 항목을 별도로 추가합니다.

## 2. CIS Benchmark와 kube-bench

시험에서는 kube-bench 결과(FAIL 항목)를 주고 특정 항목을 고치라고 하는 형태가 전형적입니다. 핵심은 항목 번호를 외우는 것이 아니라, 어느 컴포넌트의 어느 파일을 고쳐야 하는지 즉시 판단하고 안전하게 재기동하는 것입니다.

### 2.1 CIS Kubernetes Benchmark 구조

| 섹션 | 대상 | 설정 위치 (kubeadm 기준) |
| --- | --- | --- |
| 1.1 | Control plane 설정 파일 권한·소유자 | `/etc/kubernetes/manifests/*.yaml`, `/etc/kubernetes/*.conf`, PKI |
| 1.2 | kube-apiserver | `/etc/kubernetes/manifests/kube-apiserver.yaml` |
| 1.3 | kube-controller-manager | `/etc/kubernetes/manifests/kube-controller-manager.yaml` |
| 1.4 | kube-scheduler | `/etc/kubernetes/manifests/kube-scheduler.yaml` |
| 2 | etcd | `/etc/kubernetes/manifests/etcd.yaml`, 데이터 디렉터리 `/var/lib/etcd` |
| 3 | Control plane 구성 (인증, 감사 로그) | apiserver 플래그, audit policy |
| 4.1 | Worker 설정 파일 권한 | kubelet 서비스 파일, kubeconfig, `config.yaml` |
| 4.2 | kubelet | `/var/lib/kubelet/config.yaml` (+ `/etc/systemd/system/kubelet.service.d/10-kubeadm.conf`) |
| 5 | 정책 (RBAC, PSS, NetworkPolicy, Secret) | 클러스터 리소스 |

### 2.2 kube-bench 실행과 결과 읽기

```bash
# LFS260 4.4처럼 패키지로 설치된 경우
sudo kube-bench                                  # 노드 역할 자동 감지
sudo kube-bench run --targets master             # control plane만
sudo kube-bench run --targets node               # worker(kubelet)만
sudo kube-bench run --targets etcd
sudo kube-bench run --targets master --check 1.2.16   # 특정 항목만 재검사
sudo kube-bench run --targets master | grep -A3 FAIL
```

결과는 `[PASS]`/`[FAIL]`/`[WARN]`/`[INFO]`로 나오고, 맨 아래 `== Remediations ==`에 항목별 수정 방법이 그대로 적혀 있습니다. 시험에서 가장 빠른 길은 Remediations 문구를 따라 고친 뒤 `--check`로 해당 항목만 재검사하는 것입니다. WARN은 대부분 자동 검사가 불가능한 Manual 항목입니다.

같은 목적의 도구로 Kubescape(LFS260 3.7)가 있으며 `kubescape scan framework cis-v1.12.0`처럼 프레임워크를 지정해 씁니다. 시험에는 kube-bench가 주로 나옵니다.

### 2.3 컴포넌트별 핵심 조치 항목

| 컴포넌트 | 설정 | 목표 값 | 비고 |
| --- | --- | --- | --- |
| kube-apiserver | `--authorization-mode` | `Node,RBAC` | `AlwaysAllow` 포함 금지 |
| kube-apiserver | `--anonymous-auth` | `false` | 아래 함정 참고 |
| kube-apiserver | `--enable-admission-plugins` | `NodeRestriction` 포함 | `AlwaysAdmit` 금지 |
| kube-apiserver | `--profiling` | `false` | |
| kube-apiserver | `--kubelet-certificate-authority` | `/etc/kubernetes/pki/ca.crt` | kubelet 서버 인증서 검증 |
| kube-apiserver | `--audit-log-path` 외 3종 | 경로, maxage 30, maxbackup 10, maxsize 100 | 감사 로그는 Cluster Hardening·Monitoring과도 연결 |
| kube-apiserver | `--encryption-provider-config` | 설정 파일 경로 | Secret 암호화 (LFS260 5.7) |
| controller-manager | `--profiling` | `false` | |
| controller-manager | `--use-service-account-credentials` | `true` | 컨트롤러별 SA 사용 |
| controller-manager | `--bind-address` | `127.0.0.1` | |
| scheduler | `--profiling`, `--bind-address` | `false`, `127.0.0.1` | |
| etcd | `--client-cert-auth`, `--peer-client-cert-auth` | `true` | |
| etcd | `--auto-tls`, `--peer-auto-tls` | `false` | |
| etcd | `/var/lib/etcd` 권한·소유자 | `700`, `etcd:etcd` | 사용자가 없으면 생성 필요 |
| kubelet | `authentication.anonymous.enabled` | `false` | |
| kubelet | `authorization.mode` | `Webhook` | `AlwaysAllow` 금지 |
| kubelet | `readOnlyPort` | `0` | 10255 비활성화 |
| kubelet | `protectKernelDefaults` | `true` | 노드 sysctl이 맞지 않으면 kubelet 기동 실패 |
| kubelet | `rotateCertificates` | `true` | |
| 파일 | `/var/lib/kubelet/config.yaml` | `600`, `root:root` | 4.1.x |
| 파일 | `/etc/kubernetes/manifests/*.yaml` | `600` 이하, `root:root` | 1.1.x |

커리큘럼 원문의 kubedns(CoreDNS)는 CIS에 독립 섹션이 거의 없습니다. 실무적으로는 CoreDNS ConfigMap 노출 범위와 kube-system 접근 RBAC 정도를 확인하면 충분합니다.

> **함정:** `--anonymous-auth=false`를 apiserver에 걸면 `/livez`·`/readyz`에 대한 비인증 프로브가 401을 받아 apiserver가 재시작을 반복할 수 있습니다. 최신 벤치마크는 이 항목을 Manual로 분류합니다. 문제에서 명시적으로 요구할 때만 적용하고, 적용 후 Pod 상태를 끝까지 확인합니다.

### 2.4 Static Pod 매니페스트를 안전하게 고치는 절차

1. 백업은 매니페스트 디렉터리 밖에 둡니다: `sudo cp /etc/kubernetes/manifests/kube-apiserver.yaml ~/kube-apiserver.yaml.bak`. 디렉터리 안에 두면 kubelet이 그 파일도 Pod로 띄우려 합니다.
2. `sudo vim`으로 `command:` 아래 플래그를 추가·수정합니다. 이미 있는 플래그는 중복으로 넣지 말고 값을 고칩니다.
3. 파일을 참조하는 플래그(audit policy, encryption config 등)는 `volumes`/`volumeMounts`도 함께 추가합니다.
4. 저장 후 kubelet이 자동 재생성합니다. 30~60초 기다리며 `watch crictl ps`로 컨테이너를 봅니다.
5. 올라오지 않으면 `sudo crictl ps -a | grep apiserver`, `sudo crictl logs <id>`, `/var/log/pods/kube-system_kube-apiserver-*/`, `sudo journalctl -u kubelet -f` 순서로 원인을 봅니다.

kubelet 설정은 Static Pod가 아니므로 `/var/lib/kubelet/config.yaml` 수정 뒤 `sudo systemctl daemon-reload && sudo systemctl restart kubelet`이 필요합니다. 워커 노드라면 해당 노드에 ssh로 접속해 수정합니다.

## 3. Ingress에 TLS 적용

Ingress TLS 문제는 거의 항상 "인증서 생성 → `kubernetes.io/tls` Secret → Ingress `tls` 블록 → curl 검증" 4단계로 끝납니다. Secret은 반드시 Ingress와 같은 Namespace에 있어야 합니다. 공식 문서: [Ingress TLS](https://kubernetes.io/docs/concepts/services-networking/ingress/#tls).

### 3.1 인증서와 Secret

```bash
# 자체 서명 인증서 (LFS260 6.3)
openssl req -x509 -newkey rsa:2048 -nodes -days 365 \
  -keyout example.key -out example.crt \
  -subj "/CN=example.io" \
  -addext "subjectAltName=DNS:example.io"

kubectl -n <ns> create secret tls example --cert=example.crt --key=example.key
kubectl -n <ns> get secret example -o yaml   # type: kubernetes.io/tls, tls.crt / tls.key 키 확인
```

시험에서는 인증서 파일을 미리 주는 경우가 많습니다. 그때는 `create secret tls`만 하면 됩니다. `--cert`/`--key`를 바꿔 넣지 않았는지 `openssl x509 -in example.crt -noout -subject`로 한 번 확인합니다.

### 3.2 Ingress 매니페스트

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: secure-app
  namespace: web
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"        # HTTP → HTTPS 308 리다이렉트
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"  # 외부 LB에서 TLS 종료 시에도 강제
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - example.io
    secretName: example
  rules:
  - host: example.io
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-svc
            port:
              number: 80
```

`tls.hosts`와 `rules.host`가 정확히 일치해야 해당 인증서가 쓰입니다. 일치하지 않으면 컨트롤러의 기본 자체 서명 인증서("Kubernetes Ingress Controller Fake Certificate")가 응답합니다. `kubectl create ingress`로 뼈대를 만들면 빠릅니다:

```bash
kubectl -n web create ingress secure-app --class=nginx \
  --rule="example.io/*=web-svc:80,tls=example" --dry-run=client -o yaml > ing.yaml
```

### 3.3 검증

```bash
kubectl -n ingress-nginx get svc ingress-nginx-controller   # 443에 매핑된 NodePort 확인
curl -vk https://example.io:<NodePort> --resolve example.io:<NodePort>:<NodeIP>
curl -I http://example.io:<HTTP-NodePort> --resolve example.io:<HTTP-NodePort>:<NodeIP>   # 308 확인
```

`-v` 출력의 `subject: CN=example.io`가 내가 만든 인증서인지 확인합니다. Fake Certificate가 보이면 Secret 이름, Namespace, host 불일치 중 하나입니다. `kubectl describe ingress`의 Events와 `kubectl -n ingress-nginx logs deploy/ingress-nginx-controller`로 원인을 좁힙니다.

### 3.4 주의할 점

- ingress-nginx는 2026년 3월 유지보수 종료가 공지된 프로젝트입니다. 시험 환경의 컨트롤러 종류는 문제에서 주어지므로 `ingressClassName`을 `kubectl get ingressclass`로 먼저 확인합니다.
- TLS 종료 지점 이후(컨트롤러 → 백엔드)는 평문입니다. Pod 간 암호화는 다른 도메인(Minimize Microservice Vulnerabilities)의 Cilium 암호화·mTLS 주제입니다.
- Gateway API를 쓰라는 문제라면 `Gateway`의 `listeners[].tls.certificateRefs`에 같은 Secret을 지정합니다.

## 4. 노드 메타데이터와 엔드포인트 보호

이 역량은 두 가지를 묻습니다. Pod가 클라우드 메타데이터 서버(169.254.169.254)에 접근하지 못하게 막는 것, 그리고 노드가 여는 관리 포트(특히 kubelet 10250)가 인증 없이 열려 있지 않게 하는 것입니다.

### 4.1 왜 메타데이터가 위험한가

메타데이터 서버는 인스턴스 자격 증명, 사용자 데이터(user-data), 일부 클라우드에서는 kubelet 부트스트랩 정보까지 돌려줍니다. 컨테이너 하나가 침해되면 Pod 네트워크를 통해 노드 권한으로 확대되는 대표 경로입니다. OpenStack도 Nova 메타데이터를 같은 주소로 제공하므로 CAPO로 올린 클러스터에도 똑같이 해당됩니다.

### 4.2 NetworkPolicy로 차단 (시험 정석)

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-metadata
  namespace: app
spec:
  podSelector: {}
  policyTypes:
  - Egress
  egress:
  - to:
    - ipBlock:
        cidr: 0.0.0.0/0
        except:
        - 169.254.169.254/32
```

특정 Pod만 예외로 허용하라는 문제라면 정책을 하나 더 만듭니다. 정책은 합집합이므로 아래 정책이 선택한 Pod는 메타데이터에 접근할 수 있게 됩니다.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-metadata
  namespace: app
spec:
  podSelector:
    matchLabels:
      role: metadata-accessor
  policyTypes:
  - Egress
  egress:
  - to:
    - ipBlock:
        cidr: 169.254.169.254/32
```

검증: `kubectl -n app exec <pod> -- curl -s -m 3 http://169.254.169.254/` 가 timeout이면 성공입니다. 첫 번째 정책은 `0.0.0.0/0`을 허용하므로 DNS를 포함한 다른 egress는 그대로 동작합니다.

실무 보완책: 클라우드 측 IMDSv2(hop limit 1) 강제, GKE Workload Identity 같은 메타데이터 프록시, Cilium의 `CiliumClusterwideNetworkPolicy`로 클러스터 전체 일괄 차단.

### 4.3 노드가 여는 포트

| 포트 | 컴포넌트 | 노출 원칙 |
| --- | --- | --- |
| 6443 | kube-apiserver | 관리망·LB에서만 접근 |
| 2379-2380 | etcd 클라이언트·피어 | control plane 노드 간에만 |
| 10250 | kubelet API | apiserver에서만, 반드시 인증·인가 |
| 10255 | kubelet 읽기 전용 (비인증) | `readOnlyPort: 0`으로 비활성화 |
| 10257 | kube-controller-manager | `127.0.0.1` 바인드 |
| 10259 | kube-scheduler | `127.0.0.1` 바인드 |
| 30000-32767 | NodePort | 필요한 서비스만, 보안 그룹으로 범위 제한 |

노드에서 실제로 열린 포트와 바인드 주소는 `sudo ss -tlnp`로 확인합니다. `0.0.0.0:10257`처럼 전체 바인드면 `--bind-address=127.0.0.1`로 고칩니다.

### 4.4 kubelet API 잠그기

익명 인증이 켜져 있고 인가가 `AlwaysAllow`이면, 노드 IP만 알면 누구나 `/pods`, `/exec`, `/run`으로 그 노드의 모든 컨테이너에서 명령을 실행할 수 있습니다.

```bash
# 점검: 401 Unauthorized 또는 403 Forbidden이어야 정상
curl -sk https://<NodeIP>:10250/pods | head
curl -s http://<NodeIP>:10255/pods | head    # 연결 실패여야 정상
```

`/var/lib/kubelet/config.yaml`의 목표 상태:

```yaml
authentication:
  anonymous:
    enabled: false
  webhook:
    enabled: true
  x509:
    clientCAFile: /etc/kubernetes/pki/ca.crt
authorization:
  mode: Webhook
readOnlyPort: 0
```

수정 후 `sudo systemctl restart kubelet`, `kubectl get nodes`로 Ready 유지 확인. kubeadm은 기본으로 이 값을 쓰므로, 시험에서는 누군가 일부러 풀어둔 설정을 되돌리는 형태로 나옵니다. 10-kubeadm.conf의 `KUBELET_EXTRA_ARGS`나 커맨드라인 플래그가 config.yaml보다 우선한다는 점도 확인 포인트입니다.

## 참고: GUI(Dashboard) 접근 최소화

"Minimize use of, and access to, GUI elements"는 2024-10 개편으로 공식 역량에서 빠졌습니다. 오래된 문제집이나 블로그에서 나오면 우선순위를 낮춰도 됩니다. 최소한의 원칙만 알아두면 충분합니다.

- Dashboard는 `NodePort`/`LoadBalancer`로 노출하지 않고 `kubectl proxy` 또는 `kubectl port-forward`로만 접근합니다.
- Dashboard ServiceAccount에 `cluster-admin`을 바인딩하지 않고, 사용자 토큰으로 로그인하게 합니다.
- `--enable-skip-login`, `--enable-insecure-login` 인자는 쓰지 않습니다.

## 5. 플랫폼 바이너리 검증

시험에서는 "이 디렉터리의 kubelet/kubectl/kube-apiserver 바이너리 중 공식 릴리스와 해시가 다른 것을 삭제하라" 형태가 대표적입니다. 도구는 `sha512sum` 하나면 됩니다.

### 5.1 공식 해시와 비교

각 바이너리 옆에 같은 이름의 `.sha256`, `.sha512` 파일이 게시됩니다. 경로 형식: `https://dl.k8s.io/release/<버전>/bin/linux/amd64/<바이너리>`.

```bash
VER=v1.35.2
curl -LO https://dl.k8s.io/release/$VER/bin/linux/amd64/kubectl
curl -LO https://dl.k8s.io/release/$VER/bin/linux/amd64/kubectl.sha512

# 방법 1: 자동 검증 (공백 두 칸 주의)
echo "$(cat kubectl.sha512)  kubectl" | sha512sum --check
# kubectl: OK

# 방법 2: 눈으로 비교
sha512sum kubectl; cat kubectl.sha512; echo
```

서버 컴포넌트 묶음(`kubernetes-server-linux-amd64.tar.gz`)의 해시는 GitHub 릴리스의 CHANGELOG 다운로드 표에 있습니다. 시험에서 해시 목록을 텍스트로 주면 `sha512sum *`의 출력과 나란히 놓고 비교합니다. 버전을 먼저 확인해야 비교 대상이 맞습니다: `./kubelet --version`, `./kubectl version --client`.

### 5.2 실행 중인 바이너리 확인

kubelet은 호스트 바이너리이고 apiserver·controller-manager·scheduler·etcd는 컨테이너 안에 있습니다. 컨테이너 안 바이너리도 호스트에서 `/proc/<pid>/root`로 직접 읽을 수 있습니다.

```bash
sha512sum $(which kubelet)

PID=$(pgrep -f kube-apiserver | head -1)
ls -l /proc/$PID/exe                                # 실제 실행 경로
sudo sha512sum /proc/$PID/root/usr/local/bin/kube-apiserver
```

### 5.3 서명 검증 (cosign, 참고)

1.26부터 릴리스 바이너리는 Sigstore로 서명되며 `.sig`/`.cert` 파일이 함께 게시됩니다. 공급망 검증의 기본 개념이므로 Supply Chain Security 도메인과 함께 익혀 둡니다. 공식 문서: [Verify Signed Kubernetes Artifacts](https://kubernetes.io/docs/tasks/administer-cluster/verify-signed-artifacts/).

```bash
cosign verify-blob kubectl \
  --signature kubectl.sig --certificate kubectl.cert \
  --certificate-identity krel-staging@k8s-releng-prod.iam.gserviceaccount.com \
  --certificate-oidc-issuer https://accounts.google.com
```

## 6. LFS260 랩 매핑

LFS260 랩(V2026-07-30, 클러스터 v1.35.2) 중 이 도메인과 직접 연결되는 것은 7개입니다. 메타데이터 차단과 바이너리 검증은 랩이 없으므로 4·5장 명령을 랩 클러스터에서 따로 실습합니다.

| 랩 | 내용 | 이 도메인에서 추가로 해볼 것 |
| --- | --- | --- |
| 2.4 CIS Benchmarks | CIS 공식 PDF 다운로드·항목 구조 확인 | 1.2.x apiserver 항목 5개를 골라 플래그 이름과 기본값 정리 |
| 3.7 Kubescape | 프레임워크별 스캔 (`cis-v1.12.0` 등) | kube-bench 결과와 같은 항목이 같은 판정인지 비교 |
| 4.4 kube-bench | 설치, 전체 실행, Summary 확인 | FAIL 3개를 고치고 `--check`로 재검사 |
| 6.1 Network Policy | dev-ns / prod-a / prod-b 3개 Namespace, default-deny와 허용 정책 | Egress default-deny + DNS 허용, AND/OR selector 비교 실험 |
| 6.2 Ingress Controller | ingress-nginx 설치 | `kubectl get ingressclass`로 클래스 이름 확인 |
| 6.3 Secure ingress | openssl 자체 서명 → TLS Secret → Ingress tls | ssl-redirect 308 확인, host 불일치 시 Fake Certificate 재현 |
| 9.2 Practice Steps 1·5 | CIS 평가·조치, Namespace 간 정책 | 1.7절 풀이를 직접 적용하고 curl로 검증 |

### 연습 체크리스트

- [ ] 6.1 랩 netpods.yaml로 3개 Namespace 환경 재현
- [ ] Egress default-deny 후 DNS만 허용하고 이름 해석 동작 확인
- [ ] 같은 항목 AND / 별도 항목 OR 정책을 각각 적용해 접근 결과 비교
- [ ] 169.254.169.254 차단 정책 적용 후 timeout 확인
- [ ] kube-bench FAIL 항목 3개 조치 (apiserver, kubelet, etcd 각 1개)
- [ ] apiserver 매니페스트를 일부러 깨뜨리고 crictl·로그로 복구
- [ ] kubelet `readOnlyPort`, anonymous, authorization 점검 curl
- [ ] TLS Ingress 구성과 HTTPS·리다이렉트 검증
- [ ] kubectl·kubelet 바이너리 sha512 검증, `/proc/<pid>/root`로 apiserver 해시 확인

## 7. 시험 실전 팁과 치트시트

시험은 2시간, 합격선 67%이며 문제마다 지정된 클러스터·노드가 다릅니다. 실수의 대부분은 지식 부족이 아니라 컨텍스트 착오와 재기동 확인 누락에서 나옵니다.

### 7.1 매 문제 시작 루틴

1. 문제 상단의 `kubectl config use-context <ctx>` 명령을 복사해 실행합니다.
2. 노드 작업이면 안내된 `ssh <node>` 후 `sudo -i`. 끝나면 `exit`로 반드시 원래 터미널로 돌아옵니다.
3. 대상 Namespace를 `-n`으로 명시합니다. 기본 Namespace를 바꿨다면 다음 문제 전에 되돌립니다.

### 7.2 터미널 설정

```bash
alias k=kubectl
export do="--dry-run=client -o yaml"
export now="--force --grace-period=0"
echo 'set ts=2 sw=2 et ai' >> ~/.vimrc
source <(kubectl completion bash); complete -o default -F __start_kubectl k
```

### 7.3 이 도메인 명령 모음

```bash
# NetworkPolicy
k explain networkpolicy.spec.ingress.from --recursive | less
k get ns --show-labels; k get pod -n <ns> --show-labels
k -n <ns> exec <pod> -- curl -s -m 3 <svc>.<ns>.svc.cluster.local

# Control plane 플래그 현재값 확인
ps -ef | grep kube-apiserver | tr ' ' '\n' | grep -- '--'
sudo grep -n -- '--profiling\|--authorization-mode' /etc/kubernetes/manifests/kube-apiserver.yaml

# Static Pod 재기동 확인
watch sudo crictl ps
k -n kube-system get pod -l component=kube-apiserver

# kubelet
sudo vim /var/lib/kubelet/config.yaml && sudo systemctl restart kubelet && systemctl status kubelet --no-pager

# Ingress TLS
k -n <ns> create secret tls <name> --cert=x.crt --key=x.key
curl -vk https://<host>:<port> --resolve <host>:<port>:<ip>

# 바이너리
echo "$(cat kubectl.sha512)  kubectl" | sha512sum --check
```

### 7.4 북마크해 둘 공식 문서 (시험 중 열람 가능 범위)

- [Network Policies](https://kubernetes.io/docs/concepts/services-networking/network-policies/)
- [Ingress — TLS](https://kubernetes.io/docs/concepts/services-networking/ingress/#tls)
- [kube-apiserver 플래그 레퍼런스](https://kubernetes.io/docs/reference/command-line-tools-reference/kube-apiserver/)
- [Kubelet Configuration (v1beta1)](https://kubernetes.io/docs/reference/config-api/kubelet-config.v1beta1/)
- [Kubelet authentication/authorization](https://kubernetes.io/docs/reference/access-authn-authz/kubelet-authn-authz/)
- [Ports and Protocols](https://kubernetes.io/docs/reference/networking/ports-and-protocols/)
- [Verify Signed Kubernetes Artifacts](https://kubernetes.io/docs/tasks/administer-cluster/verify-signed-artifacts/)

### 7.5 시간 관리

Cluster Setup 문제는 대부분 5~8분짜리입니다. apiserver가 2분 넘게 안 올라오면 백업으로 되돌리고 플래그를 파일 단위로 다시 확인합니다. 깨진 apiserver를 둔 채 다음 문제로 넘어가면 그 클러스터의 다른 문제까지 모두 잃습니다.

## 8. 자가 점검 문제

답을 가리고 먼저 손으로 YAML이나 명령을 써 본 뒤 해설을 펼쳐 비교합니다.

**Q1.** `podSelector: {}`, `policyTypes: [Ingress, Egress]`, 규칙 없음. 이 Namespace Pod에서 `curl google.com`은 어떻게 되나?

<details>
<summary>해설</summary>

DNS(53) egress까지 막혀 이름 해석 단계에서 실패합니다. 원인을 "외부 차단"이 아니라 "DNS 차단"으로 짚을 수 있어야 합니다.

</details>

**Q2.** `from:` 아래 `- namespaceSelector: {matchLabels: {team: a}}`와 바로 다음 줄 `  podSelector: {matchLabels: {app: db}}`(대시 없음). 허용 대상은?

<details>
<summary>해설</summary>

team=a Namespace 안의 app=db Pod만 (AND). podSelector 앞에 대시가 있으면 OR로 바뀌어 team=a의 모든 Pod + 같은 Namespace의 app=db Pod가 됩니다.

</details>

**Q3.** ns-a의 정책이 `- podSelector: {matchLabels: {app: x}}`로 Ingress를 허용합니다. ns-b의 app=x Pod는 접근 가능한가?

<details>
<summary>해설</summary>

불가. namespaceSelector 없는 podSelector는 정책과 같은 Namespace로 한정됩니다.

</details>

**Q4.** kube-bench가 `1.2.x Ensure that the --profiling argument is set to false`를 FAIL로 표시했습니다. 수정 절차는?

<details>
<summary>해설</summary>

매니페스트를 디렉터리 밖에 백업 → `/etc/kubernetes/manifests/kube-apiserver.yaml`의 `command`에 `--profiling=false` → apiserver Running 확인 → `kube-bench run --targets master --check <번호>` 재검사.

</details>

**Q5.** kubelet에 `protectKernelDefaults: true`를 넣고 재시작했더니 노드가 NotReady입니다. 원인과 확인 방법은?

<details>
<summary>해설</summary>

kubelet이 요구하는 커널 sysctl(예: `vm.overcommit_memory=1`, `kernel.panic=10`)과 노드 값이 달라 kubelet이 기동을 거부한 상태입니다. `journalctl -u kubelet`에서 sysctl 오류를 보고 sysctl 값을 맞추거나 문제 요구에 따라 설정을 되돌립니다.

</details>

**Q6.** HTTPS 응답 인증서가 "Kubernetes Ingress Controller Fake Certificate"입니다. 가능한 원인 세 가지는?

<details>
<summary>해설</summary>

TLS Secret이 Ingress와 다른 Namespace, `secretName` 오타, `tls.hosts`와 요청 Host(`--resolve`의 이름) 불일치.

</details>

**Q7.** `curl -sk https://<NodeIP>:10250/pods`가 Pod 목록 JSON을 돌려줍니다. 무엇을 고치나?

<details>
<summary>해설</summary>

`/var/lib/kubelet/config.yaml`에서 `authentication.anonymous.enabled: false`, `authorization.mode: Webhook`, 이어 `systemctl restart kubelet`. 재점검 결과가 401이어야 합니다.

</details>

**Q8.** Namespace `app`의 모든 Pod에서 메타데이터 서버만 막고 나머지 egress는 유지하는 정책을 쓰세요.

<details>
<summary>해설</summary>

4.2절 `deny-metadata` 정책. `ipBlock.cidr: 0.0.0.0/0`, `except: [169.254.169.254/32]`.

</details>

**Q9.** `/opt/bins`에 kubelet, kubectl, kube-proxy가 있고 v1.35.2 공식 sha512 목록이 주어졌습니다. 어떻게 처리하나?

<details>
<summary>해설</summary>

`sha512sum /opt/bins/*` 출력과 목록을 비교하고, 일치하지 않는 파일만 삭제합니다. 비교 전에 `--version`으로 버전이 맞는지 확인합니다.

</details>

**Q10.** apiserver 매니페스트 백업을 `/etc/kubernetes/manifests/kube-apiserver.yaml.bak`으로 두면 안 되는 이유는?

<details>
<summary>해설</summary>

kubelet은 디렉터리 안 파일을 모두 Static Pod로 해석하므로 같은 이름의 Pod가 충돌하거나 예상치 않은 apiserver가 뜰 수 있습니다.

</details>

## 출처

- [Linux Foundation — CKS Program Changes (2024-10-15 시행 역량 목록)](https://training.linuxfoundation.org/cks-program-changes/)
- [Kubernetes Docs — Network Policies](https://kubernetes.io/docs/concepts/services-networking/network-policies/)
- [Kubernetes Docs — Verify Signed Kubernetes Artifacts](https://kubernetes.io/docs/tasks/administer-cluster/verify-signed-artifacts/)
- LFS260 Labs V2026-07-30: Exercise 2.4, 3.7, 4.4, 6.1–6.3, 9.2
