# Project Settings

## Auto-commit
- 작업 단위(task)가 완료될 때마다 반드시 git commit을 수행한다.
- 커밋 메시지는 변경 내용을 간결하게 요약한다.

## Repository Structure
- banana-org (root): 상위 repo
- banana-deploy, app1, app2: git submodule
- submodule 내부 변경 시 해당 submodule repo에서 먼저 커밋하고, 루트 repo에서 submodule 참조를 업데이트 커밋한다.
