# FDS (자금 이상거래 감지 시스템) - Claude Code 설계 문서

> **이 파일은 Claude Code가 새 대화에서 자동으로 읽는 프로젝트 가이드입니다.**
> 대화가 끊겨도 이 문서를 기반으로 연속 작업이 가능합니다.

## 프로젝트 한줄 요약
지주사 재무팀(6명)이 54개 그룹사 법인의 자금 최종승인 전 이상거래를 감지하는 웹 시스템.

## 기술 스택
- **Backend**: Python Flask + openpyxl + pandas + numpy
- **Frontend**: 순수 HTML/CSS/JS (프레임워크 없음, SPA 스타일)
- **배포**: Railway.app (GitHub push → 자동 재배포 1~2분)
- **데이터 저장**: 서버 파일 기반 JSON, Railway Volume 마운트
- **보안**: werkzeug password hashing, 세션 24h, 감사 로그, threading.Lock 파일 보호

## 핵심 파일 구조
```
app.py                  # 서버 전체 로직 (Flask 라우트 + 분석 엔진, ~700줄)
templates/index.html    # 메인 앱 화면 (SPA, ~790줄)
templates/login.html    # 로그인 화면
Procfile                # Railway: gunicorn 실행
requirements.txt        # flask, pandas, openpyxl, xlrd, numpy, gunicorn
docs/                   # 프로젝트 문서 및 작업 이력

# 서버 런타임 데이터 (Railway Volume: /app/data)
data/history.json       # 법인별 패턴 학습 데이터 (과거 거래 누적)
data/corps.json         # 법인 목록 + 담당자 매핑
data/users.json         # 계정 정보 (werkzeug 해시)
data/results.json       # 분석 결과 이력 (법인별 최근 30개)
data/audit.log          # 감사 로그
```

## 배포 정보
- **URL**: https://fraud-detector-production-c6a3.up.railway.app
- **GitHub**: https://github.com/emjh0602-star/fraud-detector
- **Railway Volume**: fraud-detector-volume → /app/data 마운트
- **환경변수**: `DATA_DIR=/app/data`, `SECRET_KEY` 설정됨

---

## 아키텍처 상세

### app.py 구조 (위에서 아래로)
1. **임포트 & 설정** (1~24): Flask 앱, 환경변수, 파일 경로, threading.Lock
2. **유틸 함수** (26~80): hash_pw, verify_pw, load/save JSON, audit 로깅
3. **인증 데코레이터** (82~100): login_required, admin_required
4. **페이지 라우트** (102~130): `/`, `/login`, `/logout`
5. **법인 관리 API** (132~200): CRUD `/api/corps`
6. **분석 API** (202~370): `/api/upload` (핵심), `/api/results`, `/api/export`
7. **관리자 API** (375~465): 유저 CRUD, 감사 로그, 비밀번호 관리
8. **분석 엔진** (480~): 스마트 파서, 감지 규칙, 패턴 통계

### 엑셀 스마트 파서 흐름
```
파일 업로드 → smart_parse_excel()
  → openpyxl로 워크북 열기
  → 모든 시트 순회 → parse_single_sheet()
    → 헤더 행 탐지 (1~60행, 키워드 매칭)
    → 컬럼 자동 매핑 (payee, amount, date, account, memo, bank)
    → 합계/소계 행 필터링 (SKIP_KEYWORDS)
    → 데이터 행 추출
  → 실패 시 pandas read_excel 폴백
```

### 이상거래 감지 규칙 (analyze_transactions)
| 규칙 | 가중치 | 구현 위치 |
|------|--------|-----------|
| 5억원 이상 초고액 | +3 | amt>=500_000_000 |
| 1억원 이상 고액 | +2 | amt>=100_000_000 |
| 5천만원 이상 | +1 | amt>=50_000_000 |
| 법인 평균 대비 5배 초과 | +2 | amt>avg_amt*5 |
| 상위 5% 금액의 2배 초과 | +2 | amt>p95_amt*2 |
| 신규 거래처 | +1 | payee not in hist_payees |
| 신규 거래처 + 1천만원 이상 | +1 | 위 + amt>=10_000_000 |
| 동일 수취인 당일 3건 이상 | +1 | date_payee_count 기반 |
| 동일 금액 3건 이상 반복 | +1 | amount_count 기반 |
| 긴급 처리 요청 | +1 | 적요에 긴급/urgent 등 |
| 계좌번호 미기재 + 100만원↑ | +1 | account 비어있고 amt>=1M |

**판정**: 4점↑ = 이상 / 2~3점 = 주의 / 0~1점 = 정상

### 데이터 흐름
```
[과거 데이터 업로드] → history.json에 누적 저장 (payee+amount+date 중복 제거)
[신규 거래 분석] → history 기반 패턴 비교 → 위험도 판정
                → results.json에 결과 저장 (법인별 최근 30개)
                → cumul_key(__cumul__)에 신규 거래도 누적 학습
```

### 권한 체계
- **관리자(admin)**: 모든 법인 접근, 법인/계정 CRUD, 감사 로그
- **일반 사용자(user)**: 담당 법인만 접근 (corps.json의 manager 필드)
- 분석 결과 조회/학습 데이터 삭제도 담당자/관리자만 가능

### 비밀번호 보안
- 신규: werkzeug `generate_password_hash` (pbkdf2/scrypt)
- 기존 SHA256 계정: 로그인 시 자동 마이그레이션 (`verify_pw` 함수)
- 최소 8자 요구

---

## 개발 시 주의사항

### 절대 하지 말 것
- `app.py`의 `analyze_transactions`에서 `global pattern_stats` 제거하지 말 것 (upload에서 참조)
- history.json 중복 제거를 payee만으로 하지 말 것 (payee+amount+date 조합 필수)
- `_file_lock` 제거하지 말 것 (동시 쓰기 보호)
- 기존 SHA256 해시 호환 코드(`verify_pw`) 삭제하지 말 것 (마이그레이션 전 계정 존재)

### 엑셀 파서 확장 시
- 법인마다 양식이 다름 → `header_keywords` 리스트에 새 키워드 추가
- `SKIP_KEYWORDS`에 새로운 합계/소계 패턴 추가
- `parse_single_sheet`의 컬럼 매핑 키워드 확장
- 헤더 매칭 임계값: 5개 이상이면 확정, 3개 이상이면 후보

### 프론트엔드 수정 시
- `index.html` 하나에 모든 UI가 있음 (SPA 스타일, 페이지 전환은 JS `goPage()`)
- CSS 변수는 `:root`에 정의 (`--danger`, `--warning`, `--ok` 등)
- API 호출은 모두 `fetch()`로, 401 응답 시 `/login`으로 리다이렉트

---

## 54개 법인 온보딩 작업

**작업 지침**: `docs/05_54법인_작업_지침.md` (전체 프로세스, 키워드 목록, 판단 기준)
**검증 현황**: `docs/06_법인별_검증현황.md` (법인별 진행 상태 추적표)

작업 요청이 들어오면:
1. `06_검증현황.md`에서 현재 진행 상태 확인
2. `05_작업_지침.md`의 프로세스에 따라 검증 → 학습 → 확인
3. 파서 수정이 필요하면 `app.py`의 키워드 리스트 수정
4. 완료 후 `06_검증현황.md` 표 업데이트

---

## 현재 진행 상황 & 남은 작업

### 완료
- [x] 1차 설계 및 전체 기능 구현
- [x] 3개 법인 테스트 (더편한샵, 디엔코스메틱스 등)
- [x] 코드 품질 개선 (2026-04-17 세션)
  - pattern_stats 버그 수정, 데드 코드 제거, 보안 강화 등

### 남은 작업
- [ ] **51개 법인 엑셀 파일 분석 테스트** (최우선)
- [ ] 법인별 파서 호환성 보완 (새 양식 발견 시)
- [ ] 시트 여러 개인 엑셀 → 시트별 분리 도구 (별도 기능)
- [ ] (선택) app.py 모듈 분리 리팩토링

### 상세 작업 이력
→ `docs/` 폴더의 핸드오프 문서 참조
