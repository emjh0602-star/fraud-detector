# 자금 이상거래 감지 시스템 (FDS v1.0) — 보안 버전

지주사 재무팀 전용 · 로그인 인증 + 계정 관리 + 감사 로그 포함

---

## 보안 기능

| 기능 | 설명 |
|------|------|
| 개인 계정 로그인 | 팀원별 ID/PW, 8자 이상 비밀번호 |
| 권한 분리 | 관리자(admin) / 일반 사용자(user) |
| 세션 만료 | 8시간 후 자동 로그아웃 |
| 계정 관리 | 관리자만 계정 추가/삭제/PW초기화 |
| 비밀번호 변경 | 본인이 직접 변경 가능 |
| 감사 로그 | 모든 로그인/분석/내보내기 기록 |
| API 보호 | 비로그인 시 모든 API 차단 |

---

## 초기 관리자 계정

최초 실행 시 자동 생성됩니다.

- 아이디: `admin`
- 비밀번호: `admin1234!`

**⚠️ 배포 후 반드시 비밀번호를 변경하세요.**

---

## 폴더 구조

```
fraud-detector-v2/
├── app.py                  # Flask 서버 (인증 + 분석 엔진)
├── requirements.txt
├── templates/
│   ├── login.html          # 로그인 페이지
│   └── index.html          # 메인 앱 (분석 + 계정관리)
├── data/                   # 자동 생성
│   ├── users.json          # 계정 정보 (비밀번호 해시 저장)
│   ├── history.json        # 법인별 패턴 학습 데이터
│   └── audit.log           # 접속/행동 감사 로그
└── README.md
```

---

## 로컬 테스트

```bash
pip install -r requirements.txt
python app.py
# http://localhost:5000 접속
```

---

## 클라우드 배포

### Railway.app (가장 빠름 · 무료)

1. GitHub에 이 폴더 업로드
2. `Procfile` 파일 생성:
   ```
   web: gunicorn app:app
   ```
3. railway.app → New Project → GitHub 연결
4. 환경변수 `SECRET_KEY` 설정 (임의의 긴 문자열)
5. URL 발급 → 팀원에게 공유

### AWS EC2

```bash
# 서버 접속 후
cd fraud-detector-v2
python3 -m venv venv && source venv/bin/activate
pip install -r requirements.txt

# 환경변수 설정 (중요!)
export SECRET_KEY="여기에_임의의_긴_문자열_입력"

# 백그라운드 실행
gunicorn -w 4 -b 0.0.0.0:5000 app:app --daemon

# Nginx로 80포트 연결 (선택)
sudo apt install nginx -y
# /etc/nginx/sites-available/fds 에 proxy_pass 설정
```

### SECRET_KEY 생성 방법

```python
python3 -c "import secrets; print(secrets.token_hex(32))"
```

---

## 계정 관리 방법

1. 관리자로 로그인
2. 사이드바 → **계정 관리** 탭
3. **+ 계정 추가** 버튼 → 이름, 아이디, 초기 비밀번호 입력
4. 팀원에게 아이디/초기 비밀번호 전달
5. 팀원이 **내 계정** 탭에서 비밀번호 직접 변경

---

## 팀원 사용 가이드

### 처음 시작 (패턴 학습)
1. 법인명 입력 → 파일 유형: **과거 데이터** → 과거 엑셀 업로드
2. 54개 법인 순차 등록 (법인당 5분 이내)

### 매일 사용 (승인 전 검토)
1. 법인명 입력 → 파일 유형: **신규 거래** → 오늘 리스트 업로드
2. 이상/주의 건 확인 → 승인 판단
3. 필요시 CSV 내보내기 → 보고용 활용

