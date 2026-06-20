# MoneyFlow 주도섹터 Rank Table 모듈 소스 레벨 상세 설계서

문서 버전: v3.3
작성일: 2026-06-20
프로젝트명: MoneyFlow Rank Table (MFRT)
목적: 월별 업종 수익률 순위 테이블을 기반으로 주도 섹터 탄생 → 확산 → 과열 → 피크아웃 → 주도권 교체를 탐지하고, MoneyFlow / 3중스크린 / 추세추종 스윙 투자 시스템에 연결하기 위한 소스 레벨 상세 설계

---

## 변경 이력

| 버전 | 날짜 | 변경 내용 |
|---|---|---|
| v1.0 | 2026-06-20 | 초안 |
| v2.0 | 2026-06-20 | 업종 체계 고정(KRX KOSPI 21개), 수익률 계산 인접월 검증 추가, trading_value 파이프라인 연결, 신호 엔진 정책 일치, 색상 매핑 단일화, CRUD 인터페이스 정의 |
| v3.0 | 2026-06-20 | SectorPriceCollector 3단계 fallback 설계(Path3 KRX direct는 bld·필드명 미확정 — 구현 착수 전 확정 필요), min_months_required를 수익률 계산 전 가격 데이터 월 수 기준으로 수정, LeadershipScoreEngine/RotationSignalEngine rolling window 하드코딩 제거(config 값 사용) |
| v3.1 | 2026-06-20 | Path3(_try_krx_direct) 섹션 제목·docstring·상수 주석을 "미확정 설계 계약" 상태로 명확화, settings.py YAML 로더(load_config/build_leadership_score_config/build_signal_config/get_min_months_required) 설계 추가, CLI에서 YAML → Config 실제 연결(--config 인수, build_*_config 호출, min_months YAML에서 로드) |
| v3.2 | 2026-06-20 | _fetch_month_end 실패 반환 계약 3튜플로 통일((None,None) → (None,None,"")), SignalConfig에서 미사용 중복 필드(leader_lookback_months/extended_lookback_months) 제거 — 윈도우 소스를 LeadershipScoreConfig 단일 소유로 명확화, YAML signals 섹션 주석 정합성 수정 |
| v3.3 | 2026-06-20 | LeadershipScoreConfig rolling window 주석에서 구버전 표현("SignalConfig와 반드시 일치") 제거 — 단일 소유 의도와 일치하도록 수정 |

---

## 1. 설계 목표

본 모듈의 목표는 업종별 월간 수익률 데이터를 수집·정규화한 뒤, 매월 업종 순위를 산출하고, 순위 변화·상위권 지속성·밸류체인 확산·거래대금 증가를 종합하여 주도 섹터 신호를 생성하는 것이다.

핵심 산출물은 다음 4가지다.

| 산출물 | 설명 |
|---|---|
| 월별 업종 Rank Table | 월별 수익률 순위 기준으로 업종명을 세로 정렬한 테이블 |
| 주도섹터 점수 | 순위, 지속성, 모멘텀, 확산성, 거래대금 기반 점수 |
| 섹터 Rotation Signal | NEW_LEADER, LEADER, EXTENDED, PEAK_WARNING, ROTATION_OUT 등 |
| 대장주 후보 리스트 | 주도섹터 내 거래대금·수급·차트 조건을 만족하는 종목 |

---

## 2. 정규 업종 체계 (Canonical Sector Universe)

**모든 순위 계산, 백테스트, 신호 생성의 기준이 되는 단일 업종 체계다. 소스 선택과 무관하게 이 코드 체계로 정규화한 뒤 처리한다.**

### 2.1 데이터 소스 및 업종 체계

| 항목 | 결정 |
|---|---|
| 기준 체계 | KRX KOSPI 업종 분류 |
| 업종 수 | 21개 (섹터 코드 1003~1023) |
| top_n | 21 |
| 히스토리 기간 | 2009-01 ~ 현재 |
| 수집 소스 | pykrx `get_index_ohlcv_by_date` → KRX HTTP direct fallback |

### 2.2 정규 업종 코드 테이블 (seed data)

```python
# app/config/sector_universe.py

KRX_KOSPI_SECTORS: list[dict] = [
    {"sector_code": "1003", "sector_name": "음식료품",   "market": "KOSPI", "theme_group": "소비재",   "display_color": "FFFFFF"},
    {"sector_code": "1004", "sector_name": "섬유의복",   "market": "KOSPI", "theme_group": "소비재",   "display_color": "FFFFFF"},
    {"sector_code": "1005", "sector_name": "종이목재",   "market": "KOSPI", "theme_group": "소재",     "display_color": "FFFFFF"},
    {"sector_code": "1006", "sector_name": "화학",       "market": "KOSPI", "theme_group": "소재",     "display_color": "FFC7CE"},
    {"sector_code": "1007", "sector_name": "의약품",     "market": "KOSPI", "theme_group": "바이오",   "display_color": "FFEB9C"},
    {"sector_code": "1008", "sector_name": "비금속광물", "market": "KOSPI", "theme_group": "소재",     "display_color": "FFFFFF"},
    {"sector_code": "1009", "sector_name": "철강금속",   "market": "KOSPI", "theme_group": "소재",     "display_color": "FFFFFF"},
    {"sector_code": "1010", "sector_name": "기계",       "market": "KOSPI", "theme_group": "산업재",   "display_color": "FFFFFF"},
    {"sector_code": "1011", "sector_name": "전기전자",   "market": "KOSPI", "theme_group": "반도체",   "display_color": "C6EFCE"},
    {"sector_code": "1012", "sector_name": "의료정밀",   "market": "KOSPI", "theme_group": "바이오",   "display_color": "FFEB9C"},
    {"sector_code": "1013", "sector_name": "운수장비",   "market": "KOSPI", "theme_group": "자동차",   "display_color": "FFC7CE"},
    {"sector_code": "1014", "sector_name": "유통업",     "market": "KOSPI", "theme_group": "소비재",   "display_color": "FFFFFF"},
    {"sector_code": "1015", "sector_name": "전기가스업", "market": "KOSPI", "theme_group": "유틸리티", "display_color": "D9EAD3"},
    {"sector_code": "1016", "sector_name": "건설업",     "market": "KOSPI", "theme_group": "건설",     "display_color": "FFFFFF"},
    {"sector_code": "1017", "sector_name": "운수창고업", "market": "KOSPI", "theme_group": "운송",     "display_color": "FFFFFF"},
    {"sector_code": "1018", "sector_name": "통신업",     "market": "KOSPI", "theme_group": "통신",     "display_color": "FFFFFF"},
    {"sector_code": "1019", "sector_name": "금융업",     "market": "KOSPI", "theme_group": "금융",     "display_color": "FFFFFF"},
    {"sector_code": "1020", "sector_name": "은행",       "market": "KOSPI", "theme_group": "금융",     "display_color": "FFFFFF"},
    {"sector_code": "1021", "sector_name": "증권",       "market": "KOSPI", "theme_group": "금융",     "display_color": "FFFFFF"},
    {"sector_code": "1022", "sector_name": "보험",       "market": "KOSPI", "theme_group": "금융",     "display_color": "FFFFFF"},
    {"sector_code": "1023", "sector_name": "서비스업",   "market": "KOSPI", "theme_group": "서비스",   "display_color": "FFFFFF"},
]
```

`display_color`는 sector_master에 저장되며 Excel 출력의 유일한 색상 소스다. 업종명 키워드 하드코딩은 사용하지 않는다.

### 2.3 업종명 변경 매핑

KRX 업종명이 변경될 경우 alias 테이블로 처리하고, 코드(1003~1023)를 고정 키로 사용한다.

```sql
CREATE TABLE IF NOT EXISTS sector_name_alias (
    sector_code TEXT NOT NULL,
    alias_name  TEXT NOT NULL,
    effective_from TEXT NOT NULL,  -- YYYY-MM
    effective_to   TEXT,           -- NULL이면 현재까지
    PRIMARY KEY (sector_code, alias_name)
);
```

---

## 3. 전체 아키텍처

```text
[데이터 소스]
  ├─ KRX 업종지수 (sector_code: 1003~1023)
  └─ pykrx get_index_ohlcv_by_date → KRX HTTP direct fallback
        ↓
[수집 계층]
  └─ sector_price_collector.py
        ↓
[DB 계층]
  ├─ sector_master
  ├─ sector_name_alias
  ├─ sector_monthly_price
  ├─ sector_monthly_return
  ├─ sector_rank_snapshot
  ├─ sector_leadership_score
  ├─ sector_rotation_signal
  └─ sector_leader_stocks
        ↓
[분석 계층]
  ├─ monthly_return_calculator.py
  ├─ rank_table_builder.py
  ├─ leadership_score_engine.py
  ├─ rotation_signal_engine.py
  └─ leader_stock_selector.py
        ↓
[리포트 계층]
  ├─ rank_table_excel_exporter.py
  ├─ rank_table_html_report.py
  └─ sector_briefing_writer.py
```

---

## 4. 디렉터리 구조

```text
moneyflow/
  app/
    config/
      settings.py
      sector_universe.py        ← 정규 업종 체계 (seed data)
      sector_config.yaml
      sector_theme_map.yaml
    db/
      connection.py             ← get_connection() 정의
      crud.py                   ← 모든 CRUD 함수 정의
      migrations/
        001_create_sector_tables.sql
        002_create_sector_score_tables.sql
        003_create_sector_alias_table.sql
    collectors/
      sector_price_collector.py
    services/
      monthly_return_calculator.py
      rank_table_builder.py
      leadership_score_engine.py
      rotation_signal_engine.py
      leader_stock_selector.py
    reports/
      rank_table_excel_exporter.py
      rank_table_html_report.py
      sector_briefing_writer.py
    cli/
      run_sector_monthly_pipeline.py
      run_sector_rank_report.py
      run_sector_signal_backtest.py
    tests/
      test_monthly_return_calculator.py
      test_rank_table_builder.py
      test_leadership_score_engine.py
      test_rotation_signal_engine.py
      test_leader_stock_selector.py
  data/
    raw/
    processed/
  reports/
    sector_rank/
  README.md
```

---

## 5. 데이터 모델 설계

### 5.1 sector_master

```sql
CREATE TABLE IF NOT EXISTS sector_master (
    sector_code       TEXT PRIMARY KEY,
    sector_name       TEXT NOT NULL,
    market            TEXT NOT NULL DEFAULT 'KOSPI',
    sector_level      INTEGER NOT NULL DEFAULT 1,
    parent_sector_code TEXT,
    theme_group       TEXT,
    value_chain_group TEXT,
    display_color     TEXT NOT NULL DEFAULT 'FFFFFF',  -- 6자리 hex, '#' 없이
    is_active         INTEGER NOT NULL DEFAULT 1,
    created_at        TEXT NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at        TEXT NOT NULL DEFAULT CURRENT_TIMESTAMP
);
```

### 5.2 sector_monthly_price

```sql
CREATE TABLE IF NOT EXISTS sector_monthly_price (
    trade_month   TEXT NOT NULL,   -- YYYY-MM
    sector_code   TEXT NOT NULL,
    sector_name   TEXT NOT NULL,
    close_price   REAL NOT NULL,
    trading_value REAL,            -- 월간 거래대금 (KRX에서 제공하는 경우)
    market_cap    REAL,
    source        TEXT NOT NULL,
    collected_at  TEXT NOT NULL DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (trade_month, sector_code)
);

CREATE INDEX IF NOT EXISTS idx_sector_monthly_price_month
ON sector_monthly_price(trade_month);
```

### 5.3 sector_monthly_return

```sql
CREATE TABLE IF NOT EXISTS sector_monthly_return (
    trade_month            TEXT NOT NULL,
    sector_code            TEXT NOT NULL,
    sector_name            TEXT NOT NULL,
    monthly_return         REAL NOT NULL,
    rank_no                INTEGER NOT NULL,
    prev_rank_no           INTEGER,
    rank_change            INTEGER,   -- prev_rank_no - rank_no. 양수면 순위 상승
    trading_value          REAL,      -- 해당 월 거래대금 (price에서 직접 가져옴)
    prev_trading_value     REAL,      -- 전월 거래대금 (volume_score 계산용)
    return_zscore          REAL,
    market_relative_return REAL,
    created_at             TEXT NOT NULL DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (trade_month, sector_code)
);

CREATE INDEX IF NOT EXISTS idx_sector_monthly_return_rank
ON sector_monthly_return(trade_month, rank_no);
```

`trading_value`와 `prev_trading_value`를 returns 테이블에 포함시켜 LeadershipScoreEngine이 단일 DataFrame에서 점수를 계산할 수 있도록 한다.

### 5.4 sector_rank_snapshot

```sql
CREATE TABLE IF NOT EXISTS sector_rank_snapshot (
    trade_month   TEXT NOT NULL,
    rank_no       INTEGER NOT NULL,
    sector_code   TEXT NOT NULL,
    sector_name   TEXT NOT NULL,
    monthly_return REAL NOT NULL,
    theme_group   TEXT,
    display_color TEXT,            -- sector_master에서 복사 (조회 편의)
    PRIMARY KEY (trade_month, rank_no)
);
```

### 5.5 sector_leadership_score

```sql
CREATE TABLE IF NOT EXISTS sector_leadership_score (
    trade_month         TEXT NOT NULL,
    sector_code         TEXT NOT NULL,
    sector_name         TEXT NOT NULL,
    rank_score          REAL NOT NULL DEFAULT 0,
    persistence_score   REAL NOT NULL DEFAULT 0,
    momentum_score      REAL NOT NULL DEFAULT 0,
    expansion_score     REAL NOT NULL DEFAULT 0,
    volume_score        REAL NOT NULL DEFAULT 0,
    risk_penalty        REAL NOT NULL DEFAULT 0,
    total_score         REAL NOT NULL DEFAULT 0,
    top10_in_3m         INTEGER NOT NULL DEFAULT 0,
    top10_in_6m         INTEGER NOT NULL DEFAULT 0,
    consecutive_top10   INTEGER NOT NULL DEFAULT 0,
    score_reason        TEXT,
    created_at          TEXT NOT NULL DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (trade_month, sector_code)
);
```

### 5.6 sector_rotation_signal

```sql
CREATE TABLE IF NOT EXISTS sector_rotation_signal (
    trade_month              TEXT NOT NULL,
    sector_code              TEXT NOT NULL,
    sector_name              TEXT NOT NULL,
    signal                   TEXT NOT NULL,  -- NEW_LEADER, LEADER, EXTENDED, PEAK_WARNING, ROTATION_OUT, WATCH
    signal_strength          INTEGER NOT NULL,
    total_score              REAL NOT NULL,
    rank_no                  INTEGER NOT NULL,
    rank_change              INTEGER,
    top10_in_3m              INTEGER NOT NULL DEFAULT 0,
    top10_in_6m              INTEGER NOT NULL DEFAULT 0,
    consecutive_top10        INTEGER NOT NULL DEFAULT 0,
    rank_declining_2m        INTEGER NOT NULL DEFAULT 0,
    reason                   TEXT,
    created_at               TEXT NOT NULL DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (trade_month, sector_code, signal)
);
```

### 5.7 sector_leader_stocks

```sql
CREATE TABLE IF NOT EXISTS sector_leader_stocks (
    trade_month           TEXT NOT NULL,
    sector_code           TEXT NOT NULL,
    stock_code            TEXT NOT NULL,
    stock_name            TEXT NOT NULL,
    close_price           REAL,
    monthly_return        REAL,
    trading_value_20d     REAL,
    market_cap            REAL,
    foreign_netbuy_20d    REAL,
    institution_netbuy_20d REAL,
    above_20w_ma          INTEGER,
    distance_from_20w_ma  REAL,
    breakout_flag         INTEGER,
    pullback_flag         INTEGER,
    leader_score          REAL,
    reason                TEXT,
    PRIMARY KEY (trade_month, sector_code, stock_code)
);
```

---

## 6. 설정 파일 설계

### 6.1 sector_config.yaml

```yaml
sector_universe:
  source: krx_kospi         # 고정. 변경 시 설계서 개정 필요
  market: KOSPI
  top_n: 21                 # KRX KOSPI 업종 21개에 맞춤 (1003~1023)
  min_months_required: 2    # 최소 2개월 이상 데이터가 있어야 순위에 포함

data_collection:
  pykrx_timeout_seconds: 30
  fallback_to_krx_direct: true

rank_table:
  output_formats:
    - xlsx
    - html
    - md

leadership_score:
  top3_score: 30
  top10_score: 20
  top20_score: 10
  rank_jump_threshold: 5      # 21개 체계에서 15계단은 거의 전체 이동 → 5계단으로 조정
  rank_jump_score: 20
  top10_in_3m_2_score: 20    # 최근 3개월 중 2개월 top10
  top10_in_3m_3_score: 30    # 최근 3개월 중 3개월 top10
  expansion_min_related_count: 2   # 21개 체계에서 3개는 과도 → 2개로 조정
  expansion_score: 20
  volume_growth_threshold: 0.3     # 21개 전통 섹터 기준 30% 증가
  volume_score: 20
  overheat_top10_months: 4
  overheat_penalty: -10
  # rolling window 크기 — 아래 signals 섹션의 lookback_months와 반드시 일치해야 한다
  leader_lookback_months: 3      # LeadershipScoreEngine top10_in_3m 윈도우
  extended_lookback_months: 6    # LeadershipScoreEngine top10_in_6m 윈도우

signals:
  new_leader:
    max_rank: 10
    min_rank_change: 5             # top_n=21 기준 5계단 이상 급등
    require_recent_best_rank: true # 최근 lookback_months 내 현재가 최고 순위여야 함
    lookback_months: 3             # SignalConfig.new_leader_lookback_months 에 반영
  leader:
    # rolling 윈도우 크기(top10_in_3m)는 leadership_score.leader_lookback_months 가 결정한다.
    # SignalConfig 에는 윈도우 크기 필드가 없다 — 임계값만 보유.
    min_top10_count: 2             # top10_in_3m >= 2  →  SignalConfig.leader_min_top10_count
    min_total_score: 50            # SignalConfig.leader_min_total_score
  extended:
    # rolling 윈도우 크기(top10_in_6m)는 leadership_score.extended_lookback_months 가 결정한다.
    # SignalConfig 에는 윈도우 크기 필드가 없다 — 임계값만 보유.
    min_top10_count: 4             # top10_in_6m >= 4  →  SignalConfig.extended_min_top10_count (LEADER보다 먼저 판정)
  peak_warning:
    previous_top_rank: 5
    current_rank_worse_than: 15   # top_n=21 기준 15위 이하로 이탈
  rotation_out:
    previous_top_rank: 10
    current_rank_worse_than: 18   # top_n=21 기준 18위 이하로 이탈
    require_consecutive_decline: 2  # SignalConfig.rotation_out_consecutive_decline에 반영
```

### 6.2 sector_theme_map.yaml

테마 그룹별 색상은 sector_master.display_color가 단일 소스다. 이 파일은 sector_master 초기화 시 참조하는 설정이며, 런타임에서 직접 색상 결정에 사용하지 않는다.

```yaml
반도체:
  display_color: "C6EFCE"
  sector_codes: ["1011"]          # 전기전자

자동차:
  display_color: "FFC7CE"
  sector_codes: ["1013"]          # 운수장비

바이오:
  display_color: "FFEB9C"
  sector_codes: ["1007", "1012"]  # 의약품, 의료정밀

유틸리티:
  display_color: "D9EAD3"
  sector_codes: ["1015"]          # 전기가스업

소재:
  display_color: "FFFFFF"
  sector_codes: ["1005", "1006", "1008", "1009"]

소비재:
  display_color: "FFFFFF"
  sector_codes: ["1003", "1004", "1014"]

금융:
  display_color: "FFFFFF"
  sector_codes: ["1019", "1020", "1021", "1022"]
```

### 6.3 settings.py — YAML 로더 설계

`sector_config.yaml` 값을 `LeadershipScoreConfig`, `SignalConfig` 객체로 변환하는 단일 진입점이다. CLI와 테스트 모두 이 함수를 통해 설정을 로드한다.

```python
# app/config/settings.py
from __future__ import annotations

from pathlib import Path

import yaml

from ..services.leadership_score_engine import LeadershipScoreConfig
from ..services.rotation_signal_engine import SignalConfig

DEFAULT_CONFIG_PATH = Path(__file__).parent / "sector_config.yaml"


def load_config(config_path: str | Path | None = None) -> dict:
    """YAML 파일을 읽어 dict로 반환한다."""
    path = Path(config_path) if config_path else DEFAULT_CONFIG_PATH
    with path.open("r", encoding="utf-8") as fh:
        return yaml.safe_load(fh)


def build_leadership_score_config(cfg: dict) -> LeadershipScoreConfig:
    """cfg["leadership_score"] 섹션 → LeadershipScoreConfig 객체."""
    ls = cfg.get("leadership_score", {})
    return LeadershipScoreConfig(
        top3_score=ls.get("top3_score", 30),
        top10_score=ls.get("top10_score", 20),
        top20_score=ls.get("top20_score", 10),
        rank_jump_threshold=ls.get("rank_jump_threshold", 5),
        rank_jump_score=ls.get("rank_jump_score", 20),
        top10_in_3m_2_score=ls.get("top10_in_3m_2_score", 20),
        top10_in_3m_3_score=ls.get("top10_in_3m_3_score", 30),
        expansion_min_related_count=ls.get("expansion_min_related_count", 2),
        expansion_score=ls.get("expansion_score", 20),
        volume_growth_threshold=ls.get("volume_growth_threshold", 0.3),
        volume_score=ls.get("volume_score", 20),
        overheat_top10_months=ls.get("overheat_top10_months", 4),
        overheat_penalty=ls.get("overheat_penalty", -10),
        leader_lookback_months=ls.get("leader_lookback_months", 3),
        extended_lookback_months=ls.get("extended_lookback_months", 6),
    )


def build_signal_config(cfg: dict) -> SignalConfig:
    """cfg["signals"] 섹션 → SignalConfig 객체."""
    sig = cfg.get("signals", {})
    nl  = sig.get("new_leader", {})
    ld  = sig.get("leader", {})
    ext = sig.get("extended", {})
    pw  = sig.get("peak_warning", {})
    ro  = sig.get("rotation_out", {})
    return SignalConfig(
        new_leader_max_rank=nl.get("max_rank", 10),
        new_leader_min_rank_change=nl.get("min_rank_change", 5),
        new_leader_require_recent_best=nl.get("require_recent_best_rank", True),
        new_leader_lookback_months=nl.get("lookback_months", 3),
        leader_min_top10_count=ld.get("min_top10_count", 2),
        leader_min_total_score=ld.get("min_total_score", 50.0),
        extended_min_top10_count=ext.get("min_top10_count", 4),
        peak_warning_previous_top_rank=pw.get("previous_top_rank", 5),
        peak_warning_current_worse_than=pw.get("current_rank_worse_than", 15),
        rotation_out_previous_top_rank=ro.get("previous_top_rank", 10),
        rotation_out_current_worse_than=ro.get("current_rank_worse_than", 18),
        rotation_out_consecutive_decline=ro.get("require_consecutive_decline", 2),
    )


def get_min_months_required(cfg: dict) -> int:
    """cfg["sector_universe"]["min_months_required"] 반환."""
    return cfg.get("sector_universe", {}).get("min_months_required", 2)
```

---

## 7. 핵심 알고리즘 설계

### 7.1 DB 연결 및 CRUD 인터페이스

모든 파이프라인이 공유하는 인터페이스 계약이다. 변경 시 이 섹션을 먼저 수정한다.

```python
# app/db/connection.py

import sqlite3
import os


def get_connection(db_path: str | None = None) -> sqlite3.Connection:
    path = db_path or os.getenv("MFRT_DB_PATH", "moneyflow.db")
    conn = sqlite3.connect(path)
    conn.row_factory = sqlite3.Row
    conn.execute("PRAGMA journal_mode=WAL")
    conn.execute("PRAGMA foreign_keys=ON")
    return conn
```

```python
# app/db/crud.py

import sqlite3
import pandas as pd
from typing import Optional


def load_sector_master(conn: sqlite3.Connection) -> pd.DataFrame:
    return pd.read_sql(
        "SELECT sector_code, sector_name, market, theme_group, value_chain_group, display_color "
        "FROM sector_master WHERE is_active = 1",
        conn,
    )


def upsert_sector_monthly_price(conn: sqlite3.Connection, df: pd.DataFrame) -> None:
    """df columns: trade_month, sector_code, sector_name, close_price, trading_value, market_cap, source"""
    rows = df.to_dict(orient="records")
    conn.executemany(
        """
        INSERT INTO sector_monthly_price
            (trade_month, sector_code, sector_name, close_price, trading_value, market_cap, source)
        VALUES (:trade_month, :sector_code, :sector_name, :close_price, :trading_value, :market_cap, :source)
        ON CONFLICT(trade_month, sector_code) DO UPDATE SET
            close_price   = excluded.close_price,
            trading_value = excluded.trading_value,
            market_cap    = excluded.market_cap,
            source        = excluded.source
        """,
        rows,
    )
    conn.commit()


def load_all_monthly_prices(conn: sqlite3.Connection) -> pd.DataFrame:
    return pd.read_sql(
        "SELECT trade_month, sector_code, sector_name, close_price, trading_value "
        "FROM sector_monthly_price ORDER BY sector_code, trade_month",
        conn,
    )


def save_sector_monthly_return(conn: sqlite3.Connection, df: pd.DataFrame) -> None:
    """df columns: sector_monthly_return 테이블 컬럼과 동일"""
    rows = df.where(df.notna(), other=None).to_dict(orient="records")
    conn.executemany(
        """
        INSERT INTO sector_monthly_return
            (trade_month, sector_code, sector_name, monthly_return, rank_no,
             prev_rank_no, rank_change, trading_value, prev_trading_value)
        VALUES (:trade_month, :sector_code, :sector_name, :monthly_return, :rank_no,
                :prev_rank_no, :rank_change, :trading_value, :prev_trading_value)
        ON CONFLICT(trade_month, sector_code) DO UPDATE SET
            monthly_return     = excluded.monthly_return,
            rank_no            = excluded.rank_no,
            prev_rank_no       = excluded.prev_rank_no,
            rank_change        = excluded.rank_change,
            trading_value      = excluded.trading_value,
            prev_trading_value = excluded.prev_trading_value
        """,
        rows,
    )
    conn.commit()


def save_sector_rank_snapshot(conn: sqlite3.Connection, df: pd.DataFrame) -> None:
    rows = df.where(df.notna(), other=None).to_dict(orient="records")
    conn.executemany(
        """
        INSERT INTO sector_rank_snapshot
            (trade_month, rank_no, sector_code, sector_name, monthly_return, theme_group, display_color)
        VALUES (:trade_month, :rank_no, :sector_code, :sector_name, :monthly_return, :theme_group, :display_color)
        ON CONFLICT(trade_month, rank_no) DO UPDATE SET
            sector_code   = excluded.sector_code,
            sector_name   = excluded.sector_name,
            monthly_return = excluded.monthly_return,
            theme_group   = excluded.theme_group,
            display_color = excluded.display_color
        """,
        rows,
    )
    conn.commit()


def save_sector_leadership_score(conn: sqlite3.Connection, df: pd.DataFrame) -> None:
    rows = df.where(df.notna(), other=None).to_dict(orient="records")
    conn.executemany(
        """
        INSERT INTO sector_leadership_score
            (trade_month, sector_code, sector_name, rank_score, persistence_score,
             momentum_score, expansion_score, volume_score, risk_penalty, total_score,
             top10_in_3m, top10_in_6m, consecutive_top10, score_reason)
        VALUES (:trade_month, :sector_code, :sector_name, :rank_score, :persistence_score,
                :momentum_score, :expansion_score, :volume_score, :risk_penalty, :total_score,
                :top10_in_3m, :top10_in_6m, :consecutive_top10, :score_reason)
        ON CONFLICT(trade_month, sector_code) DO UPDATE SET
            total_score      = excluded.total_score,
            rank_score       = excluded.rank_score,
            persistence_score = excluded.persistence_score,
            momentum_score   = excluded.momentum_score,
            expansion_score  = excluded.expansion_score,
            volume_score     = excluded.volume_score,
            risk_penalty     = excluded.risk_penalty,
            top10_in_3m      = excluded.top10_in_3m,
            top10_in_6m      = excluded.top10_in_6m,
            consecutive_top10 = excluded.consecutive_top10,
            score_reason     = excluded.score_reason
        """,
        rows,
    )
    conn.commit()


def save_sector_rotation_signal(conn: sqlite3.Connection, df: pd.DataFrame) -> None:
    rows = df.where(df.notna(), other=None).to_dict(orient="records")
    conn.executemany(
        """
        INSERT INTO sector_rotation_signal
            (trade_month, sector_code, sector_name, signal, signal_strength, total_score,
             rank_no, rank_change, top10_in_3m, top10_in_6m, consecutive_top10,
             rank_declining_2m, reason)
        VALUES (:trade_month, :sector_code, :sector_name, :signal, :signal_strength, :total_score,
                :rank_no, :rank_change, :top10_in_3m, :top10_in_6m, :consecutive_top10,
                :rank_declining_2m, :reason)
        ON CONFLICT(trade_month, sector_code, signal) DO UPDATE SET
            signal_strength   = excluded.signal_strength,
            total_score       = excluded.total_score,
            rank_no           = excluded.rank_no,
            rank_change       = excluded.rank_change,
            top10_in_3m       = excluded.top10_in_3m,
            top10_in_6m       = excluded.top10_in_6m,
            consecutive_top10 = excluded.consecutive_top10,
            rank_declining_2m = excluded.rank_declining_2m,
            reason            = excluded.reason
        """,
        rows,
    )
    conn.commit()
```

### 7.2 월간 수익률 계산 (MonthlyReturnCalculator)

**핵심 수정: 인접 월 검증**
`shift(1)`은 "직전 관측치"를 가져오므로 데이터 공백 시 누적 수익률이 단월 수익률로 잘못 계산된다. 반드시 직전 달이 실제로 1개월 전인지 확인한다.

```python
# app/services/monthly_return_calculator.py

from __future__ import annotations

from dataclasses import dataclass
from typing import Optional

import numpy as np
import pandas as pd


@dataclass(frozen=True)
class MonthlyReturnRecord:
    trade_month:        str
    sector_code:        str
    sector_name:        str
    monthly_return:     float
    rank_no:            int
    prev_rank_no:       Optional[int]
    rank_change:        Optional[int]
    trading_value:      Optional[float]
    prev_trading_value: Optional[float]


class MonthlyReturnCalculator:
    def calculate(
        self,
        price_df: pd.DataFrame,
        min_months_required: int = 2,
    ) -> pd.DataFrame:
        """
        input columns:
            trade_month, sector_code, sector_name, close_price
            (optional) trading_value

        output columns:
            trade_month, sector_code, sector_name, monthly_return,
            rank_no, prev_rank_no, rank_change,
            trading_value, prev_trading_value

        인접 월 규칙:
            직전 관측치가 정확히 1개월 전(YYYY-MM 기준)인 경우에만 수익률을 계산한다.
            공백이 있으면 NaN → 해당 행 제외.

        min_months_required:
            각 섹터의 가격 데이터 월 수가 이 값 이상이어야 순위에 포함된다.
            수익률 계산 개수가 아닌 가격 관측 개수 기준이다.
            예: Jan+Feb 가격 2건 → min_months_required=2 → Feb 수익률 1건이지만 포함됨.
            수익률 개수로 필터링하면 첫 달 수익률이 항상 제외되는 문제가 발생한다.
        """
        required = {"trade_month", "sector_code", "sector_name", "close_price"}
        missing = required - set(price_df.columns)
        if missing:
            raise ValueError(f"missing columns: {missing}")

        # min_months_required 필터: 가격 데이터 월 수 기준, 수익률 계산 전에 적용
        price_month_counts = price_df.groupby("sector_code")["trade_month"].nunique()
        valid_sectors = price_month_counts[price_month_counts >= min_months_required].index

        df = price_df[price_df["sector_code"].isin(valid_sectors)].copy()
        df = df.sort_values(["sector_code", "trade_month"])
        df["trade_month_dt"] = pd.to_datetime(df["trade_month"], format="%Y-%m")

        # 전월 close 및 거래대금
        df["prev_close"] = df.groupby("sector_code")["close_price"].shift(1)
        df["prev_month_dt"] = df.groupby("sector_code")["trade_month_dt"].shift(1)
        if "trading_value" in df.columns:
            df["prev_trading_value"] = df.groupby("sector_code")["trading_value"].shift(1)
        else:
            df["trading_value"] = np.nan
            df["prev_trading_value"] = np.nan

        # 인접 월 검증: 전월 레코드가 정확히 1개월 전이어야 함
        def _expected_prev(dt: pd.Timestamp) -> pd.Timestamp:
            if dt.month == 1:
                return pd.Timestamp(year=dt.year - 1, month=12, day=1)
            return pd.Timestamp(year=dt.year, month=dt.month - 1, day=1)

        mask_adjacent = df.apply(
            lambda row: (
                pd.notna(row["prev_month_dt"])
                and row["prev_month_dt"].year == _expected_prev(row["trade_month_dt"]).year
                and row["prev_month_dt"].month == _expected_prev(row["trade_month_dt"]).month
            ),
            axis=1,
        )

        df["monthly_return"] = np.where(
            mask_adjacent,
            (df["close_price"] / df["prev_close"] - 1.0) * 100.0,
            np.nan,
        )
        df = df.dropna(subset=["monthly_return"])

        # 월별 순위
        df["rank_no"] = (
            df.groupby("trade_month")["monthly_return"]
            .rank(method="first", ascending=False)
            .astype(int)
        )

        # 전월 순위 및 순위 변화
        df = df.sort_values(["sector_code", "trade_month"])
        df["prev_rank_no"] = df.groupby("sector_code")["rank_no"].shift(1)
        df["rank_change"] = df["prev_rank_no"] - df["rank_no"]

        return df[[
            "trade_month", "sector_code", "sector_name",
            "monthly_return", "rank_no", "prev_rank_no", "rank_change",
            "trading_value", "prev_trading_value",
        ]].reset_index(drop=True)
```

### 7.3 Rank Table 생성 (RankTableBuilder)

```python
# app/services/rank_table_builder.py

import pandas as pd


class RankTableBuilder:
    def build_rank_table(self, returns_df: pd.DataFrame, top_n: int = 21) -> pd.DataFrame:
        """
        월별 수익률 순위를 Rank Table 구조로 변환한다.

        output:
            index: 1위~N위
            columns: YYYY-MM
            values: sector_name
        """
        df = returns_df[returns_df["rank_no"] <= top_n].copy()
        df = df.sort_values(["trade_month", "rank_no"])

        rank_map = {}
        for trade_month, month_df in df.groupby("trade_month"):
            values = month_df.sort_values("rank_no")["sector_name"].tolist()
            rank_map[trade_month] = values

        rank_table = pd.DataFrame(rank_map)
        rank_table.index = [f"{i}위" for i in range(1, len(rank_table) + 1)]
        return rank_table

    def build_snapshot(
        self,
        returns_df: pd.DataFrame,
        sector_master_df: pd.DataFrame,
    ) -> pd.DataFrame:
        """
        sector_master의 theme_group과 display_color를 결합해 snapshot을 생성한다.
        display_color는 이후 Excel 출력의 유일한 색상 소스다.
        """
        df = returns_df.merge(
            sector_master_df[["sector_code", "theme_group", "display_color"]],
            on="sector_code",
            how="left",
        )
        return df[[
            "trade_month", "rank_no", "sector_code", "sector_name",
            "monthly_return", "theme_group", "display_color",
        ]].sort_values(["trade_month", "rank_no"]).reset_index(drop=True)

    def build_color_lookup(self, snapshot_df: pd.DataFrame) -> dict[str, str]:
        """sector_name → display_color 조회 테이블 반환."""
        return (
            snapshot_df.drop_duplicates("sector_name")
            .set_index("sector_name")["display_color"]
            .fillna("FFFFFF")
            .to_dict()
        )
```

### 7.4 주도섹터 점수 엔진 (LeadershipScoreEngine)

**핵심 수정: trading_value가 returns_df에 포함돼 있어 별도 인자 없이 volume_score를 계산한다.**

```python
# app/services/leadership_score_engine.py

from __future__ import annotations

from dataclasses import dataclass

import pandas as pd


@dataclass(frozen=True)
class LeadershipScoreConfig:
    top3_score:                 int   = 30
    top10_score:                int   = 20
    top20_score:                int   = 10
    rank_jump_threshold:        int   = 5
    rank_jump_score:            int   = 20
    top10_in_3m_2_score:        int   = 20   # top10_in_Xm (leader_lookback_months 기준)
    top10_in_3m_3_score:        int   = 30   # top10_in_Xm 최대 충족 시
    expansion_min_related_count: int  = 2
    expansion_score:            int   = 20
    volume_growth_threshold:    float = 0.3
    volume_score:               int   = 20
    overheat_top10_months:      int   = 4
    overheat_penalty:           int   = -10
    # rolling window 크기 — 단일 소유. RotationSignalEngine은 이미 계산된 컬럼값을 읽을 뿐
    leader_lookback_months:     int   = 3    # top10_in_3m 윈도우 크기
    extended_lookback_months:   int   = 6    # top10_in_6m 윈도우 크기


class LeadershipScoreEngine:
    def __init__(self, config: LeadershipScoreConfig):
        self.config = config

    def score(
        self,
        returns_df: pd.DataFrame,
        sector_master_df: pd.DataFrame,
    ) -> pd.DataFrame:
        """
        returns_df는 MonthlyReturnCalculator.calculate()의 출력이다.
        trading_value, prev_trading_value 컬럼이 포함돼 있어야 한다.

        추가되는 컬럼:
            rank_score, momentum_score, persistence_score, expansion_score,
            volume_score, risk_penalty, total_score, score_reason,
            top10_in_3m, top10_in_6m, consecutive_top10
        """
        df = returns_df.merge(
            sector_master_df[["sector_code", "theme_group", "value_chain_group"]],
            on="sector_code",
            how="left",
        )
        df = df.sort_values(["sector_code", "trade_month"]).reset_index(drop=True)

        # 윈도우 카운터 (config 값 사용 — 하드코딩 금지)
        lw = self.config.leader_lookback_months    # 기본 3
        ew = self.config.extended_lookback_months  # 기본 6
        df["_is_top10"] = (df["rank_no"] <= 10).astype(int)
        df["top10_in_3m"] = (
            df.groupby("sector_code")["_is_top10"]
            .transform(lambda s: s.rolling(lw, min_periods=1).sum())
            .astype(int)
        )
        df["top10_in_6m"] = (
            df.groupby("sector_code")["_is_top10"]
            .transform(lambda s: s.rolling(ew, min_periods=1).sum())
            .astype(int)
        )
        df["consecutive_top10"] = self._calc_consecutive_top10(df)
        df = df.drop(columns=["_is_top10"])

        df["rank_score"]      = df["rank_no"].apply(self._rank_score)
        df["momentum_score"]  = df["rank_change"].apply(self._momentum_score)
        df["persistence_score"] = df["top10_in_3m"].apply(self._persistence_score)
        df["expansion_score"] = self._calc_expansion_score(df, sector_master_df)
        df["volume_score"]    = self._calc_volume_score(df)
        df["risk_penalty"]    = df["consecutive_top10"].apply(self._risk_penalty)

        df["total_score"] = (
            df["rank_score"]
            + df["momentum_score"]
            + df["persistence_score"]
            + df["expansion_score"]
            + df["volume_score"]
            + df["risk_penalty"]
        )
        df["score_reason"] = df.apply(self._make_reason, axis=1)
        return df

    def _rank_score(self, rank_no: int) -> int:
        if rank_no <= 3:
            return self.config.top3_score
        if rank_no <= 10:
            return self.config.top10_score
        if rank_no <= 20:
            return self.config.top20_score
        return 0

    def _momentum_score(self, rank_change) -> int:
        if pd.isna(rank_change):
            return 0
        return self.config.rank_jump_score if rank_change >= self.config.rank_jump_threshold else 0

    def _persistence_score(self, top10_in_3m: int) -> int:
        if top10_in_3m >= 3:
            return self.config.top10_in_3m_3_score
        if top10_in_3m >= 2:
            return self.config.top10_in_3m_2_score
        return 0

    def _risk_penalty(self, consecutive_top10: int) -> int:
        if consecutive_top10 >= self.config.overheat_top10_months:
            return self.config.overheat_penalty
        return 0

    def _calc_consecutive_top10(self, df: pd.DataFrame) -> pd.Series:
        result = []
        for _, group in df.groupby("sector_code", sort=False):
            count = 0
            for _, row in group.iterrows():
                count = count + 1 if row["rank_no"] <= 10 else 0
                result.append(count)
        return pd.Series(result, index=df.index)

    def _calc_expansion_score(
        self,
        df: pd.DataFrame,
        sector_master_df: pd.DataFrame,
    ) -> pd.Series:
        scores = pd.Series(0, index=df.index)
        for trade_month, month_df in df.groupby("trade_month"):
            top20 = month_df[month_df["rank_no"] <= 20]
            theme_counts = top20.groupby("theme_group")["sector_code"].nunique()
            strong_themes = set(
                theme_counts[theme_counts >= self.config.expansion_min_related_count].index
            )
            idx = df[
                (df["trade_month"] == trade_month)
                & (df["theme_group"].isin(strong_themes))
            ].index
            scores.loc[idx] = self.config.expansion_score
        return scores

    def _calc_volume_score(self, df: pd.DataFrame) -> pd.Series:
        """
        trading_value와 prev_trading_value는 returns_df에서 넘어온다.
        이 컬럼이 없거나 모두 NaN이면 volume_score=0으로 처리하고 오류로 보지 않는다.
        """
        if "trading_value" not in df.columns or "prev_trading_value" not in df.columns:
            return pd.Series(0, index=df.index)
        tv = pd.to_numeric(df["trading_value"], errors="coerce")
        ptv = pd.to_numeric(df["prev_trading_value"], errors="coerce")
        growth = (tv / ptv - 1.0).where(ptv > 0)
        return growth.apply(
            lambda x: self.config.volume_score
            if pd.notna(x) and x >= self.config.volume_growth_threshold
            else 0
        )

    def _make_reason(self, row) -> str:
        reasons = []
        if row["rank_score"] > 0:
            reasons.append(f"상위 순위 {row['rank_no']}위")
        if row["momentum_score"] > 0:
            reasons.append(f"순위 급상승 {int(row['rank_change'])}계단")
        if row["persistence_score"] > 0:
            reasons.append(f"Top10 최근3개월 {int(row['top10_in_3m'])}회")
        if row["expansion_score"] > 0:
            reasons.append("관련 섹터 동반 상위권")
        if row["volume_score"] > 0:
            reasons.append("거래대금 급증")
        if row["risk_penalty"] < 0:
            reasons.append("장기 상위권 과열 패널티")
        return "; ".join(reasons)
```

### 7.5 Rotation Signal 엔진 (RotationSignalEngine)

**핵심 수정:**
1. EXTENDED 판정을 LEADER보다 먼저 수행 (신호 우선순위 재정렬)
2. `top10_in_3m`, `top10_in_6m` 윈도우 카운터 사용 (consecutive 오용 제거)
3. `require_recent_best_rank` 구현 (NEW_LEADER 조건)
4. ROTATION_OUT에 2개월 연속 순위 하락 조건 추가
5. 임계값을 top_n=21 체계에 맞게 조정

```python
# app/services/rotation_signal_engine.py

from __future__ import annotations

from dataclasses import dataclass, field

import numpy as np
import pandas as pd


@dataclass(frozen=True)
class SignalConfig:
    # NEW_LEADER 판정 파라미터
    new_leader_max_rank:               int = 10
    new_leader_min_rank_change:        int = 5
    new_leader_require_recent_best:    bool = True
    new_leader_lookback_months:        int = 3    # 최근 최고 순위 확인 윈도우

    # LEADER 판정 파라미터
    # top10_in_3m 윈도우 크기는 LeadershipScoreConfig.leader_lookback_months 가 결정한다.
    # SignalConfig 는 그 결과값에 적용할 임계값만 보유한다.
    leader_min_top10_count:            int = 2
    leader_min_total_score:            float = 50.0

    # EXTENDED 판정 파라미터
    # top10_in_6m 윈도우 크기는 LeadershipScoreConfig.extended_lookback_months 가 결정한다.
    extended_min_top10_count:          int = 4

    # PEAK_WARNING 판정 파라미터
    peak_warning_previous_top_rank:    int = 5
    peak_warning_current_worse_than:   int = 15

    # ROTATION_OUT 판정 파라미터
    rotation_out_previous_top_rank:    int = 10
    rotation_out_current_worse_than:   int = 18
    rotation_out_consecutive_decline:  int = 2


class RotationSignalEngine:
    def __init__(self, config: SignalConfig | None = None):
        self.config = config or SignalConfig()

    def generate(self, scored_df: pd.DataFrame) -> pd.DataFrame:
        """
        scored_df: LeadershipScoreEngine.score() 출력.
        top10_in_3m, top10_in_6m, consecutive_top10 컬럼이 포함돼 있어야 한다.
        """
        df = scored_df.sort_values(["sector_code", "trade_month"]).copy()

        # 연속 순위 하락 플래그 — rotation_out_consecutive_decline 설정값 사용
        n = self.config.rotation_out_consecutive_decline  # 기본 2
        df["_rank_worse"] = (df.groupby("sector_code")["rank_no"].diff() > 0).astype(int)
        df["rank_declining_2m"] = (
            df.groupby("sector_code")["_rank_worse"]
            .transform(lambda s: s.rolling(n, min_periods=n).sum() >= n)
            .fillna(False)
            .astype(int)
        )
        df = df.drop(columns=["_rank_worse"])

        # 최근 N개월 최고 순위 여부 (NEW_LEADER 용)
        df["_rolling_best_rank"] = (
            df.groupby("sector_code")["rank_no"]
            .transform(
                lambda s: s.rolling(
                    self.config.new_leader_lookback_months, min_periods=1
                ).min()
            )
        )
        df["is_recent_best_rank"] = (df["rank_no"] == df["_rolling_best_rank"]).astype(int)
        df = df.drop(columns=["_rolling_best_rank"])

        rows = []
        for _, row in df.iterrows():
            signal = self._decide_signal(row)
            if signal is None:
                continue
            rows.append({
                "trade_month":       row["trade_month"],
                "sector_code":       row["sector_code"],
                "sector_name":       row["sector_name"],
                "signal":            signal,
                "signal_strength":   self._signal_strength(signal, row),
                "total_score":       row["total_score"],
                "rank_no":           row["rank_no"],
                "rank_change":       row.get("rank_change"),
                "top10_in_3m":       int(row.get("top10_in_3m", 0)),
                "top10_in_6m":       int(row.get("top10_in_6m", 0)),
                "consecutive_top10": int(row.get("consecutive_top10", 0)),
                "rank_declining_2m": int(row.get("rank_declining_2m", 0)),
                "reason":            row.get("score_reason", ""),
            })

        return pd.DataFrame(rows)

    def _decide_signal(self, row) -> str | None:
        cfg = self.config
        rank_no       = int(row["rank_no"])
        rank_change   = row.get("rank_change")
        prev_rank     = row.get("prev_rank_no")
        total_score   = float(row.get("total_score", 0))
        top10_in_3m   = int(row.get("top10_in_3m", 0))
        top10_in_6m   = int(row.get("top10_in_6m", 0))
        declining_2m  = bool(row.get("rank_declining_2m", 0))
        recent_best   = bool(row.get("is_recent_best_rank", 0))

        # ── 이탈 신호 (우선 판정) ──────────────────────────────────
        if pd.notna(prev_rank) and int(prev_rank) <= cfg.peak_warning_previous_top_rank:
            if rank_no > cfg.peak_warning_current_worse_than:
                return "PEAK_WARNING"

        if pd.notna(prev_rank) and int(prev_rank) <= cfg.rotation_out_previous_top_rank:
            if rank_no > cfg.rotation_out_current_worse_than and declining_2m:
                return "ROTATION_OUT"

        # ── 지속 신호: EXTENDED → LEADER 순서로 판정 ─────────────
        # EXTENDED가 LEADER보다 먼저 와야 consecutive 4개월이 LEADER에 흡수되지 않는다.
        if rank_no <= 10 and top10_in_6m >= cfg.extended_min_top10_count:
            return "EXTENDED"

        if rank_no <= 10 and top10_in_3m >= cfg.leader_min_top10_count and total_score >= cfg.leader_min_total_score:
            return "LEADER"

        # ── 신규 진입 신호 ────────────────────────────────────────
        if rank_no <= cfg.new_leader_max_rank and pd.notna(rank_change) and rank_change >= cfg.new_leader_min_rank_change:
            if not cfg.new_leader_require_recent_best or recent_best:
                return "NEW_LEADER"

        if rank_no <= 20 and total_score >= 30:
            return "WATCH"

        return None

    def _signal_strength(self, signal: str, row) -> int:
        base = {
            "NEW_LEADER":   3,
            "LEADER":       4,
            "EXTENDED":     3,
            "PEAK_WARNING": 4,
            "ROTATION_OUT": 5,
            "WATCH":        2,
        }[signal]
        score = float(row.get("total_score", 0))
        if score >= 80:
            return min(5, base + 1)
        if score < 30:
            return max(1, base - 1)
        return base
```

### 7.6 대장주 후보 선별 엔진 (LeaderStockSelector)

```python
# app/services/leader_stock_selector.py

import pandas as pd


class LeaderStockSelector:
    def select(
        self,
        stock_df: pd.DataFrame,
        sector_signals_df: pd.DataFrame,
    ) -> pd.DataFrame:
        """
        stock_df columns:
            trade_month, sector_code, stock_code, stock_name,
            monthly_return, trading_value_20d, market_cap,
            foreign_netbuy_20d, institution_netbuy_20d,
            close_price, ma20w, high_52w, pullback_pct
        """
        active_sectors = sector_signals_df[
            sector_signals_df["signal"].isin(["NEW_LEADER", "LEADER", "WATCH"])
        ][["trade_month", "sector_code", "signal"]]

        df = stock_df.merge(active_sectors, on=["trade_month", "sector_code"], how="inner")

        df["above_20w_ma"]         = (df["close_price"] > df["ma20w"]).astype(int)
        df["distance_from_20w_ma"] = df["close_price"] / df["ma20w"] - 1.0
        df["breakout_flag"]        = (df["close_price"] >= df["high_52w"] * 0.97).astype(int)
        df["pullback_flag"]        = df["pullback_pct"].between(-0.25, -0.05).astype(int)

        df["leader_score"] = 0.0
        df.loc[df["monthly_return"] > 0,              "leader_score"] += 20
        df.loc[df["trading_value_20d"] >= 50_000_000_000, "leader_score"] += 20
        df.loc[df["above_20w_ma"] == 1,               "leader_score"] += 20
        df.loc[df["foreign_netbuy_20d"] > 0,          "leader_score"] += 10
        df.loc[df["institution_netbuy_20d"] > 0,      "leader_score"] += 10
        df.loc[df["breakout_flag"] == 1,              "leader_score"] += 10
        df.loc[df["pullback_flag"] == 1,              "leader_score"] += 10

        result = df[df["leader_score"] >= 50].copy()
        result = result.sort_values(
            ["trade_month", "sector_code", "leader_score"],
            ascending=[True, True, False],
        )
        return result
```

---

## 8. 수집 계층 (SectorPriceCollector)

### 8.1 fallback 수집 전략

```text
Path 1 pykrx 월 범위 쿼리          ← 구현 완료, 검증됨
  get_index_ohlcv_by_date(from_date, to_date, sector_code) → 마지막 행 추출

Path 2 pykrx 일별 역순 시도        ← 구현 완료, 검증됨
  월말 날짜부터 최대 15일 역순으로 하루씩 단일 날짜 쿼리 시도
  (휴장일이나 pykrx 범위 쿼리 버그 회피용)

Path 3 KRX JSON HTTP direct        ← 설계 계약만 확정. 구현 전 3개 파라미터 확정 필요
  data.krx.co.kr JSON 엔드포인트 직접 호출
  sugup-report krx_direct.py 패턴과 동일한 session bootstrap 사용

  [구현 착수 전 KRX 포털(data.krx.co.kr)에서 반드시 확인해야 할 항목]
  (1) bld 코드 : KRX_INDEX_BLD 상수. MDCSTAT00301 계열로 추정되나 미확정
  (2) idxIndMidclssCd : KOSPI 업종 분류 파라미터 값. "02" 추정, 미확정
  (3) 응답 필드명 : IDX_IND_CD / CLSPRC_IDX / ACC_TRDVAL 은 예측값.
                   실제 JSON 키는 KRX 포털 Network 탭에서 확인 후 확정
```

세 경로가 모두 실패하면 `(None, None, "")` 반환 → 해당 섹터 해당 월 수집 skip (로그 기록).
`_fetch_month_end()` 반환 타입: `tuple[float | None, float | None, str]` — (close, trading_value, source)

### 8.2 설계 계약 (Path 3 파라미터 미확정 — 구현 시 위 3개 항목 확정 후 반영)

```python
# app/collectors/sector_price_collector.py

from __future__ import annotations

import calendar
import logging
from datetime import datetime, timedelta

import pandas as pd
import requests
from pykrx import stock as pykrx_stock

from ..config.sector_universe import KRX_KOSPI_SECTORS

logger = logging.getLogger("moneyflow.sector_price")


class SectorPriceCollector:
    """
    KRX KOSPI 업종지수(1003~1023) 월말 종가를 수집한다.

    수집 전략 (3단계 fallback):
      Path 1. pykrx get_index_ohlcv_by_date (월 범위 쿼리)
      Path 2. pykrx get_index_ohlcv_by_date (일별 역순, 최대 15일)
      Path 3. KRX HTTP JSON direct 호출
    """

    KRX_HOME_URL  = "https://data.krx.co.kr/contents/MDC/MDI/outerLoader/index.cmd"
    KRX_JSON_URL  = "https://data.krx.co.kr/comm/bldAttendant/getJsonData.cmd"
    # TODO(구현 착수 전): KRX 포털 Network 탭에서 업종지수 bld 코드 확인 후 확정
    KRX_INDEX_BLD = "dbms/MDC/STAT/standard/MDCSTAT00301"  # 추정값 — 미확정

    SOURCE_PYKRX  = "krx_pykrx"
    SOURCE_DIRECT = "krx_direct"

    def collect_monthly_prices(self, trade_month: str) -> pd.DataFrame:
        """
        trade_month: YYYY-MM
        return: trade_month, sector_code, sector_name, close_price, trading_value, market_cap, source
        """
        year_int, month_int = int(trade_month[:4]), int(trade_month[5:7])
        from_date = f"{year_int}{month_int:02d}01"
        last_day  = calendar.monthrange(year_int, month_int)[1]
        to_date   = f"{year_int}{month_int:02d}{last_day:02d}"

        rows = []
        for sector in KRX_KOSPI_SECTORS:
            code = sector["sector_code"]
            name = sector["sector_name"]
            close_price, trading_value, source = self._fetch_month_end(code, from_date, to_date)
            if close_price is None:
                logger.warning("price_missing sector=%s month=%s", code, trade_month)
                continue
            rows.append({
                "trade_month":   trade_month,
                "sector_code":   code,
                "sector_name":   name,
                "close_price":   close_price,
                "trading_value": trading_value,
                "market_cap":    None,
                "source":        source,
            })
        return pd.DataFrame(rows)

    def _fetch_month_end(
        self, sector_code: str, from_date: str, to_date: str
    ) -> tuple[float | None, float | None, str]:
        """3단계 fallback으로 월말 종가를 반환한다. (close, trading_value, source)"""

        # Path 1: pykrx 월 범위 쿼리
        close, tvol = self._try_pykrx_range(sector_code, from_date, to_date)
        if close is not None:
            return close, tvol, self.SOURCE_PYKRX

        # Path 2: pykrx 일별 역순 (월말부터 최대 15일)
        close, tvol = self._try_pykrx_daily_fallback(sector_code, to_date)
        if close is not None:
            logger.debug("pykrx_daily_fallback ok sector=%s", sector_code)
            return close, tvol, self.SOURCE_PYKRX

        # Path 3: KRX HTTP JSON direct
        close, tvol = self._try_krx_direct(sector_code, to_date)
        if close is not None:
            logger.debug("krx_direct ok sector=%s", sector_code)
            return close, tvol, self.SOURCE_DIRECT

        return None, None, ""

    # ── Path 1 ──────────────────────────────────────────────────────────────

    def _try_pykrx_range(
        self, sector_code: str, from_date: str, to_date: str
    ) -> tuple[float | None, float | None]:
        try:
            df = pykrx_stock.get_index_ohlcv_by_date(from_date, to_date, sector_code)
            if df is not None and not df.empty:
                return self._extract_last_row(df)
        except Exception as e:
            logger.debug("pykrx_range failed sector=%s err=%s", sector_code, e)
        return None, None

    # ── Path 2 ──────────────────────────────────────────────────────────────

    def _try_pykrx_daily_fallback(
        self, sector_code: str, to_date_str: str, max_lookback: int = 15
    ) -> tuple[float | None, float | None]:
        end_dt = datetime.strptime(to_date_str, "%Y%m%d")
        for offset in range(max_lookback):
            candidate = (end_dt - timedelta(days=offset)).strftime("%Y%m%d")
            try:
                df = pykrx_stock.get_index_ohlcv_by_date(candidate, candidate, sector_code)
                if df is not None and not df.empty:
                    return self._extract_last_row(df)
            except Exception:
                continue
        return None, None

    # ── Path 3 ──────────────────────────────────────────────────────────────

    def _try_krx_direct(
        self, sector_code: str, to_date_str: str
    ) -> tuple[float | None, float | None]:
        """
        KRX JSON API를 직접 호출해 업종 지수 종가를 조회한다.

        [설계 계약 — 구현 착수 전 아래 3개 항목을 KRX 포털에서 확인·확정해야 한다]
          (1) KRX_INDEX_BLD : bld 코드. 현재 "MDCSTAT00301" 계열로 추정, 미확정.
          (2) idxIndMidclssCd : KOSPI 업종 분류 파라미터 값. "02" 추정, 미확정.
          (3) 응답 필드명 : IDX_IND_CD / CLSPRC_IDX / ACC_TRDVAL 은 예측값.
              KRX 포털 Network 탭 → 업종지수 조회 → JSON 응답 키 확인 후 확정.

        구현 참조:
          - sugup-report src/sugup_pivot/collectors/krx_direct.py
            _bootstrap_session / JSON 호출 패턴과 동일하게 작성한다.
        """
        try:
            headers = {
                "User-Agent": (
                    "Mozilla/5.0 (Windows NT 10.0; Win64; x64) "
                    "AppleWebKit/537.36 (KHTML, like Gecko) Chrome/122.0.0.0 Safari/537.36"
                ),
                "Referer":           self.KRX_HOME_URL,
                "X-Requested-With":  "XMLHttpRequest",
                "Content-Type":      "application/x-www-form-urlencoded; charset=UTF-8",
            }
            payload = {
                "locale":          "ko_KR",
                "idxIndMidclssCd": "02",        # TODO: KOSPI 업종 분류 코드 미확정 — 포털 확인 필요
                "trdDd":           to_date_str,
                "share":           "1",
                "money":           "1",
                "csvxls_isNo":     "false",
                "bld":             self.KRX_INDEX_BLD,
            }
            session = requests.Session()
            session.get(self.KRX_HOME_URL, headers={"User-Agent": headers["User-Agent"]}, timeout=10)
            resp = session.post(self.KRX_JSON_URL, data=payload, headers=headers, timeout=15)
            resp.raise_for_status()
            data = resp.json()
            rows = data.get("OutBlock_1", data.get("output", []))
            # TODO: 아래 필드명 3개는 추정값 — KRX 포털 응답 JSON에서 실제 키 확인 후 확정
            #   IDX_IND_CD  → 업종 코드 필드 (예: "1003")
            #   CLSPRC_IDX  → 종가 필드
            #   ACC_TRDVAL  → 거래대금 필드
            for row in rows:
                if str(row.get("IDX_IND_CD", "")).strip() == sector_code:
                    close_raw = str(row.get("CLSPRC_IDX", "0")).replace(",", "")
                    close = float(close_raw) if close_raw else None
                    if close and close > 0:
                        tvol_raw = row.get("ACC_TRDVAL")
                        tvol = float(str(tvol_raw).replace(",", "")) if tvol_raw else None
                        return close, tvol
        except Exception as e:
            logger.debug("krx_direct failed sector=%s to_date=%s err=%s", sector_code, to_date_str, e)
        return None, None

    # ── 공통 유틸 ───────────────────────────────────────────────────────────

    def _extract_last_row(
        self, df: pd.DataFrame
    ) -> tuple[float | None, float | None]:
        last = df.iloc[-1]
        try:
            close = float(last.get("종가", last.iloc[-1]))
        except Exception:
            return None, None
        tvol = float(last["거래대금"]) if "거래대금" in last.index else None
        return close, tvol

    # ── 히스토리 수집 ────────────────────────────────────────────────────────

    def collect_history(self, from_month: str, to_month: str) -> pd.DataFrame:
        """from_month, to_month: YYYY-MM. 범위 내 모든 월을 순차 수집한다."""
        months = pd.period_range(from_month, to_month, freq="M")
        frames = []
        for period in months:
            trade_month = str(period)
            df = self.collect_monthly_prices(trade_month)
            if not df.empty:
                frames.append(df)
            else:
                logger.warning("no_data_for_month month=%s", trade_month)
        return pd.concat(frames, ignore_index=True) if frames else pd.DataFrame()
```

---

## 9. Excel 시각화 설계

### 9.1 색상 매핑 규칙

색상 결정 체계는 다음 단일 경로를 따른다:

```text
sector_master.display_color
    → RankTableBuilder.build_snapshot() → snapshot_df.display_color
    → RankTableBuilder.build_color_lookup() → color_lookup: {sector_name: hex}
    → RankTableExcelExporter.export(color_lookup=...) → 셀 배경색
```

키워드 기반 하드코딩은 사용하지 않는다. `color_lookup`에 없는 업종명은 흰색(FFFFFF)으로 fallback한다.

### 9.2 Excel Exporter

```python
# app/reports/rank_table_excel_exporter.py

from openpyxl import Workbook
from openpyxl.styles import PatternFill, Alignment, Font, Border, Side

import pandas as pd


class RankTableExcelExporter:
    def export(
        self,
        rank_table: pd.DataFrame,
        snapshot_df: pd.DataFrame,
        output_path: str,
        color_lookup: dict[str, str] | None = None,
    ) -> None:
        """
        color_lookup: {sector_name: hex_6digit}
            RankTableBuilder.build_color_lookup(snapshot_df)로 생성한다.
            None이면 모든 셀을 흰색으로 출력한다.
        """
        wb = Workbook()
        ws = wb.active
        ws.title = "Sector Rank Table"

        header_fill = PatternFill("solid", fgColor="CC0000")
        header_font = Font(color="FFFFFF", bold=True)
        thin = Side(style="thin", color="CCCCCC")
        rank_fill = PatternFill("solid", fgColor="F2F2F2")

        # 헤더 행
        ws.cell(row=1, column=1, value="순위").fill = rank_fill
        for col_idx, col_name in enumerate(rank_table.columns, start=2):
            cell = ws.cell(row=1, column=col_idx, value=col_name)
            cell.fill = header_fill
            cell.font = header_font
            cell.alignment = Alignment(horizontal="center", vertical="center")

        # 데이터 행
        for row_idx, rank_label in enumerate(rank_table.index, start=2):
            ws.cell(row=row_idx, column=1, value=rank_label).fill = rank_fill
            for col_idx, col_name in enumerate(rank_table.columns, start=2):
                value = rank_table.loc[rank_label, col_name]
                cell = ws.cell(row=row_idx, column=col_idx, value=value)
                cell.alignment = Alignment(horizontal="center", vertical="center")
                cell.border = Border(
                    top=Side(style="thin", color="CCCCCC"),
                    bottom=Side(style="thin", color="CCCCCC"),
                    left=Side(style="thin", color="CCCCCC"),
                    right=Side(style="thin", color="CCCCCC"),
                )
                cell.fill = self._resolve_fill(str(value), color_lookup)

        # 고정 및 열 너비
        ws.freeze_panes = "B2"
        ws.column_dimensions["A"].width = 6
        for i in range(2, len(rank_table.columns) + 2):
            col_letter = ws.cell(row=1, column=i).column_letter
            ws.column_dimensions[col_letter].width = 14

        wb.save(output_path)

    def _resolve_fill(
        self, sector_name: str, color_lookup: dict[str, str] | None
    ) -> PatternFill:
        if color_lookup and sector_name in color_lookup:
            hex_color = str(color_lookup[sector_name] or "FFFFFF").lstrip("#").upper()
            return PatternFill("solid", fgColor=hex_color if len(hex_color) == 6 else "FFFFFF")
        return PatternFill("solid", fgColor="FFFFFF")
```

---

## 10. 파이프라인 실행 순서

### 10.1 월간 배치

```text
1. 업종별 월말 가격 수집 → sector_monthly_price
2. 전체 가격 로드 → 월간 수익률 계산 (인접 월 검증 포함)
3. sector_monthly_return 저장
4. Rank Snapshot 생성 (display_color 포함) → sector_rank_snapshot 저장
5. 주도섹터 점수 계산 (trading_value 포함) → sector_leadership_score 저장
6. Rotation Signal 생성 (EXTENDED → LEADER 순서) → sector_rotation_signal 저장
7. 주도섹터 내 대장주 후보 추출 (선택)
8. Excel / HTML / Markdown 리포트 생성
```

### 10.2 CLI 구현

```python
# app/cli/run_sector_monthly_pipeline.py

from __future__ import annotations

import argparse

from app.db.connection import get_connection
from app.db.crud import (
    load_sector_master,
    load_all_monthly_prices,
    upsert_sector_monthly_price,
    save_sector_monthly_return,
    save_sector_rank_snapshot,
    save_sector_leadership_score,
    save_sector_rotation_signal,
)
from app.config.sector_universe import KRX_KOSPI_SECTORS
from app.collectors.sector_price_collector import SectorPriceCollector
from app.services.monthly_return_calculator import MonthlyReturnCalculator
from app.services.rank_table_builder import RankTableBuilder
from app.services.leadership_score_engine import LeadershipScoreEngine
from app.services.rotation_signal_engine import RotationSignalEngine
from app.services.leader_stock_selector import LeaderStockSelector
from app.reports.rank_table_excel_exporter import RankTableExcelExporter
from app.config.settings import (
    load_config,
    build_leadership_score_config,
    build_signal_config,
    get_min_months_required,
)


def main() -> None:
    parser = argparse.ArgumentParser(description="MFRT 월간 파이프라인")
    parser.add_argument("--trade-month", required=True, help="YYYY-MM")
    parser.add_argument("--with-leader-stocks", action="store_true")
    parser.add_argument("--export-report", action="store_true")
    parser.add_argument("--db-path", default=None)
    parser.add_argument(
        "--config",
        default=None,
        help="sector_config.yaml 경로 (생략 시 app/config/sector_config.yaml)",
    )
    args = parser.parse_args()

    # YAML 설정 로드 — 모든 엔진 파라미터의 단일 진입점
    cfg = load_config(args.config)
    min_months = get_min_months_required(cfg)

    conn = get_connection(args.db_path)

    # 1. 수집
    collector = SectorPriceCollector()
    price_df = collector.collect_monthly_prices(args.trade_month)
    upsert_sector_monthly_price(conn, price_df)

    # 2. 전체 로드 → 수익률 계산 (min_months_required는 YAML sector_universe 에서)
    all_price_df = load_all_monthly_prices(conn)
    returns_df = MonthlyReturnCalculator().calculate(all_price_df, min_months_required=min_months)
    save_sector_monthly_return(conn, returns_df)

    # 3. Snapshot (display_color 포함)
    sector_master_df = load_sector_master(conn)
    builder = RankTableBuilder()
    snapshot_df = builder.build_snapshot(returns_df, sector_master_df)
    save_sector_rank_snapshot(conn, snapshot_df)

    # 4. 점수 계산 — rolling window 등 파라미터는 YAML leadership_score 에서
    score_engine = LeadershipScoreEngine(build_leadership_score_config(cfg))
    scored_df = score_engine.score(returns_df, sector_master_df)
    save_sector_leadership_score(conn, scored_df)

    # 5. 신호 생성 — consecutive_decline 등 파라미터는 YAML signals 에서
    signal_df = RotationSignalEngine(build_signal_config(cfg)).generate(scored_df)
    save_sector_rotation_signal(conn, signal_df)

    # 6. 대장주 후보 (--with-leader-stocks 플래그)
    if args.with_leader_stocks:
        # stock_df는 별도 수집기로 준비 (Phase 2)
        pass

    # 7. 리포트
    if args.export_report:
        rank_table  = builder.build_rank_table(returns_df)
        color_lookup = builder.build_color_lookup(snapshot_df)
        out_path = f"reports/sector_rank/sector_rank_{args.trade_month}.xlsx"
        RankTableExcelExporter().export(
            rank_table=rank_table,
            snapshot_df=snapshot_df,
            output_path=out_path,
            color_lookup=color_lookup,
        )
        print(f"[ok] report saved: {out_path}")

    conn.close()


if __name__ == "__main__":
    main()
```

### 10.3 CLI 명령

```bash
# 2026년 6월 파이프라인 실행 (기본 설정 — app/config/sector_config.yaml 자동 로드)
python -m app.cli.run_sector_monthly_pipeline \
  --trade-month 2026-06 \
  --export-report

# 커스텀 설정 파일 지정
python -m app.cli.run_sector_monthly_pipeline \
  --trade-month 2026-06 \
  --config path/to/custom_config.yaml \
  --export-report

# 대장주 후보까지 포함
python -m app.cli.run_sector_monthly_pipeline \
  --trade-month 2026-06 \
  --with-leader-stocks \
  --export-report

# 히스토리 일괄 수집 (별도 CLI)
python -m app.cli.run_sector_history_collect \
  --from-month 2009-01 \
  --to-month 2024-12
```

---

## 11. 신호 해석 정책

### 11.1 신호 우선순위

```text
PEAK_WARNING > ROTATION_OUT > EXTENDED > LEADER > NEW_LEADER > WATCH
```

동일 섹터에 복수 조건이 동시에 성립하면 우선순위가 높은 신호를 반환한다.

### 11.2 신호별 해석

| 신호 | 판정 기준 | 해석 |
|---|---|---|
| NEW_LEADER | Top10 진입 + rank_change ≥ 5 + 최근 3개월 내 최고 순위 | 신규 주도 후보. 다음 달 유지 여부 확인 후 격상 |
| LEADER | Top10 + 최근3개월 2회 이상 Top10 + total_score ≥ 50 | 주도섹터 인정. 눌림목 매수 |
| EXTENDED | Top10 + 최근6개월 4회 이상 Top10 | 강한 주도장세. 신규 추격 주의 |
| PEAK_WARNING | 직전 Top5 → 당월 Top15 이탈 | 델타 피크 가능성. 비중 축소 |
| ROTATION_OUT | 직전 Top10 → 당월 Top18 이탈 + 2개월 연속 하락 | 주도권 이탈. 신규 섹터 탐색 |
| WATCH | Top20 + total_score ≥ 30 | 모니터링 대상 |

---

## 12. 테스트 설계

### 12.1 월간 수익률 계산 테스트 (인접 월 검증 포함)

```python
# app/tests/test_monthly_return_calculator.py

import pandas as pd
import numpy as np
from app.services.monthly_return_calculator import MonthlyReturnCalculator


def _make_price(records):
    return pd.DataFrame(records)


def test_basic_return():
    df = _make_price([
        {"trade_month": "2026-01", "sector_code": "1011", "sector_name": "전기전자", "close_price": 100},
        {"trade_month": "2026-02", "sector_code": "1011", "sector_name": "전기전자", "close_price": 110},
        {"trade_month": "2026-01", "sector_code": "1013", "sector_name": "운수장비", "close_price": 100},
        {"trade_month": "2026-02", "sector_code": "1013", "sector_name": "운수장비", "close_price": 105},
    ])
    result = MonthlyReturnCalculator().calculate(df)
    s11 = result[result["sector_code"] == "1011"].iloc[0]
    s13 = result[result["sector_code"] == "1013"].iloc[0]
    assert round(s11["monthly_return"], 2) == 10.00
    assert round(s13["monthly_return"], 2) == 5.00
    assert s11["rank_no"] == 1
    assert s13["rank_no"] == 2


def test_gap_month_excluded():
    """2월 데이터가 빠진 경우 3월 수익률은 계산하지 않아야 한다."""
    df = _make_price([
        {"trade_month": "2026-01", "sector_code": "1011", "sector_name": "전기전자", "close_price": 100},
        # 2026-02 누락
        {"trade_month": "2026-03", "sector_code": "1011", "sector_name": "전기전자", "close_price": 120},
    ])
    result = MonthlyReturnCalculator().calculate(df, min_months_required=1)
    # 2026-03의 수익률은 누적치이므로 계산하면 안 됨
    assert "2026-03" not in result["trade_month"].values


def test_min_months_filter():
    """
    min_months_required는 가격 데이터 월 수 기준이다.
    - 1011: Jan+Feb 가격 2건 → min=2 충족 → Feb 수익률 포함
    - 1013: Feb 가격 1건만 → min=2 미달 → 제외
    수익률 개수(1013은 수익률 0건)가 아닌 가격 개수로 판단해야 한다.
    """
    df = _make_price([
        {"trade_month": "2026-01", "sector_code": "1011", "sector_name": "전기전자", "close_price": 100},
        {"trade_month": "2026-02", "sector_code": "1011", "sector_name": "전기전자", "close_price": 110},
        # 1013은 가격 데이터 1개월뿐 → min_months_required=2 미달
        {"trade_month": "2026-02", "sector_code": "1013", "sector_name": "운수장비", "close_price": 200},
    ])
    result = MonthlyReturnCalculator().calculate(df, min_months_required=2)
    # 1013: 가격 1건 → 제외
    assert "1013" not in result["sector_code"].values
    # 1011: 가격 2건 → Feb 수익률 1건 포함
    assert "1011" in result["sector_code"].values
    assert result[result["sector_code"] == "1011"]["trade_month"].iloc[0] == "2026-02"


def test_min_months_first_return_included():
    """
    Jan+Feb 가격 2건이면 Feb 수익률(첫 번째 수익률)이 순위에 포함돼야 한다.
    수익률 개수 기준 필터였다면 수익률이 1개라서 제외됐을 케이스다.
    """
    df = _make_price([
        {"trade_month": "2026-01", "sector_code": "1011", "sector_name": "전기전자", "close_price": 100},
        {"trade_month": "2026-02", "sector_code": "1011", "sector_name": "전기전자", "close_price": 110},
        {"trade_month": "2026-01", "sector_code": "1013", "sector_name": "운수장비", "close_price": 100},
        {"trade_month": "2026-02", "sector_code": "1013", "sector_name": "운수장비", "close_price": 105},
    ])
    result = MonthlyReturnCalculator().calculate(df, min_months_required=2)
    # 두 섹터 모두 Feb 수익률 1건씩 포함돼야 함
    assert set(result["sector_code"]) == {"1011", "1013"}
    assert set(result["trade_month"]) == {"2026-02"}


def test_trading_value_passthrough():
    """trading_value와 prev_trading_value가 반환 DataFrame에 포함돼야 한다."""
    df = _make_price([
        {"trade_month": "2026-01", "sector_code": "1011", "sector_name": "전기전자",
         "close_price": 100, "trading_value": 1_000_000},
        {"trade_month": "2026-02", "sector_code": "1011", "sector_name": "전기전자",
         "close_price": 110, "trading_value": 1_500_000},
    ])
    result = MonthlyReturnCalculator().calculate(df)
    row = result.iloc[0]
    assert row["trading_value"] == 1_500_000
    assert row["prev_trading_value"] == 1_000_000
```

### 12.2 Rank Table 생성 테스트

```python
# app/tests/test_rank_table_builder.py

import pandas as pd
from app.services.rank_table_builder import RankTableBuilder


def test_rank_table_builder():
    df = pd.DataFrame([
        {"trade_month": "2026-02", "sector_code": "1011", "sector_name": "전기전자", "rank_no": 1, "monthly_return": 10},
        {"trade_month": "2026-02", "sector_code": "1013", "sector_name": "운수장비", "rank_no": 2, "monthly_return": 5},
        {"trade_month": "2026-03", "sector_code": "1013", "sector_name": "운수장비", "rank_no": 1, "monthly_return": 8},
        {"trade_month": "2026-03", "sector_code": "1011", "sector_name": "전기전자", "rank_no": 2, "monthly_return": 3},
    ])
    table = RankTableBuilder().build_rank_table(df, top_n=2)
    assert table.loc["1위", "2026-02"] == "전기전자"
    assert table.loc["1위", "2026-03"] == "운수장비"


def test_color_lookup_from_snapshot():
    master = pd.DataFrame([
        {"sector_code": "1011", "theme_group": "반도체", "display_color": "C6EFCE"},
        {"sector_code": "1013", "theme_group": "자동차", "display_color": "FFC7CE"},
    ])
    returns = pd.DataFrame([
        {"trade_month": "2026-02", "sector_code": "1011", "sector_name": "전기전자", "rank_no": 1, "monthly_return": 10},
        {"trade_month": "2026-02", "sector_code": "1013", "sector_name": "운수장비", "rank_no": 2, "monthly_return": 5},
    ])
    builder = RankTableBuilder()
    snapshot = builder.build_snapshot(returns, master)
    color_lookup = builder.build_color_lookup(snapshot)
    assert color_lookup["전기전자"] == "C6EFCE"
    assert color_lookup["운수장비"] == "FFC7CE"
```

### 12.3 신호 엔진 테스트

```python
# app/tests/test_rotation_signal_engine.py

import pandas as pd
from app.services.rotation_signal_engine import RotationSignalEngine, SignalConfig


def _base_row(**kwargs):
    defaults = {
        "trade_month": "2026-06",
        "sector_code": "1011",
        "sector_name": "전기전자",
        "rank_no": 5,
        "prev_rank_no": None,
        "rank_change": None,
        "consecutive_top10": 1,
        "top10_in_3m": 1,
        "top10_in_6m": 1,
        "total_score": 50,
        "score_reason": "",
        "rank_declining_2m": 0,
        "is_recent_best_rank": 1,
    }
    defaults.update(kwargs)
    return pd.DataFrame([defaults])


def test_new_leader():
    df = _base_row(rank_no=3, rank_change=8, top10_in_3m=1, top10_in_6m=1, prev_rank_no=11)
    result = RotationSignalEngine().generate(df)
    assert result.iloc[0]["signal"] == "NEW_LEADER"


def test_extended_before_leader():
    """top10_in_6m >= 4이면 EXTENDED가 LEADER보다 먼저 판정돼야 한다."""
    df = _base_row(rank_no=2, top10_in_3m=3, top10_in_6m=4, total_score=80)
    result = RotationSignalEngine().generate(df)
    assert result.iloc[0]["signal"] == "EXTENDED"


def test_leader():
    df = _base_row(rank_no=5, top10_in_3m=2, top10_in_6m=2, total_score=60)
    result = RotationSignalEngine().generate(df)
    assert result.iloc[0]["signal"] == "LEADER"


def test_peak_warning():
    df = _base_row(rank_no=16, prev_rank_no=3)
    result = RotationSignalEngine().generate(df)
    assert result.iloc[0]["signal"] == "PEAK_WARNING"


def test_rotation_out_requires_2m_decline():
    """2개월 연속 하락 없으면 ROTATION_OUT 미발생."""
    df_no_decline = _base_row(rank_no=19, prev_rank_no=8, rank_declining_2m=0)
    result = RotationSignalEngine().generate(df_no_decline)
    assert result.empty or result.iloc[0]["signal"] != "ROTATION_OUT"

    df_with_decline = _base_row(rank_no=19, prev_rank_no=8, rank_declining_2m=1)
    result2 = RotationSignalEngine().generate(df_with_decline)
    assert result2.iloc[0]["signal"] == "ROTATION_OUT"
```

### 12.4 volume_score 파이프라인 통합 테스트

```python
# app/tests/test_leadership_score_engine.py

import pandas as pd
from app.services.monthly_return_calculator import MonthlyReturnCalculator
from app.services.leadership_score_engine import LeadershipScoreEngine, LeadershipScoreConfig


def test_volume_score_flows_through_pipeline():
    """trading_value가 수익률 계산을 거쳐 점수 엔진까지 전달돼야 한다."""
    price_df = pd.DataFrame([
        {"trade_month": "2026-01", "sector_code": "1011", "sector_name": "전기전자",
         "close_price": 100, "trading_value": 1_000_000},
        {"trade_month": "2026-02", "sector_code": "1011", "sector_name": "전기전자",
         "close_price": 110, "trading_value": 1_600_000},
    ])
    returns_df = MonthlyReturnCalculator().calculate(price_df)
    assert "trading_value" in returns_df.columns
    assert "prev_trading_value" in returns_df.columns

    master = pd.DataFrame([
        {"sector_code": "1011", "theme_group": "반도체", "value_chain_group": None, "display_color": "C6EFCE"}
    ])
    cfg = LeadershipScoreConfig(volume_growth_threshold=0.3)
    scored = LeadershipScoreEngine(cfg).score(returns_df, master)
    # 60% 증가 → volume_score > 0
    assert scored.iloc[0]["volume_score"] > 0
```

---

## 13. 예외 처리 정책

| 상황 | 처리 |
|---|---|
| 월말 가격 누락 | 해당 섹터 해당 월 수익률 계산 제외, 경고 로그 |
| 데이터 공백 (비인접 월) | 인접 월 검증 실패 → 수익률 NaN → 행 제외 |
| 신규 편입 섹터 | min_months_required 미달 → 순위 제외 |
| 업종명 변경 | sector_name_alias 테이블로 매핑, sector_code 불변 |
| 거래대금 없음 | trading_value=None → prev_trading_value=None → volume_score=0 |
| 동일 수익률 | rank(method='first')로 안정적 순위 부여 |
| pykrx 수집 실패 | 경고 로그 후 해당 섹터 해당 월 skip (전체 파이프라인 중단 안 함) |

---

## 14. 로깅 설계

```python
import logging

logger = logging.getLogger("moneyflow.sector_rank")

logger.info("sector pipeline started", extra={"trade_month": trade_month})
logger.warning("gap_month detected", extra={"sector_code": code, "trade_month": tm, "prev_month": prev})
logger.warning("missing monthly price", extra={"sector_code": code, "trade_month": tm})
logger.error("collector failed", exc_info=True, extra={"sector_code": code})
```

로그 파일:

```text
logs/sector_rank/202606_sector_rank_pipeline.log
```

---

## 15. 운영 스케줄

| 주기 | 작업 |
|---|---|
| 매월 마지막 거래일 장마감 후 | 공식 월간 Rank Table 생성 |
| 매월 첫 영업일 새벽 | 리포트 생성 및 MoneyFlow 신호 반영 |

---

## 16. 백테스트 설계

### 16.1 검증 에피소드

| 에피소드 | 주도섹터 | 기대 신호 발생 시점 |
|---|---|---|
| 2009~2010 차·화·정 | 운수장비(1013), 화학(1006) | 2009-06 전후 NEW_LEADER |
| 2011 조선 | (KRX 21개에 직접 없음 — 기계·운수장비로 관측) | — |
| 2019~2020 코로나 | 의약품(1007), 서비스업(1023) | 2020-03 전후 |
| 2023 이차전지 | 전기전자(1011) | 2023-01 전후 NEW_LEADER → EXTENDED |
| 2023~2024 AI | 전기전자(1011), 전기가스업(1015) | 2023-07 전후 LEADER |

KRX 21개 분류는 반도체·화장품·조선·방산 등을 직접 구분하지 않는 한계가 있다. 이 에피소드들은 상위 업종(전기전자, 운수장비 등)에서 신호를 관측하는 방식으로 검증한다.

### 16.2 백테스트 지표

| 지표 | 설명 |
|---|---|
| lead_detection_month | 주도섹터 최초 탐지 월 |
| leader_confirm_month | LEADER 확정 월 |
| peak_warning_month | PEAK_WARNING 발생 월 |
| max_return_after_signal | 신호 이후 최대 수익률 |
| false_positive_rate | 신호 후 2개월 내 Top10 이탈 비율 |

---

## 17. 구현 우선순위

### Phase 1: MVP

```text
1. DB 마이그레이션 실행 (migrations/001~003)
2. sector_master seed data 입력 (KRX_KOSPI_SECTORS)
3. SectorPriceCollector 구현 및 2009~현재 히스토리 수집
4. MonthlyReturnCalculator → 순위 산출
5. RankTableBuilder → Excel Export
6. LeadershipScoreEngine + RotationSignalEngine
7. 백테스트: 2009~2024 에피소드 검증
```

### Phase 2: MoneyFlow 연결

```text
1. 종목-섹터 매핑 (KRX 업종 구성종목)
2. 주도섹터 내 대장주 추출 (LeaderStockSelector)
3. 거래대금·수급·20주선 조건 결합
4. HTML 리포트 생성
```

### Phase 3: 자동화

```text
1. 월간 자동 배치 (cron)
2. 신호 정확도 리포트
3. 웹 대시보드 연동
```

---

## 18. 개발 체크리스트

```text
[DB]
- [ ] sector_master 생성 및 seed data 입력
- [ ] sector_name_alias 생성
- [ ] sector_monthly_price 생성
- [ ] sector_monthly_return 생성 (trading_value, prev_trading_value 컬럼 포함)
- [ ] sector_rank_snapshot 생성 (display_color 컬럼 포함)
- [ ] sector_leadership_score 생성 (top10_in_3m, top10_in_6m, consecutive_top10 컬럼 포함)
- [ ] sector_rotation_signal 생성 (rank_declining_2m 컬럼 포함)
- [ ] sector_leader_stocks 생성

[수집]
- [ ] SectorPriceCollector pykrx 경로 구현
- [ ] 월말 가격 추출 로직 (일별 → 월말)
- [ ] 히스토리 일괄 수집 CLI

[분석]
- [ ] 인접 월 검증 (gap_month 제외)
- [ ] trading_value 파이프라인 (price → returns → score)
- [ ] top10_in_3m / top10_in_6m 윈도우 카운터
- [ ] rank_declining_2m 플래그
- [ ] EXTENDED → LEADER 신호 판정 순서
- [ ] require_recent_best_rank (NEW_LEADER)
- [ ] ROTATION_OUT 2개월 연속 하락 조건

[색상 매핑]
- [ ] sector_master.display_color 단일 소스 확인
- [ ] build_snapshot() display_color 포함 확인
- [ ] build_color_lookup() 구현 확인
- [ ] RankTableExcelExporter color_lookup 경로 사용 확인

[CRUD]
- [ ] get_connection() 구현
- [ ] upsert_sector_monthly_price() 구현
- [ ] load_all_monthly_prices() 구현
- [ ] save_sector_monthly_return() 구현
- [ ] save_sector_rank_snapshot() 구현
- [ ] save_sector_leadership_score() 구현
- [ ] save_sector_rotation_signal() 구현
- [ ] load_sector_master() 구현

[리포트]
- [ ] Excel Export (color_lookup 경로)
- [ ] HTML 리포트

[검증]
- [ ] 단위 테스트 (인접 월, min_months, trading_value, volume_score 파이프라인)
- [ ] 신호 엔진 테스트 (EXTENDED vs LEADER 순서, ROTATION_OUT 조건)
- [ ] 색상 매핑 통합 테스트
- [ ] 2009~2024 에피소드 백테스트
```

---

## 19. 실전 사용 기준

Rank Table 신호는 직접 매수 신호가 아니라 **섹터 레이더**다.

실전 매수는 반드시 아래 조건을 추가로 확인한다.

```text
1. 지수 상태가 급락 추세가 아닌가?
2. 해당 섹터가 NEW_LEADER 또는 LEADER인가?
3. 관련 섹터가 동반 상승하는가?
4. 대장주가 주봉 20주선 위인가?
5. 거래대금이 증가했는가?
6. 외국인 또는 기관 수급이 동반되는가?
7. 고점 추격이 아니라 눌림 후 재돌파인가?
8. PEAK_WARNING 또는 ROTATION_OUT 섹터가 아닌가?
```

---

## 20. KRX 21개 업종 체계의 한계와 해석 지침

KRX KOSPI 21개 업종은 반도체·화장품·조선·방산·인터넷서비스를 직접 구분하지 않는다.
이 제약 하에서 레퍼런스 에피소드를 해석하는 방법은 다음과 같다.

| 레퍼런스 섹터 | KRX 대응 업종 | 비고 |
|---|---|---|
| 반도체 | 전기전자 (1011) | 삼성전자·SK하이닉스 포함 |
| 화장품 | 유통업(1014) 또는 서비스업(1023) | 직접 대응 없음. 의약품(1007) 보조 관찰 |
| 자동차 | 운수장비 (1013) | 완성차·부품 포함 |
| 조선 | 기계(1010) 또는 운수창고업(1017) | 직접 대응 없음 |
| 이차전지 | 화학(1006), 전기전자(1011) | 분산 |
| 방산·전력기기 | 전기가스업(1015), 기계(1010) | 직접 대응 없음 |
| 인터넷서비스 | 서비스업(1023) | 네이버·카카오 포함 |

Phase 2 이후에 KRX 테마 지수(별도 코드 체계)를 추가하거나, 종목-섹터 매핑을 자체 구성하면 이 한계를 극복할 수 있다. 초기에는 위 대응 관계로 분석하되, 리포트에 명시한다.
