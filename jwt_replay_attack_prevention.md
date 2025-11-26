# JWT 토큰을 사용한 Replay Attack 방어 방법

## 1. Replay Attack이란?

Replay Attack(재전송 공격)은 공격자가 합법적인 사용자의 요청을 가로채서 나중에 동일한 요청을 반복 전송하는 공격입니다. JWT 토큰만으로는 토큰이 유효한 동안 동일한 요청을 여러 번 재사용할 수 있어 취약합니다.

## 2. JWT 기반 Replay Attack 방어 방법

### 2.1 방법 1: Nonce (Number used Once) 사용

각 요청에 고유한 번호를 포함하여 동일한 요청의 재사용을 방지합니다.

**구현 방법:**
- 클라이언트가 각 요청마다 고유한 nonce 생성 (UUID, 타임스탬프+랜덤 등)
- 서버에서 사용된 nonce를 저장소에 기록
- 동일한 nonce가 재사용되면 요청 거부

**장점:**
- 확실한 재전송 방지
- 구현이 비교적 간단

**단점:**
- 서버 측 저장소 필요 (Stateless 아키텍처와 상충)
- Nonce 저장소 관리 필요

### 2.2 방법 2: Timestamp + Timeout (권장)

요청에 타임스탬프를 포함하고 일정 시간 내에만 유효하도록 제한합니다.

**구현 방법:**
- 각 API 요청에 현재 시간(timestamp) 포함
- 서버에서 요청 시간과 현재 시간 차이 검증
- 일정 시간(예: 5분)을 초과한 요청은 거부

**장점:**
- Stateless 방식으로 구현 가능
- 서버 저장소 불필요
- 구현이 간단

**단점:**
- 클라이언트와 서버 간 시간 동기화 필요
- 짧은 타임아웃은 네트워크 지연 시 문제 발생 가능

### 2.3 방법 3: JWT ID (jti) + 서버 측 블랙리스트

JWT의 `jti` (JWT ID) 클레임을 사용하여 사용된 토큰을 추적합니다.

**구현 방법:**
- JWT 발급 시 고유한 `jti` 포함
- 사용된 `jti`를 서버 측 저장소(Redis, DB 등)에 기록
- 동일한 `jti`로 재사용 시도 시 거부

**장점:**
- 토큰 레벨에서 재사용 방지
- 로그아웃 시 토큰 무효화 가능

**단점:**
- 서버 측 저장소 필요
- Stateless 아키텍처와 상충

### 2.4 방법 4: 요청 서명 (Request Signing) - QSA 방식

각 요청의 내용을 서명하여 위변조 및 재사용을 방지합니다.

**구현 방법:**
- API 요청 파라미터를 JSON으로 구성
- 클라이언트의 Private Key로 요청 데이터 서명
- 서버에서 Public Key로 서명 검증
- 서명에 타임스탬프 포함하여 재사용 방지

**장점:**
- 가장 강력한 보안 (이중 검증)
- 파라미터 위변조도 동시에 방지
- Stateless 방식 가능

**단점:**
- 구현 복잡도가 높음
- 클라이언트 측 키 관리 필요

### 2.5 방법 5: One-Time Token (OTT)

각 요청마다 새로운 토큰을 발급받아 사용합니다.

**구현 방법:**
- API 호출 전에 일회용 토큰 발급 요청
- 발급된 토큰은 한 번만 사용 가능
- 사용 후 즉시 무효화

**장점:**
- 확실한 일회성 보장

**단점:**
- 매 요청마다 토큰 발급 필요 (성능 저하)
- 서버 부하 증가

## 3. QSA 프로젝트에 권장하는 방법

현재 QSA 프로젝트는 **방법 2 (Timestamp + Timeout)**와 **방법 4 (요청 서명)**를 조합하여 사용하는 것이 가장 적합합니다.

### 3.1 하이브리드 방어 전략

```
1. JWT 토큰 검증 (기본 인증)
   ↓
2. 요청 서명 검증 (채널 검증 + 위변조 방지)
   ↓
3. Timestamp 검증 (Replay Attack 방지)
```

### 3.2 구현 예시

#### 클라이언트 측 (JavaScript)

```javascript
// API 요청 생성 시
async function makeAPICall(endpoint, params) {
  // 1. 타임스탬프 생성
  const timestamp = Date.now();
  
  // 2. 요청 데이터 구성 (타임스탬프 포함)
  const requestData = {
    ...params,
    timestamp: timestamp,
    nonce: generateNonce() // 추가 보안을 위한 nonce
  };
  
  // 3. WebSafeBox로 요청 데이터 서명
  const signature = await webSafeBox.sign(
    JSON.stringify(requestData),
    privateKey
  );
  
  // 4. API 요청 전송
  const response = await fetch(endpoint, {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${jwtToken}`,
      'Content-Type': 'application/json',
      'X-Timestamp': timestamp.toString(),
      'X-Signature': signature
    },
    body: JSON.stringify({
      ...requestData,
      signature: signature
    })
  });
  
  return response.json();
}

function generateNonce() {
  return `${Date.now()}-${Math.random().toString(36).substr(2, 9)}`;
}
```

#### 서버 측 (Node.js)

```javascript
const APIAuthAgent = require('@qsa/api-auth-agent');

async function verifyRequest(req, res, next) {
  try {
    // 1. JWT 토큰 검증
    const token = req.headers.authorization?.replace('Bearer ', '');
    if (!token) {
      return res.status(401).json({ error: 'Missing token' });
    }
    
    const tokenPayload = await APIAuthAgent.verifyToken(token);
    if (!tokenPayload) {
      return res.status(401).json({ error: 'Invalid token' });
    }
    
    // 2. 타임스탬프 검증 (Replay Attack 방지)
    const timestamp = parseInt(req.headers['x-timestamp'] || req.body.timestamp);
    if (!timestamp) {
      return res.status(401).json({ error: 'Missing timestamp' });
    }
    
    const currentTime = Date.now();
    const timeDiff = Math.abs(currentTime - timestamp);
    const TIMEOUT_MS = 5 * 60 * 1000; // 5분
    
    if (timeDiff > TIMEOUT_MS) {
      return res.status(401).json({ 
        error: 'Request expired',
        detail: `Time difference: ${timeDiff}ms (max: ${TIMEOUT_MS}ms)`
      });
    }
    
    // 3. 서명 검증
    const signature = req.headers['x-signature'] || req.body.signature;
    if (!signature) {
      return res.status(401).json({ error: 'Missing signature' });
    }
    
    // 서명값 제외한 원본 데이터 추출
    const requestBody = { ...req.body };
    delete requestBody.signature;
    
    const isValid = await APIAuthAgent.verifySignature(
      JSON.stringify(requestBody),
      signature,
      tokenPayload.mediaPublicKey // JWT에서 추출한 Public Key
    );
    
    if (!isValid) {
      return res.status(401).json({ error: 'Invalid signature' });
    }
    
    // 4. (선택) Nonce 검증 (추가 보안)
    if (requestBody.nonce) {
      const nonceKey = `nonce:${requestBody.nonce}`;
      const used = await redis.get(nonceKey);
      if (used) {
        return res.status(401).json({ error: 'Duplicate request (nonce reused)' });
      }
      // Nonce를 짧은 시간(예: 10분) 동안 저장
      await redis.setex(nonceKey, 600, '1');
    }
    
    // 검증 통과
    req.user = tokenPayload.user;
    req.verified = true;
    next();
    
  } catch (error) {
    console.error('Request verification error:', error);
    return res.status(500).json({ error: 'Verification failed' });
  }
}
```

## 4. 보안 강화를 위한 추가 고려사항

### 4.1 타임스탬프 검증의 한계

- **클라이언트-서버 시간 동기화**: NTP를 통한 시간 동기화 필요
- **네트워크 지연**: 타임아웃을 너무 짧게 설정하면 정상 요청도 거부될 수 있음
- **권장 타임아웃**: 5분 (300초) - 네트워크 지연을 고려한 적절한 값

### 4.2 Nonce와 Timestamp 조합

타임스탬프만으로는 완벽하지 않으므로, Nonce를 추가하면 더욱 안전합니다:

```javascript
// 클라이언트
const requestData = {
  ...params,
  timestamp: Date.now(),
  nonce: crypto.randomUUID() // 또는 타임스탬프 기반 고유값
};
```

### 4.3 서명에 타임스탬프 포함

서명 생성 시 타임스탬프를 포함하면 더욱 안전합니다:

```javascript
// 서명할 데이터에 타임스탬프 포함
const dataToSign = {
  ...params,
  timestamp: Date.now(),
  endpoint: '/api/transfer' // 엔드포인트도 포함
};

const signature = await sign(JSON.stringify(dataToSign), privateKey);
```

### 4.4 요청 순서 검증 (선택)

중요한 작업의 경우 요청 순서를 검증할 수 있습니다:

```javascript
// 클라이언트가 요청 번호를 증가시키며 전송
const requestData = {
  ...params,
  timestamp: Date.now(),
  requestId: getNextRequestId() // 순차적으로 증가하는 ID
};
```

## 5. 각 방법의 비교표

| 방법 | Stateless | 구현 복잡도 | 보안 수준 | 성능 영향 |
|------|-----------|------------|----------|----------|
| Nonce | ❌ | 낮음 | 높음 | 중간 |
| Timestamp + Timeout | ✅ | 낮음 | 중간 | 낮음 |
| jti + 블랙리스트 | ❌ | 중간 | 높음 | 중간 |
| 요청 서명 | ✅ | 높음 | 매우 높음 | 낮음 |
| One-Time Token | ✅ | 중간 | 매우 높음 | 높음 |

## 6. 결론

QSA 프로젝트에서는 **Timestamp + Timeout + 요청 서명**을 조합하여 사용하는 것을 권장합니다:

1. ✅ **Stateless 아키텍처 유지** (확장성)
2. ✅ **강력한 보안** (이중 검증)
3. ✅ **Replay Attack 완벽 방지**
4. ✅ **파라미터 위변조도 동시에 방지**

이 방식은 현재 프로젝트의 아키텍처와도 잘 맞으며, 이미 구현된 이중 서명 검증 메커니즘과 자연스럽게 통합됩니다.


