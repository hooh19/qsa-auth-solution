# QSA (Quantum Safe API) - 인증 솔루션

JWT 기반 이중 인증 시스템을 제공하는 Quantum Safe API 인증 솔루션입니다.

## 개요

QSA는 매체 인증 서명(wSafeBox X 양자암호화)과 JWT 토큰을 결합한 이중 안전 API 인증 솔루션입니다.

## 주요 특징

- 이중 인증 메커니즘 (JWT 토큰 + 서명 검증)
- PQC(Post-Quantum Cryptography) 알고리즘 기반 암호화
- 클라이언트 키 관리 (WSB)
- 검증(APIAuth Agent lib)
- 토큰 위조 방지
- Stateless 아키텍처

## 프로젝트 구조

```
.
├── index.html          # 메인 웹페이지
├── styles.css          # 스타일시트
├── script.js           # JavaScript
├── qsaAuthAPI .png     # 프로세스 다이어그램
└── *.md                # 문서 파일들
```

## 배포

이 프로젝트는 Netlify를 통해 자동 배포됩니다.

## 문서

- `railway_qsa_integration.md`: Railway CLI와 QSA 통합 방안
- `oauth_token_transfer.md`: OAuth 토큰 전달 방식 설명
- `cli_local_server_comparison.md`: CLI 로컬 서버 선택 가이드

## 라이선스

Copyright 2024 QSA (Quantum Safe API). All rights reserved.

