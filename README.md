# speckit-fastapi-claude-config

[speckit-fastapi-claude](https://github.com/cstpeace0191/speckit-fastapi-claude) 앱의 **Config 저장소**입니다.
클러스터가 따라야 할 상태(Kubernetes 매니페스트)만 담고, 앱 소스 코드는 담지 않습니다.
Argo CD가 이 저장소를 Pull해서 클러스터에 동기화합니다.

```
speckit-fastapi-claude-config
├── base/
│   ├── deployment.yaml          # todo-backend (FastAPI + React, 포트 3000)
│   ├── service.yaml             # ClusterIP :80 → 3000
│   └── kustomization.yaml
└── overlays/
    ├── dev/kustomization.yaml   # namespace todo-dev,  newTag: <main 최신 SHA>
    ├── stg/kustomization.yaml   # namespace todo-stg,  newTag: <dev에서 검증한 SHA>
    └── prod/kustomization.yaml  # namespace todo-prod, newTag: <stg에서 검증한 SHA>, LoadBalancer
```

## 배포 = `newTag` 한 줄 변경

| 단계 | 누가 | 무엇을 |
|---|---|---|
| dev | App 저장소 `build-and-promote.yml` (main 푸시 시 자동) | 새 이미지 빌드 → `overlays/dev` newTag PR |
| stg · prod | 사람이 수동 실행 (`Run workflow`, `target_env`, `image_tag`) | 검증된 태그로 `overlays/stg` · `overlays/prod` PR |
| 반영 | Argo CD | PR 머지를 감지해 해당 네임스페이스에 동기화 |
| 롤백 | 사람 | 승격 PR의 머지 커밋을 `git revert` |

## 로컬에서 확인

```bash
kubectl kustomize overlays/dev
kubectl kustomize overlays/prod | kubeconform -strict -summary
```

## Argo CD 연결 (예시)

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: todo-app
  namespace: argocd
spec:
  generators:
    - list:
        elements:
          - env: dev
          - env: stg
          - env: prod
  template:
    metadata:
      name: 'todo-{{env}}'
    spec:
      project: default
      source:
        repoURL: https://github.com/cstpeace0191/speckit-fastapi-claude-config.git
        targetRevision: main
        path: 'overlays/{{env}}'
      destination:
        server: https://kubernetes.default.svc
        namespace: 'todo-{{env}}'
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
        syncOptions:
          - CreateNamespace=true
```

## 참고

- 데이터베이스: `MYSQL_HOST`가 없으면 앱은 SQLite(`/data/todo.db`, emptyDir)를 사용합니다. 파드가 재시작되면 데이터가 사라지므로 replicas는 1로 둡니다. 데이터를 유지하려면 MySQL(App 저장소 `helm/todo-app` 참조)을 붙이세요.
- 초기 `newTag: prod`는 기존 `deploy.yml`이 올린 태그입니다. 첫 승격 PR부터 커밋 SHA로 바뀝니다.
