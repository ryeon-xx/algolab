# AlgoLab 설계 문서

> 상태: v1.0 확정 (2026-11-01)
> 구현하면서 바뀌는 내용은 이 문서에 반영하고 변경 이력에 남긴다.

## 1. 개요

정렬 알고리즘을 같은 입력으로 실행해 실행 시간과 연산 횟수를 측정하고, 결과를 저장해 표와 차트로 비교하는 실험 플랫폼.

답하려는 질문:

- 데이터가 많아지면 실행 시간은 어떻게 달라지는가?
- 알고리즘마다 비교 연산 횟수는 어떻게 다른가?
- 무작위 데이터와 정렬된 데이터에서 결과는 어떻게 다른가?
- 이론적 시간 복잡도와 실제 측정값은 어떤 관계인가?

측정값이 이론과 항상 일치한다고 가정하지 않는다. 측정 환경, 구현 방식, 입력 특성을 결과와 함께 기록한다.

## 2. 요구사항 검토 결과

| 항목 | 문제 | 결정 |
|---|---|---|
| 화면 5개 | 결과 화면과 상세 화면이 같은 데이터를 보여줌 | 결과 화면 = 상세 화면, 총 4개 |
| 데이터 크기별 변화 | 실험 1번에 크기는 1개 | 기록 목록에서 여러 실험을 골라 비교 |
| 오름차순/내림차순 정의 | `1..n`으로 만들면 무작위 데이터와 값 분포가 달라짐 | 같은 seed로 무작위 값을 만든 뒤 정렬. 값은 같고 순서만 다름 |
| 원본 배열 저장 | 저장하지 않으면 재현 불가 | seed와 생성 조건만 저장. `java.util.Random`은 알고리즘이 명세로 고정돼 있어 같은 seed면 같은 배열이 나옴 |
| 연산 횟수 측정 | 카운터 증가가 실행 시간을 늘림 | MVP에서는 감수하고 한계로 기록. 3단계 개선 후보 |
| 동시 실행 | 실험이 동시에 돌면 측정값이 서로 오염됨 | 동시에 1개만 실행, 실행 중이면 429 |

## 3. MVP 범위

### 포함

| 기능 | 내용 |
|---|---|
| 데이터 생성 | RANDOM / ASCENDING / DESCENDING, 크기, 값 범위(min/max), seed(비우면 서버가 생성), 중복 허용 |
| 알고리즘 | Bubble(조기 종료 버전) → Insertion → Merge |
| 측정 | 워밍업 1회 + 측정 N회(1~5, 기본 3). 중앙값·최소·최대 실행 시간, 비교 횟수, 쓰기 횟수, 정렬 성공 여부, 데이터 생성 시간 |
| 저장/조회 | 실험 + 알고리즘별 결과, 목록(페이징), 상세 |
| 비교 | 실험 내 알고리즘별 표·막대 차트, 실험 간 크기별 선 차트 |
| 예외 처리 | Bean Validation, 크기 제한, ProblemDetail 오류 형식, 동시 실행 제한 |

### 제외

| 기능 | 이유 |
|---|---|
| 인증 | 개인 실험 도구라 MVP 가치가 낮음 |
| Redis, AI | 필요성이 확인되기 전에는 도입하지 않음 |
| 브라우저 실행, 정렬 애니메이션 | 측정 환경이 하나 더 생기고 핵심 기능이 아님 |
| 중복 불허, 거의 정렬된 데이터, Quick/Heap Sort | MVP 이후 첫 추가 후보 |
| 비동기 작업 큐, 실행 취소 | 크기 제한으로 최대 실행 시간이 짧게 묶임 |
| 배포, CI/CD | 5단계 |

## 4. 기술 스택과 버전

버전은 2026-11-01 기준 공식 문서(spring.io, docs.spring.io의 System Requirements)로 확인했다.

| 구분 | 선택 | 근거 |
|---|---|---|
| Spring Boot | 4.1.2 | 최신 Stable. 4.2.0은 RC라 제외. OSS 지원 종료 2027-07-31 (endoflife.date) |
| Java | 21 (LTS) | 4.1.2는 Java 17 이상, 27까지 호환. 25 전용 기능이 필요 없고 21의 도구·배포 환경 호환 이력이 가장 넓음 |
| Spring Framework | 7.0.10 이상 | Boot가 관리 |
| Hibernate 등 | Boot 관리 버전 사용 | 직접 고정하지 않음. 실제 버전은 의존성 트리로 확인 |
| 빌드 | Gradle (Groovy DSL), Gradle Wrapper | start.spring.io 기본값. 요구사항: Gradle 8.14 이상 또는 9.x. 전역 설치 없이 Wrapper 사용 |
| DB | MySQL 8.4.10 (로컬 설치본) | 이미 설치·실행 중인 서버 사용 |
| 프론트 | React + TypeScript + Vite | 버전은 프로젝트 생성 시 확인해 기록 |
| 라우팅 | React Router | 화면 4개의 URL 기반 이동 |
| 차트 | Recharts | React 컴포넌트 방식, 막대·선 차트면 충분 |
| HTTP | fetch + 얇은 래퍼 | API 5개뿐이라 Axios 불필요 |
| 오류 응답 | Spring ProblemDetail (RFC 9457) | 표준 형식이 내장돼 있음 |

## 5. 시스템 구조

```
[Browser] React (Vite :5173)
     │  /api/* → Vite proxy (CORS 설정 불필요)
     ▼
[Spring Boot :8080]
  Controller ── 요청 검증(DTO + Bean Validation)
     │
  ExperimentService ── 동시 실행 제한, 트랜잭션
     ├─ DataGenerator     : seed로 원본 배열 생성 (생성 시간 별도 측정)
     ├─ BenchmarkRunner   : 알고리즘마다 원본 복사 → 워밍업 → N회 측정 → 정렬 검증
     │    └─ 정렬 구현체 + OperationCounter(비교/쓰기)
     └─ Repository (JPA) ──► MySQL
```

- 저장소: 모노레포 `backend/`, `frontend/`, `docs/`
- 백엔드 패키지는 기능별: `algorithm`, `experiment`, `common`
- 알고리즘 인터페이스는 처음부터 만들지 않는다. 두 번째 알고리즘을 추가해 교체 필요가 생길 때 도입한다.
- 측정 때마다 `Arrays.copyOf`로 원본을 복사하고, 복사 시간은 측정 구간에서 뺀다.
- 알고리즘 메타데이터(이름, 설명, 시간·공간 복잡도)는 Java enum에 둔다. 코드와 함께 바뀌는 정적 정보이기 때문이다.

## 6. ERD

```
experiment                          algorithm_result
──────────────────────────          ──────────────────────────────
id            BIGINT PK AI          id               BIGINT PK AI
data_size     INT                   experiment_id    BIGINT FK → experiment.id
data_pattern  VARCHAR(20)           algorithm        VARCHAR(30)
value_min     INT                   median_time_ns   BIGINT
value_max     INT                   min_time_ns      BIGINT
seed          BIGINT                max_time_ns      BIGINT
repeat_count  INT                   comparison_count BIGINT
generation_ns BIGINT                write_count      BIGINT
java_version  VARCHAR(50)           sorted_correctly BOOLEAN
os_name       VARCHAR(100)          UNIQUE(experiment_id, algorithm)
cpu_cores     INT
created_at    DATETIME(6)

experiment 1 ─── N algorithm_result
```

- 실험 조건이 알고리즘 수만큼 중복 저장되지 않도록 두 테이블로 나눈다.
- 실행 환경 컬럼은 다른 환경(로컬/배포 서버)의 측정값을 구분하기 위해 둔다.
- 카운트는 BIGINT(long)로 둔다.
- JPA: `AlgorithmResult → Experiment` 를 `@ManyToOne(fetch = LAZY)` 단방향으로만 매핑하고 Cascade는 쓰지 않는다. 상세는 `findByExperimentId`로 따로 조회한다. 목록 조회의 N+1 문제는 3단계에서 재현하고 개선한다.

## 7. API

| Method | URL | 설명 | 성공 |
|---|---|---|---|
| GET | `/api/algorithms` | 알고리즘 목록, 메타데이터, 최대 크기 | 200 |
| POST | `/api/experiments` | 실험 실행 + 저장 + 결과 반환 | 201 |
| GET | `/api/experiments?page=0&size=20` | 기록 목록 (최신순, 결과 미포함) | 200 |
| GET | `/api/experiments/{id}` | 상세 (조건 + 알고리즘별 결과) | 200 |
| GET | `/api/experiments/compare?ids=1,2,3` | 여러 실험 결과 (크기별 차트용) | 200 |

요청 예시:

```json
{ "size": 5000, "pattern": "RANDOM", "minValue": 0, "maxValue": 100000,
  "seed": null, "repeatCount": 3, "algorithms": ["BUBBLE", "MERGE"] }
```

크기 제한(잠정, T4 실측 후 확정): O(n²) 알고리즘 10,000 / Merge 1,000,000. 선택한 알고리즘 중 하나라도 넘으면 400.

오류 응답(ProblemDetail + 확장 필드):

```json
{ "type": "about:blank", "title": "Bad Request", "status": 400,
  "detail": "BUBBLE의 최대 크기는 10000입니다.", "code": "SIZE_LIMIT_EXCEEDED",
  "errors": [{ "field": "size", "reason": "..." }] }
```

| 상황 | 상태 코드 |
|---|---|
| 검증 실패, 지원하지 않는 알고리즘, min > max | 400 |
| 존재하지 않는 기록 | 404 |
| 다른 실험이 실행 중 | 429 |
| 그 밖의 서버 오류 | 500 (내부 메시지 미노출) |

## 8. 측정 정의

- 실행 시간: `System.nanoTime()`, 알고리즘 호출 직전부터 직후까지. 원본 복사와 정렬 검증은 제외한다.
- 비교 횟수: 원소끼리 대소를 비교한 횟수.
- 쓰기 횟수: 배열 원소에 값을 대입한 횟수. swap 1번은 쓰기 2번이다. 보조 배열에 대입한 것도 포함한다.
- 비교와 쓰기는 합산하지 않는다.
- 연산 횟수는 같은 입력이면 결정적이므로 측정 1회 값을 저장한다. 실행 시간은 N회의 중앙값·최소·최대를 저장한다.
- 정렬 성공: 결과가 비내림차순인지, 그리고 `Arrays.sort` 결과와 같은지 확인한다.
- Bubble Sort는 한 바퀴 동안 교환이 없으면 종료하는 버전이다. 정렬된 입력에서는 O(n)이다.

## 9. 작업 순서와 완료 기준

| # | 작업 | 완료 기준 | 예정일 |
|---|---|---|---|
| T0 | 환경 확인 + 설계 문서 커밋 | 도구 버전 확인, 이 문서 커밋 | 11/1 |
| T1 | 백엔드 프로젝트 생성 (Web, JPA, Validation, MySQL Driver) | `./gradlew bootRun` 기동 (DB 연결 제외) | 11/2 |
| T2 | Bubble Sort + OperationCounter (순수 Java) | 빈 배열/1개/중복/음수/정렬/역순/무작위 결과가 `Arrays.sort`와 같음, `[3,2,1]` 카운트가 손 계산과 같음 | 11/2 |
| T3 | DataGenerator | 같은 seed면 같은 배열, 패턴별 순서 검증, 범위 준수 | 11/2 |
| T4 | BenchmarkRunner | 원본 불변 테스트, 중앙값 테스트, 실측으로 크기 제한 확정 | 11/3 |
| T5 | `POST /api/experiments` (DB 없이) + 검증 + ProblemDetail | 성공 1개, 400 케이스 3개 확인 | 11/3 |
| T6 | 프론트 생성 + 설정 폼 + 결과 표 | 브라우저에서 실행 → 표 표시 | 11/4 |
| T7 | MySQL 연결(환경변수) + Entity/Repository + 저장 | INSERT SQL 로그와 테이블 행 확인 | 11/5 |
| T8 | 목록/상세 API + 404 | 페이징 동작, 없는 id면 404 | 11/5 |
| T9 | Insertion, Merge (인터페이스 도입) | T2와 같은 테스트 통과 | 11/6 |
| T10 | 목록·상세 화면 + 막대 차트 | 목록 → 상세 이동, 차트 표시 | 11/6 |
| T11 | 비교 API + 크기별 선 차트 | 실험 3개로 크기-시간 그래프 | 11/7 |
| T12 | 동시 실행 제한, 오류 처리 정리, README 초안 | 동시 요청 2개 중 1개가 429 | 11/7 |

일정이 밀릴 때 자르는 순서: T12 → T11의 비교 API(프론트에서 상세를 여러 번 호출하는 방식으로 축소) → Merge Sort 연기.

## 10. 예상되는 어려움

1. JIT 워밍업: 워밍업 1회를 버리고 중앙값을 쓴다. JMH 수준의 정밀도는 아니다.
2. 짧은 실행 시간: 중앙값이 1ms 미만이면 "측정 신뢰도 낮음"을 표시한다.
3. GC 개입: 최소/최대를 함께 저장해 편차를 보인다.
4. 카운터 오버헤드: 3단계에서 카운터 없는 실행과 비교한다.
5. 쓰기 횟수 정의: 8장 정의를 모든 알고리즘에 똑같이 적용한다.
6. 긴 요청이 서버 스레드를 붙잡음: 크기 제한과 동시 실행 1개 제한으로 막는다.
7. JPA 학습: 실행되는 SQL을 로그로 확인하고 MyBatis와 비교한다. `ddl-auto=update`는 로컬에서만 쓰고, 생성된 DDL을 기록한다.

## 11. 개발 환경 (2026-11-01 확인)

| 도구 | 상태 |
|---|---|
| JDK | 17(17+35)만 설치돼 있음 → **JDK 21 설치 필요** |
| Node.js / npm | v22.23.1 / 10.9.8 |
| MySQL | 8.4.10 Community, Windows 서비스 `MySQL84` 실행 중 |
| Gradle | 전역 설치 없음 → 프로젝트의 Gradle Wrapper 사용 |
| Git | 설치됨 |
| Docker | 없음 (사용하지 않음) |

## 변경 이력

- 2026-11-01: v1.0 확정
