# ELK Stack Distribution 프로젝트 검토 보고서

## 요약

프로젝트는 ELK Stack을 3가지 방식(바이너리, Docker, Kubernetes)으로 배포하는 가이드 문서입니다. 전반적으로 **개발/테스트 환경에는 적합**하지만 **프로덕션 환경에는 심각한 보안 및 운영 문제**가 있습니다.

---

## 🔴 심각한 보안 문제 (Critical)

### 1. 보안 기능 비활성화 (모든 배포)
- **위치**: docker/instruction.md:49, k8s/instruction.md:64, prebuilt/instruction.md:63
- **문제**: `xpack.security.enabled: false`
- **영향**: 인증 없이 누구나 데이터 접근/수정 가능
- **권장**: 프로덕션 보안 설정 섹션 추가

### 2. 전체 인터페이스 노출 (모든 배포)
- **문제**: 모든 서비스가 `0.0.0.0`에 바인딩
- **영향**: 외부 네트워크에서 직접 접근 가능
- **권장**: `127.0.0.1` 또는 내부 IP만 바인딩

### 3. TLS/HTTPS 미적용 (모든 배포)
- **문제**: 모든 통신이 평문 HTTP
- **영향**: 데이터 도청 및 중간자 공격 위험
- **권장**: TLS 인증서 설정 가이드 추가

### 4. 플레이스홀더 비밀번호
- **위치**: k8s/instruction.md:149-150
- **문제**: `"your_secure_password"` 예제 그대로 사용 가능
- **권장**: 강력한 비밀번호 생성 및 Secret 관리 가이드 추가

### 5. NetworkPolicy 부재 (Kubernetes)
- **문제**: Pod 간 네트워크 격리 없음
- **영향**: 클러스터 내 모든 Pod가 ELK에 접근 가능
- **권장**: NetworkPolicy 매니페스트 추가

### 6. Privileged 컨테이너 (Kubernetes)
- **위치**: k8s/instruction.md:241-242
- **문제**: init container가 `privileged: true` 사용
- **권장**: 특정 capability만 부여 (`SYS_RESOURCE`)

---

## 🟠 운영 문제 (High)

### 7. SecurityContext 부재 (Kubernetes)
- **문제**: 모든 Pod에서 미설정
- **권장**: `runAsNonRoot`, `readOnlyRootFilesystem` 추가

### 8. PodDisruptionBudget 없음 (Kubernetes)
- **문제**: 클러스터 업그레이드 시 서비스 중단 가능
- **권장**: PDB 매니페스트 추가

### 9. 리소스 제한 부족 (Docker)
- **문제**: `deploy.resources.limits` 미설정
- **영향**: 컨테이너가 호스트 리소스 고갈 가능
- **권장**: CPU/메모리 제한 추가

### 10. 재시작 정책 없음 (Docker)
- **문제**: `restart_policy` 미정의
- **영향**: 컨테이너 충돌 시 자동 복구 안됨
- **권장**: `on-failure` 또는 `unless-stopped` 추가

### 11. 불완전한 헬스체크
- **Docker**: Logstash, OTel Collector 헬스체크 없음
- **Kubernetes**: livenessProbe 일부 누락
- **권장**: 모든 서비스에 완전한 헬스체크 추가

---

## 🟡 구조적 문제 (Medium)

### 12. 실제 설정 파일 부재
- **문제**: 모든 YAML이 instruction.md에 인라인으로 포함
- **영향**: 복사-붙여넣기 필요, 버전 관리 어려움
- **권장**: 실제 배포 파일 분리 (또는 Helm/Kustomize 사용)

### 13. 버전 하드코딩
- **문제**: `8.12.0`, `0.96.0` 등 직접 명시
- **권장**: 변수화 또는 환경변수 사용

### 14. RBAC 불완전 (Kubernetes)
- **문제**: OTel Collector만 ServiceAccount 정의
- **권장**: 모든 Pod에 최소 권한 RBAC 적용

### 15. Pod Affinity 없음 (Kubernetes)
- **문제**: 단일 노드 장애 시 모든 Pod 손실 가능
- **권장**: Anti-affinity 규칙 추가

### 16. 로그 로테이션 미설정 (Docker)
- **문제**: 로그 파일 무제한 증가
- **권장**: `json-file` 드라이버 + `max-size`, `max-file` 설정

---

## 📊 프로덕션 준비도

| 배포 방식 | 보안 | 운영 | 전체 |
|----------|------|------|------|
| Docker | 3/10 | 5/10 | 4/10 |
| Kubernetes | 3/10 | 6/10 | 4.5/10 |
| Prebuilt | 2/10 | 4/10 | 3/10 |

---

## 권장 개선 방향

### 즉시 조치 (P1)
1. 보안 활성화 가이드 섹션 추가
2. TLS/HTTPS 설정 가이드 추가
3. NetworkPolicy 매니페스트 추가 (K8s)
4. 리소스 제한 설정 추가 (Docker)

### 단기 개선 (P2)
5. SecurityContext 완성 (K8s)
6. PodDisruptionBudget 추가 (K8s)
7. 완전한 헬스체크 구성
8. 재시작 정책 추가 (Docker)

### 장기 개선 (P3)
9. 실제 배포 파일 분리 또는 Helm Chart 제공
10. 프로덕션 체크리스트 문서화
11. 백업/복구 가이드 추가

---

## 긍정적 요소

- 명확한 3가지 배포 방식 분리
- 일관된 ELK 8.12.0 버전 사용
- 코드 컨벤션 준수 (YAML 2-space, kebab-case 등)
- 기본적인 헬스체크 구성 (Elasticsearch, Kibana)
- OpenTelemetry Collector RBAC 기본 구성
- PVC/StorageClass 설정 (K8s)
