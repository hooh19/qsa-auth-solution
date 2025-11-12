# Railway CLI 인증 방식 분석 및 QSA 적용 방안

## 1. Railway CLI 현재 인증 방식 분석

### 1.1 인증 프로세스

Railway CLI는 **OAuth 2.0 Device Flow** 또는 **Authorization Code Flow with PKCE** 방식을 사용합니다:

```
1. railway login 실행
   ↓
2. 브라우저 자동 열림 (OAuth 인증 페이지)
   ↓
3. 사용자가 Railway 웹사이트에서 로그인
   ↓
4. 권한 승인 (Authorize)
   ↓
5. 인증 코드 또는 토큰이 CLI로 전달
   ↓
6. CLI가 액세스 토큰을 받아 로컬에 저장 (~/.railway/token)
   ↓
7. 이후 모든 CLI 명령어는 이 토큰을 사용하여 Railway API 호출
```

### 1.2 현재 방식의 특징

**장점:**
- ✅ 사용자 친화적 (브라우저 기반 로그인)
- ✅ 표준 OAuth 2.0 프로토콜 사용
- ✅ 토큰 기반 인증 (Stateless)
- ✅ 세션 관리 불필요

**단점:**
- ❌ 브라우저 의존성 (브라우저가 없는 환경에서 제한적)
- ❌ 토큰 탈취 시 재사용 가능 (재전송 공격 취약)
- ❌ 전통적인 암호화 방식 (양자 컴퓨터 공격에 취약)
- ❌ 토큰만으로 인증 (추가 서명 검증 없음)

### 1.3 저장되는 정보

```bash
~/.railway/
├── token          # 액세스 토큰 (JWT 또는 Bearer Token)
├── config.json    # 프로젝트 연결 정보
└── ...
```

## 2. QSA 인증 방식과의 비교

### 2.1 QSA 인증의 핵심 특징

| 항목 | Railway (현재) | QSA |
|------|---------------|-----|
| **인증 방식** | OAuth 2.0 (단일 토큰) | JWT + 서명 (이중 인증) |
| **토큰 검증** | 서버 측 세션/토큰 검증 | JWT 검증 + 서명 검증 |
| **암호화** | RSA/ECDSA (전통적) | PQC (Post-Quantum Cryptography) |
| **재전송 공격 방지** | 없음 | 서명 검증으로 방지 |
| **키 관리** | 서버 측 관리 | 클라이언트 측 키 쌍 (WSB) |

### 2.2 QSA의 장점

1. **이중 인증 메커니즘**
   - JWT 토큰 검증: 사용자 인증 확인
   - 서명 검증: 요청의 진위성 확인

2. **양자 안전 암호화**
   - PQC 알고리즘 사용
   - 양자 컴퓨터 공격에도 안전

3. **재전송 공격 방지**
   - API 원문 서명 검증
   - 토큰 탈취 후에도 재사용 불가

4. **클라이언트 측 키 관리**
   - WSB를 통한 안전한 키 생성/저장
   - Private Key는 클라이언트에만 존재

## 3. QSA 인증을 Railway CLI에 적용하는 방안

### 3.1 적용 시나리오

#### 시나리오 A: 하이브리드 방식 (권장)

```
[초기 로그인]
1. railway login 실행
   ↓
2. 브라우저에서 Railway 웹사이트 로그인 (기존 방식)
   ↓
3. Railway 서버가 APIAuthToken 발급 (JWT 기반)
   ↓
4. CLI가 APIAuthToken + 매체 인증용 Public Key 받음
   ↓
5. WSB를 통해 매체 인증용 키 쌍 생성
   ↓
6. APIAuthToken과 키 쌍을 로컬에 안전하게 저장
   ↓

[CLI 명령어 실행 시]
1. railway logs 실행
   ↓
2. WSB로 API 요청 원문에 서명 생성
   ↓
3. Railway API 호출:
   - Header: Authorization: Bearer <APIAuthToken>
   - Header: X-Signature: <매체 인증용 서명값>
   ↓
4. Railway 서버 (APIAuth Agent lib):
   - APIAuthToken 검증 (JWT)
   - 서명값 검증 (토큰 내 Public Key 사용)
   ↓
5. 검증 성공 시 API 응답 반환
```

#### 시나리오 B: 완전 QSA 방식

```
[초기 로그인]
1. railway login
   ↓
2. 브라우저에서 Railway 웹사이트 로그인
   ↓
3. Railway 서버가 매체 인증용 Public Key 요청
   ↓
4. CLI가 WSB로 키 쌍 생성 후 Public Key 전송
   ↓
5. Railway 서버가 APIAuthToken 발급 (Public Key 포함)
   ↓
6. CLI가 APIAuthToken 저장
   ↓

[CLI 명령어 실행 시]
- 동일한 프로세스 (시나리오 A와 동일)
```

### 3.2 구현 구조

#### 3.2.1 CLI 측 구현

```javascript
// railway-cli-qsa/src/auth/qsa-auth.js

class QSAAuth {
  constructor() {
    this.wsb = new WebSecurityBox();
    this.tokenStorage = new TokenStorage();
  }

  async login() {
    // 1. 브라우저에서 Railway 로그인 (기존 방식)
    const authCode = await this.browserLogin();
    
    // 2. APIAuthToken 발급 요청
    const tokenResponse = await this.requestToken(authCode);
    
    // 3. 매체 인증용 키 쌍 생성
    const keyPair = await this.wsb.generateKeyPair();
    
    // 4. Public Key를 서버에 등록
    await this.registerPublicKey(tokenResponse.token, keyPair.publicKey);
    
    // 5. 토큰과 키 쌍 저장
    await this.tokenStorage.save({
      token: tokenResponse.token,
      publicKey: keyPair.publicKey,
      // Private Key는 WSB 내부에만 저장
    });
  }

  async signRequest(method, url, body) {
    // API 요청 원문 생성
    const requestData = {
      method,
      url,
      body: JSON.stringify(body),
      timestamp: Date.now(),
    };
    
    // WSB로 서명 생성
    const signature = await this.wsb.sign(
      JSON.stringify(requestData),
      this.tokenStorage.getPrivateKey()
    );
    
    return signature;
  }

  async makeRequest(method, url, body) {
    const token = this.tokenStorage.getToken();
    const signature = await this.signRequest(method, url, body);
    
    return fetch(url, {
      method,
      headers: {
        'Authorization': `Bearer ${token}`,
        'X-Signature': signature,
        'X-Timestamp': Date.now().toString(),
        'Content-Type': 'application/json',
      },
      body: JSON.stringify(body),
    });
  }
}
```

#### 3.2.2 서버 측 구현 (Railway)

```javascript
// Railway API Gateway에 APIAuth Agent lib 통합

const APIAuthAgent = require('@qsa/api-auth-agent');

async function authenticateRequest(req, res, next) {
  const token = req.headers.authorization?.replace('Bearer ', '');
  const signature = req.headers['x-signature'];
  const timestamp = req.headers['x-timestamp'];
  
  if (!token || !signature) {
    return res.status(401).json({ error: 'Missing authentication' });
  }
  
  // 1. APIAuthToken 검증
  const tokenPayload = await APIAuthAgent.verifyToken(token);
  if (!tokenPayload) {
    return res.status(401).json({ error: 'Invalid token' });
  }
  
  // 2. 매체 인증용 서명 검증
  const requestData = {
    method: req.method,
    url: req.url,
    body: JSON.stringify(req.body),
    timestamp: parseInt(timestamp),
  };
  
  const isValid = await APIAuthAgent.verifySignature(
    JSON.stringify(requestData),
    signature,
    tokenPayload.mediaPublicKey // 토큰 내에 포함된 Public Key
  );
  
  if (!isValid) {
    return res.status(401).json({ error: 'Invalid signature' });
  }
  
  // 3. 타임스탬프 검증 (재전송 공격 방지)
  const timeDiff = Date.now() - parseInt(timestamp);
  if (Math.abs(timeDiff) > 300000) { // 5분
    return res.status(401).json({ error: 'Request expired' });
  }
  
  req.user = tokenPayload.user;
  next();
}
```

### 3.3 파일 구조

```
railway-cli-qsa/
├── src/
│   ├── auth/
│   │   ├── qsa-auth.js      # QSA 인증 로직
│   │   ├── wsb.js           # Web Security Box 래퍼
│   │   └── token-storage.js  # 토큰 저장소
│   ├── commands/
│   │   ├── login.js         # railway login 명령어
│   │   ├── logs.js          # railway logs 명령어
│   │   └── ...
│   └── ...
└── package.json
```

### 3.4 저장 구조 변경

```bash
~/.railway/
├── token              # APIAuthToken (JWT)
├── config.json        # 프로젝트 연결 정보
└── .wsb/              # WSB 키 저장소 (암호화됨)
    └── keys.json      # Public Key만 저장 (Private Key는 WSB 내부)
```

## 4. 적용 시 기대 효과

### 4.1 보안 강화

1. **재전송 공격 방지**
   - 토큰만 탈취해도 서명 없이는 사용 불가
   - 타임스탬프 검증으로 오래된 요청 차단

2. **양자 안전 암호화**
   - PQC 알고리즘 사용
   - 미래 양자 컴퓨터 공격에도 안전

3. **이중 인증**
   - 토큰 검증 + 서명 검증
   - 다층 보안 체계

### 4.2 사용자 경험

1. **기존 방식과 유사**
   - `railway login` 명령어는 동일
   - 브라우저 로그인 방식 유지
   - 사용자는 추가 작업 불필요

2. **투명한 보안**
   - 백그라운드에서 자동으로 서명 생성
   - 사용자는 기존처럼 사용

### 4.3 운영 편의성

1. **Stateless 아키텍처 유지**
   - 서버 측 세션 관리 불필요
   - 수평 확장 용이

2. **모니터링 강화**
   - 서명 검증 실패 로그로 공격 탐지
   - 이상 패턴 분석 가능

## 5. 마이그레이션 전략

### 5.1 단계별 적용

**Phase 1: 하이브리드 모드 (3개월)**
- 기존 OAuth 방식과 QSA 방식 병행
- `railway login --qsa` 옵션으로 선택 가능
- 점진적 사용자 전환

**Phase 2: 기본 QSA 모드 (6개월)**
- QSA를 기본 인증 방식으로 설정
- 기존 방식은 `--legacy` 옵션으로 사용 가능
- 대부분의 사용자가 QSA로 전환

**Phase 3: 완전 전환 (12개월)**
- 기존 OAuth 방식 제거
- 모든 사용자가 QSA 사용
- 보안 강화 완료

### 5.2 호환성 유지

- 기존 토큰은 유효 기간 동안 계속 사용 가능
- 점진적 토큰 갱신 시 QSA 방식으로 전환
- 사용자에게 강제 전환 없음

## 6. 결론 및 권장사항

### 6.1 적용 권장

**QSA 인증을 Railway CLI에 적용하는 것을 강력히 권장합니다:**

1. **보안 강화**: 재전송 공격 방지 및 양자 안전 암호화
2. **사용자 경험**: 기존 방식과 유사하여 학습 곡선 최소화
3. **미래 대비**: 양자 컴퓨팅 시대에 대비한 선제적 보안
4. **운영 효율**: Stateless 아키텍처 유지로 확장성 확보

### 6.2 구현 우선순위

1. **높음**: CLI 측 WSB 통합 및 서명 생성
2. **높음**: 서버 측 APIAuth Agent lib 통합
3. **중간**: 하이브리드 모드 지원
4. **낮음**: 모니터링 및 로깅 강화

### 6.3 추가 고려사항

- **WSB 구현**: 브라우저 환경이 아닌 Node.js 환경에서의 WSB 구현 필요
- **키 복구**: Private Key 손실 시 복구 메커니즘 필요
- **다중 디바이스**: 여러 터미널/디바이스에서 동일 계정 사용 시 키 동기화
- **성능**: 서명 생성/검증 오버헤드 최소화

---

**작성일**: 2024년
**버전**: 1.0

