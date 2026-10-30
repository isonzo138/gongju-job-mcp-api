# 공주잡(Gongju Job) API 통합 정리 문서

> 2026-10-30 기준. 실제 API 호출로 검증된 내용만 포함합니다.

---

## 1. 전체 구조 한눈에 보기

```
[공공데이터 원천]                [MCP 서버]                  [사용자]
KOSIS 국가통계포털  ──연결──▶  gongju-job MCP  ◀──연결──  Claude/Cursor/
(지역별고용조사 통계)          (FastMCP, STDIO)           MCP Inspector
                                     │
고용24(워크넷) 채용정보 ──미연동──▶   │  (기업회원 전용,
                                     │   코드는 준비 완료)
                                     ▼
                          7개 Tool이 JSON 결과 반환
                          (출처·기준시점·단위 포함)
```

- **사용 중**: KOSIS Open API 1종 (통계 2개 표)
- **미연동**: 고용24 채용정보 API (기업회원 전용이라 인증키 발급 불가 → `WORK24_ENABLED=false`)

---

## 2. KOSIS Open API (사용 중)

### 2-1. 기본 정보

| 항목 | 값 |
|---|---|
| 제공기관 | 국가데이터처 (조사: 지역별고용조사) |
| 인증키 발급 | https://kosis.kr/openapi/ (회원가입 → 활용신청) |
| API 문서 | https://kosis.kr/openapi/devGuide/devGuide_0201List.do |
| 형식 | JSON (HTTP GET) |
| 인증키 저장 위치 | 프로젝트 폴더 `.env` → `KOSIS_API_KEY` |
| 호출 제한 | 분당 호출 수 제한 있음 (2026-07-09 공지) |

### 2-2. 사용 중인 통계표 (2개)

공주시·천안시를 **동일 조사·동일 표·동일 시점·동일 연령 기준(15세 이상)**으로 비교하기 위해 선정한 시군구 단위 표입니다.

**① 시군구 경제활동인구 총괄** — `orgId=101`, `tblId=DT_1ES3A01S`, 주기 H(반기)

| 항목 ID | 항목명 | 단위 |
|---|---|---|
| T1 | 15세이상인구 | 천명 |
| T2 | 경제활동인구 | 천명 |
| T3 | 취업자 | 천명 |
| T4 | 실업자 | 천명 |
| T5 | 비경제활동인구 | 천명 |
| T6 | 경제활동참가율 | % |
| **T7** | **고용률** | % |
| **T8** | **실업률** | % |
| T11 | 15~64세 고용률 | % |

**② 시군구/산업별 취업자(10차)** — `orgId=101`, `tblId=DT_1ES3A05S`, 주기 H(반기)

| 산업 분류 ID (I분류) | 산업명 |
|---|---|
| 00 | 계 |
| 01 | 농업, 임업 및 어업 (A) |
| 04 | 광·제조업 (B,C) |
| 41 | 건설업 (F) |
| 42 | 도소매·숙박음식업 (G,I) |
| 43 | 전기·운수·통신·금융 (D,H,J,K) |
| 44 | 사업·개인·공공서비스 및 기타 (E,L~U) |

- 이 표의 항목 ID는 `ALL`로 호출해야 취업자(T3) 값이 반환됩니다 (실제 검증).

### 2-3. 통계자료 API (통계표선택 방식) — 실제 호출 방법

**요청 URL**
```
https://kosis.kr/openapi/Param/statisticsParameterData.do
```

**요청 파라미터 (GET)**

| 파라미터 | 필수 | 설명 | 사용 예 |
|---|---|---|---|
| apiKey | Y | 발급받은 인증키 | `KZ6IE17B...` |
| orgId | Y | 기관 ID (국가데이터처) | `101` |
| tblId | Y | 통계표 ID | `DT_1ES3A01S` |
| objL1 | Y | 분류1 값 (행정구역) | `3402`(공주시) |
| objL2 | 산업별 표에서만 | 분류2 값 (산업) | `44` |
| itmId | Y | 항목 ID | `T7` 또는 `ALL` |
| prdSe | Y | 수록주기 | `H`(반기) |
| format | Y | 결과 유형 | `json` |
| newEstPrdCnt | 최신기준 | 최근 N개 시점 | `6` |
| startPrdDe / endPrdDe | 시점기준 | 시작/종료 시점 | `202301` / `202601` |

- 시점 조건은 (start+end) 또는 (newEstPrdCnt) 중 택1. 미지정 시 최근 1개 시점.

**지역 코드 (지역별고용조사 자체 코드)**

| 지역 | 코드 |
|---|---|
| 천안시 | `3401` |
| 공주시 | `3402` |

**시점 코드 표기**: `202601` = 2026년 상반기, `202502` = 2025년 하반기

**호출 예시 (공주시 고용률 최근 2개 시점)**
```
https://kosis.kr/openapi/Param/statisticsParameterData.do
  ?apiKey=인증키&orgId=101&tblId=DT_1ES3A01S
  &objL1=3402&itmId=T7&prdSe=H&newEstPrdCnt=2&format=json
```

**응답 형식 (실제 검증)**
```
[{DT:"60.1"},{DT:"59.1"}]
```
`outputFields`를 생략하면 전체 필드가 옵니다:
```
[{TBL_ID:"DT_1ES3A01S", ORG_ID:"101", ITM_NM:"고용률 (%)",
  C1:"3402", C1_NM:"공주시", PRD_DE:"202502", DT:"60.1",
  PRD_SE:"S", LST_CHN_DE:"2026-02-12", ...}]
```

**출력 필드 정리**

| 필드 | 의미 |
|---|---|
| PRD_DE | 수록시점 (예: 202601) |
| DT | 수치값 (문자열) |
| C1 / C1_NM | 분류1 코드/명 (행정구역) |
| C2 / C2_NM | 분류2 코드/명 (산업, 산업별 표만) |
| ITM_ID / ITM_NM | 항목 ID/명 |
| LST_CHN_DE | 최종수정일 |

### 2-4. KOSIS 실무 발견 사항 (실제 응답 검증)

1. **비표준 JSON**: 키에 따옴표가 없는 형태(`{DT:"59.1"}`)로 응답 → 파서에서 정규식으로 키에 따옴표 보강 후 파싱 (`kosis_api.py`)
2. **outputFields 함정**: 이 파라미터를 지정하면 전체 필드가 아니라 **DT 값만** 반환됨 → 반드시 생략
3. **산업별 표의 itmId**: `T1`이 아니라 `ALL`이어야 취업자 값 반환 (`T1`은 오류 21: 잘못된 요청 변수)
4. **결측/비밀 표기**: `-`, `...`, `*`(비밀조치)는 0으로 대체하지 않고 `None`(미제공) 처리
5. **오류 응답**: `{err:"21", errMsg:"잘못된 요청 변수를 호출 하였습니다."}`, `{err:"20", errMsg:"필수요청변수값이 누락..."}` 형태
6. **메타 API**(참고): `https://kosis.kr/openapi/statisticsData.do?method=getMeta&type=TBL` (통계표 명칭), `type=ITM` (분류/항목)

### 2-5. 보조 API (KOSIS 웹에서 코드 확인 시 사용)

- 통계표 화면: `https://kosis.kr/statHtml/statHtml.do?orgId=101&tblId=DT_1ES3A01S`
- 검색: `https://kosis.kr/search/search.do?query=지역별고용조사`
- OpenAPI 활용신청: `https://kosis.kr/openapi/`

---

## 3. 고용24(워크넷) 채용정보 API (미연동, 코드 준비 완료)

### 3-1. 기본 정보

| 항목 | 값 |
|---|---|
| 제공기관 | 한국고용정보원 |
| API 명 | 채용정보 API - 채용정보목록/상세 |
| 엔드포인트 | `https://www.work24.go.kr/cm/openApi/call/wk/callOpenApiSvcInfo210L01.do` |
| 형식 | XML (`returnType=XML` 필수) |
| 갱신 주기 | 실시간 (워크넷 인증 구인공고) |
| 이용 자격 | **기업회원 전용** (개인회원 인증키로 호출 시 `개인회원은 사용할 수 없는 OPEN-API입니다` 오류 — 실제 확인) |
| 현재 상태 | 미연동 (`WORK24_ENABLED=false`) |

### 3-2. 요청 파라미터 (GET)

| 파라미터 | 필수 | 설명 |
|---|---|---|
| authKey | Y | 인증키 |
| callTp | Y | `L` 목록 / `D` 상세 |
| returnType | Y | `XML` 고정 |
| startPage | Y | 시작 위치 (1~1000) |
| display | Y | 출력건수 (1~100) |
| region | 선택 | 근무지역코드 (다중 가능) |
| occupation | 선택 | 직종코드 (다중 가능, `|` 구분) |
| salTp | 선택 | 임금형태: `D` 일급 / `H` 시급 / `M` 월급 / `Y` 연봉 |
| minPay / maxPay | salTp 지정시 | 최소/최대급여 (원 단위) |
| education | 선택 | `00` 학력무관 ~ `07` 박사 |
| career | 선택 | `N` 신입 / `E` 경력 / `Z` 무관 (E일 때 minCareerM/maxCareerM 필수) |
| keyword | 선택 | 키워드 (UTF-8) |
| wantedAuthNo | 상세 조회시 | 구인인증번호 (callTp=D) |

**사용 코드 (실제 코드표에서 확인)**

| 구분 | 코드 |
|---|---|
| 공주시 | `44150` |
| 천안시 (전체) | `44130` |
| 천안시 동남구 | `44131` |
| 천안시 서북구 | `44133` |
| 데이터 분석가 | `135200` |
| 데이터 시스템 전문가 | `135100` |
| 데이터 설계 및 프로그래머 | `135101` |
| 데이터베이스 운영·관리자 | `135102` |
| 응용 소프트웨어 개발자 | `133200` |
| 시스템 소프트웨어 개발자 | `133100` |
| 컴퓨터시스템 전문가 | `132000` |
| IT 기술지원 전문가 | `136103` |

**호출 예시 (공주시 채용공고 20건)**
```
https://www.work24.go.kr/cm/openApi/call/wk/callOpenApiSvcInfo210L01.do
  ?authKey=인증키&callTp=L&returnType=XML&startPage=1&display=20
  &region=44150
```

**출력 필드 (wantedRoot > wanted 반복)**

| 필드 | 의미 |
|---|---|
| wantedAuthNo | 구인인증번호 |
| company / busino | 회사명 / 사업자등록번호 |
| indTpNm | 업종 |
| title | 채용제목 |
| salTpNm / sal / minSal / maxSal | 임금형태 / 급여 |
| region / basicAddr / detailAddr | 근무지역 / 주소 |
| minEdubg / maxEdubg | 최소/최대 학력 |
| career | 경력 |
| empTpCd | 고용형태코드 (10 무기한, 20 유기한 등) |
| jobsCd | 직종코드 |
| regDt / closeDt | 등록일자 / 마감일자 |
| wantedInfoUrl | 채용정보 URL |
| infoSvc | 정보제공처 (VALIDATION = 워크넷 인증) |

### 3-3. 미연동 사유와 전환 방법

- **사유**: OPEN-API가 기업회원 전용이라 일반 사용자가 인증키를 발급받을 수 없음 (실제 발급 후 테스트에서 오류 확인)
- **전환 방법**: 기업회원 인증키 확보 시 `.env`에서 `WORK24_ENABLED=true` + `WORK24_API_KEY` 입력 → 코드 수정 없이 `get_job_postings`/`recommend_jobs`가 실제 공고 기반으로 전환
- **대안 (현재)**: 워크넷 웹 https://www.work.go.kr 에서 지역·직종 검색

---

## 4. MCP Tool 7개와 API 매핑

| # | Tool | 사용 API | 동작 |
|---|---|---|---|
| 1 | `get_employment_stats` | KOSIS DT_1ES3A01S | 지역의 9개 지표를 반기 시계열로 반환 |
| 2 | `compare_regions` | KOSIS DT_1ES3A01S (+DT_1ES3A05S) | 두 지역 지표 비교·격차·우위·전년 대비 증감·산업구조 |
| 3 | `get_industry_stats` | KOSIS DT_1ES3A05S | 산업별 취업자 + 지역 내 비중(%) |
| 4 | `get_job_postings` | 고용24 (미연동) | 미연동 안내 + 워크넷 웹 안내 반환 |
| 5 | `analyze_employment_gap` | KOSIS 두 표 | 고용 문제 진단: 확인된 사실 / 가설 / 데이터 한계 분리 |
| 6 | `recommend_jobs` | 현재: KOSIS / 연동 시: 고용24 | 현재: 희망 직무 관련 산업군 비중 등 지역 일자리 환경 참고 정보. 연동 시: 공고 매칭 스코어링(지역30+직무30+급여20+학력10+경력10) |
| 7 | `get_data_source_info` | - | 출처·통계표 ID·기준연령·단위·운영모드 메타 |

**모든 Tool 응답 공통 원칙**
1. 실제 API 응답 값만 사용 (없는 값은 임의 생성 금지)
2. 결측은 0 대체 없이 `null`/미제공 표기
3. 공주-천안 비교는 동일 표·동일 시점·동일 연령 기준(15세 이상)만 수행
4. 15~29세/19~34세 청년 구분은 시군구 표에 없어 추정하지 않음 (`youth_age_note`로 명시)
5. `source`(제공기관·조사·표·표ID·단위) + `ok` 필드를 항상 포함

---

## 5. 실데이터 검증 결과 (2026-10-30)

MCP 서버 실구동 후 프로토콜 호출로 확인한 값:

**공주시 (2026년 상반기, 최신)**
- 고용률 59.1% / 실업률 3.3% / 경제활동참가율 61.1%
- 취업자 57.2천명 / 경제활동인구 59.2천명 / 15세이상인구 96.8천명
- 15~64세 고용률 64.3%

**천안시 (2026년 상반기)**
- 고용률 64.8% / 실업률 1.8% / 경제활동참가율 66.0%
- 취업자 405.2천명 / 경제활동인구 412.8천명 / 15세이상인구 625.8천명

**2025년 하반기 고용률**: 공주 60.1% vs 천안 63.4% (격차 3.3%p)

**산업구조 (2024년 상반기)**
- 공주시: 농림어업 23.9% / 사업·공공서비스 37.4% / 도소매·숙박음식 15.7% / 광·제조 11.7% / 건설 6.3% / 전기·통신·금융 5.0%
- 천안시: 광·제조 29.1% / 사업·공공서비스 36.2% / 도소매·숙박음식 15.1% / 전기·통신·금융 8.9% / 건설 6.8% / 농림어업 3.9%(값 `*` 부분은 미제공 처리)
- 격차 최대: 농림어업 +20.0%p (공주 우위), 광·제조 -17.4%p (천안 우위)

**Tool별 PASS 결과**: 7개 전부 PASS (get_job_postings는 미연동 안내 정상 반환 포함)

---

## 6. 실행 방법 요약

```bat
:: MCP 서버 실행 (STDIO)
cd gongju-job-mcp
.venv\Scripts\python server.py

:: MCP Inspector (브라우저 테스트 도구)
run_inspector.bat  더블클릭 → 브라우저 자동 열림 → Connect → Tools 탭 → Execute Tool
```

- Claude Desktop 연결: `%APPDATA%\Claude\claude_desktop_config.json`의 `mcpServers`에 python.exe 경로 + server.py 경로 등록 (README 6-3 참조)
- 테스트 스크립트: `test_modules.py`(단위), `test_mcp_client.py`(프로토콜), `test_real_data.py`(실데이터 5시나리오), `test_final_real.py`(최종 통합)
