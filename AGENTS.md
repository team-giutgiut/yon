# AGENTS.md

Y:ON 프로젝트의 공통 AI 개발 지침이다. 팀원과 AI 개발 도구는 작업 영역에 관계없이 본 문서를 기준으로 작업한다.

## 1. Project

Y:ON은 청년정책·직업훈련 정보를 통합 수집·정규화하고, 사용자 조건에 따른 신청 가능성 판정, 검색·추천, 변경 추적·알림을 제공하는 서비스다.

프로그래머스 데브코스 BE11 최종 팀 프로젝트를 기반으로 지속적인 개선과 운영을 진행한다.

단순한 기능 추가보다 실제 사용자 요구사항과 운영 문제를 해결하는 것을 우선한다.

## 2. Architecture

여러 서비스를 하나의 저장소에서 관리하는 모노레포 구조를 유지한다. `backend/`, `data-service/`, `notification-service/`는 각각 독립 Gradle 프로젝트이며 루트 빌드는 없다.

| 경로 | 역할 |
| --- | --- |
| `backend/` | 사용자 API, 인증, 신청 가능성 판정, 검색·추천, 변경 추적·이벤트 발행, 관리자 API |
| `data-service/` | 외부 API 수집, 원본 데이터 관리, 정규화 |
| `notification-service/` | 이벤트 소비, 인앱 알림, 웹 푸시 |
| `frontend/` | React 기반 사용자·관리자 화면 |
| `contracts/` | 서비스 간 공통 계약 (gRPC proto, 메시지 JSON) |
| `database/` | 수동 적용 DDL 및 운영 SQL |
| `docker/` | 로컬 실행 환경 |
| `deploy/` | 서버별 배포 및 운영 설정 |
| `infra/` | Terraform 인프라 코드 |
| `perf/` | 성능·부하 테스트 |
| `scripts/` | CI 테스트 리포트·정책, E2E 보조 스크립트 |
| `tools/` | 수동 점검 도구 |
| `docs/` | 설계·운영 문서 |

### 기술 스택

- Java 25, Spring Boot 4.1.1, PostgreSQL 16, React(Vite, TypeScript)
- JSON 처리는 Jackson 3(`tools.jackson.*`)을 사용한다. `com.fasterxml.jackson.databind`를 import하지 않는다.
- Spring Boot 3 이하 기준의 예시 코드를 그대로 가져오지 않는다. 기존 import와 의존성을 먼저 확인한다.

### 서비스 간 연결

- backend → data-service: gRPC(`contracts/proto/collection/v1/collection.proto`). 수집 관리자 명령·조회에만 사용한다.
- 공고·조건·지역 데이터: data-service가 쓰고 backend는 같은 DB에서 읽는다.
- backend → notification-service: Outbox 발행(HTTP 또는 SQS). 메시지 형식은 `contracts/*.json`을 따른다.
- data-service → notification-service: 수집 완료 관리자 알림(SQS).

### 원칙

- 서비스 간 책임과 의존성 경계를 유지한다.
- 공유 데이터 구조나 서비스 간 계약을 변경할 때 영향받는 서비스를 함께 확인한다.
- 상세 아키텍처와 기술적 결정은 관련 코드와 `docs/`를, 배포·운영 절차는 `deploy/README.md`를 확인한다.

## 3. Domain Rules

- 외부 API 원본 데이터와 정규화 데이터를 구분한다.
- 데이터 출처, 외부 식별자, 버전 및 변경 이력을 보존한다.
- 중복 수집·재실행·재처리·부분 실패 상황에서도 데이터 정합성을 유지한다.
- 불명확하거나 충돌하는 자격조건을 임의로 확정하지 않는다.
- 신청 가능성 판정(`ELIGIBLE`, `NOT_ELIGIBLE`, `POSSIBLY_ELIGIBLE`, `UNKNOWN`)과 모집 상태(`UPCOMING`, `OPEN`, `ALWAYS_OPEN`, `CLOSED`, `UNKNOWN`)를 구분한다.
- 청년정책의 신청기간과 직업훈련의 교육기간을 혼동하지 않는다.
- 정책 정보 변경, 사용자 조건 변경, 이벤트 발생 및 알림 전달 기준을 구분한다.
- 인증·인가와 개인정보 보호 요구사항을 유지한다.

## 4. Development Rules

### 기본 원칙

- YAGNI 원칙을 따른다. 불필요한 추상화, 계층 및 의존성을 추가하지 않는다.
- 기존 코드와 표준 라이브러리를 우선 재사용하고, 기존 코딩 스타일과 패키지 구조를 따른다.
- 요청 범위를 벗어난 변경을 하지 않는다.
- 보안, 예외 처리, 트랜잭션 및 데이터 정합성을 생략하지 않는다.
- 문제가 명확하지 않은 상태에서 기술 도입이나 구조 변경을 우선하지 않는다.
- AI가 생성한 코드는 실제 프로젝트 구조와 요구사항에 맞는지 검증한다.

### 서비스 경계

- 외부 데이터 수집·정규화 실행 로직은 `data-service/`에서 관리한다.
- 정규화 규칙은 `data-service/`에서 수정한다. `backend/`에 남아 있는 같은 이름의 정규화 코드(`opportunity.normalization`, `opportunity.service`, `opportunity.conditionset`)는 data-service와 내용이 다른 이전 복사본이다. 이 복사본을 수정해야 하는 경우 이유를 먼저 보고한다.
- `backend.collection`에는 수집 관리자 HTTP API·DTO·오류 코드만 둔다(`controller`, `dto`, `exception`).
- 알림 소비·저장·전달 기능은 `notification-service/`에서 관리한다. `backend/`에 `com.team3.backend.notification` 패키지를 만들지 않는다.
- 알림 이벤트 생성, 대상 선별 및 Outbox 발행은 `backend`의 `tracking`·`outbox`가 담당한다. notification-service는 수신한 메시지만으로 처리한다.
- `backend/`의 메인·공용 패키지는 수집·정규화 패키지를 import하지 않는다. 패키지 목록은 `IngestBoundaryTest`의 `MAIN_AND_SHARED`와 `INGEST_IMPORT`를 기준으로 한다.
- 서비스 경계 변경 시 `docs/ingest-split.md`, `docs/database-boundaries.md`와 경계 테스트(`IngestBoundaryTest`, `ApplicationPersistenceCompletenessTest`, `DatabaseBoundaryPostgresTest`)를 확인한다.

### 데이터베이스

- 현재 마이그레이션 도구(Flyway 등)를 사용하지 않으며 임의로 도입하지 않는다. 스키마 변경 SQL은 `database/scripts/`에 두고 `deploy/README.md`의 절차에 따라 수동 적용한다.
- `ddl-auto`는 backend·data-service가 `update`, notification-service가 `validate`다. notification-service 엔티티를 변경하면 대응하는 SQL을 함께 추가한다.
- `application` 스키마(backend·data-service)와 `notification` 스키마(notification-service) 사이에 FK를 만들지 않고, 서로의 테이블을 직접 읽거나 쓰지 않는다.
- 공고 관련 엔티티는 backend(읽기)와 data-service(쓰기)에 각각 있다. 한쪽을 변경하면 다른 쪽도 함께 맞춘다.
- `raw_external_record`, `user_behavior_events`는 파티션 전환 대상 테이블이다. 구조 변경은 `docs/`의 파티셔닝 문서와 `database/scripts/`를 따른다.
- DB 변경 시 기존 데이터와 운영 환경에 미치는 영향을 검토한다.

### 변경 관리

- 기존 API, DB 스키마 및 메시지 계약을 임의로 변경하지 않으며, 변경 시 하위 호환성을 확인한다.
- 성능 개선 시 기존 동작과 결과의 동일성을 검증한다.
- 운영 설정, 배포 방식, 인증·인가 정책을 임의로 변경하지 않는다.
- data-service는 한 개만 실행하는 것을 전제로 한다(수집 스케줄러·대조 중복 방지). 다중 인스턴스를 전제로 한 변경은 먼저 보고한다.
- 비밀번호, API 키, 토큰 및 인증정보를 코드나 로그에 노출하지 않는다.

## 5. Git & Collaboration

Git 협업 규칙은 `README.md`의 Git Convention을 기준으로 한다.

### Branch Strategy

| 브랜치 | 용도 |
| --- | --- |
| `main` | 운영 배포 기준 브랜치. push되면 이미지 게시·배포 워크플로가 실행된다 |
| `dev` | 기본 개발 브랜치. 작업 브랜치의 분기·병합 대상 |
| `feat/*` | 신규 기능 개발 |
| `fix/*` | 오류 수정 |
| `hotfix/*` | 운영 환경 긴급 오류 수정 |
| `test/*` | 테스트 코드 작성 및 검증 환경 개선 |
| `docs/*` | 문서 작성·수정 |
| `refactor/*` | 코드 구조 개선 |

1. 작업 브랜치는 `dev`에서 생성한다.
2. 구현·테스트 후 Pull Request로 `dev`에 병합한다.
3. `dev`의 테스트가 성공하면 `dev` → `main` Pull Request로 운영에 반영한다.
4. 운영 긴급 수정은 `main`에서 `hotfix/*`로 분기하고, 수정 내용을 `main`과 `dev`에 모두 반영한다.

- `main`과 `dev`에 직접 Push하지 않고 Pull Request로 변경한다.
- PR 검증 워크플로는 현재 `main` 대상 Pull Request에서만 실행된다. `dev`에 병합하기 전에는 영향받는 테스트를 로컬에서 실행하고 결과를 Pull Request에 기재한다.

### Commit Convention

- 커밋 메시지는 **`Type: 작업 내용`** 형식을 사용하며, Type의 첫 글자는 대문자로 작성한다. 유형 목록은 `README.md`를 따른다.
- 변경한 파일 목록보다 작업의 핵심 목적과 내용을 중심으로 작성한다.
- 서로 다른 목적의 변경은 가능한 한 커밋을 분리한다.
- 기존 코드 변경과 관계없는 파일을 함께 커밋하지 않는다.
- 커밋 메시지는 실제 변경 내용과 일치해야 한다.

```text
Feat: 개인화 추천 필터 기능 추가
Fix: 검색 조건 누락 오류 수정
Test: 검색 API 통합 테스트 추가
```

### Pull Request Convention

- 기능 단위로 Pull Request를 생성하고 목적·변경 사항·테스트 결과를 기재한다. 본문은 `.github/PULL_REQUEST_TEMPLATE.md` 형식을 따른다.
- 미완료 작업과 확인이 필요한 사항을 명시한다.
- 코드 리뷰와 검증 후 `dev`에 병합한다.
- 운영 반영은 `dev`에서 `main`으로 별도 Pull Request를 생성해 진행한다.

### AI 도구의 Git 사용

- 사용자 또는 작업 담당자의 명시적 요청 없이 커밋, Push, Merge, 배포 및 파괴적인 Git 명령을 실행하지 않는다.
- 서비스별 `AGENTS.md`가 더 엄격하면 그 규칙을 따른다. `frontend/`에서는 조회 외의 Git 작업을 실행하지 않는다.

## 6. Testing & Verification

### 실행 방법

- Java 서비스는 각 서비스 디렉터리에서 실행한다: `./gradlew test` (Windows는 `gradlew.bat`).
- 테스트 그룹은 `-PtestGroup=`으로 고른다: `fast`(PostgreSQL 불필요), `postgres`(PostgreSQL 필요), `manual`(벤치마크 등 명시적 실행). 지정하지 않으면 전체를 실행한다.
- PostgreSQL 테스트는 Docker(Testcontainers)가 필요하다. backend의 일부 테스트는 `YON_TEST_DB_URL`, `YON_TEST_DB_USERNAME`, `YON_TEST_DB_PASSWORD`가 없으면 실행되지 않고 건너뛴다.
- `YON_TEST_DB_URL`에는 일회용 DB만 지정한다. 테스트가 `create-drop`으로 테이블을 삭제한다.
- frontend는 `frontend/`에서 `npm run lint`, `npm test`, `npm run build`를 실행한다.
- CI는 `scripts/test_report.py`와 `scripts/test-policy.json`으로 필수 테스트 누락과 예상하지 못한 skip을 실패로 처리한다. 정책에 등록된 테스트의 이름을 바꾸거나 삭제하면 정책 파일도 함께 수정한다.

### 검증 원칙

- 변경된 기능에 직접 영향을 받는 테스트를 우선 실행한다.
- 관련 회귀 테스트와 서비스 간 계약 테스트를 확인한다.
- 데이터 정합성, 트랜잭션 및 PostgreSQL 의존 동작은 필요한 경우 통합 테스트로 검증한다.
- 프런트엔드 변경 시 타입 검사, 빌드 및 관련 UI 동작을 확인한다.
- 테스트가 통과하지 않았거나 실행되지 않은 상태를 성공으로 보고하지 않는다. 빌드 성공만으로 건너뛴 테스트를 검증 완료로 보고하지 않는다.

### 성능 개선

성능 개선은 다음 순서로 진행한다. 측정 도구와 환경은 `perf/README.md`를 확인한다.

1. 목표와 문제가 발생하는 상황을 정의한다.
2. 로그·메트릭·쿼리 실행 계획 등 근거를 수집한다.
3. 병목과 원인을 분석한다.
4. 개선 방법과 트레이드오프를 비교한다.
5. 개선 전후를 동일한 조건에서 측정한다.
6. 결과와 한계, 재현 조건을 기록한다.

가능하면 응답 시간, p95/p99, TPS, SQL 실행 횟수, 자원 사용량 등의 수치로 검증한다.

## 7. Workflow

1. 요청 목적과 요구사항을 확인한다.
2. `README.md`, 관련 코드, 테스트 및 설계 문서를 확인한다.
3. 작업 범위와 변경 영향도를 파악한다.
4. 기존 구조를 유지하면서 필요한 범위만 수정한다.
5. 영향을 받는 테스트와 검증 절차를 실행한다.
6. 변경 내용, 검증 결과 및 남은 문제를 요약한다.

작업 결과에는 다음 정보를 포함한다.

- 변경한 파일과 주요 내용
- 변경 이유 및 기술적 판단
- 실행한 테스트와 결과
- 미실행 테스트와 이유, 미검증 범위
- 확인이 필요한 사항 및 후속 작업

## 8. Additional Instructions

- 서비스별 상세 지침이 존재하면 해당 디렉터리의 `AGENTS.md`도 함께 확인한다(현재 `frontend/AGENTS.md`).
- 공통 지침과 서비스별 지침이 충돌하면 임의로 판단하지 않고 충돌 내용을 알린다.
- README와 설계 문서를 참고하되, 실제 구현 여부는 코드와 테스트를 기준으로 확인한다. 구현과 문서가 다르면 차이를 보고한다.
- 기능 구현과 운영 적용 상태를 구분하여 설명한다.