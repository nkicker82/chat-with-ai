# Claude → GitHub 연동 방법 정리

## 거버닝 메시지
Claude 채팅에서 정리한 내용을 GitHub 리포지토리에 반영하는 방법은 크게 3가지이며, 자동화 수준과 설정 난이도에 따라 선택 가능함.

## 1. Claude Code 활용 (자동화 권장안)
- 로컬 터미널/VS Code/JetBrains에서 동작
- 로컬에 인증된 git 설정을 그대로 사용해 파일 생성 → commit → push 일괄 처리
- 반복적인 문서 업데이트 워크플로우 구성에 적합

## 2. Claude in Chrome 활용 (현재 사용 방식)
- claude.ai 채팅 내에서 브라우저 자동화로 GitHub 웹 UI 조작
- "Add file → Create new file" 페이지에 직접 내용 입력 후 commit
- 커밋(제출) 직전 항상 사용자 확인 절차를 거침
- 리포지토리 로그인 상태가 사전에 필요함 (자격 증명 입력은 사용자가 직접 수행)

## 3. 수동 업로드
- 채팅에서 파일만 생성 → 사용자가 직접 GitHub 웹 UI 드래그앤드롭 또는 git push
- 설정 불필요, 자동화 수준 최소

## 참고사항
- 이 세션(claude.ai 채팅) 자체의 코드 샌드박스는 네트워크 접근이 차단되어 있어 직접 git push 불가
- GitHub 전용 커넥터는 현재 미제공 (2026-09 기준, 레지스트리 검색 결과 없음)
- GitHub Pages로 게시하려면 별도로 Settings → Pages에서 소스 브랜치/폴더 지정 필요
