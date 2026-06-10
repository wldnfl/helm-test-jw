# 멀티 차트(Umbrella) 배포 예시 — `sample-umbrella`

> "멀티 레포 차트 = root chart + 그 아래 서브차트" 구조가 우리 플랫폼에서 어떻게 배포되는지 보여주는 예시.
> 사내 GitLab 에 통째로 clone/저장하는 방식을 전제로, **서브차트를 `charts/` 에 vendoring** 한 형태.

## 0. 배포용 Helm Chart 작성 규칙 (필수)

DevOpsIT 의 `HELM_CHART_GIT` 모드로 배포하려면 차트를 아래 규칙대로 구성해야 한다.
검증(`validateChart`)은 chartPath 의 **root `Chart.yaml` 존재 + name/version** 만 확인한다 — 통과해도 "배포 가능"까지 보장하진 않으므로 아래 규칙은 사용자 책임이다.

1. **root 에 `Chart.yaml`** — 배포할 디렉토리 최상단에 `Chart.yaml` 이 있어야 한다. 티켓의 chartPath 를 그 위치로 지정한다 (repo 루트면 `/`, 하위면 `/path/to/chart`).
2. **여러 컴포넌트를 한 번에 = umbrella** — 하나의 ArgoCD Application(=release)으로 묶어 배포하려면 root `Chart.yaml` + 서브차트를 `charts/` 아래에 둔다. Helm 은 umbrella 를 "차트 1개"로 본다.
3. **서브차트는 `charts/` 에 실물 vendoring** — 외부 helm repo 의존(`dependencies: repository: https://...`)은 사내망 fetch 실패 위험이 있어 금지. 차트를 사내 GitLab 에 통째로 올리면 자연히 충족된다. (`charts/` 는 Helm 스펙상 고정 위치)
4. **서브차트 설정은 root `values.yaml`** — `global:`(공통) + `<서브차트이름>:`(개별) 키로 내려준다.
5. **차트는 자기 `values.yaml` 로 완결** — 플랫폼은 git 차트의 `values.yaml` 을 건드리지 않고 그대로 배포한다. 배포에 필요한 모든 설정이 차트 안에 있어야 하고, 외부 노출(Ingress)·Service·ConfigMap 등 필요한 k8s 리소스도 **차트가 직접 정의**해야 한다 (플랫폼이 만들어주지 않음).
6. **독립 차트 여러 개를 따로 배포** — umbrella 로 묶지 않을 거면 **release 를 N 개**로 만들고 각 release 의 chartPath 를 각 차트 디렉토리로 지정한다. (1 release : 1 chart : 1 app)

---

## 1. 디렉토리 구조

```
sample-umbrella/
├── Chart.yaml                      # root(umbrella) 차트 — type: application
├── values.yaml                     # ★ 여기서 global + 서브차트별 설정을 모두 내려줌
└── charts/                         # 서브차트 "실물"을 vendoring (외부 repo 접근 불필요)
    ├── frontend/                   # Deployment + Service
    ├── backend/                    # Deployment + Service (+ env)
    ├── db/                         # StatefulSet + Service + Secret (PostgreSQL, 영속볼륨)
    ├── jenkins/                    # Deployment + Service + PVC (오픈소스 CI)
    └── argocd/                     # Deployment + Service (오픈소스 CD, server 단순화)
        ├── Chart.yaml
        ├── values.yaml             # 서브차트 기본값 (root 가 덮어씀)
        └── templates/all.yaml
```

> 서브차트마다 워크로드 종류가 달라도(=Deployment / StatefulSet / +PVC / +Secret) 한 umbrella 안에 공존한다.
> jenkins·argocd 는 **개념 예시로 단순화**한 것(실제 upstream 차트는 다컴포넌트) — "오픈소스도 같은 방식으로 서브차트가 된다"를 보여주는 용도.

**Helm 관점에서 이건 "차트 1개"다.** root `Chart.yaml` 하나를 렌더링하면 `charts/` 안의 서브차트들이 **함께** 펼쳐진다 → **ArgoCD Application 1개 = 우리 플랫폼의 release 1개**로 배포된다. 즉 `1 release : 1 (umbrella) chart : 1 app` 모델과 충돌하지 않는다.

## 2. umbrella 의 핵심 — root `values.yaml` 이 서브차트를 제어

```yaml
global:                 # 모든 서브차트가 .Values.global 로 공유
  imageRegistry: harbor.sre.local
frontend:               # charts/frontend 의 values 를 덮어씀 (서브차트 이름이 키)
  replicaCount: 2
  image: { repository: myteam/frontend, tag: "1.4.0" }
backend:                # charts/backend 의 values 를 덮어씀
  replicaCount: 3
  image: { repository: myteam/backend, tag: "2.1.0" }
```

이 "서브차트 이름 키 + `global:`로 값 전파" 가 umbrella 패턴의 생명이다. → **이 부분이 아래 §4 의 플랫폼 주의점과 직결된다.**

## 3. 우리 플랫폼 배포 흐름에 매핑

1. 팀이 작성한 umbrella 차트를 **사내 GitLab repo** 에 push (또는 외부에서 clone 해 사내로 미러).
2. 배포 티켓 생성 시 `validateChart(url, branch, chartPath)` 호출
   - `chartPath` = umbrella 의 **root 경로** (이 예시를 repo 루트에 두면 `/`, 하위에 두면 `/sample-umbrella`)
   - 검증은 **root `Chart.yaml` 의 name/version 만** 확인 → 통과. (서브차트는 차트의 일부라 따로 검증 안 함)
3. 응답 `resolvedRef`(commit SHA) 를 deploy 티켓의 `chartSource.resolvedChartRef` 로 그대로 전달.
4. 승인 → finalize → `cloneChartAtPinnedRef`(SHA checkout) → `chartPath` 디렉토리(= `charts/` 포함)를 staging repo 에 복사 → ArgoCD 가 Helm 으로 부모+자식 렌더링.

> 멀티 **레포** (서브차트가 각각 별도 git repo) 인 경우: 우리는 "clone 해서 사내 GitLab 에 저장" 하므로, **서브차트들을 root 의 `charts/` 아래로 모아 한 repo 로 vendoring** 해서 올리면 위 흐름 그대로 동작한다.

## 4. ⚠️ 현재 코드(jiwoo `a79864d`)에서의 주의점 — 그대로는 안 됨

현재 `commitAndPushForDeployment` 는 HELM_CHART_GIT 차트도 기존 `writeHelmChart` 를 타서 **차트의 `values.yaml` 을 플랫폼 HelmValues(JSONB)로 덮어쓴다.** umbrella 에서는 이게 치명적:

- root `values.yaml` 의 `frontend:` / `backend:` / `global:` 가 **통째로 사라지고** 플랫폼 스키마(`deployments: [...]`)로 교체됨
- → 서브차트들은 자기 `charts/<name>/values.yaml` **기본값으로만** 렌더링되거나, `global` 참조가 비어 의도와 다르게 배포됨
- (이 예시는 그 상황에서도 죽지 않게 `global` 접근에 `default dict` 가드를 넣어뒀지만, **원하는 설정은 반영 안 됨**)

**필요한 수정 (작음):** finalize 분기에서 `source_type == HELM_CHART_GIT` 이면 **values.yaml 덮어쓰기·이미지 주입을 skip** 하고 차트를 가공 없이 그대로 push. (= 5차 설계 D5-5 "가공 없이 통째 복사". `docs/helm-chart-git-source-impl-spec.md` §5 참조)

수정 후에는 root `values.yaml` 이 보존되어 umbrella 가 의도대로 배포된다.

## 5. 멀티 차트 정리

| 케이스 | 지원 |
|---|---|
| umbrella (root + `charts/` 서브차트) = 차트 1개 → release 1개 | ✅ (단, §4 values skip 필요) |
| 서로 다른 차트 N개를 한 티켓에 = **release N개** | ✅ (release 마다 `chartSource` 지정, 이미 동작) |
| 한 release(app 1개)에 독립 차트 여러 개 | ❌ (Helm/ArgoCD 모델상 불가 — umbrella 로 묶거나 release 분리) |

## 6. 로컬 검증 (참고)

```bash
# 렌더링 확인 (서브차트 5개가 모두 함께 나오는지)
helm template myrel docs/examples/helm-umbrella-sample/ | grep -E "^kind:|name: myrel"
# 기대: Deployment 4 · StatefulSet 1 · Service 5 · Ingress 3 · PVC 1 · Secret 1
#   frontend/jenkins/argocd = +Ingress, db = StatefulSet+Secret, jenkins = +PVC
```

> ⚠️ `helm lint` 은 `[ERROR] chart metadata is missing these dependencies: ...` 를 낸다.
> 이는 **서브차트를 charts/ 에 vendoring 하면서 dependencies 를 선언하지 않은** (의도된) 패턴에 대한 helm lint 의 알려진 동작이다.
> **`helm template`/ArgoCD 렌더에는 영향 없음** (위 렌더 결과가 증거). dependencies 를 선언하면 lint 는 통과하지만
> Chart.lock 동기화 + ArgoCD `helm dependency build` 부담이 생기므로, 손으로 수정하는 예시에서는 vendoring 을 택했다.

## 7. 외부 접속 (브라우저 테스트)

ArgoCD 의 sync/Healthy 와 "브라우저로 접속" 은 **별개**다. 기본 Service 는 `ClusterIP` 라 **클러스터 내부 전용** → 브라우저(외부)에서는 못 친다. 노출 방법:

| 방법 | 접속 | 전제 |
|---|---|---|
| **Ingress** (이 예시 frontend 에 포함, `ingress.enabled: true`) | `http://frontend.sample.local/` | 클러스터에 **Ingress Controller(nginx 등) 설치** + DNS/hosts 가 host → Ingress LB IP 매핑. 없으면 Ingress 객체만 생기고 안 열림 |
| `kubectl port-forward svc/<release>-frontend 8080:80` | `http://localhost:8080` | **무설정 빠른 확인** |
| Service `type: NodePort` / `LoadBalancer` | 노드IP:포트 / 외부IP | 컨트롤러 불필요 / 클라우드·MetalLB |

> **git 모드는 플랫폼이 Ingress 를 만들어주지 않는다** (BUNDLED 폼의 역할이었음). 그래서 외부 노출이 필요하면 **차트가 Ingress 를 직접** 가져야 한다 → 그래서 frontend 차트에 `templates/ingress.yaml` 을 넣었다. 다른 서브차트(jenkins/argocd 등)도 외부 노출하려면 같은 방식으로 각 차트에 Ingress 를 추가하면 된다.
