# Banana-Org 배포 아키텍처

## 레포지토리 구조

```
banana-org/
├── banana-deploy/              ← 배포 전용 repo (git)
│   ├── common-chart/           ← 공용 Helm chart
│   │   ├── Chart.yaml
│   │   ├── values.yaml         ← 차트 기본값
│   │   └── templates/
│   │       ├── _helpers.tpl
│   │       ├── deployment.yaml
│   │       ├── service.yaml
│   │       └── ingressroute.yaml
│   │
│   ├── app1/
│   │   ├── Dockerfile          ← app1 빌드용 (build context: ../../app1)
│   │   ├── common.yaml         ← app1 공통 Helm values (이미지명, 포트 등)
│   │   └── image/
│   │       ├── prod.yaml       ← prod 이미지 태그만 오버라이드
│   │       └── stage.yaml      ← stage 이미지 태그만 오버라이드
│   │
│   ├── app2/
│   │   ├── Dockerfile
│   │   ├── common.yaml
│   │   └── image/
│   │       ├── prod.yaml
│   │       └── stage.yaml
│   │
│   ├── helm-deploy.sh          ← 배포 스크립트
│   └── rollback-helm-deploy.sh ← 롤백 스크립트
│
├── app1/                       ← app1 소스코드 repo (git)
│   ├── index.html
│   ├── version.txt
│   └── bump-version.sh
│
└── app2/                       ← app2 소스코드 repo (git)
    ├── index.html
    ├── version.txt
    └── bump-version.sh
```

## 브랜치 & 버전 전략

### App Repo (app1, app2)

| 브랜치 | 환경 | 태그 형식 | 예시 |
|--------|------|----------|------|
| `master` | prod | `vX.Y.Z` | `v1.2.0` |
| `release` | stage | `vX.Y.ZrcN` | `v1.2.0rc1` |

### banana-deploy Repo

app repo에서 이미지 태그가 업데이트된 후 banana-deploy에도 커밋 & 태그 생성:

| 태그 형식 | 예시 | 의미 |
|----------|------|------|
| `{appname}-{env}-{version}` | `app1-prod-v1.2.0` | app1의 prod 이미지가 v1.2.0으로 변경됨 |
| `{appname}-{env}-{version}` | `app2-stage-v1.0.0rc1` | app2의 stage 이미지가 v1.0.0rc1로 변경됨 |

## 커밋 → 자동 배포 흐름

```
[app1 repo에서 커밋 발생]
         │
         ├── master 브랜치
         │     ├─ 1. git tag v1.2.0 (app1 repo)
         │     ├─ 2. docker build → app1:v1.2.0
         │     ├─ 3. banana-deploy/app1/image/prod.yaml 이미지 태그 자동 수정
         │     ├─ 4. banana-deploy repo 커밋 "app1 prod v1.2.0"
         │     └─ 5. banana-deploy repo 태그: app1-prod-v1.2.0
         │
         └── release 브랜치
               ├─ 1. git tag v1.2.0rc1 (app1 repo)
               ├─ 2. docker build → app1:v1.2.0rc1
               ├─ 3. banana-deploy/app1/image/stage.yaml 이미지 태그 자동 수정
               ├─ 4. banana-deploy repo 커밋 "app1 stage v1.2.0rc1"
               └─ 5. banana-deploy repo 태그: app1-stage-v1.2.0rc1
```

**자동 업데이트 방식**: app 커밋 시 post-commit hook이:
1. 브랜치 판별 (master → prod, release → stage)
2. banana-deploy repo의 `{appname}/image/{env}.yaml` 이미지 태그 수정
3. banana-deploy repo에서 커밋
4. `{appname}-{env}-{version}` 형식으로 태그

## 배포 스크립트

### helm-deploy.sh — 배포

```bash
bash helm-deploy.sh {appname} {env}
# 예시
bash helm-deploy.sh app1 prod
bash helm-deploy.sh app2 stage
```

내부 동작:
1. `{appname}/common.yaml` + `{appname}/image/{env}.yaml` 을 values로 사용
2. namespace `{appname}-{env}` 에 배포
3. `helm upgrade --install` 사용 (최초 배포 / 업데이트 모두 대응)

실행되는 명령:
```bash
helm upgrade --install {appname} ./common-chart \
  -f ./{appname}/common.yaml \
  -f ./{appname}/image/{env}.yaml \
  --set env={env} \
  --namespace {appname}-{env} --create-namespace
```

### rollback-helm-deploy.sh — 롤백

```bash
bash rollback-helm-deploy.sh {banana-deploy-tag}
# 예시
bash rollback-helm-deploy.sh app1-prod-v1.0.0
```

내부 동작:
1. 태그 파싱 → appname(`app1`), env(`prod`), version(`v1.0.0`) 추출
2. 해당 태그 시점의 `{appname}/image/{env}.yaml` 파일 내용을 가져옴 (`git show {tag}:{path}`)
3. 현재 브랜치에 해당 파일 덮어쓰기
4. 새 커밋 생성: `"rollback: app1 prod to v1.0.0"`
5. banana-deploy repo 태그: `app1-prod-v1.0.0` (이미 존재하면 스킵)
6. 자동으로 `helm-deploy.sh {appname} {env}` 실행

```
롤백 흐름 예시: app1-prod 가 v1.0.0 → v1.1.0 → v1.2.0 인 상태에서

bash rollback-helm-deploy.sh app1-prod-v1.0.0

결과:
  - app1/image/prod.yaml 이 v1.0.0 시점으로 되돌아감
  - 새 커밋이 최신 HEAD에 생성 (히스토리 보존, reset 아님)
  - helm-deploy.sh app1 prod 자동 실행 → v1.0.0 으로 재배포
```

## Namespace 규칙

`{appname}-{env}` 형식:
- `app1-prod`
- `app1-stage`
- `app2-prod`
- `app2-stage`

## IngressRoute 라우팅 규칙

| URL 패턴 | 대상 |
|----------|------|
| `localhost/prod/app1` | app1 prod 환경 |
| `localhost/stage/app1` | app1 stage 환경 |
| `localhost/prod/app2` | app2 prod 환경 |
| `localhost/stage/app2` | app2 stage 환경 |

IngressRoute match 규칙: `Host(`localhost`) && PathPrefix(`/{env}/{appname}`)`

## Values 파일 역할 (최소 인자)

### common.yaml (앱별 공통 — 변경 거의 없음)
```yaml
image:
  repository: app1       # 이미지 이름
containerPort: 80
```

### image/prod.yaml (환경별 — 이미지 태그만)
```yaml
image:
  tag: "v1.2.0"
```

### image/stage.yaml
```yaml
image:
  tag: "v1.2.0rc1"
```

나머지 값(namespace, ingress path, env 등)은 `helm-deploy.sh`가 `--set`으로 자동 전달.

## Docker 빌드

```bash
# banana-org/ 루트에서 실행
docker build -f banana-deploy/app1/Dockerfile -t app1:v1.2.0 ./app1
docker build -f banana-deploy/app2/Dockerfile -t app2:v1.0.0 ./app2
```

Dockerfile은 nginx 기반으로 static HTML을 서빙.

## 태그 히스토리 예시

```
banana-deploy repo tags:
  app1-prod-v0.1.0      ← 최초 배포
  app1-prod-v0.2.0
  app1-prod-v1.0.0
  app1-stage-v1.0.0rc1
  app1-stage-v1.0.0rc2
  app2-prod-v0.1.0
  app2-stage-v0.1.0rc1
```

이 태그들이 rollback-helm-deploy.sh 의 입력으로 사용됨.
롤백 시 해당 태그 시점의 yaml을 복원하여 새 커밋으로 만들고 재배포.
