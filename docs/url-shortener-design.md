# URL 단축 서비스 (Bitly 클론) 시스템 디자인

> 목표: 단일 서버로 시작해, 트래픽이 늘 때마다 **도메인 코드는 그대로 두고 어댑터만 갈아끼우는** 방식으로 확장한다.
>
> 기술을 하나씩 도입하는 13단계 다이어그램 버전: [url-shortener-roadmap.html](./url-shortener-roadmap.html) (브라우저로 열기)

---

## 1. 요구사항

### 기능 요구사항
| 우선순위 | 기능 | 비고 |
|---|---|---|
| P0 | 긴 URL → 짧은 키 생성 (`POST /api/urls`) | 같은 URL 재요청 시 새 키 발급 (단순화) |
| P0 | 짧은 키 → 원본 URL 리다이렉트 (`GET /{key}`) | 302 기본, 설정으로 301 전환 |
| P1 | 만료 시간 지정 | TTL 지난 키는 410 Gone |
| P1 | 커스텀 alias (`shrt.ly/my-promo`) | 충돌 시 409 |
| P2 | 클릭 통계 (횟수, 리퍼러, 국가) | 비동기 집계 |
| P2 | 악성 URL 차단, rate limit | |

### 비기능 요구사항
- **읽기 >> 쓰기**: 리다이렉트 : 생성 ≈ 100 : 1
- **리다이렉트 지연**: p99 < 50ms (캐시 히트 시 < 10ms)
- **가용성 > 일관성**: 생성 직후 몇 초 전파 지연은 허용, 리다이렉트는 절대 죽으면 안 됨
- **키 충돌 0**: 서로 다른 URL에 같은 키가 나가면 안 됨
- **키는 예측 불가**: `abc123` 다음이 `abc124` 면 전체 크롤링 당함

### 규모 추정 (설계 기준점)
| 항목 | 값 | 계산 |
|---|---|---|
| 쓰기 | 100 TPS | 월 2.6억 건 |
| 읽기 | 10,000 TPS | 쓰기 × 100 |
| 10년 누적 URL | 약 300억 건 | 100 × 86400 × 365 × 10 |
| 저장 용량 | 약 15 TB | 500 B × 300억 |
| 키 길이 | **7자 base62** | 62^7 ≈ 3.5조 ≫ 300억 |
| 캐시 (20% 핫셋, 일 단위) | 약 90 GB | 읽기 10k × 86400 × 0.2 × 500 B |

→ 7자면 10년치 여유가 100배 이상. 6자(568억)는 아슬아슬하다.

---

## 2. 핵심 설계 결정 (단계와 무관하게 먼저 고정)

### 2-1. 키 생성 방식: 해시 vs 카운터
| 방식 | 장점 | 단점 | 판단 |
|---|---|---|---|
| MD5/SHA 후 앞 7자 | 무상태, 같은 URL → 같은 키 | 충돌 발생 → DB 조회 후 재시도 필요, 쓰기가 느려짐 | ✗ |
| 유일 ID → base62 | 충돌 없음, 빠름 | ID 생성기가 필요, 순차 ID는 예측 가능 | ✓ |

**결정**: 유일 ID를 base62 로 인코딩한다. 예측 가능성은 ID 에 비트 셔플(또는 Snowflake 의 시간+노드 비트)을 섞어 해결한다.

### 2-2. 301 vs 302
- **301 (영구)**: 브라우저가 캐시해 서버 부하 ↓, 대신 클릭 통계가 안 잡힘
- **302 (임시)**: 매번 서버 경유, 통계 가능, 부하 ↑

**결정**: 302 기본. 통계가 필요 없는 링크(설정값)는 301 로 내린다.

### 2-3. 리다이렉트 경로와 생성 경로의 분리
읽기 경로(`GET /{key}`)는 **DB 를 거치지 않는 것**이 최종 목표다. 처음부터 두 경로를 별도 클래스/패키지로 나눠, 나중에 별도 서비스로 쪼갤 수 있게 한다.

---

## 3. 단계별 아키텍처

### Stage 0 — 단일 서버, 단일 DB (지금 이 레포)

```
Client ──► Spring Boot (Kotlin) ──► H2 / PostgreSQL
              ├ UrlController
              ├ UrlShortener (domain)
              ├ IdGenerator   = DB sequence + bit shuffle
              └ UrlRepository = JPA
```

- **ID**: DB 시퀀스(`BIGINT`) → 비트 셔플(예: Feistel 또는 XOR + rotate) → base62 7자
- **테이블**
  ```sql
  CREATE TABLE short_url (
    id          BIGINT PRIMARY KEY,          -- 셔플 전 원본 시퀀스 값
    short_key   VARCHAR(16) NOT NULL UNIQUE, -- base62 결과 또는 custom alias
    long_url    TEXT        NOT NULL,
    expires_at  TIMESTAMP   NULL,
    created_at  TIMESTAMP   NOT NULL
  );
  CREATE INDEX idx_short_url_key ON short_url(short_key);
  ```
- **처리량**: 단일 PostgreSQL 이면 쓰기 100 TPS 는 여유. 읽기 수천 TPS 까지는 인덱스 조회로 버팀.
- **여기서 끝나는 병목**: 읽기 DB 부하, 앱 서버 단일 장애점.

### Stage 1 — 읽기 확장 (캐시 + 수평 확장)

```
Client ──► CDN/LB ──► App ×N ──► Redis (key → long_url)
                         │           miss ↓
                         └────────► DB Primary ──► Read Replica ×N
```

- **Redis**: `short_key → long_url` 를 TTL 과 함께 캐시. 생성 시 write-through 로 미리 넣어 첫 조회 미스를 없앤다.
- **없는 키 공격 대응**: 존재하지 않는 키 요청이 매번 DB 로 가면 캐시가 무력화된다. → **네거티브 캐시**(짧은 TTL) 또는 **블룸 필터**.
- **읽기는 Replica 로**: 생성 직후 Replica 지연은 캐시 write-through 가 가려준다.
- **앱 무상태화**: 세션 없음, ID 생성은 여전히 DB 시퀀스라 다중 인스턴스에서도 안전.
- **코드 변경**: `UrlCache` 어댑터를 `NoopCache → RedisCache` 로 교체. 도메인 코드 무변경.
- **여기서 끝나는 병목**: 쓰기가 DB 시퀀스 한 곳에 몰림, 단일 DB 용량 한계.

### Stage 2 — 쓰기·저장 확장 (ID 생성기 분리 + 샤딩)

```
                    ┌─► Redirect Service ×N ──► Redis Cluster ──► DB Shard[hash(key)]
Client ─► LB ───────┤
                    └─► Create Service ×N ──► IdGenerator ──► DB Shard[hash(key)]
                                                 │
                                         Snowflake(노드ID) 또는
                                         Key Generation Service(미리 발급된 키 풀)
```

- **ID 생성기 선택지**
  | 방식 | 설명 | 장단점 |
  |---|---|---|
  | Snowflake | 41bit 시간 + 10bit 노드 + 12bit 시퀀스, 앱 내부에서 생성 | 외부 의존 없음, 시간 역전 주의, base62 로 11자 → 앞부분 잘라 7자로 쓰려면 별도 처리 |
  | KGS (Key Generation Service) | 미리 7자 키를 대량 생성해 두고, 각 앱 인스턴스가 1,000개씩 가져가 사용 | 키 길이 정확히 7자, 충돌 0, KGS 자체 HA 필요 |
  
  **추천**: 처음엔 **Snowflake 류 (48bit 이내로 줄인 변형)** 로 시작. 키 길이 통제가 중요해지면 KGS 로.
- **샤딩 키**: `short_key` 의 해시. 리다이렉트가 항상 키로 조회하므로 단일 샤드 접근이 보장된다. (사용자별 조회는 별도 인덱스 테이블 또는 Stage 3 의 분석 DB 에서)
- **서비스 분리**: 읽기/쓰기 서비스를 물리적으로 분리해 배포·스케일 단위를 다르게 가져간다. Stage 0 에서 패키지를 나눠둔 게 여기서 빛난다.
- **여기서 끝나는 병목**: 클릭 통계가 리다이렉트 경로를 느리게 함, 단일 리전 지연.

### Stage 3 — 분석 분리 + 글로벌

```
Redirect Service ──(비동기)──► Kafka ──► Consumer ──► ClickHouse / BigQuery
      │                                   └──► Redis (실시간 카운터)
      ▼
   302 즉시 반환 (통계 기록을 기다리지 않음)

GeoDNS ──► Region A (풀 스택)   Region B (풀 스택)   ... DB 는 리전 간 비동기 복제
```

- **클릭 이벤트**: 리다이렉트 스레드에서 Kafka 에 fire-and-forget. 유실 허용 (가용성 > 정확도).
- **분석 DB**: 시계열·집계용 컬럼 DB. 운영 DB 와 완전 분리.
- **멀티 리전**: 리다이렉트는 가까운 리전에서, 생성은 ID 충돌 방지를 위해 리전별 노드 ID 범위 분리.
- **Rate limit / 악성 URL**: LB 또는 API Gateway 단에서 처리. Safe Browsing API 조회는 생성 시 비동기로.

---

## 4. 코드 구조 (Stage 0 부터 적용)

확장할 때 바뀌는 건 **어댑터뿐**이어야 한다. 포트(인터페이스)를 처음부터 둔다.

```
com.architecturepractice.shortener
├── domain
│   ├── ShortUrl.kt            # 엔티티 (shortKey, longUrl, expiresAt)
│   ├── ShortKey.kt            # 값 객체, base62 검증
│   └── Base62.kt
├── application
│   ├── CreateShortUrl.kt      # 쓰기 유스케이스
│   └── ResolveShortUrl.kt     # 읽기 유스케이스 (캐시 → 저장소 순서)
├── port
│   ├── IdGenerator.kt         # Long nextId()
│   ├── ShortUrlRepository.kt  # save / findByKey
│   └── ShortUrlCache.kt       # get / put / putNegative
└── adapter
    ├── web/        UrlController, RedirectController   # 읽기·쓰기 컨트롤러 분리
    ├── persistence/ JpaShortUrlRepository
    ├── cache/      NoopShortUrlCache   → (Stage 1) RedisShortUrlCache
    └── id/         DbSequenceIdGenerator → (Stage 2) SnowflakeIdGenerator / KgsIdGenerator
```

| Stage | 바뀌는 어댑터 | 안 바뀌는 것 |
|---|---|---|
| 0 → 1 | `cache/` Noop → Redis, 인프라에 LB·Replica | domain, application, port |
| 1 → 2 | `id/` DbSequence → Snowflake, `persistence/` 샤드 라우팅 | domain, application, port |
| 2 → 3 | `ResolveShortUrl` 에 `ClickEventPublisher` 포트 1개 추가 | 나머지 전부 |

---

## 5. Stage 0 API 스펙

```
POST /api/urls
  { "longUrl": "https://...", "customAlias": "promo"?, "expiresAt": "2026-12-31T00:00:00Z"? }
  → 201 { "shortKey": "aZ3kQ9x", "shortUrl": "https://shrt.ly/aZ3kQ9x" }
  → 400 잘못된 URL / 409 alias 중복

GET /{shortKey}
  → 302 Location: {longUrl}
  → 404 없음 / 410 만료
```

---

## 6. 다음 할 일
1. Stage 0 구현 (위 패키지 구조, H2 + JPA, 테스트 포함)
2. 부하 테스트로 Stage 0 한계 수치 확인 (k6 또는 Gatling)
3. Stage 1: Redis 어댑터 추가, docker-compose 로 로컬 재현
