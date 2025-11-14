# CLAUDE.md

이 파일은 이 리포지토리에서 Claude Code(claude.ai/code)가 코드를 작업할 때 참고할 정보를 제공합니다.

## 빠른 시작 명령어

```bash
# 의존성 설치
npm install

# TypeScript 빌드
npm run build

# 개발 모드 (변경 감지)
npm run dev

# MCP 서버 시작 (v3 - 권장)
npm run start:v3

# HTTP 전송 모드로 시작 (stdio 대신)
npm run start:v3:http
```

## 프로젝트 개요

**Google Research MCP Server v3.0.0** - Google 검색 기능을 제공하는 Model Context Protocol(MCP) 서버입니다. AI 기반 연구 종합 기능을 포함하며, Claude Code, Claude Desktop 및 기타 MCP 호환 클라이언트와 통합됩니다.

### 주요 기능
- **Google Custom Search API 통합** - 품질 점수 및 중복 제거 포함
- **웹페이지 콘텐츠 추출** - 여러 형식 지원 (마크다운, HTML, 텍스트)
- **에이전트 기반 연구 종합** - 별도의 API 키 불필요 (기존 Claude 세션 활용)
- **소스 품질 평가** - 권위성 및 최신성 순위 매김
- **연구 깊이 레벨** - basic/intermediate/advanced (분석 복잡도 차이)

## 아키텍처

### 서비스 계층 (`src/services/`)

코드베이스는 연구의 다양한 측면을 처리하는 전문화된 서비스로 구성됩니다.

| 서비스 | 목적 |
|--------|------|
| `google-search.service.ts` | Google Custom Search API 통합 및 캐싱(5분 TTL). 검색 요청, 페이지네이션, API 상호작용 관리 |
| `content-extractor.service.ts` | Mozilla Readability를 사용한 웹페이지 콘텐츠 추출. 마크다운, HTML, 텍스트 포맷 지원. 배치 추출 지원(최대 5개 URL) |
| `source-quality.service.ts` | 권위성, 최신성, 도메인 타입, 콘텐츠 신선도를 기반으로 소스 순위 및 점수 매김. 연구 결과 우선순위 지정에 사용 |
| `deduplication.service.ts` | 검색 결과에서 중복 URL 및 유사 콘텐츠 식별 및 제거(약 30% 중복 제거율) |
| `research-synthesis.service.ts` | 에이전트 기반 분석(기본값) 또는 직접 API 호출(`USE_DIRECT_API=true`일 때)을 사용한 연구 결과 종합. 3가지 연구 깊이 레벨 지원 |

### 데이터 흐름

```
검색 쿼리
    ↓
Google Custom Search API (캐싱 포함)
    ↓
중복 제거 서비스 (중복 제거)
    ↓
소스 품질 서비스 (순위 지정 및 점수 매김)
    ↓
콘텐츠 추출 (상위 소스에서 추출)
    ↓
연구 종합 서비스
    ├─ 에이전트 모드 (기본값): 연구 데이터로 에이전트 실행
    └─ 직접 API 모드: Anthropic API 직접 호출
    ↓
포괄적인 연구 보고서
```

### MCP 진입점 (`src/google-search-v3.ts`)

다음을 수행하는 메인 서버 파일:
1. MCP 서버 초기화 (환경 변수 `MCP_TRANSPORT`에 따라 stdio 또는 HTTP 전송)
2. 4가지 도구 등록: `google_search`, `extract_webpage_content`, `extract_multiple_webpages`, `research_topic`
3. 입력 검증 및 출력 타이핑을 위해 Zod 스키마 사용
4. 모든 서비스를 통합하여 완전한 연구 기능 제공

### 중요한 설계 세부사항

- **에이전트 모드 (기본값)**: `ANTHROPIC_API_KEY` 또는 `USE_DIRECT_API=true`가 없을 때, 서버는 특수한 `[AGENT_SYNTHESIS_REQUIRED]` 마커를 반환합니다. Claude Code가 이를 감지하고 프롬프트에 패킹된 연구 데이터로 자동으로 에이전트를 실행합니다.
- **캐싱**: Google Search API 결과에 5분 TTL을 적용하여 할당량 사용을 줄입니다. 캐시 키는 쿼리 + 필터를 포함합니다.
- **품질 점수**: 각 결과는 여러 요소를 기반으로 0-10 품질 점수를 받습니다 (도메인 권위성, 결과 최신성, 소스 카테고리 신뢰도).
- **연구 깊이**:
  - `basic`: 3개 소스, 빠른 개요
  - `intermediate`: 5개 소스, 포괄적 분석 (기본값)
  - `advanced`: 8-10개 소스, 모순 및 권장사항 포함 심층 분석

## 환경 설정

프로젝트 루트에 `.env` 파일을 생성하고 다음을 추가하세요:

```
GOOGLE_API_KEY=your_api_key
GOOGLE_SEARCH_ENGINE_ID=your_search_engine_id
```

**중요**: `dotenv` 패키지가 자동으로 `.env` 파일을 로드하므로 별도 설정 없이 작동합니다.

선택사항:
```
ANTHROPIC_API_KEY=...         # 직접 API 모드용 (에이전트 모드에서는 불필요)
USE_DIRECT_API=true           # 에이전트 모드 대신 직접 API 모드 활성화
MCP_TRANSPORT=http            # stdio 대신 HTTP 사용 (기본값)
PORT=3000                     # HTTP 모드 포트 (기본값: 3000)
```

## Claude Desktop과 MCP 통합

### Claude Desktop 설정

1. Claude Desktop 설정 파일 열기:

**macOS/Linux:**
```bash
nano ~/.claude/claude_desktop_config.json
```

**Windows:**
```bash
notepad %APPDATA%\Claude\claude_desktop_config.json
```

2. 다음 설정 추가:

```json
{
  "mcpServers": {
    "google-research": {
      "command": "node",
      "args": ["/path/to/Google-Search-MCP-Server/dist/google-search-v3.js"]
    }
  }
}
```

경로를 실제 위치로 변경하세요. 예:
- macOS/Linux: `/home/username/Google-Search-MCP-Server/dist/google-search-v3.js`
- Windows: `C:\\Users\\username\\Google-Search-MCP-Server\\dist\\google-search-v3.js`

3. Claude Desktop 재시작

### Cursor 설정 (MCP 미지원)

Cursor는 현재 MCP를 지원하지 않습니다. Claude Desktop 또는 Claude Code 사용을 권장합니다.

### Claude Code 통합

Claude Code는 MCP 서버를 자동으로 감지하며, 별도 설정이 필요하지 않습니다.

## 서버 테스트

`npm run build`로 빌드한 후:

1. **서버 시작**:
   ```bash
   npm run start:v3
   ```

2. **별도 터미널에서 클라이언트로 테스트** (Claude Code, Claude Desktop 또는 직접 HTTP):
   - 서버가 어떤 종합 모드가 활성화되었는지 표시하는 시작 정보 출력
   - 연구 쿼리가 실행될 때, 출력에서 `[AGENT_SYNTHESIS_REQUIRED]` (에이전트 모드) 또는 직접 API 호출 (직접 모드) 확인

3. **예상 시작 출력**:
   ```
   Google Research MCP Server v3.0.0 (Enhanced)
   ✓ Source quality assessment
   ✓ Deduplication
   ✓ AI synthesis: AGENT MODE (Claude will launch agents)
   ```

## 주요 코드 위치

- **도구 등록**: `src/google-search-v3.ts:200+` (도구가 MCP에 등록되는 부분)
- **검색 로직**: `src/services/google-search.service.ts` (googleapis 패키지 사용)
- **품질 점수 알고리즘**: `src/services/source-quality.service.ts` (0-10 점수 계산)
- **콘텐츠 추출**: `src/services/content-extractor.service.ts` (@mozilla/readability 사용)
- **에이전트 종합 감지**: `src/services/research-synthesis.service.ts:20-40` (에이전트 모드 설정)

## 일반적인 개발 작업

### 새 도구 추가

1. `src/google-search-v3.ts`에서 Zod 입력/출력 스키마 정의
2. 관련 서비스를 호출하는 핸들러 함수 생성
3. 메인 서버 설정에서 `server.setRequestHandler(Tool, handler)`로 등록
4. 필요시 `src/types.ts`의 타입 업데이트

### 품질 점수 수정

품질 점수 알고리즘은 `src/services/source-quality.service.ts`에 있습니다. 점수는 0-10이며 다음에 기반합니다:
- 도메인 권위성 (학술, 공식, 뉴스, 포럼 등)
- 결과 최신성 (최근 결과에 신선도 부스트)
- 카테고리 신뢰도 가중치

### 연구 종합 동작 변경

에이전트 모드 vs 직접 API 모드는 `src/services/research-synthesis.service.ts` 생성자에서 제어됩니다. 서비스는 모드 선택을 위해 환경 변수를 감지하고 모드 선택에 따라 다른 응답 형식을 반환합니다.

## 의존성

주요 패키지:
- `@modelcontextprotocol/sdk` - MCP 프로토콜 구현
- `googleapis` - Google Custom Search API 클라이언트
- `@mozilla/readability` - 깔끔한 콘텐츠 추출
- `zod` - 스키마 검증 및 타이핑
- `axios` - 콘텐츠 추출용 HTTP 요청
- `cheerio` - HTML 파싱
- `turndown` - HTML을 마크다운으로 변환
- `jsdom` - 콘텐츠 추출을 위한 DOM 시뮬레이션

## 문제 해결

### 서버가 시작되지 않음
- Node.js 버전 확인 (18 이상 필요)
- 환경 변수가 설정되어 있는지 확인
- 먼저 `npm run build`를 실행하여 TypeScript 컴파일

### 검색에서 결과 없음
- Google API 키가 유효한지 확인
- Custom Search Engine ID 존재 및 인덱싱 활성화 여부 확인
- 더 광범위한 검색어로 시도

### 에이전트 모드가 트리거되지 않음
- `npm run start:v3`이 실행 중인지 확인 (v2 아님)
- 서버 시작 시 "AGENT MODE" 메시지 확인
- Claude Code/Desktop과 MCP 통합이 있는지 확인
