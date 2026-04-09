# Red/Blue Team 실시간 공격·방어 실습 환경

OWASP Juice Shop + Docker Compose + Suricata + Web Dashboard

`bash start.sh` 한 번이면 Docker 설치부터 **공격 12종 + IDS 탐지 11종 + 웹 대시보드**까지 로컬에서 즉시 동작합니다.

> **Windows 사용자**: `start.bat` 대신 **WSL2 Ubuntu** 또는 **Git Bash** 에서 `bash start.sh`를 실행하세요. `.sh` 스크립트가 Docker 자동 설치, 패킷 캡처 도구 설치 등을 모두 처리합니다. `.bat` 파일은 이러한 자동화가 포함되어 있지 않습니다.

### 설치 및 실습 데모

https://github.com/kentech-stis-lab/teaching_Realtime-Attack_Defense/blob/main/%EC%84%A4%EC%B9%98%20%EB%B0%8F%20%EC%8B%A4%EC%8A%B5%EB%8D%B0%EB%AA%A8.mp4


### AI-IDS 실습 구글 드라이브

https://buly.kr/GZzOwo5
---

## 사용법

### 1. 환경 시작

```bash
git clone https://github.com/kentech-stis-lab/teaching_Realtime-Attack_Defense.git
cd teaching_Realtime-Attack_Defense

# Linux / Mac / Windows(WSL2, Git Bash) — 권장
bash start.sh
# → sudo 비밀번호 입력 → Docker 자동 설치 → 빌드 → 8개 컨테이너 기동
```

`start.sh`와 `capture-wireshark.sh`는 내부적으로 `install-deps.sh`를 호출하여 모든 의존성을 자동으로 해결합니다 (Docker Engine, Docker 데몬, tcpdump, tshark, docker 그룹 등). 의존성만 별도로 설치하려면 `bash install-deps.sh`.

| 서비스 | URL | 용도 |
|--------|-----|------|
| **Dashboard** | http://localhost:8080 | 공격/방어/모니터링 웹 UI |
| Juice Shop | http://localhost:3000 | 공격 대상 웹앱 |
| API Docs | http://localhost:8000/docs | FastAPI Swagger |
| SSH | `ssh admin@localhost -p 2222` | Brute Force 대상 (pw: admin123) |
| FTP | `ftp localhost 2121` | Brute Force 대상 (pw: admin123) |

### 2. 대시보드에서 공격 실행

브라우저에서 **http://localhost:8080** 접속 → Attack Panel에서 공격 선택 → **[Run]** 클릭

12종의 PoC 공격을 웹 UI에서 바로 실행하고, 실시간으로 Suricata IDS 탐지 결과를 확인할 수 있습니다.

공격을 실행하면 **터미널 위에 패킷 필터 바**가 자동으로 표시됩니다. 해당 공격의 Wireshark/tshark 필터 검색식이 나타나므로, 별도 터미널에서 `capture-wireshark.sh`를 실행한 후 이 필터를 적용하면 해당 공격 패킷만 골라볼 수 있습니다.

### 3. 패킷 캡처 (공격 payload 분석)

> **중요**: 공격 payload (예: `' OR 1=1--`, `<script>alert()</script>`)는 Docker 내부 **backend-net** (172.20.1.0/24)에서 발생합니다. Windows Wireshark의 loopback 캡처로는 이 트래픽을 볼 수 없으며, **WSL2에서 `capture-wireshark.sh`를 실행**해야 합니다.

```bash
bash capture-wireshark.sh
# → sudo 비밀번호 입력 → 캡처 모드 선택 → Docker 내부 공격 패킷 캡처
```

캡처 모드:
| 모드 | 설명 | 용도 |
|------|------|------|
| 1) tshark -V (기본) | 터미널에 패킷 + payload 상세 출력 | 실시간 확인 |
| 2) pcap 파일 저장 | `.pcap` 파일로 저장, Wireshark에서 열기 | Windows Wireshark 사용 시 |
| 3) tcpdump | 경량 터미널 + hex dump | 상세 분석 |
| 4) wireshark GUI | Wireshark GUI 창 | Linux GUI 환경 |

**pcap 파일 모드** (모드 2)가 가장 권장됩니다:
1. `capture-wireshark.sh` 실행 → 모드 2 선택
2. 대시보드에서 공격 실행
3. Ctrl+C로 캡처 종료 → `.pcap` 파일 생성
4. Windows Wireshark에서 `.pcap` 파일 열기
5. **Analyze → Decode As → TCP port 3000 = HTTP** (필수)
6. Display filter 적용: `ip.dst == 172.20.1.10 && http`
7. 패킷 우클릭 → **Follow → HTTP Stream** → payload 확인

공격별 payload 예시:
| 공격 | 대상 | payload 위치 |
|------|------|-------------|
| **SQLi** | 172.20.1.10 (Juice Shop) | POST body: `{"email":"' OR 1=1--"}` |
| **XSS** | 172.20.1.10 (Juice Shop) | URI/body: `<iframe>`, `<script>` |
| **Brute Force (Web)** | 172.20.1.10 (Juice Shop) | POST body: 반복 login |
| **Brute Force (SSH)** | 172.20.0.11:2222 | SSH auth attempts |
| **Brute Force (FTP)** | 172.20.0.12:21 | FTP USER/PASS commands |
| **Bot** | 172.20.1.10 (Juice Shop) | rapid GET /api/Products |
| **DoS** | 172.20.1.10:3000 | partial HTTP headers (Slowloris) |
| **Port Scan** | 172.20.0.0/24 | SYN to many hosts/ports |
| **Infiltration** | 172.20.1.30/172.20.1.20 | internal API/DB access |

> **HTTP Decode 자동 적용**: `capture-wireshark.sh`는 `-d tcp.port==3000,http` 플래그를 자동으로 적용합니다. pcap 파일을 Wireshark GUI에서 열 때는 수동으로 **Analyze → Decode As → TCP port 3000 = HTTP** 설정이 필요합니다.

### 4. 환경 종료

```bash
bash stop.sh       # Linux/WSL2
stop.bat           # Windows
```

---

## 트러블슈팅

| 증상 | 원인 | 해결 |
|------|------|------|
| `docker: command not found` | Docker 미설치 | `start.sh`가 자동 설치. 수동: 아래 가이드 참고 |
| `Cannot connect to the Docker daemon` | Docker 데몬 미실행 | `start.sh`가 자동 시작. 수동: `sudo service docker start` |
| `apt-get` 실행 시 lock 에러 | 자동 업데이트(unattended-upgr) 점유 | 잠시 기다린 후 재시도, 또는 `sudo kill <PID>` |
| `port is already allocated` | 다른 프로세스가 해당 포트 사용 중 | `bash stop.sh` 후 재시작, 또는 충돌 프로세스 종료 |
| 패킷 캡처 권한 에러 | root 권한 필요 | `capture-wireshark.sh`가 sudo로 실행. 비밀번호 입력 필요 |
| 캡처에 공격 패킷 안 보임 | 잘못된 인터페이스 | `capture-wireshark.sh`가 Docker bridge를 자동 탐지 |
| `http` 필터에 패킷 안 잡힘 | 비표준 포트(3000/5000/8000) | `capture-wireshark.sh`가 `-d` 플래그로 자동 적용. Wireshark GUI: Analyze → Decode As → TCP port 3000 = HTTP |
| Wireshark에서 캡처 안 됨 (Windows) | Npcap 미설치 | `capture-wireshark.bat`가 자동 설치. 수동: [npcap.com](https://npcap.com) |
| Dashboard 접속 안 됨 (`localhost:8080`) | backend 미기동 | `docker logs backend`로 확인 후 `bash start.sh` 재실행 |

> **WSL2 참고**: WSL2를 재시작할 때마다 `sudo service docker start`를 실행해야 합니다. `start.sh`가 자동으로 처리합니다.

---

## 아키텍처

```
            ┌────────────── frontend-net (172.20.0.0/24) ──────────────┐
            │                                                           │
Student ──→ [Dashboard .2]──→ [Backend .100:8000]──→ [Juice Shop .10:3000]
Browser     [:8080]                  │                [SSH .11:2222]
            │                        │                [FTP .12:21]
            │                        │                [Suricata .13]
            │                        │
            │     ┌─── backend-net (172.20.1.0/24) ──────────┐
            │     │                                           │
            │     │  Backend .100 ──→ Juice Shop .10:3000     │
            │     │                ──→ internal-api .30:5000   │
            │     │                ──→ MySQL .20:3306          │
            │     └───────────────────────────────────────────┘
            │
            └── alert ← eve.json ← 공격 실행 시 자동 기록
```

> 모든 컨테이너 IP는 `docker-compose.yml`에서 고정 할당됩니다 (DHCP 아님).

**8개 Docker 서비스**: Juice Shop, SSH, FTP, Suricata, MySQL, Flask API, FastAPI Backend, React Dashboard

## 사전 준비된 공격 PoC (12종)

| ID | 유형 | 대상 | 언어 | CIC-IDS Label |
|----|------|------|------|---------------|
| sqli_login_bypass | SQLi | Juice Shop | Python | Web Attack - Sql Injection |
| sqli_union_select | SQLi | Juice Shop | Python | Web Attack - Sql Injection |
| xss_dom | XSS | Juice Shop | Python | Web Attack - XSS |
| xss_reflected | XSS | Juice Shop | Python | Web Attack - XSS |
| xss_stored | XSS | Juice Shop | Python | Web Attack - XSS |
| brute_web | BruteForce | Juice Shop | Python | Web Attack - Brute Force |
| brute_ssh | BruteForce | SSH 서버 | Bash | SSH-Patator |
| brute_ftp | BruteForce | FTP 서버 | Bash | FTP-Patator |
| bot_scraper | Bot | Juice Shop | Python | Bot |
| dos_slowloris | DoS | Juice Shop | Python | DoS slowloris |
| portscan | PortScan | Docker net | Bash | PortScan |
| infiltration | Infiltration | Internal | Python | Infiltration |

## Suricata IDS 규칙 (11종)

| SID | 탐지 대상 | 시그니처 핵심 |
|-----|----------|-------------|
| 100001 | SQLi Attempt | SELECT + '/--/# in URI |
| 100002 | SQLi UNION | UNION+SELECT in URI |
| 100003 | XSS Script Tag | `<script` in HTTP |
| 100004 | XSS Event Handler | onload/onerror pattern |
| 100005 | Web Brute Force | POST /rest/user/login x5/30s |
| 100006 | SSH Brute Force | port 2222 SYN x5/60s |
| 100007 | FTP Brute Force | port 21 USER x5/60s |
| 100008 | Bot Scraping | GET /api/Products x30/10s |
| 100009 | DoS Slowloris | SYN x20/10s to HTTP |
| 100010 | Port Scan | SYN x20/5s multi-port |
| 100011 | Infiltration | frontend -> MySQL (3306) |

## 웹 대시보드

3-Panel 레이아웃:

- **Attack Panel** (Red): PoC 12종 목록, [Run] 버튼으로 실행, [View Code]로 소스 확인, 학생이 새 PoC 추가 가능
- **Defense Panel** (Blue): Suricata 규칙 11종, ON/OFF 토글, 학생이 새 규칙 추가 가능
- **Live Monitor**: 실시간 alert 피드 (3초 폴링), 공격 유형별 통계 차트, 컨테이너 상태
- **Terminal**: 공격 실행 결과 stdout/stderr, 드래그로 상하 크기 조절

## 프로젝트 구조

```
teaching/
├── docker-compose.yml              # Docker 8개 서비스 구성
├── install-deps.sh                 # 의존성 자동 설치 (Docker, tcpdump, tshark 등)
├── start.sh / stop.sh              # Linux/Mac 실행/종료 (install-deps.sh 자동 호출)
├── capture-wireshark.sh            # 패킷 캡처 (install-deps.sh 자동 호출)
├── start.bat / stop.bat            # Windows 실행/종료
├── capture-wireshark.bat           # Windows 패킷 캡처 (Wireshark 자동 설치)
├── setup-windows.md                # Windows 설치 가이드
│
├── backend/                        # FastAPI 서버
│   ├── main.py                     # 앱 진입점
│   ├── routers/
│   │   ├── attacks.py              # PoC CRUD + 실행 API
│   │   ├── rules.py                # Suricata 규칙 CRUD + 토글
│   │   └── monitor.py              # Alert 폴링 + 상태
│   ├── pocs/
│   │   ├── preset/                 # 사전 준비 PoC 12종 (.py/.sh + .json)
│   │   └── custom/                 # 학생 추가 PoC
│   └── rules/
│       ├── preset/local.rules      # Suricata 규칙 11종
│       └── custom/                 # 학생 추가 규칙
│
├── dashboard/                      # React 웹 대시보드
│   ├── src/
│   │   ├── App.jsx                 # 3-panel + terminal 레이아웃
│   │   ├── components/
│   │   │   ├── AttackPanel/        # 공격 목록/실행/에디터
│   │   │   ├── DefensePanel/       # 규칙 목록/토글/에디터
│   │   │   ├── MonitorPanel/       # Alert 피드/통계/시스템 상태
│   │   │   └── common/             # Terminal, CodeModal, PacketFilter
│   │   └── hooks/useWebSocket.js   # 실시간 연결
│   ├── nginx.conf                  # API/WS 프록시
│   └── Dockerfile                  # multi-stage build (node + nginx)
│
├── suricata/                       # IDS 설정
│   ├── suricata.yaml               # 경량 설정 (eth0, 메모리 제한)
│   └── rules/local.rules           # 탐지 규칙 11종
│
├── services/                       # 추가 서비스
│   ├── internal-api/               # Flask (Infiltration 대상)
│   └── internal-db/init.sql        # MySQL 초기화 (FLAG 테이블)
│
├── wordlists/common.txt            # Brute Force용 50개 (admin123 포함)
│
├── slides/                         # 강의 슬라이드
│   ├── main.tex / main.pdf         # 내부 공유용 (아키텍처, 설계)
│   └── main1.tex / main1.pdf       # Day 1 학생 강의용
│
└── docs/                           # 설계 문서
    ├── 3day-curriculum.md           # 3일 과정 상세 커리큘럼
    ├── 3day-curriculum-summary.md   # 3일 과정 요약
    ├── workshop-design.md           # 수업 설계
    ├── implementation-design.md     # 구현 설계서
    └── cicids/                      # CIC-IDS 2017 데이터 분석
        ├── scripts/
        └── results/
```

## 요구사항

- Docker + Docker Compose (또는 Docker Desktop)
- 인터넷 불필요 (오프라인 가능, 이미지만 사전 pull)
- 최소 RAM 4GB 권장
- Windows: [setup-windows.md](setup-windows.md) 참고

---

## 3일 과정 커리큘럼

- 대상: 고등학생 | 3일 × 6시간 = 18시간
- 시간: 하루 1시간 30분 (19:00-20:30)
- 스토리: 웹 및 네트워크 취약점을 직접 뚫어보고(Red) → 룰을 만들어 막아보며(Blue) 기존 방어 체계의 한계를 체감한 뒤 → AI 기술로 이를 극복하는 흐름
- 담당: Day 1 이선우 연구원, Day 2 김수하 연구원, Day 3 박민서 연구원

### 기본 설치
- Docker Desktop: OWASP Juice Shop 및 Suricata(Rule-based IDS) 컨테이너 구동용
- Google Colab: Day 3 AI 모델 학습용

### Overview
- OWASP Juice Shop 소개
- 공격/방어/AI-IDS 흐름

---

### Day 1: Red Team — 웹 해킹의 원리 (담당: 이선우 연구원)

방향: Juice Shop 실습 + Gemini 활용 탐지 우회 체험. SQLi와 XSS에 집중.

#### 배경지식

- 인터넷과 웹: 식당 비유 (브라우저=손님, 서버=종업원, DB=주방)
- DevTools 사용법 (F12, Network, Console, Elements)
- URL과 라우팅: #/경로, main.js에 페이지 목록
- 해킹이란: 시스템이 예상하지 못한 입력을 넣는 것
- 웹-서버-DB 구조: 각 지점의 보안 중요성

#### SQL Injection

- SQL 기초: SELECT, FROM, WHERE (엑셀 비유)
- 로그인 시 SQL 쿼리가 만들어지는 과정
- `' OR 1=1--` 한 글자씩 분해 설명
- UNION SELECT: 다른 테이블 데이터 읽기
- 브라우저에서 안 됨 → curl(cmd)에서 됨 → 프론트엔드 검증 ≠ 보안
- 대시보드에서 PoC 실행 + 결과 확인
- SQLi 방어 방법 (Prepared Statement, 입력 검증)

#### XSS (Cross-Site Scripting)

- HTML/JavaScript 기초 (뼈대/두뇌)
- XSS 비유: 학교 게시판에 함정 코드
- DOM/Reflected/Stored 3가지 유형
- Juice Shop DOM XSS 실습 (검색창 iframe)
- 쿠키 탈취 시나리오
- XSS 방어 방법 (이스케이프, CSP, HttpOnly)

#### Gemini로 탐지 우회

- IDS 탐지 규칙 확인 → Gemini에게 우회 요청
- SQLi 우회: `' OR 1=1--` → `' OR '1'='1'`
- XSS 우회: `<script>` → `<img onerror>`, `<svg onload>`
- Gemini가 방어 규칙도 만들어줌 (이벤트 핸들러 감시)
- Bot 우회: Low and Slow (속도 조절)
- 전체 사이클: 공격 → 탐지 → 우회 → 방어 → 재탐지

#### 실습 문제

- 문제 0: Score Board 찾기 (코드 분석)
- 문제 1: Bender 계정으로 SQLi 로그인
- 문제 2: 관리자 페이지 찾기 (Code Analysis)

---

### Day 2: Blue Team — 규칙 기반 방어 (담당: 김수하 연구원)

방향: 공격 스크립트는 자동화를 위함. 여기선 rule 작성에 초점

#### IDS 개념, 규칙 작성

- 배경지식
  - 방화벽 vs IDS vs IPS 차이
  - Suricata 소개, 패킷 캡처 → 규칙 매칭 → 알림 흐름
  - 규칙 문법: content, pcre, http_method, http_uri, threshold, flow
  - SID, msg, classtype 메타데이터
  - Day 1 공격별 네트워크 시그니처 분석
- 실습
  - preset 규칙 11종 구조 파악
  - 규칙 직접 작성 1: SQLi 탐지 (UNION SELECT 패턴)
  - 규칙 직접 작성 2: XSS 탐지 (`<script` 패턴)
  - 공격 재실행 → 탐지 확인
- 리뷰
  - 작성한 규칙 해설
  - content 매칭이 패킷에서 동작하는 방식
  - 오탐 가능성 논의

#### 오후 — 규칙 실전, 대결, 한계

- 배경지식
  - threshold 심화: count/seconds 조합, by_src vs by_dst
  - 규칙 튜닝: 느슨하면 미탐, 빡빡하면 오탐
  - 규칙 기반의 한계: 알려진 패턴만 탐지
  - 우회 기법: 인코딩, 대소문자, 분할
  - 변형 공격 예시: `' OR 1=1--` → `' OR '1'='1`, `<script>` → `<ScRiPt>`
  - 제로데이에 무력
- 실습
  - 규칙 직접 작성 3: Brute Force 탐지 (threshold 설정)
  - 규칙 직접 작성 4: Bot 탐지 (빈도 기반)
  - 변형 공격 대결: 강사가 변형 → 학생 규칙이 잡는지 확인
- 리뷰
  - 규칙 4종 전체 정리
  - 대결 결과 분석
  - 한계 정리: "변형하면 못 잡는다", "제로데이에 무력"
  - CIC-IDS 2017/2018 데이터셋 소개 (Day 3 예고로)
  - "rule-based 대신 feature-based로 판단하면?" → Day 3 예고

---

### Day 3: Blue Team — AI IDS 및 종합 (담당: 박민서 연구원)

방향: Rule-based(SNORT/Suricata)의 한계를 수치적/논리적으로 짚어주고, AI가 특징(Feature) 기반으로 유연하게 탐지하는 과정을 경험 후 발표

#### AI-IDS 도입 및 모델 학습

배경지식 + 실습

- 배경지식
  - 지난 시간 요약 및 Rule-based의 한계 (논문 근거)
  - Rule-based와 ML-based 접근 방식의 차이
  - 침입탐지에 사용되는 주요 AI 모델 소개
  - 학습에 사용되는 데이터 (CIC-IDS 2017/2018)의 문제점 (데이터 불균형) → 모델을 어떻게 잘 평가할 수 있는지도 함께 설명
- 실습
  - 환경 구축: Google Colab
  - 사전 준비된 경량 데이터셋 사용 (CIC-IDS에서 주요 공격+benign 샘플링, 데이터 수 맞춤)
  - 데이터 전처리 방식 간단 설명 (코드는 사전 구현)
  - 학생들이 AI (Gemini, GPT, Claude 등) 활용하여 모델 코드 구현
  - AI 모델 학습: Decision Tree, SVM 등 선택하여 학습
  - 탐지 실행

> **데이터 참고**: CIC-IDS 2017/2018 전체는 Colab 기본 런타임에 무거움. 주요 공격과 benign의 데이터 수를 맞춰 경량 데이터셋을 구성하여 실습. ML 모델(Decision Tree, SVM)이므로 무거운 데이터 불필요.

#### 오후 — 실습 발표 및 한계 종합

- 발표
  - 일부 학생들이 구현한 모델 설명 및 발표
- 배경지식
  - 모델 성능 평가지표
    - 네트워크 보안 데이터의 불균형 특징 설명이 주(主)
    - accuracy만으로 불충분한 이유 (예: 99% benign이면 다 benign으로 찍어도 99%)
    - precision/recall/F1 등 다양한 지표는 서브(sub)로
  - AI 탐지의 유동성 입증 (rule-based가 놓친 공격을 AI가 성공적으로 분류하는지)
  - AI-IDS의 한계 및 과제
  - Deep learning 기반 최신 연구 소개 (Transformer 등, ML 실습과의 차이)
- 마무리
  - 3일간의 수업 종합
  - 질의 응답

---

### 기존 과정 개선 방향 (팀 피드백)

- **Day 1**: 시간 배분이 되면 Heartbleed 공격에 대해 간단히 설명 추가
- **Day 2**: CIC-IDS 2017/2018 데이터셋은 AI 학습용이므로, 이론이 아닌 오후 리뷰에서 Day 3 예고로 설명
- **Day 3**:
  - 데이터셋/전처리는 실습 중 간단 설명, 배경지식에서는 Rule-based vs ML-based 차이에 집중
  - 성능 평가: 데이터 불균형이 주, 평가지표가 서브
  - 실습 시간을 오전/오후 중 한쪽으로 몰아넣기 가능
  - 경량 데이터셋 구성 필요 (CIC-IDS 전체는 Colab에 무거움)
