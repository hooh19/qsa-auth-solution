# 브라우저 인증 후 터미널로 토큰 전달 방식

## 개요

OAuth 2.0 인증에서 브라우저를 통해 인증한 후, 토큰을 터미널 CLI로 전달하는 방법은 여러 가지가 있습니다. 가장 일반적이고 안전한 방법은 **Localhost Redirect 방식**입니다.

## 주요 방식 비교

### 1. Localhost Redirect 방식 (가장 일반적) ⭐

**동작 원리:**
1. CLI가 임시 로컬 서버를 시작 (예: `http://localhost:12345`)
2. 브라우저를 열어 인증 URL로 이동
3. 사용자가 브라우저에서 로그인 및 권한 승인
4. 인증 서버가 `http://localhost:12345/callback?code=AUTH_CODE`로 리다이렉트
5. CLI의 로컬 서버가 리다이렉트를 받아 인증 코드 추출
6. CLI가 인증 코드를 사용하여 토큰 교환
7. 토큰을 로컬 파일에 저장

**장점:**
- ✅ 자동화되어 사용자 편의성 높음
- ✅ 보안성 높음 (토큰이 브라우저를 거치지 않음)
- ✅ 표준 OAuth 2.0 방식
- ✅ 파일 다운로드나 수동 입력 불필요

**단점:**
- ❌ 로컬 포트 충돌 가능성
- ❌ 방화벽이 localhost 연결을 차단할 수 있음

**구현 예시:**

```javascript
// railway-cli/src/auth/oauth-flow.js

const http = require('http');
const { URL } = require('url');
const open = require('open'); // 브라우저 열기

class OAuthFlow {
  constructor() {
    this.port = 12345;
    this.callbackUrl = `http://localhost:${this.port}/callback`;
  }

  async login() {
    return new Promise((resolve, reject) => {
      // 1. 로컬 서버 시작
      const server = http.createServer((req, res) => {
        const url = new URL(req.url, `http://localhost:${this.port}`);
        
        if (url.pathname === '/callback') {
          // 2. 인증 코드 추출
          const code = url.searchParams.get('code');
          const error = url.searchParams.get('error');
          
          if (error) {
            res.writeHead(400, { 'Content-Type': 'text/html' });
            res.end('<h1>인증 실패</h1><p>브라우저를 닫아주세요.</p>');
            server.close();
            reject(new Error(error));
            return;
          }
          
          if (code) {
            res.writeHead(200, { 'Content-Type': 'text/html' });
            res.end('<h1>인증 성공!</h1><p>이 창을 닫아주세요.</p>');
            server.close();
            
            // 3. 인증 코드로 토큰 교환
            this.exchangeToken(code)
              .then(token => {
                this.saveToken(token);
                resolve(token);
              })
              .catch(reject);
          }
        }
      });

      server.listen(this.port, () => {
        console.log('브라우저를 열어 인증을 진행합니다...');
        
        // 4. 브라우저 열기
        const authUrl = `https://railway.app/oauth/authorize?` +
          `client_id=${CLIENT_ID}&` +
          `redirect_uri=${encodeURIComponent(this.callbackUrl)}&` +
          `response_type=code&` +
          `scope=read write`;
        
        open(authUrl);
      });

      // 타임아웃 설정 (5분)
      setTimeout(() => {
        server.close();
        reject(new Error('인증 시간이 초과되었습니다.'));
      }, 300000);
    });
  }

  async exchangeToken(code) {
    // 5. 인증 코드를 액세스 토큰으로 교환
    const response = await fetch('https://railway.app/oauth/token', {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
      },
      body: JSON.stringify({
        grant_type: 'authorization_code',
        code: code,
        redirect_uri: this.callbackUrl,
        client_id: CLIENT_ID,
        client_secret: CLIENT_SECRET,
      }),
    });

    const data = await response.json();
    return data.access_token;
  }

  saveToken(token) {
    // 6. 토큰을 파일에 저장
    const tokenPath = path.join(os.homedir(), '.railway', 'token');
    fs.mkdirSync(path.dirname(tokenPath), { recursive: true });
    fs.writeFileSync(tokenPath, token, 'utf8');
    console.log('인증이 완료되었습니다!');
  }
}
```

### 2. Device Flow 방식

**동작 원리:**
1. CLI가 디바이스 코드 요청
2. 사용자 코드와 확인 URL을 표시
3. 사용자가 브라우저에서 코드 입력 또는 URL 접속
4. CLI가 폴링 방식으로 토큰 확인
5. 인증 완료 시 토큰 수신

**장점:**
- ✅ 로컬 서버 불필요
- ✅ 방화벽 문제 없음
- ✅ 원격 서버에서도 사용 가능

**단점:**
- ❌ 사용자가 수동으로 코드 입력 필요
- ❌ 폴링으로 인한 지연

**구현 예시:**

```javascript
async deviceFlow() {
  // 1. 디바이스 코드 요청
  const response = await fetch('https://railway.app/oauth/device', {
    method: 'POST',
    body: JSON.stringify({
      client_id: CLIENT_ID,
      scope: 'read write',
    }),
  });

  const { device_code, user_code, verification_uri, interval } = await response.json();

  // 2. 사용자에게 코드 표시
  console.log(`\n다음 URL에 접속하여 코드를 입력하세요:`);
  console.log(`${verification_uri}`);
  console.log(`코드: ${user_code}\n`);

  // 3. 브라우저 열기
  open(`${verification_uri}?user_code=${user_code}`);

  // 4. 폴링으로 토큰 확인
  while (true) {
    await new Promise(resolve => setTimeout(resolve, interval * 1000));

    const tokenResponse = await fetch('https://railway.app/oauth/token', {
      method: 'POST',
      body: JSON.stringify({
        grant_type: 'urn:ietf:params:oauth:grant-type:device_code',
        device_code: device_code,
        client_id: CLIENT_ID,
      }),
    });

    const data = await tokenResponse.json();

    if (data.access_token) {
      return data.access_token; // 토큰 획득 성공
    }

    if (data.error === 'authorization_pending') {
      process.stdout.write('.'); // 진행 중 표시
      continue;
    }

    if (data.error === 'expired_token') {
      throw new Error('인증 시간이 만료되었습니다.');
    }
  }
}
```

### 3. 파일 다운로드 방식 (비권장)

**동작 원리:**
1. 브라우저에서 인증 완료
2. 토큰을 JSON 파일로 다운로드
3. 사용자가 특정 폴더에 저장
4. CLI가 해당 폴더에서 파일 읽기

**장점:**
- ✅ 로컬 서버 불필요
- ✅ 구현이 간단

**단점:**
- ❌ 사용자가 수동으로 파일 저장 필요
- ❌ 보안 위험 (파일 노출 가능)
- ❌ 사용자 경험 저하

**구현 예시:**

```javascript
async fileBasedAuth() {
  const tokenFile = path.join(os.homedir(), '.railway', 'auth-token.json');
  
  console.log('브라우저에서 인증을 완료한 후, 토큰 파일을 다운로드하세요.');
  console.log(`다운로드한 파일을 다음 위치에 저장하세요: ${tokenFile}`);
  console.log('\n브라우저를 엽니다...');
  
  open('https://railway.app/oauth/authorize?download_token=true');
  
  // 파일이 생성될 때까지 대기
  let attempts = 0;
  while (attempts < 60) {
    if (fs.existsSync(tokenFile)) {
      const tokenData = JSON.parse(fs.readFileSync(tokenFile, 'utf8'));
      fs.unlinkSync(tokenFile); // 보안을 위해 파일 삭제
      return tokenData.access_token;
    }
    await new Promise(resolve => setTimeout(resolve, 1000));
    attempts++;
  }
  
  throw new Error('토큰 파일을 찾을 수 없습니다.');
}
```

### 4. 클립보드 방식 (비권장)

**동작 원리:**
1. 브라우저에서 인증 완료
2. 토큰을 클립보드에 복사
3. CLI가 클립보드에서 읽기

**장점:**
- ✅ 간단한 구현

**단점:**
- ❌ 플랫폼별 구현 필요
- ❌ 사용자가 수동으로 복사 필요
- ❌ 보안 위험 (클립보드 노출)

## Railway CLI의 실제 구현

Railway CLI는 **Localhost Redirect 방식**을 사용합니다:

```bash
$ railway login
> Open the browser? Yes
```

**실제 동작:**
1. CLI가 `http://localhost:PORT` 서버 시작
2. 브라우저가 `https://railway.app/oauth/authorize?...` 열림
3. 사용자 로그인 및 승인
4. Railway 서버가 `http://localhost:PORT/callback?code=XXX`로 리다이렉트
5. CLI가 코드를 받아 토큰으로 교환
6. `~/.railway/token`에 저장

## QSA 인증 적용 시 토큰 전달 방식

QSA 인증을 적용할 때도 동일한 Localhost Redirect 방식을 사용하되, 추가 단계가 필요합니다:

### 개선된 QSA 인증 플로우

```javascript
class QSAOAuthFlow {
  async login() {
    return new Promise((resolve, reject) => {
      const server = http.createServer(async (req, res) => {
        const url = new URL(req.url, `http://localhost:${this.port}`);
        
        if (url.pathname === '/callback') {
          const code = url.searchParams.get('code');
          
          if (code) {
            res.writeHead(200, { 'Content-Type': 'text/html' });
            res.end('<h1>인증 성공!</h1><p>이 창을 닫아주세요.</p>');
            server.close();
            
            try {
              // 1. 인증 코드로 APIAuthToken 교환
              const tokenResponse = await this.exchangeToken(code);
              const apiAuthToken = tokenResponse.token;
              
              // 2. WSB로 매체 인증용 키 쌍 생성
              const keyPair = await this.wsb.generateKeyPair();
              
              // 3. Public Key를 서버에 등록
              await this.registerPublicKey(apiAuthToken, keyPair.publicKey);
              
              // 4. 토큰과 키 정보 저장
              await this.saveAuthData({
                token: apiAuthToken,
                publicKey: keyPair.publicKey,
                // Private Key는 WSB 내부에 저장
              });
              
              resolve(apiAuthToken);
            } catch (error) {
              reject(error);
            }
          }
        }
      });

      server.listen(this.port, () => {
        const authUrl = `https://railway.app/oauth/authorize?` +
          `client_id=${CLIENT_ID}&` +
          `redirect_uri=${encodeURIComponent(this.callbackUrl)}&` +
          `response_type=code&` +
          `scope=read write&` +
          `qsa_enabled=true`; // QSA 활성화 플래그
        
        open(authUrl);
      });
    });
  }

  async registerPublicKey(token, publicKey) {
    // Public Key를 서버에 등록
    await fetch('https://railway.app/api/auth/register-key', {
      method: 'POST',
      headers: {
        'Authorization': `Bearer ${token}`,
        'Content-Type': 'application/json',
      },
      body: JSON.stringify({
        publicKey: publicKey,
        algorithm: 'PQC', // Post-Quantum Cryptography
      }),
    });
  }
}
```

## 보안 고려사항

### 1. Localhost Redirect 보안

- ✅ 토큰이 브라우저를 거치지 않음
- ✅ HTTPS로 인증 서버와 통신
- ✅ 인증 코드는 단일 사용 (One-time use)
- ✅ 짧은 유효 기간 (보통 10분)

### 2. 토큰 저장 보안

```javascript
// 안전한 토큰 저장
const crypto = require('crypto');

function saveTokenSecurely(token) {
  const tokenPath = path.join(os.homedir(), '.railway', 'token');
  
  // 파일 권한 설정 (소유자만 읽기/쓰기)
  fs.writeFileSync(tokenPath, token, {
    mode: 0o600, // rw-------
    encoding: 'utf8',
  });
  
  // macOS/Linux에서 추가 보안
  if (process.platform !== 'win32') {
    fs.chmodSync(tokenPath, 0o600);
  }
}
```

### 3. QSA 적용 시 추가 보안

- Private Key는 WSB 내부에만 저장 (파일 시스템에 저장하지 않음)
- 키는 암호화되어 메모리에만 존재
- Public Key만 서버에 전송
- 서명 검증으로 재전송 공격 방지

## 결론

**Railway CLI는 Localhost Redirect 방식을 사용합니다:**

1. ✅ **파일 다운로드 방식이 아님** - 자동으로 처리됨
2. ✅ **로컬 서버를 통한 자동 전달** - 사용자 개입 최소화
3. ✅ **보안성 높음** - 토큰이 브라우저를 거치지 않음
4. ✅ **표준 OAuth 2.0 방식** - 안정적이고 검증된 방법

QSA 인증을 적용할 때도 동일한 방식을 사용하되, 추가로 키 쌍 생성 및 Public Key 등록 단계가 필요합니다.

