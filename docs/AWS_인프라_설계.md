# AWS 인프라 설계 — IoT Core · Aurora MySQL

HEXA FocusMate의 IoT 데이터 파이프라인과 데이터베이스 설계 문서입니다. (담당: 도동원)

> 보안을 위해 엔드포인트 주소, 계정 ID 등 식별 정보는 제외했습니다.

---

## 1. 전체 데이터 흐름

| 단계 | 구성 요소 | 역할 |
|---|---|---|
| 1 | 라즈베리파이 (EAR 센서) | EAR 수치 수집 및 MQTT 발행 |
| 2 | AWS IoT Core (토픽 룰) | 토픽 수신 → SQL 필터링 → Lambda 호출 |
| 3 | AWS Lambda | 집중도 판단 로직 처리 |
| 4 | Aurora MySQL | 결과 데이터 저장 및 이벤트 로깅 |

라즈베리파이 → IoT Core → Lambda → Aurora MySQL로 이어지는 단방향 이벤트 기반 파이프라인입니다. 각 계층의 역할을 분리하고, IoT Core SQL 필터링으로 Lambda 전달 데이터를 최소화했으며, Aurora Serverless v2로 비용 효율과 확장성을 함께 확보했습니다.

## 2. AWS IoT Core

### 2.1 디바이스 등록과 인증
| 항목 | 내용 |
|---|---|
| 리전 | ap-northeast-2 (서울) |
| 등록 디바이스 | 3대 (라즈베리파이, 테스트용 노트북 등) |
| 인증 방식 | X.509 인증서 기반 상호 인증 (mTLS) |

**설계 근거**
- MQTT는 저전력·저대역폭 환경에 맞는 경량 프로토콜로, 라즈베리파이 같은 엣지 디바이스에 적합합니다.
- 인증서 기반 상호 인증으로 디바이스 위장 접속을 차단하고, 클라이언트 ID 단위로 연결을 제어해 인가된 기기만 통신하도록 했습니다.

### 2.2 IoT 정책 (최소 권한)
| 허용 동작 | 용도 |
|---|---|
| iot:Connect | 디바이스 연결 |
| iot:Publish | 지정 토픽으로 데이터 발행 |
| iot:Subscribe | 토픽 구독 |
| iot:Receive | 메시지 수신 |

- 네 가지 동작만 허용하고 토픽 범위를 `sensor/ear/data` 하나로 한정했습니다.
- 디바이스가 탈취되더라도 다른 토픽에는 접근할 수 없습니다.

### 2.3 토픽 룰
| 항목 | 내용 |
|---|---|
| 구독 토픽 | `sensor/ear/data` |
| SQL 필터 | `SELECT ear, user_id FROM 'sensor/ear/data'` |
| 액션 | 집중도 판단 Lambda 호출 |

**설계 근거**
- 원시 데이터에서 필요한 필드만 추출해 Lambda로 전달합니다. 불필요한 필드는 처리 비용과 후속 처리의 복잡도를 높이기 때문입니다.
- 중간 큐 없이 IoT Core에서 Lambda를 직접 호출해 지연을 최소화했습니다.

## 3. Aurora MySQL

### 3.1 클러스터 구성
| 항목 | 내용 |
|---|---|
| 엔진 | Aurora MySQL 8.0 (3.10.3) |
| 용량 모드 | Serverless v2 — 최소 0 ACU / 최대 4 ACU, 유휴 300초 후 자동 일시정지 |
| 인스턴스 | Writer 1 + Reader 1 |
| 암호화 | AWS KMS |
| 백업 | 7일 자동 백업 |
| 기타 | Data API 활성화 |

**Serverless v2 선택 근거**
- 개발 단계에는 사용량이 불규칙하고 야간에는 트래픽이 거의 없습니다.
- 프로비저닝 인스턴스는 유휴 시간에도 과금되지만, Serverless v2는 최소 0 ACU 설정으로 유휴 과금을 최소화하면서 요청 시 즉시 확장할 수 있어 실서비스 전환에도 대응할 수 있습니다.

### 3.2 스키마 설계
사용자 관리, 그룹 관리, 집중도 데이터 저장, 이벤트 로깅, 배치 이력까지 서비스 전반을 다루는 8개 테이블을 설계했습니다.

| 테이블 | 핵심 제약조건 | 역할 |
|---|---|---|
| users | id PK AUTO_INCREMENT, user_id · phone · email · nickname UNIQUE | 사용자 계정 |
| study_groups | group_id PK, invite_code UNIQUE, created_by FK → users.id | 스터디 그룹 |
| group_members | id PK, group_id FK, user_id FK | 그룹-사용자 관계 |
| user_rankings | user_id PK + FK, total_study_time UNSIGNED, updated_at 자동 갱신 | 집중도 랭킹 |
| focus_summary_logs | user_id FK, avg_focus_score TINYINT UNSIGNED, drowsy_count · away_count | 세션 요약 |
| critical_event_logs | user_id FK, event_type ENUM('DROWSY','AWAY'), snapshot_s3 | 이벤트 로그 |
| batch_execution_log | status ENUM('RUNNING','SUCCESS','FAILED') DEFAULT 'RUNNING', error_msg | 배치 실행 이력 |
| user_deletion_audit | original_user_id, deleted_at, reason | 계정 삭제 감사 |

**설계 원칙**
- `users`의 아이디 · 전화번호 · 이메일 · 닉네임에 UNIQUE 제약을 걸어 중복 가입을 방지했습니다.
- `user_rankings.updated_at`에 `ON UPDATE CURRENT_TIMESTAMP`를 설정해 랭킹 갱신 시각을 자동 관리했습니다.
- ENUM으로 허용 값을 DB 단에서 강제해 애플리케이션의 유효성 검사 부담을 줄였습니다.
