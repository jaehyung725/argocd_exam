# ArgoCD 테스트 요약

## 1. ArgoCD 핵심 개념

ArgoCD는 GitOps 기반 Kubernetes 배포 도구입니다.

```text
GitHub의 YAML = Kubernetes 클러스터가 유지해야 할 정답
```

즉, Kubernetes 리소스를 직접 계속 수정하는 대신 Git에 변경사항을 올리면 ArgoCD가 클러스터에 자동 반영합니다.

```text
YAML 수정 → git commit → git push → ArgoCD Sync → Kubernetes 반영
```

## 2. 이번 테스트 구조

ArgoCD가 바라보는 GitHub 저장소입니다.

```text
https://github.com/jaehyung725/argocd_exam.git
```

ArgoCD가 배포 기준으로 사용하는 경로입니다.

```text
app_exam-k8s/k8s
```

주요 파일입니다.

```text
app_exam-k8s/argocd/echo-hostname.yaml  # ArgoCD Application
app_exam-k8s/k8s/deployment.yaml        # Deployment
app_exam-k8s/k8s/svc.yaml               # Service
```

## 3. Application 핵심 설정

```yaml
source:
  repoURL: https://github.com/jaehyung725/argocd_exam.git
  targetRevision: main
  path: app_exam-k8s/k8s

syncPolicy:
  automated:
    prune: true
    selfHeal: true
```

의미는 다음과 같습니다.

```text
automated  Git 변경을 자동 반영
prune      Git에서 삭제된 리소스를 클러스터에서도 삭제
selfHeal   kubectl로 직접 바꾼 내용을 Git 상태로 자동 복구
```

## 4. 현재 확인한 테스트 결과

Application 생성:

```bash
kubectl apply -f app_exam-k8s/argocd/echo-hostname.yaml
```

ArgoCD 상태 확인:

```bash
kubectl -n argocd get application echo-hostname
```

정상 결과:

```text
echo-hostname   Synced   Healthy
```

배포 리소스 확인:

```bash
kubectl get deploy,svc,pod -n default
```

서비스 호출 확인:

```bash
kubectl run curl-test --rm -it --image=curlimages/curl --restart=Never -- \
  curl http://echo-hostname-svc.default.svc.cluster.local:8080
```

응답이 오면 Service → Pod 연결이 정상입니다.

## 5. Git Push 자동 배포 테스트

`deployment.yaml`에서 replica 수를 변경합니다.

```yaml
replicas: 2
```

GitHub에 push합니다.

```bash
cd ~/argocd_exam
git add app_exam-k8s/k8s/deployment.yaml
git commit -m "test: change replicas"
git push origin main
```

확인:

```bash
kubectl get deploy hostname-deployment
```

결과가 Git에 적힌 replica 수와 같으면 성공입니다.

```text
GitHub 변경 → ArgoCD 감지 → Kubernetes 자동 반영
```

## 6. Self-Heal 테스트

클러스터에서 직접 replica 수를 강제로 변경합니다.

```bash
kubectl scale deployment hostname-deployment --replicas=1
kubectl get deploy hostname-deployment
```

처음에는 `1/1`로 줄어듭니다.

잠시 후 다시 확인합니다.

```bash
kubectl get deploy hostname-deployment
```

Git에 `replicas: 2`가 있으면 다시 `2/2`로 복구됩니다.

이 테스트로 확인한 내용:

```text
누군가 kubectl로 직접 변경해도 ArgoCD가 Git 상태로 자동 복구한다
```

## 7. Prune 확인

설정 확인:

```bash
kubectl -n argocd get application echo-hostname -o yaml | grep -A4 syncPolicy
```

정상 설정:

```yaml
syncPolicy:
  automated:
    prune: true
    selfHeal: true
```

`prune: true`는 Git에서 삭제된 리소스를 Kubernetes에서도 삭제한다는 뜻입니다.

운영 환경에서는 실수로 YAML을 삭제하면 실제 리소스도 삭제될 수 있으므로 주의가 필요합니다.

## 8. ArgoCD UI 접속

포트포워딩:

```bash
kubectl -n argocd port-forward svc/argocd-server 8080:443
```

브라우저 접속:

```text
https://localhost:8080
```

로그인 ID:

```text
admin
```

초기 비밀번호 확인:

```bash
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d
```

## 9. UI에서 확인할 내용

ArgoCD UI에서 `echo-hostname` Application을 보면 됩니다.

확인 포인트:

```text
Status: Healthy / Synced
Repository: https://github.com/jaehyung725/argocd_exam.git
Path: app_exam-k8s/k8s
Namespace: default
```

Application 상세 화면에서는 다음 리소스 관계를 볼 수 있습니다.

```text
Application
 ├── Service echo-hostname-svc
 └── Deployment hostname-deployment
      └── ReplicaSet
           └── Pods
```

## 10. 팀원 시연용 시나리오

### 시나리오 1: Git Push 자동 배포

1. `deployment.yaml`의 `replicas` 값을 변경
2. commit / push
3. ArgoCD UI에서 `echo-hostname` 상태 확인
4. `kubectl get deploy hostname-deployment`로 replica 수 확인

핵심 메시지:

```text
Git에 올린 변경이 Kubernetes에 자동 반영된다
```

### 시나리오 2: Self-Heal 자동 복구

1. 터미널에서 강제로 replica 수 변경

```bash
kubectl scale deployment hostname-deployment --replicas=1
```

2. 잠시 후 다시 확인

```bash
kubectl get deploy hostname-deployment
```

3. Git에 정의된 replica 수로 자동 복구되는지 확인

핵심 메시지:

```text
클러스터를 직접 바꿔도 ArgoCD가 Git 상태로 되돌린다
```

## 11. 결론

ArgoCD에서 가장 중요한 원칙은 다음입니다.

```text
Git = 정답
Cluster = Git을 따라가는 실행 환경
ArgoCD = Git과 Cluster를 맞춰주는 도구
```
