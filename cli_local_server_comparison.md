# CLI 로컬 서버 선택 가이드

## 개요

OAuth 인증 플로우를 위한 CLI의 임시 로컬 서버 선택 시 고려사항과 추천 사항입니다.

## 요구사항 분석

### CLI 서버의 특성

1. **임시 서버**: 인증 완료 후 즉시 종료
2. **단순한 기능**: 리다이렉트 받기만 하면 됨
3. **빠른 시작/종료**: 사용자 대기 시간 최소화
4. **의존성 최소화**: CLI 설치 크기 최소화
5. **크로스 플랫폼**: Windows, macOS, Linux 모두 지원

### 필요한 기능

- ✅ HTTP GET 요청 처리
- ✅ URL 파라미터 파싱
- ✅ 간단한 HTML 응답
- ✅ 포트 충돌 처리
- ✅ 타임아웃 처리

## 옵션 비교

### 1. Node.js 내장 `http` 모듈 ⭐⭐⭐⭐⭐ (최고 추천)

**장점:**
- ✅ **의존성 없음** - Node.js에 내장
- ✅ **가벼움** - 추가 패키지 불필요
- ✅ **빠른 시작** - 오버헤드 최소
- ✅ **충분한 기능** - OAuth 콜백 처리에 충분

**단점:**
- ❌ URL 파싱을 수동으로 해야 함 (하지만 간단함)
- ❌ 라우팅이 수동

**구현 예시:**

```javascript
const http = require('http');
const { URL } = require('url');

class LocalServer {
  constructor(port = 0) {
    this.port = port;
    this.server = null;
  }

  start(callback) {
    return new Promise((resolve, reject) => {
      this.server = http.createServer((req, res) => {
        const url = new URL(req.url, `http://localhost:${this.port}`);
        
        if (url.pathname === '/callback') {
          const code = url.searchParams.get('code');
          const error = url.searchParams.get('error');
          
          if (error) {
            res.writeHead(400, { 'Content-Type': 'text/html; charset=utf-8' });
            res.end(`
              <html>
                <head><title>인증 실패</title></head>
                <body>
                  <h1>인증에 실패했습니다</h1>
                  <p>에러: ${error}</p>
                  <p>이 창을 닫아주세요.</p>
                </body>
              </html>
            `);
            this.server.close();
            reject(new Error(error));
            return;
          }
          
          if (code) {
            res.writeHead(200, { 'Content-Type': 'text/html; charset=utf-8' });
            res.end(`
              <html>
                <head>
                  <title>인증 성공</title>
                  <style>
                    body { font-family: Arial, sans-serif; text-align: center; padding: 50px; }
                    h1 { color: #4CAF50; }
                  </style>
                </head>
                <body>
                  <h1>✓ 인증 성공!</h1>
                  <p>이 창을 닫아주세요.</p>
                  <script>setTimeout(() => window.close(), 2000);</script>
                </body>
              </html>
            `);
            this.server.close();
            callback(code);
            resolve(code);
          } else {
            res.writeHead(400, { 'Content-Type': 'text/html' });
            res.end('<h1>인증 코드를 받을 수 없습니다.</h1>');
          }
        } else {
          res.writeHead(404, { 'Content-Type': 'text/plain' });
          res.end('Not Found');
        }
      });

      this.server.on('error', (err) => {
        if (err.code === 'EADDRINUSE') {
          // 포트가 사용 중이면 다른 포트 시도
          this.port++;
          this.start(callback).then(resolve).catch(reject);
        } else {
          reject(err);
        }
      });

      this.server.listen(this.port, '127.0.0.1', () => {
        const actualPort = this.server.address().port;
        this.port = actualPort;
        resolve(actualPort);
      });
    });
  }

  stop() {
    if (this.server) {
      this.server.close();
      this.server = null;
    }
  }

  getCallbackUrl() {
    return `http://localhost:${this.port}/callback`;
  }
}

// 사용 예시
async function login() {
  const server = new LocalServer(0); // 0 = 사용 가능한 포트 자동 선택
  
  try {
    const port = await server.start((code) => {
      console.log('인증 코드를 받았습니다:', code);
      // 토큰 교환 로직
    });
    
    console.log(`로컬 서버가 시작되었습니다: http://localhost:${port}`);
    console.log('브라우저를 엽니다...');
    
    // 브라우저 열기
    const authUrl = `https://railway.app/oauth/authorize?` +
      `client_id=${CLIENT_ID}&` +
      `redirect_uri=${encodeURIComponent(server.getCallbackUrl())}&` +
      `response_type=code`;
    
    await open(authUrl);
    
    // 타임아웃 설정 (5분)
    setTimeout(() => {
      server.stop();
      throw new Error('인증 시간이 초과되었습니다.');
    }, 300000);
    
  } catch (error) {
    server.stop();
    throw error;
  }
}
```

**추천도: ⭐⭐⭐⭐⭐ (5/5)**

---

### 2. `http-server` (경량 패키지) ⭐⭐⭐

**장점:**
- ✅ 간단한 API
- ✅ 포트 자동 선택
- ✅ 정적 파일 서빙 가능

**단점:**
- ❌ 추가 의존성 필요
- ❌ OAuth 콜백 처리에 과함
- ❌ CLI 도구에 불필요한 기능

**설치:**
```bash
npm install http-server
```

**구현 예시:**
```javascript
const httpServer = require('http-server');

// 너무 단순해서 OAuth 콜백 처리에 부적합
```

**추천도: ⭐⭐⭐ (3/5) - CLI에는 과함**

---

### 3. Express (경량 버전) ⭐⭐

**장점:**
- ✅ 풍부한 기능
- ✅ 미들웨어 지원
- ✅ 널리 사용됨

**단점:**
- ❌ **의존성이 큼** (CLI에 부적합)
- ❌ 시작 시간 느림
- ❌ OAuth 콜백에는 과함

**설치:**
```bash
npm install express
```

**구현 예시:**
```javascript
const express = require('express');

const app = express();
app.get('/callback', (req, res) => {
  const code = req.query.code;
  res.send('<h1>인증 성공!</h1>');
  // ...
});

app.listen(0, () => {
  // ...
});
```

**추천도: ⭐⭐ (2/5) - CLI에는 과함**

---

### 4. Fastify ⭐⭐

**장점:**
- ✅ 빠른 성능
- ✅ Express보다 가벼움

**단점:**
- ❌ 여전히 의존성 필요
- ❌ OAuth 콜백에는 과함

**추천도: ⭐⭐ (2/5) - CLI에는 과함**

---

### 5. `local-web-server` ⭐

**장점:**
- ✅ 간단한 설정

**단점:**
- ❌ 추가 의존성
- ❌ CLI에 부적합

**추천도: ⭐ (1/5)**

---

## 최종 추천: Node.js 내장 `http` 모듈

### 이유

1. **의존성 없음**
   - Node.js에 내장되어 있어 추가 설치 불필요
   - CLI 설치 크기 최소화
   - 의존성 관리 문제 없음

2. **충분한 기능**
   - OAuth 콜백 처리에 필요한 모든 기능 제공
   - URL 파싱은 `url` 모듈로 간단히 처리

3. **빠른 시작**
   - 오버헤드 최소
   - 사용자 대기 시간 최소화

4. **크로스 플랫폼**
   - 모든 플랫폼에서 동일하게 동작

### 개선된 구현 (완전한 예시)

```javascript
// railway-cli/src/auth/local-server.js

const http = require('http');
const { URL } = require('url');
const { promisify } = require('util');

class OAuthLocalServer {
  constructor(options = {}) {
    this.port = options.port || 0; // 0 = 자동 포트 선택
    this.host = options.host || '127.0.0.1';
    this.timeout = options.timeout || 300000; // 5분
    this.server = null;
    this.callbackPromise = null;
  }

  /**
   * 서버 시작 및 인증 코드 대기
   * @returns {Promise<string>} 인증 코드
   */
  async start() {
    return new Promise((resolve, reject) => {
      // 포트 충돌 시 재시도
      const tryStart = (port) => {
        this.server = http.createServer((req, res) => {
          this.handleRequest(req, res, resolve, reject);
        });

        this.server.on('error', (err) => {
          if (err.code === 'EADDRINUSE') {
            // 다른 포트 시도
            tryStart(port + 1);
          } else {
            reject(err);
          }
        });

        this.server.listen(port, this.host, () => {
          const actualPort = this.server.address().port;
          this.port = actualPort;
          console.log(`로컬 서버 시작: http://${this.host}:${actualPort}`);
        });
      };

      tryStart(this.port);

      // 타임아웃 설정
      setTimeout(() => {
        this.stop();
        reject(new Error('인증 시간이 초과되었습니다. 다시 시도해주세요.'));
      }, this.timeout);
    });
  }

  /**
   * HTTP 요청 처리
   */
  handleRequest(req, res, resolve, reject) {
    const url = new URL(req.url, `http://${this.host}:${this.port}`);
    
    // CORS 헤더 (필요한 경우)
    const headers = {
      'Content-Type': 'text/html; charset=utf-8',
      'Access-Control-Allow-Origin': '*',
    };

    if (url.pathname === '/callback') {
      const code = url.searchParams.get('code');
      const error = url.searchParams.get('error');
      const errorDescription = url.searchParams.get('error_description');

      if (error) {
        res.writeHead(400, headers);
        res.end(this.getErrorPage(error, errorDescription));
        this.stop();
        reject(new Error(errorDescription || error));
        return;
      }

      if (code) {
        res.writeHead(200, headers);
        res.end(this.getSuccessPage());
        this.stop();
        resolve(code);
      } else {
        res.writeHead(400, headers);
        res.end(this.getErrorPage('missing_code', '인증 코드를 받을 수 없습니다.'));
        this.stop();
        reject(new Error('인증 코드를 받을 수 없습니다.'));
      }
    } else if (url.pathname === '/health') {
      // 헬스 체크 엔드포인트
      res.writeHead(200, { 'Content-Type': 'application/json' });
      res.end(JSON.stringify({ status: 'ok', port: this.port }));
    } else {
      res.writeHead(404, headers);
      res.end(this.getNotFoundPage());
    }
  }

  /**
   * 성공 페이지 HTML
   */
  getSuccessPage() {
    return `
<!DOCTYPE html>
<html lang="ko">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>인증 성공</title>
  <style>
    * { margin: 0; padding: 0; box-sizing: border-box; }
    body {
      font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', sans-serif;
      display: flex;
      justify-content: center;
      align-items: center;
      min-height: 100vh;
      background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
      color: white;
    }
    .container {
      text-align: center;
      padding: 2rem;
      background: rgba(255, 255, 255, 0.1);
      border-radius: 16px;
      backdrop-filter: blur(10px);
    }
    h1 { font-size: 2rem; margin-bottom: 1rem; }
    p { font-size: 1.1rem; opacity: 0.9; }
    .checkmark {
      width: 64px;
      height: 64px;
      border-radius: 50%;
      background: #4CAF50;
      display: inline-flex;
      align-items: center;
      justify-content: center;
      margin-bottom: 1rem;
      font-size: 2rem;
    }
  </style>
</head>
<body>
  <div class="container">
    <div class="checkmark">✓</div>
    <h1>인증 성공!</h1>
    <p>이 창을 닫아주세요.</p>
  </div>
  <script>
    // 2초 후 자동으로 창 닫기 (선택사항)
    setTimeout(() => {
      window.close();
    }, 2000);
  </script>
</body>
</html>
    `;
  }

  /**
   * 에러 페이지 HTML
   */
  getErrorPage(error, description) {
    return `
<!DOCTYPE html>
<html lang="ko">
<head>
  <meta charset="UTF-8">
  <title>인증 실패</title>
  <style>
    body {
      font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', sans-serif;
      display: flex;
      justify-content: center;
      align-items: center;
      min-height: 100vh;
      background: #f5f5f5;
    }
    .container {
      text-align: center;
      padding: 2rem;
      background: white;
      border-radius: 8px;
      box-shadow: 0 2px 8px rgba(0,0,0,0.1);
    }
    h1 { color: #f44336; margin-bottom: 1rem; }
    p { color: #666; }
  </style>
</head>
<body>
  <div class="container">
    <h1>✗ 인증 실패</h1>
    <p>${description || error}</p>
    <p style="margin-top: 1rem; font-size: 0.9rem; color: #999;">이 창을 닫아주세요.</p>
  </div>
</body>
</html>
    `;
  }

  /**
   * 404 페이지 HTML
   */
  getNotFoundPage() {
    return `
<!DOCTYPE html>
<html lang="ko">
<head>
  <meta charset="UTF-8">
  <title>404 Not Found</title>
</head>
<body>
  <h1>404 Not Found</h1>
  <p>요청한 페이지를 찾을 수 없습니다.</p>
</body>
</html>
    `;
  }

  /**
   * 서버 중지
   */
  stop() {
    if (this.server) {
      this.server.close();
      this.server = null;
    }
  }

  /**
   * 콜백 URL 반환
   */
  getCallbackUrl() {
    return `http://${this.host}:${this.port}/callback`;
  }
}

module.exports = OAuthLocalServer;
```

### 사용 예시

```javascript
// railway-cli/src/commands/login.js

const OAuthLocalServer = require('../auth/local-server');
const open = require('open');

async function login() {
  const server = new OAuthLocalServer({
    port: 0, // 자동 포트 선택
    timeout: 300000, // 5분
  });

  try {
    // 서버 시작 (비동기)
    const serverPromise = server.start();
    
    // 콜백 URL 가져오기
    const callbackUrl = server.getCallbackUrl();
    console.log(`콜백 URL: ${callbackUrl}`);
    
    // 인증 URL 생성
    const authUrl = `https://railway.app/oauth/authorize?` +
      `client_id=${CLIENT_ID}&` +
      `redirect_uri=${encodeURIComponent(callbackUrl)}&` +
      `response_type=code&` +
      `scope=read write`;
    
    // 브라우저 열기
    console.log('브라우저를 엽니다...');
    await open(authUrl);
    
    // 인증 코드 대기
    const code = await serverPromise;
    console.log('인증 코드를 받았습니다.');
    
    // 토큰 교환
    const token = await exchangeToken(code, callbackUrl);
    
    // 토큰 저장
    saveToken(token);
    
    console.log('✓ 로그인 성공!');
    
  } catch (error) {
    server.stop();
    console.error('✗ 로그인 실패:', error.message);
    process.exit(1);
  }
}
```

## 추가 고려사항

### 1. 포트 충돌 처리

```javascript
// 사용 가능한 포트 찾기
const net = require('net');

function findAvailablePort(startPort = 3000) {
  return new Promise((resolve, reject) => {
    const server = net.createServer();
    server.listen(startPort, () => {
      const port = server.address().port;
      server.close(() => resolve(port));
    });
    server.on('error', (err) => {
      if (err.code === 'EADDRINUSE') {
        findAvailablePort(startPort + 1).then(resolve).catch(reject);
      } else {
        reject(err);
      }
    });
  });
}
```

### 2. 보안 고려사항

- ✅ `127.0.0.1`만 리스닝 (외부 접근 차단)
- ✅ 타임아웃 설정 (무한 대기 방지)
- ✅ 인증 코드는 단일 사용 (One-time use)
- ✅ 서버는 인증 완료 후 즉시 종료

### 3. 에러 처리

```javascript
// 다양한 에러 상황 처리
server.on('error', (err) => {
  if (err.code === 'EADDRINUSE') {
    // 포트 충돌 - 다른 포트 시도
  } else if (err.code === 'EACCES') {
    // 권한 없음 - 관리자 권한 필요
  } else {
    // 기타 에러
  }
});
```

## 결론

**CLI 로컬 서버로는 Node.js 내장 `http` 모듈을 강력히 추천합니다:**

1. ✅ **의존성 없음** - 추가 패키지 불필요
2. ✅ **가벼움** - CLI 설치 크기 최소화
3. ✅ **충분한 기능** - OAuth 콜백 처리에 완벽
4. ✅ **빠른 시작** - 오버헤드 최소
5. ✅ **크로스 플랫폼** - 모든 OS에서 동작

Express나 다른 프레임워크는 CLI 도구에는 과하고, 의존성만 증가시킵니다.

