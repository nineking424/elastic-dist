# ELK Stack Distribution Project

## 프로젝트 개요
ELK(Elasticsearch, Logstash, Kibana) 스택 배포를 위한 매니페스트 및 설정 파일 모음

## 프로젝트 구조
```
elastic-dist/
├── prebuilt/     # 사전 빌드된 바이너리 설치 스크립트
├── docker/       # Docker Compose 기반 설치
└── k8s/          # Kubernetes 매니페스트
```

## 설치 방법
1. **prebuilt**: 공식 바이너리를 직접 다운로드하여 설치
2. **docker**: Docker Compose를 사용한 컨테이너 기반 설치
3. **k8s**: Kubernetes 클러스터에 배포

## 코드 컨벤션
- YAML 파일: 2 space 들여쓰기
- 환경변수: UPPER_SNAKE_CASE
- Kubernetes 리소스명: kebab-case
- Docker 서비스명: lowercase with underscores

## ELK 버전
- 기본 버전: 8.x (최신 안정 버전 사용 권장)

## 주요 구성요소
- **Elasticsearch**: 분산 검색 및 분석 엔진
- **Logstash**: 데이터 처리 파이프라인
- **Kibana**: 데이터 시각화 대시보드
- **Filebeat** (선택): 로그 파일 수집기

## Git 워크플로우

### 기본 원칙
- 모든 코드 및 문서 변경사항은 반드시 Git에 커밋하고 push까지 완료해야 함
- 작업 단위별로 논리적인 커밋 분리

### 커밋 메시지 컨벤션
형식: `<type>: <description>`

**타입 종류:**
- `feat`: 새로운 기능 추가
- `fix`: 버그 수정
- `docs`: 문서 변경
- `refactor`: 코드 리팩토링
- `chore`: 빌드, 설정 파일 변경

**작성 규칙:**
- 영어로 작성
- 소문자로 시작
- 명령형 현재 시제 사용 (예: "add", "fix", "update")
- 50자 이내로 간결하게

**예시:**
- `feat: add kubernetes deployment manifests`
- `fix: correct elasticsearch port configuration`
- `docs: update installation guide`
