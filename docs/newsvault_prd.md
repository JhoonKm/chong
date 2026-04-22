# NewsVault 제품/기술 설계서 (v1.0)

## 1) 제품 한 줄 정의
로컬 원장을 기준으로 신문 스크랩 이미지를 구조화된 지식 객체로 변환하고, 요약 중심으로 안전하게 외부 동기화/공유하는 Local-first 아카이빙 앱.

## 2) 문제 정의
- 이미지 기반 스크랩은 본문 검색/재활용이 어렵다.
- 클라우드 OCR 중심 도구는 원문 외부 반출 리스크가 있다.
- 50~100건 단위 수집 시 수작업 보정/업로드가 병목이다.

## 3) 목표 사용자
- 리서치/PR/법무/데스크/PKM 사용자
- 니즈: **로컬에는 전문 저장**, 외부에는 **요약 중심 공유**

## 4) 핵심 사용자 시나리오
1. 아침에 이미지 100장 드롭(폴더 포함).
2. 배치 생성 후 백그라운드 처리(보정→OCR→요약).
3. 처리 완료 목록에서 50건 다중 선택.
4. 카카오톡 공유용 요약 복사.
5. 퇴근 전 Notion DB로 summary_only 일괄 동기화.

## 5) 기능 범위
### MVP
- 대량 업로드/배치 생성
- 전처리(회전/기울기/노이즈)
- OCR+구조화 추출+요약+키워드
- SQLite 원장 저장
- 카카오톡 복사 포맷
- 단건 Notion 동기화(summary_only 기본)

### Phase 2
- 50건+ 다중 선택 일괄 액션
- 실패 건 재시도/부분성공 UX
- 중복 탐지(fingerprint)
- 배치 큐 모니터링

### Phase 3
- 자체 웹앱 동기화/마이그레이션
- FTS 전문 검색
- 고급 이미지 보정(OpenCV)

## 6) 상세 PRD
- 입력: 단건/다건/폴더 업로드, 배치 즉시 생성, user_note/source_url 입력
- 전처리: 원본 보존 + 분석용 파생본 생성, 미리보기 제공
- 추출: 제목/부제/본문/기자/매체/발행일/요약박스/캡션/신뢰도
- 지식화: 1문장+3~5문장 요약, 키워드 5~10, 태그 2~8, 카테고리 1, NER
- 저장: local full storage + JSON/CSV/SQLite dump
- 공유: full/summary/metadata 선택, 기본값 summary_only
- 운영: 큐, 재시도, idempotency, chunking, rate-limit 대응

## 7) IA
- Dashboard
- Inbox(대기/처리중/실패)
- Archive(검색/필터/다중선택)
- Article Detail(이미지+추출폼)
- Batch Manager(일괄 동기화/공유)
- Settings(API키/정책/저장소)

## 8) 주요 화면 목록
- Dashboard
- Upload Dropzone + Upload Modal
- Archive Table/Grid
- Article Detail Split View
- Batch Manager
- Settings

## 9) 핵심 UI 요소
- 상태 뱃지: QUEUED/PROCESSING/COMPLETED/FAILED/NEEDS_REVIEW
- 일괄 액션 바: 노션 동기화/웹 업로드/카톡 복사/카테고리 변경
- 정책 토글: 저장 범위, 공유 범위
- 실패 재시도 버튼

## 10) 사용자 플로우
업로드 → 배치 생성 → 큐 처리 → 완료 알림 → 다중 선택 → 일괄 공유/동기화 → 감사로그 기록

## 11) 시스템 아키텍처
- **Desktop**: Electron + React + TypeScript
- **Main**: IPC API, Queue Worker, OCR/LLM 서비스, Sync 서비스
- **DB**: SQLite(원장)
- **외부**: OpenAI Vision/Text, Notion API, Web Sync API(Phase3)
- 원칙: Notion/Web은 타깃, 원장은 항상 SQLite

## 12) 배치 처리 아키텍처
- DB-backed queue
- chunk size: 3~5
- worker concurrency: 2~4
- 상태 전이: QUEUED→PREPROCESSING→OCR→AI_EXTRACT→DONE/FAILED
- retry: 최대 3회, exponential backoff
- partial success 보장

## 13) 로컬 저장 구조
```text
~/.newsvault/
  data.db
  backups/
  logs/app.log
  media/original/
  media/processed/
  exports/json/
  exports/csv/
```

## 14) DB 스키마 (SQLite)
```sql
CREATE TABLE batches (
  id TEXT PRIMARY KEY,
  created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
  total_count INTEGER NOT NULL,
  processed_count INTEGER DEFAULT 0,
  failed_count INTEGER DEFAULT 0,
  status TEXT NOT NULL
);

CREATE TABLE articles (
  id TEXT PRIMARY KEY,
  batch_id TEXT,
  created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
  uploaded_at DATETIME,
  published_at TEXT,
  title TEXT,
  subtitle TEXT,
  source TEXT,
  reporter TEXT,
  category TEXT,
  tags TEXT,
  keywords TEXT,
  article_body_text TEXT,
  cleaned_text TEXT,
  summary_short TEXT,
  summary_long TEXT,
  summary_box_text TEXT,
  source_url TEXT,
  user_note TEXT,
  image_hash TEXT,
  content_hash TEXT,
  duplicate_score REAL,
  extraction_confidence REAL,
  image_quality_score REAL,
  orientation_detected TEXT,
  orientation_corrected INTEGER,
  preprocess_status TEXT,
  review_status TEXT,
  share_mode_default TEXT DEFAULT 'summary_only',
  store_mode_default TEXT DEFAULT 'full_article',
  legal_scope TEXT DEFAULT 'internal_only',
  notion_sync_status TEXT DEFAULT 'UNSYNCED',
  web_sync_status TEXT DEFAULT 'UNSYNCED',
  processing_stage TEXT,
  processing_error_code TEXT,
  processing_error_message TEXT,
  idempotency_key TEXT UNIQUE,
  original_image_path TEXT,
  processed_image_path TEXT,
  FOREIGN KEY(batch_id) REFERENCES batches(id)
);

CREATE TABLE sync_jobs (
  id TEXT PRIMARY KEY,
  target TEXT NOT NULL,
  article_id TEXT NOT NULL,
  mode TEXT NOT NULL,
  status TEXT NOT NULL,
  retry_count INTEGER DEFAULT 0,
  error_message TEXT,
  created_at DATETIME DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE action_logs (
  id TEXT PRIMARY KEY,
  article_id TEXT,
  action_type TEXT NOT NULL,
  action_scope TEXT,
  actor TEXT DEFAULT 'local_user',
  created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
  detail_json TEXT
);
```

## 15) 주요 엔터티
- Batch: 업로드 단위 운영 객체
- Article: 검색 가능한 지식 객체(핵심)
- SyncJob: 외부 동기화 큐 엔터티
- ActionLog: 정책/감사 이력

## 16) 기사 1건 처리 파이프라인
1. 업로드 저장 + hash 계산
2. 전처리(회전/왜곡/노이즈)
3. Vision OCR + 구조화 JSON 추출
4. 요약/키워드/카테고리 생성
5. 신뢰도 계산 및 NEEDS_REVIEW 판정
6. DB update + 완료 이벤트 발행

## 17) 배치 업로드 파이프라인
- Renderer에서 filePaths[] 전달
- Main에서 batch_id 생성
- 트랜잭션으로 articles 일괄 insert(QUEUED)
- 즉시 batch_id 반환
- UI는 progress polling/IPC event 구독

## 18) 방향 보정 로직
- 1차: EXIF 기반 보정
- 2차: Vision orientation_detected(0/90/180/270)
- 3차: 필요 시 sharp rotate + flip 적용
- 결과: processed_image_path만 분석/뷰어 기본값으로 사용

## 19) OCR/추출/요약/분류 파이프라인
- 모델 출력은 JSON schema 강제
- 요약 박스 없으면 생성 요약으로 대체
- 단계별 실패 코드 분리: OCR_TIMEOUT, JSON_PARSE_FAIL, LOW_CONFIDENCE

## 20) 저장 정책
- 로컬: full_article 고정(원장)
- Notion/Web: summary_only 기본, full_article은 명시적 선택
- Export: JSON/CSV/SQLite dump 제공

## 21) 공유 정책
- 기본: summary_only
- 외부 기본 권장: metadata_only
- full_article 공유는 경고 + 감사로그 필수

## 22) 법무/운영 가드레일
- full_article 외부 전송 시 확인 모달 + 로그
- default 설정은 관리자도 summary_only로 시작
- 정책 위반 가능 액션은 2단계 확인

## 23) 노션 동기화 설계
- Notion DB 1 row = 기사 1 페이지
- Adapter 패턴으로 Article→Notion payload 변환
- 중복 방지: notion_page_id upsert
- 본문 기본: summary + note (전문 제외)

## 24) 웹앱 마이그레이션 설계
- local SQLite → JSON dump → /api/v1/sync bulk upsert
- idempotency_key 기준 멱등 보장
- 단계적 동기화: metadata→summary→(선택)full

## 25) 카카오톡 복사 포맷
```text
📰 [NewsVault 요약 공유]
▪ 제목: {title}
▪ 출처: {source} ({published_at})
▪ 기자: {reporter}

💡 핵심 요약
{summary_short}

- {summary_long bullet...}
🏷 키워드: {keywords}
💬 메모: {user_note}
```

## 26) 실패 처리/재시도
- 기사 단위 실패 격리
- retry_count <= 3
- 실패 건만 재큐잉
- 배치는 partial success 허용

## 27) 성능 최적화
- 업로드 시 이미지 2000px 리사이즈
- 리스트 virtualized rendering
- SQLite index(batch_id,status,created_at,published_at)

## 28) 관측성/로그/감사로그
- app.log: 처리시간/에러스택/API 응답코드
- action_logs: 동기화/공유 정책 이벤트
- metrics: throughput, fail_rate, avg_processing_ms

## 29) 권한/설정
- 단일 사용자 로컬 앱
- API 키는 keytar 저장
- 정책 기본값: 저장 full_article, 공유 summary_only

## 30) API 명세 초안 (IPC)
```ts
// articles:upload
req: { filePaths: string[]; userNote?: string; sourceUrl?: string }
res: { batchId: string; queuedCount: number }

// articles:get-list
req: { page: number; limit: number; status?: string }
res: { data: ArticleDto[]; total: number }

// articles:sync-notion-batch
req: { articleIds: string[]; syncMode: 'summary_only'|'full_article'|'metadata_only' }
res: { successCount: number; failCount: number; errors: {id:string;message:string}[] }
```

## 31) 프론트엔드 컴포넌트 구조
- features/upload: Dropzone, UploadModal, BatchProgress
- features/archive: ArticleTable, Filters, BulkActionBar
- features/detail: ImagePane, ExtractForm, PolicyPanel
- features/settings: ProviderSettings, PolicySettings, StorageSettings

## 32) 백엔드/서비스 모듈 구조
- db/: schema, repo
- services/: ImageService, OCRService, AIService, SyncService, ExportService
- workers/: ProcessWorker, SyncWorker
- ipc/: handlers

## 33) 폴더 구조 제안
```text
src/
  main/
    db/
    ipc/
    services/
    workers/
  renderer/
    components/
    features/
    store/
    lib/
  shared/
    types.ts
```

## 34) 구현 우선순위
1. DB/IPC 골격
2. 단건 처리 end-to-end
3. 배치 큐 + 재시도
4. 목록/상세 UI
5. Notion 동기화
6. 웹 동기화/내보내기

## 35) 1~4주 로드맵
- 1주차: 보일러플레이트 + DB + IPC
- 2주차: 전처리/OCR/요약 파이프라인
- 3주차: 배치 큐/실패재시도/다중선택
- 4주차: 동기화/정책 가드레일/패키징

## 36) 테스트 전략
- 단위: Adapter, JSON parser, policy guard
- 통합: 10~100건 업로드 큐 처리
- E2E: 업로드→완료→일괄동기화

## 37) QA 체크리스트
- 180도/90도 회전 보정 정상
- 요약박스 없는 기사 자동 요약 정상
- 100건 업로드 시 UI 프리징 없음
- summary_only 정책 기본 적용
- 재시작 후 QUEUED 재개

## 38) 리스크/대응
- 429 제한: backoff + concurrency downshift
- 환각: confidence+검수필요 뱃지
- 저작권: full 공유 경고/로그

## 39) 샘플 데이터 구조
```json
{
  "id": "art_001",
  "batch_id": "bat_001",
  "title": "...",
  "summary_short": "...",
  "keywords": ["AI", "저작권"],
  "status": "COMPLETED"
}
```

## 40) 실제 예시 JSON
```json
{
  "orientation_detected": 90,
  "extraction": {
    "title": "AI 기술 발전과 저작권 이슈",
    "subtitle": "학습 데이터 적법성 논란",
    "source": "A일보",
    "reporter": "홍길동",
    "published_at": "2023-10-25",
    "article_body_text": "...",
    "summary_box_text": null,
    "summary_short": "생성형 AI 확산으로 저작권 논쟁이 커졌다.",
    "summary_long": "- 소송 증가\n- 가이드라인 필요\n- 산업 영향 확대",
    "category": "IT/과학",
    "keywords": ["AI", "저작권", "가이드라인"],
    "confidence_score": 0.95
  }
}
```

## 41) 노션 DB 속성 예시
- Name(Title)
- PublishedAt(Date)
- Source(Select)
- Reporter(RichText)
- Category(Select)
- Keywords(MultiSelect)
- SyncMode(Select)
- ArchiveLink(URL)

## 42) 공유 텍스트 예시
```text
[NewsVault 요약 공유]
▪ 제목: AI 기술 발전과 저작권 이슈
▪ 출처: A일보 (2023.10.25)
▪ 기자: 홍길동

💡 핵심 요약:
생성형 AI 확산으로 저작권 논쟁이 커졌고 기준 마련이 시급하다.

주요 내용:
- 학습 데이터 적법성 이슈 확대
- 관련 소송 증가
- 정책 가이드라인 정비 필요
```

## 43) 대량 처리 UX 문구
- 업로드: "총 52건이 큐에 등록되었습니다. 백그라운드 분석을 시작합니다."
- 정책경고: "현재 원문 포함 동기화가 선택되었습니다. 요약만으로 전환하시겠습니까?"
- 일괄복사: "선택 30건 중 완료 28건의 요약이 복사되었습니다."

## 44) 최종 추천안
### 1. 가장 추천하는 기술스택
- Electron + React + TypeScript + Vite
- SQLite(better-sqlite3)
- Zustand + TanStack Query/Table
- Sharp(OpenCV는 Phase3)
- OpenAI(vision+text) structured output
- Notion SDK + Adapter

### 2. 초기 구현 순서
1) Electron+React 보일러플레이트
2) SQLite 스키마/리포지토리
3) IPC 업로드/목록 API
4) 전처리+OCR 단건 파이프라인
5) 배치 워커/재시도
6) Archive UI/다중 선택
7) Notion sync
8) Export(JSON/CSV)

### 3. 바이브코딩 첫 태스크 10개
1. 프로젝트 스캐폴딩
2. DB 초기화 스크립트
3. 업로드 Dropzone
4. articles:upload IPC
5. queue worker
6. OCR+JSON parser
7. archive table
8. detail split view
9. notion adapter
10. bulk actions + retry

### 4. 하위 프롬프트 5개
1. "Electron+React+TS+SQLite 초기화 및 ping/pong IPC 구현"
2. "폴더/다중 업로드와 batch_id 발급 및 QUEUED 저장 구현"
3. "setInterval worker + OpenAI Vision JSON structured output 구현"
4. "Archive 테이블 다중선택 + 하단 bulk action bar 구현"
5. "Notion adapter로 summary_only 기본 동기화 및 sync status 업데이트 구현"
