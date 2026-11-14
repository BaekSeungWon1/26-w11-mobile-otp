# guide.md — 나만의 모바일 OTP (TOTP 2단계 인증) (`26-w11-mobile-otp`)

> **이번 주의 규칙은 딱 하나입니다: "복사할 프롬프트가 없습니다."**  
> 지난주(10주차)에 배운 **Antigravity IDE**와 **Gemini(웹)**를 활용해, 이번 주는 **기획부터 구현·검증까지 여러분이 처음부터 스스로 사고해서** 완성합니다.

---

## 🌟 오늘 우리는 무엇을 만드나요

스마트폰의 Google Authenticator, Microsoft Authenticator처럼 **인터넷 없이도 30초마다 똑같은 6자리 인증 번호를 만들어내는 모바일 OTP(TOTP) 웹앱**을 직접 기획하고 구현합니다.

- **정해진 정답 화면이 없습니다**: 서비스 로그인 + 2단계 인증 시뮬레이터로 만들어도 좋고, 여러 계정의 OTP를 관리하는 모바일 웹 인증기 앱으로 만들어도 좋고, 둘을 한 화면에 합쳐도 좋습니다.
- **기획과 구현을 스스로 설계합니다**: 무료 Gemini 웹에서 먼저 대화하며 기획서를 정리한 뒤 Antigravity IDE로 구현할 수도 있고, Antigravity IDE 안에서 Implementation Plan을 세워가며 직접 개발할 수도 있습니다.
- **사이버보안과의 눈**: `Math.random()` 대신 암호학적 난수(`crypto.getRandomValues`)와 브라우저 내장 Web Crypto API(`crypto.subtle`)를 사용해 **외부 서버 전송 없이(Zero-Transmission)** 내 브라우저 안에서만 안전하게 인증 로직을 구현합니다.

---

## ⏱️ 오늘의 수업 타임라인 (총 130분)

| 시간 | 단계 | 진행 내용 |
|---|---|---|
| **준비 (10분)** | 환경 준비 | 저장소 Fork → 로컬 Clone → Antigravity IDE(또는 VS Code) 열기 → `dev` 브랜치 생성 → `npm install && npm run dev` |
| **1교시 (45분)** | 방향 브리핑 & 자율 기획 | TOTP 3요소·완성 기준 브리핑(15분) → AI 기획 방식 선택 & 앱 컨셉·구조 직접 기획 후 구현 착수(30분) |
| *(휴식 15분)* | — | 스마트폰에 Authenticator 앱(Google / Microsoft / 2FAS 등) 설치 확인 |
| **2교시 (45분)** | 자율 구현 & 실전 검증 | QR 생성·30초 타이머·6자리 OTP 구현 → **내 스마트폰 앱과 번호 실시간 일치 확인** → 디버깅 및 고도화 |
| *(휴식 15분)* | — | — |
| **3교시 (30분)** | 회고·듀얼 배포·제출 | `dev` 병합 → `README.md` 회고 작성 → Netlify & GitHub Pages 배포 → `v1.0` 태그 & PR 제출 |

---

## 1단계 — 환경 준비 & `dev` 브랜치 시작 (10분)

1. [`26-w11-mobile-otp`](https://github.com/kopo-kangseo-cybersec/26-w11-mobile-otp) 저장소를 본인 GitHub 계정으로 **Fork**하고 내 컴퓨터로 Clone합니다:
   ```bash
   git clone https://github.com/<내-GitHub-아이디>/26-w11-mobile-otp.git
   cd 26-w11-mobile-otp
   ```
2. **Antigravity IDE**에서 `26-w11-mobile-otp` 폴더를 엽니다. *(Codespaces/VS Code 환경이 편하다면 VS Code로 진행해도 무방합니다.)*
3. 하단 터미널에서 `dev` 브랜치를 생성하고 원격에 푸시한 뒤, 작업용 기능 브랜치(예: `otp`)를 만듭니다:
   ```bash
   git checkout -b dev
   git push -u origin dev
   git checkout -b otp
   ```
4. 패키지를 설치하고 기본 Vite 화면이 뜨는지 확인합니다:
   ```bash
   npm install
   npm run dev
   ```
   > 💡 `package.json`에는 `react`, `lucide-react`, `qrcode.react`, `gh-pages`가 이미 포함되어 있습니다. 별도 설치 오류 없이 바로 설계에 집중할 수 있습니다.

---

## 2단계 — 핵심 원리와 최소 완성 기준 이해 (1교시)

### 🔑 TOTP (RFC 6238)가 인터넷 없이 작동하는 3요소
```
[1. 비밀 키 (Base32 Secret)]  +  [2. 현재 시각 (UnixTime / 30초)]
                  │
                  ▼   HMAC-SHA1 연산 (Web Crypto API)
          [3. 6자리 OTP 번호]
```
서버와 내 스마트폰이 **같은 비밀 키**와 **같은 현재 시각**을 가지고 있다면, 네트워크 통신 없이도 매 30초마다 정확히 같은 6자리 숫자를 계산해 냅니다.

### 🔄 내 웹앱과 스마트폰 Authenticator 앱의 역할 흐름
```
[내 웹앱 (서비스/인증기 역할)]                    [스마트폰 Authenticator 앱]
1. 비밀 키 생성 & QR 코드 화면 표시  ──(QR 스캔)──> 2. 앱에 계정 등록 완료
4. 사용자가 입력한 6자리 번호 검증    <──(번호 확인)── 3. 30초마다 6자리 번호 표시
   (화면 한쪽에 예상 OTP도 함께 표시해 일치 여부 확인!)
```

### ✅ 반드시 만족해야 할 4가지 완성 기준
1. **안전한 비밀 키 생성**: 버튼 클릭으로 새로운 Base32 비밀 키를 발급할 수 있다.
2. **QR 코드 연동**: `otpauth://totp/<발급자>:<계정>?secret=<비밀키>&issuer=<발급자>` 형식의 QR 코드를 표시해 **스마트폰 Authenticator 앱으로 스캔 등록**할 수 있다.
3. **실시간 동기화**: 30초 카운트다운과 함께 6자리 코드가 실시간 갱신되며, **스마트폰 앱에 뜬 6자리 숫자와 일치**한다.
4. **코드 검증**: 사용자가 6자리 숫자를 입력했을 때 일치/불일치(인증 성공·실패) 결과를 판정한다.

### 🛡️ 사이버보안과 필수 체크포인트 2가지
1. **`Math.random()` 절대 금지**: 일반 난수는 예측 가능하므로 보안 키 생성에 사용할 수 없습니다. 반드시 `window.crypto.getRandomValues()`를 사용하세요.
2. **Zero-Transmission (무전송 원칙)**: 비밀 키 생성과 HMAC-SHA1 연산은 외부 API 서버가 아닌 브라우저 내장 `crypto.subtle`로 처리합니다. 개발자 도구(`F12`) → **Network** 탭을 열어 외부 서버로 키가 새어나가지 않는지 직접 검증해 보세요.

---

## 3단계 — 나만의 AI 기획·구현 방식 선택 & 제작 (1~2교시)

### 🧭 선택 가능한 AI 협업 방식
- **방식 A. Gemini(웹 무료)에서 먼저 기획 → Antigravity IDE에서 구현** *(할당량 절약 추천!)*
  - 먼저 웹 Gemini와 대화하며 *"내가 만들고 싶은 모바일 OTP 앱의 화면 구성, 사용자 흐름, 기능 명세(PRD)"*를 충분히 정리합니다.
  - 기획이 확정되면 그 명세서를 **Antigravity IDE**로 가져와 코드로 구현하고 다듬습니다. (무료 계정 사용량 부담을 크게 줄일 수 있습니다!)
- **방식 B. Antigravity IDE에서 직접 대화형 기획(Implementation Plan) → 단계별 구현**
  - 지난주(10주차)에 배운 방식대로 Antigravity IDE Chat에 내 아이디어를 설명하고 먼저 **Implementation Plan**을 받아 검토·수정한 뒤, 한 단계씩 직접 확인하며 구현합니다.
- **방식 C. VS Code + GitHub Copilot 병행 (보조 트랙)**
  - Google AI 무료 사용량 제한에 걸리거나 VS Code가 더 편한 학생은 5~9주차에 배운 GitHub Copilot(Plan/Chat)을 활용해 이어서 구현해도 됩니다.

### 💡 스스로에게 던져볼 기획 질문 5가지 (힌트)
> AI에게 무작정 *"OTP 앱 만들어줘"*라고 하기 전에, 아래 질문을 먼저 정리해서 전달하면 훨씬 완성도 높은 결과가 나옵니다.

1. **앱의 정체성**: 내 앱은 *"구글 OTP처럼 코드를 보관하는 인증기 앱"*인가요, *"OTP를 등록·인증하는 로그인 시뮬레이터"*인가요, 아니면 *"둘을 양쪽 화면에서 비교해 주는 체험 스튜디오"*인가요?
2. **화면과 상태 흐름**: 사용자가 처음 들어왔을 때 가장 먼저 무엇을 누르고, 그다음 어떤 화면 변화가 일어나야 자연스러울까요?
3. **표준 규격(RFC 6238) 준수**: 브라우저는 `Base32` 디코딩과 `64-bit Big-Endian` 카운터 버퍼, `HMAC-SHA1` 동적 절단(Dynamic Truncation)을 정확히 계산해야 스마트폰 앱과 번호가 일치합니다. 이를 어떻게 별도 모듈(예: `src/utils/totp.js`)로 분리하고 검증할까요?
4. **스마트폰 실전 테스트**: 내 폰의 Google/Microsoft Authenticator로 QR을 스캔했는데 **6자리 숫자가 안 맞는다면**, 어떤 원인(Base32 패딩, PC 시계 오차, 빅엔디안 비트 연산 오류 등)을 의심하고 AI에게 점검을 요청할까요?
5. **보안과 UX 확장(선택)**: 기본 기능이 완성되었다면 어떤 기능을 더 붙일까요?
   - 예: 다중 계정(`GitHub`, `Naver`, `AWS` 등) 등록·삭제 / 1회용 백업 복구 코드 발급 / 시간 오차 ±1 스텝(앞뒤 30초) 허용 토글 / `localStorage` 저장 시 주의점 표시 등

> 💾 **중간 커밋 습관**: 기능이 하나 동작할 때마다 프로젝트에 탑재된 `GEMINI.md` 규칙에 맞춰 커밋을 요청하거나 직접 커밋해 세이브포인트를 남기세요!
> - Antigravity IDE Chat: `"지금까지 작업한 내용 프로젝트 커밋 규칙에 맞춰서 커밋해줘"`
> - 터미널 직접 입력: `git add . && git commit -m "feat: TOTP 비밀 키 생성 및 QR 연동 구현"`

---

## 4단계 — `dev` 병합, 회고 작성 및 듀얼 배포 (3교시)

### 1. 통합 브랜치(`dev`)로 병합 및 빌드 확인
```bash
git checkout dev
git merge otp
npm run build
npm run lint
```

### 2. `README.md` 회고 작성 및 커밋
`dev` 브랜치의 `README.md`를 열고 아래 내용을 작성합니다:
- 내가 기획한 앱의 컨셉과 화면 구성
- 선택한 AI 기획·구현 방식(Gemini 웹 기획 + Antigravity IDE 구현 등)과 시행착오(특히 스마트폰 앱 연동 디버깅 경험)
- `crypto.getRandomValues` / `crypto.subtle` / Network 탭 무전송 보안 점검 결과

### 3. Netlify & GitHub Pages 듀얼 배포
1. **Netlify 배포**: [app.netlify.com](https://app.netlify.com) → Import project → 본인의 `26-w11-mobile-otp` 선택 → Branch: `dev`, Build: `npm run build`, Publish: `dist` → 배포 후 `https://<사이트이름>.netlify.app` 복사
2. **GitHub Pages 배포**:
   ```bash
   npm run build
   npx gh-pages -d dist
   ```
   - GitHub 저장소 **Settings → Pages**에서 `gh-pages` 브랜치가 연결되었는지 확인하고 `https://<내아이디>.github.io/26-w11-mobile-otp/` 주소를 확인합니다.
   - ⚠️ **주의**: 배포 URL에 `.git` 주소나 `http://127.0.0.1:5173` 로컬 주소를 적지 마세요!
3. `README.md`에 두 배포 URL을 채워 넣고 `dev` 브랜치에 푸시합니다:
   ```bash
   git add README.md
   git commit -m "docs: README 회고 및 배포 링크 작성"
   git push origin dev
   ```

---

## 5단계 — 마일스톤 태그 & PR 제출

1. `v1.0` 태그 생성 및 푸시:
   ```bash
   git tag v1.0
   git push origin v1.0
   ```
2. GitHub에서 **Pull requests → New pull request** 클릭:
   - **base**: `kopo-kangseo-cybersec/26-w11-mobile-otp` (`main`)
   - **compare**: `<내아이디>/26-w11-mobile-otp` (`dev`)
3. PR 템플릿의 이름, 배포 URL 2개, 태그(`v1.0`), 체크리스트, 한 줄 소감을 작성해 제출합니다.
4. 1~2분 뒤 AI 자동 채점 리포트와 `'통과'` 라벨을 확인합니다!
