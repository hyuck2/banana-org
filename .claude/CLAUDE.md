# Project Settings

## Auto-commit
- 작업 단위(task)가 완료될 때마다 반드시 git commit을 수행한다.
- 커밋 메시지는 변경 내용을 간결하게 요약한다.

## Repository Structure
- banana-org (root): 상위 repo
- admin-dashboard, banana-deploy, app1, app2: git submodule
- submodule 내부 변경 시 해당 submodule repo에서 먼저 커밋하고, 루트 repo에서 submodule 참조를 업데이트 커밋한다.

## 배포 프로세스 (Kind 클러스터)

### Kind 클러스터 정보
- 클러스터: `cluster-dev`, `cluster-staging` (현재 주로 `cluster-staging` 사용)
- Traefik NodePort: `30080` (HTTP)
- 프론트엔드 접속: `http://localhost:30080/{env}/{appname}/`

### Docker 이미지 빌드 & Kind 로드
```bash
# 루트(banana-org)에서 실행 — Dockerfile은 banana-deploy 안에 있음
docker build -t {appname}:{tag} -f banana-deploy/{appname}/Dockerfile .
kind load docker-image {appname}:{tag} --name cluster-staging
```

### Helm 배포 (반드시 helm-deploy.sh 사용)
```bash
# 1. banana-deploy에서 image tag 업데이트
#    banana-deploy/{appname}/image/{env}.yaml 의 image.tag 수정
# 2. banana-deploy에서 커밋
# 3. helm-deploy.sh로 배포
cd banana-deploy
bash helm-deploy.sh {appname} {env}
# 예: bash helm-deploy.sh admin-dashboard-backend prod
#     bash helm-deploy.sh admin-dashboard-frontend prod
```
- `helm-deploy.sh`는 `common.yaml` + `image/{env}.yaml`을 합쳐서 `helm upgrade --install` 수행
- 직접 `helm upgrade` 하지 말 것 — 반드시 스크립트 사용
- 롤백: `bash rollback-helm-deploy.sh {appname}-{env}-{tag}`

### 배포 순서 요약 (admin-dashboard 예시)
1. admin-dashboard 코드 수정 → admin-dashboard submodule에서 커밋
2. Docker 이미지 빌드 (backend & frontend)
3. Kind에 이미지 로드
4. banana-deploy에서 image tag yaml 업데이트 → 커밋
5. `bash helm-deploy.sh` 로 배포
6. 루트 repo에서 submodule 참조 업데이트 커밋

## admin-dashboard 아키텍처

### 백엔드 (Python FastAPI)
- 경로: `admin-dashboard/backend/`
- 포트: 8000 (uvicorn)
- 인증: base64 JSON 토큰 (JWT 아님, `deps.py` 참고)
- K8s 접근: kubeconfig Secret 마운트 (`/root/.kube/config`)
- DB: MySQL (`admin-db` namespace)

### 프론트엔드 (React + Vite + TypeScript)
- 경로: `admin-dashboard/src/`
- 빌드: `npm run build` (tsc + vite)
- 배포: nginx (multi-stage Docker build)
- base: `./` (상대경로) — Traefik sub-path 라우팅과 호환
- API 프록시: nginx `/api/` → backend service
- WebSocket 프록시: nginx `/api/ws/` → backend `/k8s/ws/` (k8s 라우터 prefix 포함)

### Helm Chart
- 공통 차트: `banana-deploy/common-chart/`
- 앱별 설정: `banana-deploy/{appname}/common.yaml` + `image/{env}.yaml`

### WebSocket 프록시 (nginx)
- `/api/ws/ssh` → backend `/servers/ws/ssh` (SSH 터미널)
- `/api/ws/ansible` → backend `/ansible/ws/ansible` (Ansible 로그)
- `/api/ws/` → backend `/k8s/ws/` (K8s exec)

### 현재 이미지 버전
- admin-dashboard-backend: **v0.1.7**
- admin-dashboard-frontend: **v0.2.5**

## Phase 2 K8s 관리 기능 (구현 완료)

### 기본 기능
- 클러스터 목록/상태, 노드 목록, 네임스페이스 목록, Deployment 목록
- Deployment 액션: Scale, Restart, Logs

### Phase 2 보강 (backend v0.1.7 / frontend v0.2.5)
- Deployment `updatedAt` 컬럼 (상대시간 표시)
- Deployment Edit (YAML 편집) — GET/PUT `/yaml` 엔드포인트
- Deployment Exec (파드 쉘) — WebSocket `/k8s/ws/exec` + xterm.js
- Node 목록: 총 CPU cores / 메모리 표시, IP 컬럼, 컬럼 정렬, 검색
- Deployments 탭: 클러스터 전체 Deployment 목록 (이름+네임스페이스 검색, 액션)
- 네임스페이스 클릭 → Deployments 탭으로 이동 (해당 네임스페이스 검색 자동입력)

## Phase 3 서버 관리 기능 (구현 완료)

### 서버/그룹 관리
- 서버 CRUD: hostname, IP, SSH 포트, 사용자, 비밀번호(Fernet 암호화)
- 서버 그룹 CRUD: 그룹별 서버 분류
- 대량 등록: 클립보드 붙여넣기 (TSV/CSV 자동감지) → 편집 가능 테이블 → 일괄 등록

### SSH 기능
- SSH 접속 테스트: 단건/대량 (paramiko)
- 웹 SSH 터미널: WebSocket `/servers/ws/ssh` + xterm.js
- 그룹 명령 실행: 그룹 내 전 서버 SSH 병렬 실행 → 서버별 결과 표시

### 메트릭 소스 (Prometheus/VictoriaMetrics)
- 메트릭 소스 CRUD + 연결 테스트
- 타겟 목록 + 등록 서버 IP 매칭
- 서버 메트릭 조회: CPU%, Mem%, Disk% (PromQL, Canvas 차트)

### Ansible 관리
- Playbook CRUD (YAML 편집)
- Inventory CRUD + 그룹 기반 자동생성
- Playbook 실행 (subprocess) + 실시간 로그 스트리밍 (WebSocket)
- 실행 이력 조회

### DB 테이블 (Phase 3)
- server_groups, servers, metric_sources
- ansible_playbooks, ansible_inventories, ansible_executions

### 개발 환경
- `docker-compose.dev.yml`: SSH 서버 3대 + node-exporter 2대 + Prometheus
- 테스트: testuser/testpass, 포트 2221-2223
