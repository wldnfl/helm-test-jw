# validateSource type 테스트용 예시 소스

ArgoCD `getAppDetails`가 소스 구조로 타입을 자동 판별한다.
한 repo에 push 후, validateSource 의 `path` 만 바꿔 세 타입을 모두 테스트할 수 있다.

| path        | 판별 근거                | 기대 type   |
|-------------|-------------------------|-------------|
| `/helm`     | `Chart.yaml` 존재        | `Helm`      |
| `/kustomize`| `kustomization.yaml` 존재| `Kustomize` |
| `/directory`| 생 매니페스트(둘 다 없음) | `Directory` |

## 사용법

1. 이 폴더 전체를 테스트용 git repo(예: helm-test-jw)에 push
2. validateSource 호출 시 path 만 변경:

```graphql
mutation {
  deploy {
    validateSource(input: {
      url: "<repo url>", branch: "main",
      path: "/kustomize",            # /helm, /directory 로 바꿔가며
      svcName: "<정상 svc>"
    }) { valid type name resolvedRef reason }
  }
}
```

## 주의 — 검증과 실제 배포는 다름

- **검증(validateSource)**: getAppDetails 가 helm 을 강제하지 않으므로 세 타입 모두 정확히 판별된다.
- **실제 배포**: 현재 `ArgocdClient.createSource()` 가 항상 `source.helm`(releaseName)을 붙여 ArgoCD 가 무조건 Helm 으로 렌더한다.
  → 따라서 Kustomize/Directory 소스는 **타입 판별은 되지만 실제 배포(sync)는 실패**한다. 비-Helm 배포를 지원하려면 createSource 가 Chart.yaml 있을 때만 helm 을 붙이도록 바꿔야 한다.
