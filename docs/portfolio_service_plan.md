# Portfolio Management Service Plan

## 1. 목표와 핵심 요구사항
- 개인 투자자가 보유 자본, 부채, 목표 포트폴리오 구성 비중을 입력하고 자동으로 매수 수량과 배당 추정치를 계산
- 최초 진입 가격과 시장 상황에 따라 리밸런싱 시점을 안내
- 전체 포트폴리오 구성 및 수익률을 시각화하고, S&P 500 등 벤치마크 지수와 비교 제공
- 향후 자본금 증가/감소, 부채 조정에 대한 시나리오 분석 지원

## 2. 주요 사용자 플로우
1. **자산/부채 입력**
   - 보유 자본금, 보유 부채 잔액, 부채 이자율, 미사용 한도 입력
   - 자산/부채 요약 카드와 함께 가용 투자금 계산
2. **포트폴리오 구성 입력**
   - 자산 클래스(주식, ETF, 채권, 현금 등), 종목/티커, 목표 비중 입력
   - 비중 합산 검증, 추천 모델(예: 기존 템플릿) 제공
3. **투자 실행 가이드**
   - 시세 데이터 수집 후 목표 비중에 맞는 매수 수량, 잔여 현금, 예상 배당금 계산
   - 배당 캘린더 표시, 세후 수령액 추정
4. **리밸런싱 가이드**
   - 최초 진입가격 기록, 현재 시세와 비교하여 편차 계산
   - 편차 허용 범위(예: 5%) 설정 후 리밸런싱 알림 생성
5. **성과 분석 및 시각화**
   - 히스토리컬 수익률, 자산 비중 변화, S&P 500과의 비교 차트 제공
   - 시나리오 분석(추가 입금, 부채 상환) 시뮬레이션

## 3. 시스템 아키텍처
```
[Frontend (Next.js + TypeScript)]
    |
    v
[API Gateway / BFF (NestJS or FastAPI)]
    |
    +-->[Portfolio Service]
    +-->[Market Data Service]
    +-->[Dividend Service]
    +-->[Analytics Service]
    |
    v
[Data Store]
  - PostgreSQL (거래 내역, 포트폴리오 설정)
  - Redis (캐시)
  - S3/Blob Storage (리포트, 스냅샷)

[ETL/Batch Layer]
  - Airflow/Prefect로 시세·배당 데이터 수집
  - Snowflake/BigQuery/Redshift 등 DW (선택 사항)
```

### 3.1 프론트엔드
- **기술 스택**: Next.js(React 기반), TypeScript, Tailwind CSS, Zustand/Redux Toolkit 상태관리
- **주요 화면**
  - 자산/부채 입력 폼 및 Summary 카드
  - 포트폴리오 빌더 (드래그&드롭, 비중 슬라이더)
  - 투자 가이드(매수 수량, 배당 추정치) 표시 카드
  - 리밸런싱 알림 타임라인
  - 분석 대시보드: Pie Chart(비중), Line Chart(수익률), Benchmark 비교
- **차트 라이브러리**: Recharts, ECharts, 혹은 D3 기반 래퍼
- **인증/권한**: NextAuth, Auth0, Cognito 등 연계

### 3.2 백엔드
- **API 설계** (예시)
  - `POST /users/{id}/portfolio` : 포트폴리오 구성 저장
  - `GET /users/{id}/portfolio/summary` : 현재 비중, 수익률, 매수 추천 반환
  - `POST /users/{id}/rebalance/simulate` : 리밸런싱 결과 시뮬레이션
  - `GET /market-data/{ticker}` : 실시간/지연 시세, 배당 정보
  - `GET /benchmarks/sp500/performance` : 벤치마크 수익률
- **로직 구성**
  - 포트폴리오 계산 서비스: 비중에 따른 매수 수량 계산, 리밸런싱 드리프트 분석
  - 배당 추정 서비스: 과거 배당 데이터 기반 연간 배당 추정, 배당 캘린더 생성
  - 리밸런싱 가이드: 목표 대비 편차, 거래 비용 추정, 세금 고려
- **프레임워크 후보**
  - Python FastAPI (빠른 개발, 데이터 분석 친화적)
  - Node.js NestJS (TypeScript 일관성)

### 3.3 데이터 파이프라인
- **수집 대상**: 가격(일/분), 배당, 벤치마크, 금리, 환율
- **API 소스**: Yahoo Finance, Alpha Vantage, Polygon.io, Quandl 등
- **흐름**
  1. ETL 스케줄러가 외부 API 호출하여 원천 데이터 수집
  2. 원천 데이터 검증 후 Data Lake 저장
  3. 표준화/정규화 파이프라인을 통해 DB 적재
  4. 캐시 및 API 응답용으로 최신 데이터 Redis 저장
- **분석/머신러닝**: 리밸런싱 최적화, 몬테카를로 시뮬레이션, 위험 지표(Sharpe, Max Drawdown)

## 4. 데이터 모델 개요
- `users`: 사용자 기본 정보, 위험 성향
- `assets`: 티커, 자산 유형, 통화
- `portfolios`: 사용자별 포트폴리오 마스터, 목표 수익률, 리밸런싱 정책
- `portfolio_positions`: 목표 비중, 최초 진입가, 목표 수량
- `portfolio_snapshots`: 일별 평가액, 수익률 기록
- `transactions`: 매수/매도 기록, 수수료
- `dividends`: 배당 지급 일정, 세율
- `benchmarks`: 지수 가격 기록

## 5. 핵심 기능 상세 설계
### 5.1 자산/부채 입력 및 투자 가능 금액 계산
- 프론트에서 입력 받은 자본, 부채, 이자율, 사용 가능 한도 → `Net Investable Capital = Cash + Available Credit - (Debt * Buffer)`
- 이자 비용 계산 및 월별 현금 흐름 시각화

### 5.2 포트폴리오 구성 입력
- 자유도 높은 구성 지원: 커스텀 티커 추가, 비중 입력 슬라이더, CSV 업로드
- Risk Level에 따라 추천 비중 템플릿 제공 (예: 60/40)
- 비중 합계 검증 및 경고 표시

### 5.3 매수 수량 및 배당 계산
- 최신 가격과 배당 수익률 조회 → `목표 투자금 * 비중 / 현재 가격 = shares_to_buy`
- `Expected Dividend = shares_to_buy * trailing_dividend_per_share`
- 배당 캘린더: 지급일과 금액 표시, 세후 금액 계산(국내/해외 세율 반영)

### 5.4 리밸런싱 가이드
- 최초 진입가 대비 현재 가치 비교: `drift = (current_weight - target_weight)`
- 허용 편차 설정 후 자동 알림
- 거래 비용, 세금 추정 반영하여 순효과 제시
- 시나리오 분석: 추가 입금/출금, 부채 상환 영향

### 5.5 수익률 시각화 및 벤치마크 비교
- Time Series Chart: 포트폴리오 vs S&P 500 vs Custom Benchmark
- Pie/Donut Chart: 현 시점 비중, 누적 배당 재투자 비중 표시
- 성과 지표: CAGR, MDD, Sharpe Ratio, 배당수익률
- 리포트 PDF 생성 기능 (주간/월간)

## 6. 기술 선택 및 인프라
- **호스팅**: AWS (Amplify/EC2/Lambda), Vercel (Frontend), Render/Heroku (Backend)
- **CI/CD**: GitHub Actions (테스트, 린트, 배포)
- **모니터링**: Datadog/New Relic, Sentry(프론트), OpenTelemetry(백엔드)
- **보안**: JWT/OAuth 인증, KMS로 API 키 암호화, 비밀번호 해시(Argon2)

## 7. 로드맵
1. **MVP (4~6주)**
   - 사용자 인증, 자산/부채 입력, 포트폴리오 생성
   - 외부 시세 API 연동, 매수 수량 계산, 기본 배당 추정
   - 기본 리밸런싱 알림(이메일), 수익률 차트(S&P 500 비교)
2. **Phase 2 (6~10주)**
   - 고급 분석: 몬테카를로 시뮬레이션, VaR
   - 알림 채널 확장(Slack, 모바일 푸시)
   - 다중 통화 지원, 세금 계산 고도화
3. **Phase 3 (이후)**
   - 알고리즘 리밸런싱, 자동 주문 API 연계(증권사 계좌)
   - 소셜 포트폴리오 공유, 커뮤니티 기능

## 8. 데이터 시각화 & 분석 도구
- 프론트: Recharts/ECharts + Tailwind
- 백엔드: Pandas, NumPy, Prophet(예측), scikit-learn(리스크 분석)
- 노트북 환경: Jupyter + Papermill로 리포트 자동화

## 9. 향후 확장 아이디어
- ESG 점수 기반 필터링, 테마 포트폴리오 추천
- 머신러닝을 통한 배당 컷 예측, 리스크 시그널 감지
- 사용자 맞춤 뉴스 피드 및 이벤트 기반 경고

## 10. 개발 및 협업 프로세스
- GitHub Issues + Projects로 태스크 관리
- Conventional Commits 적용
- 코드 리뷰 및 CI 파이프라인 구축
