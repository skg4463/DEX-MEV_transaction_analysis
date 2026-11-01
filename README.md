# Decentralized Exchange Transaction Analysis and MEV Attack Identification: Focusing on Uniswap USDC3

**Nakhoon Choi, Heeyoul Kim**

MDPI 논문 *"Decentralized exchange transaction analysis and MEV attack identification: Focusing on Uniswap USDC3"* 의 **데이터 수집 코드, 분석용 데이터셋, 그래프 생성 코드** 저장소입니다.

이더리움 Uniswap V3 **USDC/WETH 풀(USDC3)** 의 스왑·유동성 트랜잭션과 가스 가격을 수집하고, MEV 봇의 거래 비중, 아비트라지, 샌드위치 공격, JIT(Just-In-Time) 유동성 공격의 흔적을 분석합니다.

![Data collection and analysis process for DEX analysis.](https://github.com/skg4463/MDPI_DEX-MEV_transaction_analysis/assets/45844164/43da90b2-011e-4b10-bba9-e10236a608fc)

*Figure. Data collection and analysis process for DEX analysis.*

## 분석 대상

| 항목 | 값 |
|---|---|
| 체인 | Ethereum Mainnet |
| 풀 | Uniswap V3 USDC/WETH (0.05%) — `0x88e6A0c2dDD26FEEb64F039a2c41296FcB3f5640` |
| 라우터 | Uniswap Universal Router — `0x3fC91A3afd70395Cd496C647d5a6CC9D4B2b7FAD` |
| 토큰 | USDC `0xa0b86991c6218b36c1d19d4a2e9eb0ce3606eb48`, WETH `0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2` |
| 장기 분석 기간 | 2022-01-01 ~ 2023-12-30 (블록 13,916,330 ~ 18,901,697) |
| 상세 트랜잭션 분석 구간 | 2023-01-16 ~ 2023-01-17 (블록 16,415,635 ~ 16,426,657) |

## 저장소 구성

파일 이름의 접두어로 역할을 구분합니다.

| 접두어 | 역할 |
|---|---|
| `dataEX-*` | 데이터 수집(Extraction) 스크립트 |
| `data-*` | 분석에 사용한 데이터셋 |
| `grpdrw-*` | 데이터로 그래프를 그리는 스크립트 |
| `figures/` | 논문에 사용한 그림 (Figure 1 ~ 12) |
| `graph/` | 분석 과정에서 생성한 그래프 (유동성, 가스 가격, 트랜잭션 수 등) |

```
.
├── dataEX-EtherAPI_get_token_transfer.py            # Etherscan: 풀 Swap 이벤트 로그 수집
├── dataEX-EtherAPI_get_token_transfer_gasPriceOnly.py  # Etherscan: 가스 가격 수집 + 1시간 평균
├── dataEX-sdk_base_endpoint_swaps.py                # Transpose SDK: USDC 스왑 조회
├── dataEX-sdk_custom_endpoint_liqui.py              # Transpose 커스텀 엔드포인트: 유동성 추가/제거 수집
├── grpdrw-DEXtrading_piechart.py                    # DEX별 거래량 파이 차트
├── grpdrw-bot_trading_volume.py                     # 봇 vs 일반 사용자 거래 수/거래량
├── grpdrw-liquidity_graph.py                        # 유동성 추가·제거, 풀 잔액, 거래량
├── liquidity_query.txt                              # Transpose 커스텀 엔드포인트용 SQL
├── token.txt                                        # 주소, 블록 범위, 이벤트 구조 등 작업 메모
├── data-*.csv / data-*.xlsx                         # 데이터셋 (아래 표)
├── figures/                                         # 논문 그림
└── graph/                                           # 분석 그래프
```

## 데이터 수집 (`dataEX-*`)

수집에는 세 가지 경로를 사용했습니다.

1. **Etherscan.io 웹 내보내기** — 주소별 토큰 전송 CSV 다운로드
2. **Etherscan API** — `logs/getLogs`로 풀 컨트랙트의 이벤트 로그 수집
3. **Transpose.io** — Python SDK 및 SQL 기반 커스텀 엔드포인트

> **API 키는 직접 준비해야 합니다.** 각 스크립트의 `YOUR_OWN_APIKEY` / `YOUR_API_KEY` 부분을 본인 키로 바꿔 사용하세요.

| 스크립트 | 소스 | 동작 | 출력 |
|---|---|---|---|
| `dataEX-EtherAPI_get_token_transfer.py` | Etherscan API | USDC3 풀의 이벤트 로그를 1,000개씩 페이지 단위로 수집 (블록 16,422,226 ~ 16,426,657) | `data-(from-etherscanAPI)_uniswap_transactions2.csv` |
| `dataEX-EtherAPI_get_token_transfer_gasPriceOnly.py` | Etherscan API | 로그의 `gasPrice`를 모아 **1시간 평균**으로 리샘플링 | `data-aggregated_gas_prices.csv` |
| `dataEX-sdk_base_endpoint_swaps.py` | Transpose SDK | USDC 컨트랙트의 스왑 중 WETH 관련 Uniswap 스왑 조회 (2023-01-16 하루) | 콘솔 출력 |
| `dataEX-sdk_custom_endpoint_liqui.py` | Transpose 커스텀 엔드포인트 | 48시간 단위로 USDC3 풀의 유동성 추가/제거 이벤트 수집 (쿼리: `liquidity_query.txt`) | `data-(from-trans)_uniswap_data_*.csv` |

### Swap 이벤트 `data` 필드 디코딩

Etherscan API로 받은 CSV의 `data` 컬럼은 Uniswap V3 `Swap` 이벤트의 비인덱스 인자를 32바이트(64 hex)씩 이어 붙인 값입니다.

```
Swap(address indexed sender, address indexed recipient,
     int256 amount0, int256 amount1, uint160 sqrtPriceX96, uint128 liquidity, int24 tick)
```

| 순서 | 필드 | 타입 | 의미 |
|---|---|---|---|
| 1 | `amount0` | int256 | 풀의 token0(USDC) 잔액 변화량 |
| 2 | `amount1` | int256 | 풀의 token1(WETH) 잔액 변화량 |
| 3 | `sqrtPriceX96` | uint160 | 스왑 후 풀의 √가격 (Q64.96) |
| 4 | `liquidity` | uint128 | 스왑 후 풀의 유동성 |
| 5 | `tick` | int24 | 스왑 후 가격의 log₁.₀₀₀₁ |

디코딩 예시는 `data-(from-etherscanAPI)_uniswap_transaction_csv_info`에 있습니다.
참고: [IUniswapV3PoolEvents](https://docs.uniswap.org/contracts/v3/reference/core/interfaces/pool/IUniswapV3PoolEvents)

## 데이터셋 (`data-*`)

### 직접 수집한 데이터

| 파일 | 내용 | 기간 | 주요 컬럼 |
|---|---|---|---|
| `data-(from-etherscan)_export-address-token.csv` | Etherscan 웹에서 내보낸 USDC3 풀의 토큰 전송 내역 | 2023-01-16 ~ | `Txhash, Blockno, DateTime, From, To, TokenValue, TokenSymbol` |
| `data-(from-etherscanAPI)_uniswap_transactions.csv` | USDC3 풀 Swap 이벤트 로그 | 2023-01-16 00:00 ~ 01-17 12:55 | `blockNumber, timeStamp, transactionHash, sender, to, data, gasPrice, gasUsed` |
| `data-(from-etherscanAPI)_uniswap_transactions2.csv` | 위와 같음 (블록 16,422,226부터) | 2023-01-16 22:06 ~ 01-17 12:55 | 위와 같음 |
| `data-(from-trans)_uniswap_data_22.01-22.12.csv` | USDC3 풀 유동성 추가/제거 이벤트 | 2022년 | `block_number, category(deposit/withdraw), liquidity, liquidity_delta, pool_balance, quantity, sender_address, tick_lower, tick_upper, timestamp, token_address` |
| `data-(from-trans)_uniswap_data_23.01-23.12.csv` | 위와 같음 | 2023년 | 위와 같음 (컬럼 순서 다름) |
| `data-(from-trans)_uniswap_data_22.01-23.12.csv` | 위 두 파일 통합본 | 2022 ~ 2023 | 위와 같음 |
| `data-aggregated_gas_prices.csv` | USDC3 풀 트랜잭션의 1시간 평균 가스 가격 (wei) | 2022-01-01 ~ 01-07 | `timeStamp, blockNumber, gasPrice` |
| `data-aggregated_gas_prices1.csv` | 위와 같음 | 2022-01-05 23시 ~ 01-07 09시 | 위와 같음 |

### 외부 집계 데이터

| 파일 | 내용 | 기간 | 주요 컬럼 |
|---|---|---|---|
| `data-ethereum_price.csv` | ETH 시간별 평균 가격 (USD) | 2022 | `average_price, timestamp` |
| `data-bot_trading_volume.csv` | 주간 거래 수/거래량 및 봇 비중 | 2022-01 ~ 2023-11 | `time, number_of_trades, trade_volume_usd, percent_trades, percent_volume` |
| `data-MEVarbitrage_uniswap.csv` | Uniswap 일별 아비트라지 건수·금액·봇 수 | 2023 | `block_date, arbitrage_transaction_count, arbitrage_amount, arbitrage_average_amount, arbitrage_bots_count` |
| `data-MEVbot_profit_bySYM.csv` | MEV 봇 수익의 토큰별 합계 | — | `symbol, profit` |
| `data-DEXattacker_protocols.csv` | 공격자가 사용한 프로토콜/컨트랙트 상위 100개 | — | `project_name, name, counter` |
| `data-Gas2AttackVolume.xlsx` | 월별 피해 거래량과 공격 가스비 | 2021-10 ~ | `month, Minimal attacked Trade Volume, Minimal Gas fee of attack` |
| `data-DEX trading volume.csv` | DEX별 7일/24시간 거래량 상위 20개 (**실제 형식은 xlsx**) | — | `Project, 7 Days Volume, 24 Hours Volume` |

## 그래프 생성 (`grpdrw-*`)

| 스크립트 | 입력 | 그래프 |
|---|---|---|
| `grpdrw-DEXtrading_piechart.py` | `data-DEX trading volume.csv` | 7일 거래량 기준 상위 5개 DEX + 기타 파이 차트 |
| `grpdrw-bot_trading_volume.py` | `data-bot_trading_volume.csv` | 봇 vs 일반 사용자의 거래 수, 거래량(USD) 추이 |
| `grpdrw-liquidity_graph.py` | 유동성 데이터, `data-ethereum_price.csv`, `data-bot_trading_volume.csv` | 72시간 단위 유동성 추가/제거, 평균 풀 잔액, 주간 거래량 |

`graph/` 폴더의 파일명 약어:

- `Liqu_balance_{12T,24T,72T}` — 유동성/풀 잔액 그래프 (리샘플링 간격 12·24·72시간)
- `*_ETHprice`, `*_botBene` — ETH 가격, 봇 거래량을 함께 표시한 버전
- `SL_{30T,1H,2H}` — 30분·1시간·2시간 단위 집계
- `gasPrice_overtime`, `gasPrise_gasUsed`, `numberOfTransactions_byTime` — 가스 가격 추이, 가스 가격-사용량 관계, 시간대별 트랜잭션 수

## 실행 환경

```bash
pip install requests pandas matplotlib openpyxl transpose
```

- Python 3
- Etherscan API 키 (무료 플랜 기준 초당 5회 제한 → 스크립트에 0.2~0.3초 대기 포함)
- Transpose.io API 키 (초당 1회 요청 기준으로 대기)

### 실행 전 참고

`grpdrw-*` 스크립트는 분석 당시 노트북 환경에서 쓰던 코드라 단독 실행 시 수정이 필요합니다.

- `grpdrw-DEXtrading_piechart.py`, `grpdrw-bot_trading_volume.py`: 데이터 로딩 코드가 없습니다 (`dex_data`, `df` 정의, `import pandas as pd` 추가 필요). DEX 거래량 파일은 확장자가 `.csv`지만 `pd.read_excel()`로 읽어야 합니다.
- `grpdrw-liquidity_graph.py`: 입력 파일명이 `uniswap_data.csv`로 되어 있으니 `data-(from-trans)_uniswap_data_*.csv` 중 하나로 바꿔야 합니다.
- `dataEX-*` 스크립트는 출력 파일에 **append**하므로, 다시 실행하면 기존 CSV 뒤에 이어 붙습니다. `dataEX-EtherAPI_get_token_transfer.py`는 페이지마다 헤더 행을 다시 씁니다.

## 인용

이 저장소의 데이터나 코드를 사용하신다면 위 논문을 인용해 주세요.
