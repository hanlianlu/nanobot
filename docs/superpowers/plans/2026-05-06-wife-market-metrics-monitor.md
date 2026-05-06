# Wife Market Metrics Monitor Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a wife-only nanobot skill that maintains an independent US stock metrics archive, detects trends/anomalies, and delivers integrated PNG monitor cards in WeChat/Telegram.

**Architecture:** Implement the monitor as a wife-logical workspace skill under `/Users/hanlianlyu/.nanobot-wife/workspace/skills/market-metrics-monitor/`. On this Mac that path is a symlink into `/Users/hanlianlyu/.nanobot/workspace/skills/market-metrics-monitor/`; the main nanobot instance disables the skill through `agents.defaults.disabledSkills`, while wife leaves it enabled. The skill owns a small Python package in `scripts/market_metrics_monitor/`, stores data under `/Users/hanlianlyu/.nanobot-wife/workspace/data/market-metrics-monitor/`, and uses launchd for deterministic daily updates. It never writes finvesto data or calls the existing professional finance pipeline.

**Tech Stack:** Python 3.11+, `uv run` with PEP 723 script dependencies, pandas/numpy/pyarrow for local storage and metrics, httpx for EOD providers, Jinja2 for HTML cards, Playwright for HTML-to-PNG rendering, pytest for verification, launchd for macOS scheduling.

---

## File Structure

Create these files under the wife workspace:

```text
/Users/hanlianlyu/.nanobot-wife/workspace/skills/market-metrics-monitor/
  SKILL.md
  references/
    metric_glossary.md
  scripts/
    update_metrics.py
    install_launchd.py
    market_metrics_monitor/
      __init__.py
      paths.py
      config.py
      atomic.py
      registry.py
      storage.py
      providers.py
      metrics.py
      anomalies.py
      snapshots.py
      render.py
      cli.py
    tests/
      conftest.py
      test_skill_skeleton.py
      test_registry.py
      test_storage.py
      test_providers.py
      test_metrics.py
      test_anomalies.py
      test_snapshots.py
      test_render.py
      test_cli.py
      test_launchd.py
```

Implementation note for this machine: because `~/.nanobot-wife/workspace/skills`
is a symlink to `~/.nanobot/workspace/skills`, commit skill-file changes from
`/Users/hanlianlyu/.nanobot/workspace` with `git add -f
skills/market-metrics-monitor`. Do not try to track files through the wife
workspace symlink. Runtime data still belongs under the wife data directory.

The implementation writes runtime data under:

```text
/Users/hanlianlyu/.nanobot-wife/workspace/data/market-metrics-monitor/
```

Use this command form for tests throughout the plan:

```bash
cd /Users/hanlianlyu/.nanobot-wife/workspace/skills/market-metrics-monitor
PYTHONPATH=scripts uv run \
  --with pytest \
  --with pandas \
  --with numpy \
  --with pyarrow \
  --with httpx \
  --with jinja2 \
  --with playwright \
  --with python-dotenv \
  python -m pytest scripts/tests -q
```

## Task 1: Skill Skeleton and Script Entrypoints

**Files:**
- Create: `/Users/hanlianlyu/.nanobot-wife/workspace/skills/market-metrics-monitor/SKILL.md`
- Create: `/Users/hanlianlyu/.nanobot-wife/workspace/skills/market-metrics-monitor/references/metric_glossary.md`
- Create: `/Users/hanlianlyu/.nanobot-wife/workspace/skills/market-metrics-monitor/scripts/update_metrics.py`
- Create: `/Users/hanlianlyu/.nanobot-wife/workspace/skills/market-metrics-monitor/scripts/market_metrics_monitor/__init__.py`
- Create: `/Users/hanlianlyu/.nanobot-wife/workspace/skills/market-metrics-monitor/scripts/tests/conftest.py`
- Create: `/Users/hanlianlyu/.nanobot-wife/workspace/skills/market-metrics-monitor/scripts/tests/test_skill_skeleton.py`

- [ ] **Step 1: Create directories**

Run:

```bash
mkdir -p /Users/hanlianlyu/.nanobot-wife/workspace/skills/market-metrics-monitor/{references,scripts/market_metrics_monitor,scripts/tests}
```

Expected: command exits 0 and creates the directories.

- [ ] **Step 2: Write the failing skeleton test**

Create `/Users/hanlianlyu/.nanobot-wife/workspace/skills/market-metrics-monitor/scripts/tests/test_skill_skeleton.py`:

```python
from pathlib import Path

import yaml


SKILL_ROOT = Path(__file__).resolve().parents[2]


def test_skill_frontmatter_is_discoverable():
    text = (SKILL_ROOT / "SKILL.md").read_text(encoding="utf-8")
    assert text.startswith("---\n")
    _, frontmatter, _ = text.split("---", 2)
    meta = yaml.safe_load(frontmatter)
    assert meta["name"] == "market-metrics-monitor"
    assert "stocks" in meta["description"].lower()
    assert "buy/sell" in text
    assert "invest_portfolio" in text


def test_update_metrics_script_has_pep723_dependencies():
    text = (SKILL_ROOT / "scripts" / "update_metrics.py").read_text(encoding="utf-8")
    assert "# /// script" in text
    assert "pandas" in text
    assert "playwright" in text
    assert "market_metrics_monitor.cli" in text
```

- [ ] **Step 3: Run the skeleton test and verify it fails**

Run:

```bash
cd /Users/hanlianlyu/.nanobot-wife/workspace/skills/market-metrics-monitor
PYTHONPATH=scripts uv run --with pytest --with pyyaml python -m pytest scripts/tests/test_skill_skeleton.py -q
```

Expected: FAIL because `SKILL.md` and `update_metrics.py` do not exist yet.

- [ ] **Step 4: Create the initial skill files**

Create `SKILL.md`:

```markdown
---
name: market-metrics-monitor
description: >
  Use when the user asks to monitor US stocks, stock metrics, trendlines,
  trend tables, indicator evolution, anomalies, key levels, RSI/MACD,
  or watchlist status without asking for buy/sell, rebalancing, portfolio
  optimization, model prediction, or investment advice.
metadata: '{"nanobot":{"emoji":"📈"}}'
---

# Market Metrics Monitor

This skill is for the wife nanobot instance. It provides independent US stock
metrics monitoring for user-selected tickers and watchlists.

## Hard Boundary

This skill is not the existing `finance` skill and not the finvesto professional
investment pipeline.

Do not call:
- `invest_portfolio`
- finvesto alpha prediction
- finvesto optimizer
- finvesto executor
- finvesto professional report tools

Do not provide buy/sell, rebalance, or portfolio optimization advice.

Use this skill for observation: daily metrics, trendlines, trend tables,
indicator evolution, anomaly explanations, and key price levels.

If the user asks an ambiguous question such as "TSLA 怎么样", ask:
"你想看指标趋势监控，还是投资/调仓建议？"

## Data Contract

Read and write only under:

`/Users/hanlianlyu/.nanobot-wife/workspace/data/market-metrics-monitor/`

The monitor may read finvesto credentials or raw price cache as read-only source
material. It must not write finvesto files and must not present monitor metrics
as finvesto model output.

## Workflows

Add monitoring:

```text
update_metrics.py add-watchlist wife_core AAPL MSFT NVDA
update_metrics.py update-all
```

Single ticker card:

```text
update_metrics.py render AAPL --kind ticker
```

Watchlist card:

```text
update_metrics.py render wife_core --kind watchlist
```

## Response Style

Default language is Chinese. Send one concise text summary plus one integrated
PNG card. Attach the HTML file only when the user asks for the interactive or
desktop version.

Always show:
- `as_of`
- data source
- data quality flags when present
- short explanations for displayed indicators

End with: "这是指标监控，不是 finvesto 模型建议。"
```

Create `references/metric_glossary.md`:

```markdown
# Metric Glossary

## MA5/20/50/200
Moving averages smooth price. Short averages above long averages usually mean
the trend structure is stronger.

## RSI-14
RSI measures short-term momentum pressure. Above 70 is hot; below 30 is cold.

## MACD
MACD compares fast and slow moving averages to show momentum acceleration or
deceleration.

## 20D Volatility
Recent price swing intensity. Higher volatility means price movement is less
stable.

## Volume Ratio 20D
Current volume divided by 20-day average volume. High values mean attention or
participation has increased.

## Drawdown From 52W High
How far the price is below its 52-week high.

## Relative Strength vs SPY
Whether the ticker has outperformed or underperformed the broad US market.
```

Create `scripts/update_metrics.py`:

```python
# /// script
# requires-python = ">=3.11"
# dependencies = [
#   "pandas>=2.2",
#   "numpy>=1.26",
#   "pyarrow>=15",
#   "httpx>=0.28",
#   "jinja2>=3.1",
#   "playwright>=1.45",
#   "python-dotenv>=1.0",
# ]
# ///

from market_metrics_monitor.cli import main


if __name__ == "__main__":
    raise SystemExit(main())
```

Create `scripts/market_metrics_monitor/__init__.py`:

```python
"""Independent wife-instance US stock metrics monitor."""

__all__ = ["__version__"]
__version__ = "0.1.0"
```

Create `scripts/tests/conftest.py`:

```python
from pathlib import Path

import pytest


@pytest.fixture
def temp_monitor_root(tmp_path: Path) -> Path:
    root = tmp_path / "market-metrics-monitor"
    root.mkdir()
    return root
```

- [ ] **Step 5: Run the skeleton test and verify it passes**

Run:

```bash
cd /Users/hanlianlyu/.nanobot-wife/workspace/skills/market-metrics-monitor
PYTHONPATH=scripts uv run --with pytest --with pyyaml python -m pytest scripts/tests/test_skill_skeleton.py -q
```

Expected: PASS.

- [ ] **Step 6: Commit**

Run:

```bash
cd /Users/hanlianlyu/.nanobot-wife/workspace
git add skills/market-metrics-monitor
git commit -m "feat: add market metrics monitor skill skeleton"
```

Expected: commit succeeds with no AI attribution or co-author trailer.

## Task 2: Paths, Config, and Atomic Writes

**Files:**
- Create: `/Users/hanlianlyu/.nanobot-wife/workspace/skills/market-metrics-monitor/scripts/market_metrics_monitor/paths.py`
- Create: `/Users/hanlianlyu/.nanobot-wife/workspace/skills/market-metrics-monitor/scripts/market_metrics_monitor/config.py`
- Create: `/Users/hanlianlyu/.nanobot-wife/workspace/skills/market-metrics-monitor/scripts/market_metrics_monitor/atomic.py`
- Create: `/Users/hanlianlyu/.nanobot-wife/workspace/skills/market-metrics-monitor/scripts/tests/test_storage.py`

- [ ] **Step 1: Write failing tests for paths and atomic writes**

Create `scripts/tests/test_storage.py`:

```python
import json
from pathlib import Path

import pandas as pd

from market_metrics_monitor.atomic import atomic_write_json, atomic_write_parquet
from market_metrics_monitor.config import ProviderConfig, load_provider_config
from market_metrics_monitor.paths import MonitorPaths


def test_monitor_paths_create_expected_directories(temp_monitor_root: Path):
    paths = MonitorPaths(temp_monitor_root)
    paths.ensure()
    assert paths.registry_dir.is_dir()
    assert paths.prices_dir.is_dir()
    assert paths.metrics_dir.is_dir()
    assert paths.anomalies_dir.is_dir()
    assert paths.snapshots_dir.is_dir()
    assert paths.cards_dir.is_dir()
    assert paths.runs_dir.is_dir()


def test_atomic_json_write_replaces_file(temp_monitor_root: Path):
    target = temp_monitor_root / "sample.json"
    atomic_write_json(target, {"a": 1})
    atomic_write_json(target, {"a": 2})
    assert json.loads(target.read_text(encoding="utf-8")) == {"a": 2}
    assert not list(temp_monitor_root.glob("*.tmp"))


def test_atomic_parquet_write_replaces_file(temp_monitor_root: Path):
    target = temp_monitor_root / "sample.parquet"
    atomic_write_parquet(target, pd.DataFrame([{"symbol": "AAPL", "close": 10.0}]))
    df = pd.read_parquet(target)
    assert df.to_dict("records") == [{"symbol": "AAPL", "close": 10.0}]


def test_provider_config_uses_defaults_when_file_missing(temp_monitor_root: Path):
    cfg = load_provider_config(temp_monitor_root / "missing.json")
    assert isinstance(cfg, ProviderConfig)
    assert cfg.priority[0] == "alpaca"
    assert "stooq" in cfg.priority
```

- [ ] **Step 2: Run tests and verify they fail**

Run:

```bash
cd /Users/hanlianlyu/.nanobot-wife/workspace/skills/market-metrics-monitor
PYTHONPATH=scripts uv run --with pytest --with pandas --with pyarrow python -m pytest scripts/tests/test_storage.py -q
```

Expected: FAIL because modules are not implemented.

- [ ] **Step 3: Implement paths, config, and atomic writes**

Create `scripts/market_metrics_monitor/paths.py`:

```python
from __future__ import annotations

from dataclasses import dataclass
from pathlib import Path


DEFAULT_ROOT = Path.home() / ".nanobot-wife/workspace/data/market-metrics-monitor"


@dataclass(frozen=True)
class MonitorPaths:
    root: Path = DEFAULT_ROOT

    @property
    def registry_dir(self) -> Path:
        return self.root / "registry"

    @property
    def us_dir(self) -> Path:
        return self.root / "us"

    @property
    def prices_dir(self) -> Path:
        return self.us_dir / "prices"

    @property
    def metrics_dir(self) -> Path:
        return self.us_dir / "metrics"

    @property
    def anomalies_dir(self) -> Path:
        return self.us_dir / "anomalies"

    @property
    def snapshots_dir(self) -> Path:
        return self.us_dir / "snapshots"

    @property
    def cards_dir(self) -> Path:
        return self.us_dir / "cards"

    @property
    def runs_dir(self) -> Path:
        return self.root / "runs"

    def ensure(self) -> None:
        for path in (
            self.registry_dir,
            self.prices_dir,
            self.metrics_dir,
            self.anomalies_dir,
            self.snapshots_dir,
            self.cards_dir,
            self.runs_dir,
        ):
            path.mkdir(parents=True, exist_ok=True)
```

Create `scripts/market_metrics_monitor/config.py`:

```python
from __future__ import annotations

import json
from dataclasses import dataclass, field
from pathlib import Path
from typing import Any


@dataclass(frozen=True)
class ProviderConfig:
    priority: list[str] = field(
        default_factory=lambda: ["alpaca", "stooq", "finvesto_raw_cache"]
    )
    credential_refs: dict[str, dict[str, Any]] = field(default_factory=dict)


def load_provider_config(path: Path) -> ProviderConfig:
    if not path.exists():
        return ProviderConfig()
    payload = json.loads(path.read_text(encoding="utf-8"))
    return ProviderConfig(
        priority=list(payload.get("priority") or ProviderConfig().priority),
        credential_refs=dict(payload.get("credential_refs") or {}),
    )
```

Create `scripts/market_metrics_monitor/atomic.py`:

```python
from __future__ import annotations

import json
import os
import tempfile
from pathlib import Path
from typing import Any

import pandas as pd


def _replace_bytes(path: Path, data: bytes) -> None:
    path.parent.mkdir(parents=True, exist_ok=True)
    fd, tmp_name = tempfile.mkstemp(prefix=f".{path.name}.", suffix=".tmp", dir=path.parent)
    try:
        with os.fdopen(fd, "wb") as handle:
            handle.write(data)
            handle.flush()
            os.fsync(handle.fileno())
        os.replace(tmp_name, path)
    finally:
        tmp = Path(tmp_name)
        if tmp.exists():
            tmp.unlink()


def atomic_write_json(path: Path, payload: Any) -> None:
    data = json.dumps(payload, ensure_ascii=False, indent=2, sort_keys=True).encode("utf-8")
    _replace_bytes(path, data)


def atomic_write_parquet(path: Path, frame: pd.DataFrame) -> None:
    path.parent.mkdir(parents=True, exist_ok=True)
    fd, tmp_name = tempfile.mkstemp(prefix=f".{path.name}.", suffix=".tmp", dir=path.parent)
    os.close(fd)
    tmp = Path(tmp_name)
    try:
        frame.to_parquet(tmp, index=False)
        os.replace(tmp, path)
    finally:
        if tmp.exists():
            tmp.unlink()
```

- [ ] **Step 4: Run tests and verify they pass**

Run:

```bash
cd /Users/hanlianlyu/.nanobot-wife/workspace/skills/market-metrics-monitor
PYTHONPATH=scripts uv run --with pytest --with pandas --with pyarrow python -m pytest scripts/tests/test_storage.py -q
```

Expected: PASS.

- [ ] **Step 5: Commit**

Run:

```bash
cd /Users/hanlianlyu/.nanobot-wife/workspace
git add skills/market-metrics-monitor
git commit -m "feat: add monitor storage primitives"
```

Expected: commit succeeds with no AI attribution or co-author trailer.

## Task 3: Registry Manager

**Files:**
- Create: `/Users/hanlianlyu/.nanobot-wife/workspace/skills/market-metrics-monitor/scripts/market_metrics_monitor/registry.py`
- Create: `/Users/hanlianlyu/.nanobot-wife/workspace/skills/market-metrics-monitor/scripts/tests/test_registry.py`

- [ ] **Step 1: Write failing registry tests**

Create `scripts/tests/test_registry.py`:

```python
from pathlib import Path

from market_metrics_monitor.paths import MonitorPaths
from market_metrics_monitor.registry import Registry


def test_add_watchlist_registers_symbols(temp_monitor_root: Path):
    paths = MonitorPaths(temp_monitor_root)
    paths.ensure()
    reg = Registry(paths)
    reg.add_watchlist("wife_core", ["aapl", "MSFT", " nvda "])
    assert reg.load_watchlists()["wife_core"]["symbols"] == ["AAPL", "MSFT", "NVDA"]
    tickers = reg.load_tickers()
    assert set(tickers) == {"AAPL", "MSFT", "NVDA"}
    assert tickers["AAPL"]["market"] == "us"
    assert tickers["AAPL"]["active"] is True


def test_remove_symbol_deactivates_ticker_when_unused(temp_monitor_root: Path):
    paths = MonitorPaths(temp_monitor_root)
    paths.ensure()
    reg = Registry(paths)
    reg.add_watchlist("wife_core", ["AAPL", "MSFT"])
    reg.remove_from_watchlist("wife_core", ["AAPL"])
    assert reg.load_watchlists()["wife_core"]["symbols"] == ["MSFT"]
    assert reg.load_tickers()["AAPL"]["active"] is False
    assert reg.load_tickers()["MSFT"]["active"] is True


def test_active_symbols_are_unique_across_watchlists(temp_monitor_root: Path):
    paths = MonitorPaths(temp_monitor_root)
    paths.ensure()
    reg = Registry(paths)
    reg.add_watchlist("core", ["AAPL", "MSFT"])
    reg.add_watchlist("ai", ["MSFT", "NVDA"])
    assert reg.active_symbols() == ["AAPL", "MSFT", "NVDA"]
```

- [ ] **Step 2: Run tests and verify they fail**

Run:

```bash
cd /Users/hanlianlyu/.nanobot-wife/workspace/skills/market-metrics-monitor
PYTHONPATH=scripts uv run --with pytest --with pandas --with pyarrow python -m pytest scripts/tests/test_registry.py -q
```

Expected: FAIL because `registry.py` is missing.

- [ ] **Step 3: Implement the registry**

Create `scripts/market_metrics_monitor/registry.py`:

```python
from __future__ import annotations

import json
from datetime import datetime
from pathlib import Path

from market_metrics_monitor.atomic import atomic_write_json
from market_metrics_monitor.paths import MonitorPaths


def normalize_symbol(symbol: str) -> str:
    return symbol.strip().upper()


def _now() -> str:
    return datetime.now().astimezone().isoformat(timespec="seconds")


class Registry:
    def __init__(self, paths: MonitorPaths):
        self.paths = paths
        self.paths.ensure()

    @property
    def tickers_path(self) -> Path:
        return self.paths.registry_dir / "tickers.json"

    @property
    def watchlists_path(self) -> Path:
        return self.paths.registry_dir / "watchlists.json"

    def load_tickers(self) -> dict:
        if not self.tickers_path.exists():
            return {}
        return json.loads(self.tickers_path.read_text(encoding="utf-8"))

    def load_watchlists(self) -> dict:
        if not self.watchlists_path.exists():
            return {}
        return json.loads(self.watchlists_path.read_text(encoding="utf-8"))

    def save_tickers(self, tickers: dict) -> None:
        atomic_write_json(self.tickers_path, tickers)

    def save_watchlists(self, watchlists: dict) -> None:
        atomic_write_json(self.watchlists_path, watchlists)

    def add_watchlist(self, name: str, symbols: list[str]) -> None:
        clean = sorted({normalize_symbol(s) for s in symbols if normalize_symbol(s)})
        if not clean:
            raise ValueError("watchlist must contain at least one symbol")

        tickers = self.load_tickers()
        watchlists = self.load_watchlists()
        now = _now()

        for symbol in clean:
            entry = tickers.get(symbol, {})
            entry.update(
                {
                    "symbol": symbol,
                    "market": "us",
                    "active": True,
                    "source": entry.get("source", "user"),
                    "added_at": entry.get("added_at", now),
                }
            )
            tickers[symbol] = entry

        watchlists[name] = {
            "market": "us",
            "symbols": clean,
            "active": True,
            "created_at": watchlists.get(name, {}).get("created_at", now),
            "updated_at": now,
        }

        self.save_tickers(tickers)
        self.save_watchlists(watchlists)

    def remove_from_watchlist(self, name: str, symbols: list[str]) -> None:
        remove = {normalize_symbol(s) for s in symbols}
        watchlists = self.load_watchlists()
        if name not in watchlists:
            raise KeyError(f"unknown watchlist: {name}")
        kept = [s for s in watchlists[name]["symbols"] if s not in remove]
        watchlists[name]["symbols"] = kept
        watchlists[name]["updated_at"] = _now()
        self.save_watchlists(watchlists)
        self._deactivate_unused_symbols()

    def _deactivate_unused_symbols(self) -> None:
        watchlists = self.load_watchlists()
        used = {
            symbol
            for entry in watchlists.values()
            if entry.get("active", True)
            for symbol in entry.get("symbols", [])
        }
        tickers = self.load_tickers()
        for symbol, entry in tickers.items():
            entry["active"] = symbol in used
        self.save_tickers(tickers)

    def active_symbols(self) -> list[str]:
        watchlists = self.load_watchlists()
        symbols = {
            symbol
            for entry in watchlists.values()
            if entry.get("active", True)
            for symbol in entry.get("symbols", [])
        }
        return sorted(symbols)
```

- [ ] **Step 4: Run tests and verify they pass**

Run:

```bash
cd /Users/hanlianlyu/.nanobot-wife/workspace/skills/market-metrics-monitor
PYTHONPATH=scripts uv run --with pytest --with pandas --with pyarrow python -m pytest scripts/tests/test_registry.py -q
```

Expected: PASS.

- [ ] **Step 5: Commit**

Run:

```bash
cd /Users/hanlianlyu/.nanobot-wife/workspace
git add skills/market-metrics-monitor
git commit -m "feat: add market monitor registry"
```

Expected: commit succeeds with no AI attribution or co-author trailer.

## Task 4: Price Storage and Upsert

**Files:**
- Create: `/Users/hanlianlyu/.nanobot-wife/workspace/skills/market-metrics-monitor/scripts/market_metrics_monitor/storage.py`
- Modify: `/Users/hanlianlyu/.nanobot-wife/workspace/skills/market-metrics-monitor/scripts/tests/test_storage.py`

- [ ] **Step 1: Add failing price storage tests**

Append to `scripts/tests/test_storage.py`:

```python
from market_metrics_monitor.storage import PriceStore


def test_price_store_upserts_by_date_and_symbol(temp_monitor_root: Path):
    paths = MonitorPaths(temp_monitor_root)
    paths.ensure()
    store = PriceStore(paths)
    first = pd.DataFrame(
        [
            {"date": "2026-05-01", "symbol": "AAPL", "close": 100.0, "volume": 10},
            {"date": "2026-05-02", "symbol": "AAPL", "close": 101.0, "volume": 11},
        ]
    )
    second = pd.DataFrame(
        [
            {"date": "2026-05-02", "symbol": "AAPL", "close": 102.0, "volume": 12},
            {"date": "2026-05-03", "symbol": "AAPL", "close": 103.0, "volume": 13},
        ]
    )
    store.upsert_prices("AAPL", first)
    store.upsert_prices("AAPL", second)
    rows = store.read_prices("AAPL")[["date", "symbol", "close", "volume"]].to_dict("records")
    assert rows == [
        {"date": "2026-05-01", "symbol": "AAPL", "close": 100.0, "volume": 10},
        {"date": "2026-05-02", "symbol": "AAPL", "close": 102.0, "volume": 12},
        {"date": "2026-05-03", "symbol": "AAPL", "close": 103.0, "volume": 13},
    ]
```

- [ ] **Step 2: Run the new test and verify it fails**

Run:

```bash
cd /Users/hanlianlyu/.nanobot-wife/workspace/skills/market-metrics-monitor
PYTHONPATH=scripts uv run --with pytest --with pandas --with pyarrow python -m pytest scripts/tests/test_storage.py::test_price_store_upserts_by_date_and_symbol -q
```

Expected: FAIL because `PriceStore` is missing.

- [ ] **Step 3: Implement price storage**

Create `scripts/market_metrics_monitor/storage.py`:

```python
from __future__ import annotations

from pathlib import Path

import pandas as pd

from market_metrics_monitor.atomic import atomic_write_parquet
from market_metrics_monitor.paths import MonitorPaths


PRICE_COLUMNS = [
    "date",
    "symbol",
    "open",
    "high",
    "low",
    "close",
    "adjusted_close",
    "volume",
    "currency",
    "provider",
    "provider_symbol",
    "fetched_at",
    "quality_flags",
]


class PriceStore:
    def __init__(self, paths: MonitorPaths):
        self.paths = paths
        self.paths.ensure()

    def price_path(self, symbol: str) -> Path:
        return self.paths.prices_dir / f"{symbol.upper()}.parquet"

    def metrics_path(self, symbol: str) -> Path:
        return self.paths.metrics_dir / f"{symbol.upper()}.parquet"

    def read_prices(self, symbol: str) -> pd.DataFrame:
        path = self.price_path(symbol)
        if not path.exists():
            return pd.DataFrame(columns=PRICE_COLUMNS)
        frame = pd.read_parquet(path)
        return frame.sort_values(["date", "symbol"]).reset_index(drop=True)

    def upsert_prices(self, symbol: str, frame: pd.DataFrame) -> pd.DataFrame:
        symbol = symbol.upper()
        incoming = frame.copy()
        incoming["symbol"] = symbol
        if "date" not in incoming:
            raise ValueError("price frame missing date column")
        existing = self.read_prices(symbol)
        merged = pd.concat([existing, incoming], ignore_index=True)
        merged = merged.drop_duplicates(["date", "symbol"], keep="last")
        merged = merged.sort_values(["date", "symbol"]).reset_index(drop=True)
        atomic_write_parquet(self.price_path(symbol), merged)
        return merged

    def write_metrics(self, symbol: str, frame: pd.DataFrame) -> None:
        atomic_write_parquet(self.metrics_path(symbol), frame.sort_values("date").reset_index(drop=True))

    def read_metrics(self, symbol: str) -> pd.DataFrame:
        path = self.metrics_path(symbol)
        if not path.exists():
            return pd.DataFrame()
        return pd.read_parquet(path).sort_values("date").reset_index(drop=True)
```

- [ ] **Step 4: Run storage tests and verify they pass**

Run:

```bash
cd /Users/hanlianlyu/.nanobot-wife/workspace/skills/market-metrics-monitor
PYTHONPATH=scripts uv run --with pytest --with pandas --with pyarrow python -m pytest scripts/tests/test_storage.py -q
```

Expected: PASS.

- [ ] **Step 5: Commit**

Run:

```bash
cd /Users/hanlianlyu/.nanobot-wife/workspace
git add skills/market-metrics-monitor
git commit -m "feat: add market monitor price storage"
```

Expected: commit succeeds with no AI attribution or co-author trailer.

## Task 5: Provider Adapters

**Files:**
- Create: `/Users/hanlianlyu/.nanobot-wife/workspace/skills/market-metrics-monitor/scripts/market_metrics_monitor/providers.py`
- Create: `/Users/hanlianlyu/.nanobot-wife/workspace/skills/market-metrics-monitor/scripts/tests/test_providers.py`

- [ ] **Step 1: Write failing provider tests**

Create `scripts/tests/test_providers.py`:

```python
from datetime import date

import httpx
import pandas as pd
import pytest

from market_metrics_monitor.providers import (
    AlpacaProvider,
    FinvestoRawCacheProvider,
    ProviderChain,
    ProviderResult,
    StooqProvider,
    parse_env_file,
)


def test_parse_env_file_reads_alpaca_keys(tmp_path):
    env = tmp_path / ".env"
    env.write_text("APCA_API_KEY_ID=key\nAPCA_API_SECRET_KEY=secret\n", encoding="utf-8")
    assert parse_env_file(env)["APCA_API_KEY_ID"] == "key"
    assert parse_env_file(env)["APCA_API_SECRET_KEY"] == "secret"


def test_stooq_provider_parses_csv(monkeypatch):
    csv = "Date,Open,High,Low,Close,Volume\n2026-05-01,10,11,9,10.5,1000\n"

    def handler(request: httpx.Request) -> httpx.Response:
        assert "aapl.us" in str(request.url)
        return httpx.Response(200, text=csv)

    provider = StooqProvider(client=httpx.Client(transport=httpx.MockTransport(handler)))
    frame = provider.fetch_daily("AAPL", start=date(2026, 5, 1), end=date(2026, 5, 2))
    assert frame[["date", "symbol", "close", "volume", "provider"]].to_dict("records") == [
        {
            "date": "2026-05-01",
            "symbol": "AAPL",
            "close": 10.5,
            "volume": 1000,
            "provider": "stooq",
        }
    ]


def test_provider_chain_uses_first_non_empty_provider():
    class EmptyProvider:
        name = "empty"

        def fetch_daily(self, symbol, start, end):
            return pd.DataFrame()

    class GoodProvider:
        name = "good"

        def fetch_daily(self, symbol, start, end):
            return pd.DataFrame([{"date": "2026-05-01", "symbol": symbol, "close": 10.0}])

    chain = ProviderChain([EmptyProvider(), GoodProvider()])
    result = chain.fetch_daily("AAPL", start=date(2026, 5, 1), end=date(2026, 5, 2))
    assert isinstance(result, ProviderResult)
    assert result.provider == "good"
    assert len(result.frame) == 1


def test_alpaca_provider_returns_empty_when_credentials_missing():
    provider = AlpacaProvider(env={})
    frame = provider.fetch_daily("AAPL", start=date(2026, 5, 1), end=date(2026, 5, 2))
    assert frame.empty


def test_finvesto_raw_cache_provider_reads_explicit_parquet_dir(tmp_path):
    root = tmp_path / "raw"
    root.mkdir()
    pd.DataFrame(
        [
            {"date": "2026-05-01", "open": 10.0, "high": 11.0, "low": 9.0, "close": 10.5, "volume": 1000},
            {"date": "2026-05-02", "open": 11.0, "high": 12.0, "low": 10.0, "close": 11.5, "volume": 1200},
        ]
    ).to_parquet(root / "AAPL.parquet", index=False)
    provider = FinvestoRawCacheProvider(root)
    frame = provider.fetch_daily("AAPL", start=date(2026, 5, 2), end=date(2026, 5, 3))
    assert frame[["date", "symbol", "close", "provider"]].to_dict("records") == [
        {"date": "2026-05-02", "symbol": "AAPL", "close": 11.5, "provider": "finvesto_raw_cache"}
    ]
```

- [ ] **Step 2: Run provider tests and verify they fail**

Run:

```bash
cd /Users/hanlianlyu/.nanobot-wife/workspace/skills/market-metrics-monitor
PYTHONPATH=scripts uv run --with pytest --with pandas --with httpx python -m pytest scripts/tests/test_providers.py -q
```

Expected: FAIL because `providers.py` is missing.

- [ ] **Step 3: Implement providers**

Create `scripts/market_metrics_monitor/providers.py`:

```python
from __future__ import annotations

import csv
import io
from dataclasses import dataclass
from datetime import date, datetime
from pathlib import Path
from typing import Protocol

import httpx
import pandas as pd


class DailyProvider(Protocol):
    name: str

    def fetch_daily(self, symbol: str, start: date, end: date) -> pd.DataFrame:
        raise NotImplementedError


@dataclass(frozen=True)
class ProviderResult:
    provider: str
    frame: pd.DataFrame


def parse_env_file(path: Path) -> dict[str, str]:
    if not path.exists():
        return {}
    result: dict[str, str] = {}
    for raw in path.read_text(encoding="utf-8").splitlines():
        line = raw.strip()
        if not line or line.startswith("#") or "=" not in line:
            continue
        key, value = line.split("=", 1)
        result[key.strip()] = value.strip().strip('"').strip("'")
    return result


def _standardize(frame: pd.DataFrame, symbol: str, provider: str, provider_symbol: str) -> pd.DataFrame:
    if frame.empty:
        return frame
    out = frame.copy()
    out["symbol"] = symbol.upper()
    out["currency"] = "USD"
    out["provider"] = provider
    out["provider_symbol"] = provider_symbol
    out["fetched_at"] = datetime.now().astimezone().isoformat(timespec="seconds")
    out["quality_flags"] = [[] for _ in range(len(out))]
    if "adjusted_close" not in out:
        out["adjusted_close"] = out["close"]
    cols = [
        "date",
        "symbol",
        "open",
        "high",
        "low",
        "close",
        "adjusted_close",
        "volume",
        "currency",
        "provider",
        "provider_symbol",
        "fetched_at",
        "quality_flags",
    ]
    return out[cols]


class StooqProvider:
    name = "stooq"

    def __init__(self, client: httpx.Client | None = None):
        self.client = client or httpx.Client(timeout=30)

    def fetch_daily(self, symbol: str, start: date, end: date) -> pd.DataFrame:
        provider_symbol = f"{symbol.lower()}.us"
        url = "https://stooq.com/q/d/l/"
        resp = self.client.get(
            url,
            params={"s": provider_symbol, "d1": start.strftime("%Y%m%d"), "d2": end.strftime("%Y%m%d"), "i": "d"},
        )
        resp.raise_for_status()
        rows = list(csv.DictReader(io.StringIO(resp.text)))
        if not rows or "Date" not in rows[0]:
            return pd.DataFrame()
        frame = pd.DataFrame(rows)
        if frame.empty:
            return frame
        frame = frame.rename(
            columns={
                "Date": "date",
                "Open": "open",
                "High": "high",
                "Low": "low",
                "Close": "close",
                "Volume": "volume",
            }
        )
        for col in ["open", "high", "low", "close", "volume"]:
            frame[col] = pd.to_numeric(frame[col], errors="coerce")
        frame = frame.dropna(subset=["date", "close"])
        return _standardize(frame, symbol, self.name, provider_symbol)


class AlpacaProvider:
    name = "alpaca"

    def __init__(self, env: dict[str, str], client: httpx.Client | None = None):
        self.env = env
        self.client = client or httpx.Client(timeout=30)

    def fetch_daily(self, symbol: str, start: date, end: date) -> pd.DataFrame:
        key = self.env.get("APCA_API_KEY_ID") or self.env.get("ALPACA_API_KEY")
        secret = self.env.get("APCA_API_SECRET_KEY") or self.env.get("ALPACA_SECRET_KEY")
        if not key or not secret:
            return pd.DataFrame()
        resp = self.client.get(
            "https://data.alpaca.markets/v2/stocks/bars",
            params={
                "symbols": symbol.upper(),
                "timeframe": "1Day",
                "start": start.isoformat(),
                "end": end.isoformat(),
                "adjustment": "all",
                "feed": "iex",
            },
            headers={"APCA-API-KEY-ID": key, "APCA-API-SECRET-KEY": secret},
        )
        resp.raise_for_status()
        payload = resp.json()
        bars = payload.get("bars", {}).get(symbol.upper(), [])
        rows = [
            {
                "date": item["t"][:10],
                "open": item["o"],
                "high": item["h"],
                "low": item["l"],
                "close": item["c"],
                "adjusted_close": item["c"],
                "volume": item["v"],
            }
            for item in bars
        ]
        return _standardize(pd.DataFrame(rows), symbol, self.name, symbol.upper())


class FinvestoRawCacheProvider:
    name = "finvesto_raw_cache"

    def __init__(self, root: Path | None = None):
        self.root = root

    def fetch_daily(self, symbol: str, start: date, end: date) -> pd.DataFrame:
        if self.root is None:
            return pd.DataFrame()
        path = self.root / f"{symbol.upper()}.parquet"
        if not path.exists():
            return pd.DataFrame()
        frame = pd.read_parquet(path)
        if frame.empty:
            return frame
        frame["date"] = pd.to_datetime(frame["date"]).dt.strftime("%Y-%m-%d")
        start_s = start.isoformat()
        end_s = end.isoformat()
        frame = frame[(frame["date"] >= start_s) & (frame["date"] <= end_s)]
        for col in ["open", "high", "low", "close", "volume"]:
            if col not in frame:
                frame[col] = pd.NA
        return _standardize(frame, symbol, self.name, symbol.upper())


class ProviderChain:
    def __init__(self, providers: list[DailyProvider]):
        self.providers = providers

    def fetch_daily(self, symbol: str, start: date, end: date) -> ProviderResult:
        errors: list[str] = []
        for provider in self.providers:
            try:
                frame = provider.fetch_daily(symbol, start, end)
            except Exception as exc:
                errors.append(f"{provider.name}: {exc}")
                continue
            if not frame.empty:
                return ProviderResult(provider=provider.name, frame=frame)
        empty = pd.DataFrame()
        empty.attrs["errors"] = errors
        return ProviderResult(provider="", frame=empty)
```

- [ ] **Step 4: Run provider tests and verify they pass**

Run:

```bash
cd /Users/hanlianlyu/.nanobot-wife/workspace/skills/market-metrics-monitor
PYTHONPATH=scripts uv run --with pytest --with pandas --with httpx python -m pytest scripts/tests/test_providers.py -q
```

Expected: PASS.

- [ ] **Step 5: Commit**

Run:

```bash
cd /Users/hanlianlyu/.nanobot-wife/workspace
git add skills/market-metrics-monitor
git commit -m "feat: add monitor market data providers"
```

Expected: commit succeeds with no AI attribution or co-author trailer.

## Task 6: Metrics Engine

**Files:**
- Create: `/Users/hanlianlyu/.nanobot-wife/workspace/skills/market-metrics-monitor/scripts/market_metrics_monitor/metrics.py`
- Create: `/Users/hanlianlyu/.nanobot-wife/workspace/skills/market-metrics-monitor/scripts/tests/test_metrics.py`

- [ ] **Step 1: Write failing metrics tests**

Create `scripts/tests/test_metrics.py`:

```python
import pandas as pd

from market_metrics_monitor.metrics import compute_metrics


def sample_prices(n=260):
    dates = pd.date_range("2025-01-01", periods=n, freq="B")
    close = pd.Series(range(100, 100 + n), dtype="float")
    return pd.DataFrame(
        {
            "date": dates.strftime("%Y-%m-%d"),
            "symbol": "AAPL",
            "open": close - 0.5,
            "high": close + 1,
            "low": close - 1,
            "close": close,
            "adjusted_close": close,
            "volume": [1_000_000 + i * 100 for i in range(n)],
        }
    )


def test_compute_metrics_has_expected_columns():
    metrics = compute_metrics(sample_prices())
    expected = {
        "ma_5",
        "ma_20",
        "ma_50",
        "ma_200",
        "rsi_14",
        "macd",
        "macd_signal",
        "volatility_20d",
        "volume_ratio_20d",
        "drawdown_from_52w_high",
        "trend_slope_20d",
        "trend_slope_60d",
        "price_action_state",
        "quality_flags",
    }
    assert expected.issubset(metrics.columns)


def test_compute_metrics_marks_short_history():
    metrics = compute_metrics(sample_prices(30))
    flags = metrics.iloc[-1]["quality_flags"]
    assert "insufficient_history_ma200" in flags
    assert "insufficient_history_52w" in flags


def test_uptrend_series_gets_uptrend_state():
    metrics = compute_metrics(sample_prices(260))
    assert metrics.iloc[-1]["ma_stack_state"] == "bullish_stack"
    assert metrics.iloc[-1]["price_action_state"] in {"uptrend", "natural_rally"}
```

- [ ] **Step 2: Run metrics tests and verify they fail**

Run:

```bash
cd /Users/hanlianlyu/.nanobot-wife/workspace/skills/market-metrics-monitor
PYTHONPATH=scripts uv run --with pytest --with pandas --with numpy python -m pytest scripts/tests/test_metrics.py -q
```

Expected: FAIL because `metrics.py` is missing.

- [ ] **Step 3: Implement metrics**

Create `scripts/market_metrics_monitor/metrics.py`:

```python
from __future__ import annotations

import numpy as np
import pandas as pd


def _rsi(close: pd.Series, window: int = 14) -> pd.Series:
    delta = close.diff()
    gain = delta.clip(lower=0).rolling(window).mean()
    loss = (-delta.clip(upper=0)).rolling(window).mean()
    rs = gain / loss.replace(0, np.nan)
    rsi = 100 - (100 / (1 + rs))
    return rsi.fillna(50)


def _slope(series: pd.Series, window: int) -> pd.Series:
    def calc(values):
        y = np.asarray(values, dtype=float)
        x = np.arange(len(y), dtype=float)
        if np.isnan(y).any() or len(y) < 2:
            return np.nan
        return float(np.polyfit(x, y, 1)[0])

    return series.rolling(window).apply(calc, raw=True)


def _ma_stack(row: pd.Series) -> str:
    if pd.isna(row["ma_200"]):
        return "insufficient_history"
    if row["ma_5"] > row["ma_20"] > row["ma_50"] > row["ma_200"]:
        return "bullish_stack"
    if row["ma_5"] < row["ma_20"] < row["ma_50"] < row["ma_200"]:
        return "bearish_stack"
    return "mixed"


def _price_action(row: pd.Series) -> str:
    if row["ma_stack_state"] == "bullish_stack" and row["trend_slope_20d"] > 0:
        return "uptrend"
    if row["ma_stack_state"] == "bearish_stack" and row["trend_slope_20d"] < 0:
        return "downtrend"
    if row["trend_slope_20d"] > 0 and row["trend_slope_60d"] <= 0:
        return "secondary_rally"
    if row["trend_slope_20d"] < 0 and row["trend_slope_60d"] >= 0:
        return "secondary_reaction"
    return "range_bound"


def compute_metrics(prices: pd.DataFrame, spy_metrics: pd.DataFrame | None = None) -> pd.DataFrame:
    frame = prices.copy()
    frame["date"] = pd.to_datetime(frame["date"]).dt.strftime("%Y-%m-%d")
    frame = frame.sort_values("date").reset_index(drop=True)
    close = frame["adjusted_close"].fillna(frame["close"]).astype(float)
    volume = frame.get("volume", pd.Series([np.nan] * len(frame))).astype(float)

    out = pd.DataFrame({"date": frame["date"], "symbol": frame["symbol"].str.upper(), "close": close})
    out["return_1d"] = close.pct_change()
    out["return_5d"] = close.pct_change(5)
    out["return_20d"] = close.pct_change(20)
    out["ma_5"] = close.rolling(5).mean()
    out["ma_20"] = close.rolling(20).mean()
    out["ma_50"] = close.rolling(50).mean()
    out["ma_200"] = close.rolling(200).mean()
    out["rsi_14"] = _rsi(close)
    ema_12 = close.ewm(span=12, adjust=False).mean()
    ema_26 = close.ewm(span=26, adjust=False).mean()
    out["macd"] = ema_12 - ema_26
    out["macd_signal"] = out["macd"].ewm(span=9, adjust=False).mean()
    out["macd_hist"] = out["macd"] - out["macd_signal"]
    out["volatility_20d"] = out["return_1d"].rolling(20).std() * np.sqrt(252)
    out["volume_ratio_20d"] = volume / volume.rolling(20).mean()
    out["drawdown_from_52w_high"] = close / close.rolling(252).max() - 1
    out["trend_slope_20d"] = _slope(close, 20)
    out["trend_slope_60d"] = _slope(close, 60)
    out["relative_strength_spy_20d"] = np.nan
    out["relative_strength_spy_60d"] = np.nan
    if spy_metrics is not None and not spy_metrics.empty:
        spy = spy_metrics.set_index("date")
        out["relative_strength_spy_20d"] = out["return_20d"] - out["date"].map(spy["return_20d"])
        out["relative_strength_spy_60d"] = close.pct_change(60) - out["date"].map(spy["close"]).pct_change(60)
    out["ma_stack_state"] = out.apply(_ma_stack, axis=1)
    out["price_action_state"] = out.apply(_price_action, axis=1)
    out["anomaly_score"] = 0.0

    flags = []
    for idx in range(len(out)):
        row_flags = []
        if idx < 199:
            row_flags.append("insufficient_history_ma200")
        if idx < 251:
            row_flags.append("insufficient_history_52w")
        if pd.isna(volume.iloc[idx]):
            row_flags.append("missing_volume")
        flags.append(row_flags)
    out["quality_flags"] = flags
    out["computed_at"] = pd.Timestamp.now(tz="UTC").isoformat()
    return out
```

- [ ] **Step 4: Run metrics tests and verify they pass**

Run:

```bash
cd /Users/hanlianlyu/.nanobot-wife/workspace/skills/market-metrics-monitor
PYTHONPATH=scripts uv run --with pytest --with pandas --with numpy python -m pytest scripts/tests/test_metrics.py -q
```

Expected: PASS.

- [ ] **Step 5: Commit**

Run:

```bash
cd /Users/hanlianlyu/.nanobot-wife/workspace
git add skills/market-metrics-monitor
git commit -m "feat: compute market monitor metrics"
```

Expected: commit succeeds with no AI attribution or co-author trailer.

## Task 7: Anomaly Detection

**Files:**
- Create: `/Users/hanlianlyu/.nanobot-wife/workspace/skills/market-metrics-monitor/scripts/market_metrics_monitor/anomalies.py`
- Create: `/Users/hanlianlyu/.nanobot-wife/workspace/skills/market-metrics-monitor/scripts/tests/test_anomalies.py`

- [ ] **Step 1: Write failing anomaly tests**

Create `scripts/tests/test_anomalies.py`:

```python
import pandas as pd

from market_metrics_monitor.anomalies import detect_anomalies


def test_detects_volume_spike_and_rsi_extreme():
    metrics = pd.DataFrame(
        [
            {"date": "2026-05-01", "symbol": "AAPL", "volume_ratio_20d": 1.0, "rsi_14": 50, "ma_stack_state": "mixed", "price_action_state": "range_bound"},
            {"date": "2026-05-02", "symbol": "AAPL", "volume_ratio_20d": 2.6, "rsi_14": 74, "ma_stack_state": "mixed", "price_action_state": "range_bound"},
        ]
    )
    events = detect_anomalies(metrics)
    assert {event["type"] for event in events} == {"volume_spike", "rsi_extreme"}
    assert all("explanation" in event for event in events)


def test_detects_trend_shift():
    metrics = pd.DataFrame(
        [
            {"date": "2026-05-01", "symbol": "AAPL", "volume_ratio_20d": 1.0, "rsi_14": 50, "ma_stack_state": "mixed", "price_action_state": "range_bound"},
            {"date": "2026-05-02", "symbol": "AAPL", "volume_ratio_20d": 1.0, "rsi_14": 55, "ma_stack_state": "bullish_stack", "price_action_state": "uptrend"},
        ]
    )
    events = detect_anomalies(metrics)
    assert any(event["type"] == "trend_shift" for event in events)
```

- [ ] **Step 2: Run anomaly tests and verify they fail**

Run:

```bash
cd /Users/hanlianlyu/.nanobot-wife/workspace/skills/market-metrics-monitor
PYTHONPATH=scripts uv run --with pytest --with pandas python -m pytest scripts/tests/test_anomalies.py -q
```

Expected: FAIL because `anomalies.py` is missing.

- [ ] **Step 3: Implement anomaly detection**

Create `scripts/market_metrics_monitor/anomalies.py`:

```python
from __future__ import annotations

import json
from pathlib import Path

import pandas as pd


def _event(row, event_type: str, severity: str, metric: str, value, explanation: str) -> dict:
    return {
        "date": row["date"],
        "symbol": row["symbol"],
        "type": event_type,
        "severity": severity,
        "metric": metric,
        "value": None if pd.isna(value) else float(value) if isinstance(value, (int, float)) else value,
        "explanation": explanation,
    }


def detect_anomalies(metrics: pd.DataFrame, lookback_rows: int = 2) -> list[dict]:
    if metrics.empty:
        return []
    recent = metrics.tail(lookback_rows).reset_index(drop=True)
    events: list[dict] = []
    latest = recent.iloc[-1]

    if latest.get("volume_ratio_20d", 0) >= 2.0:
        events.append(
            _event(
                latest,
                "volume_spike",
                "medium" if latest["volume_ratio_20d"] < 3 else "high",
                "volume_ratio_20d",
                latest["volume_ratio_20d"],
                "成交量显著高于20日均量，说明市场关注或交易参与度突然升高。",
            )
        )

    if latest.get("rsi_14", 50) >= 70 or latest.get("rsi_14", 50) <= 30:
        events.append(
            _event(
                latest,
                "rsi_extreme",
                "medium",
                "rsi_14",
                latest["rsi_14"],
                "RSI进入极端区间，短线动能可能偏热或偏冷。",
            )
        )

    if len(recent) >= 2 and recent.iloc[-2].get("price_action_state") != latest.get("price_action_state"):
        events.append(
            _event(
                latest,
                "trend_shift",
                "medium",
                "price_action_state",
                latest.get("price_action_state"),
                "价格行为状态发生切换，说明趋势结构和前一交易日不同。",
            )
        )

    return events


def append_events(path: Path, events: list[dict]) -> None:
    if not events:
        return
    path.parent.mkdir(parents=True, exist_ok=True)
    with path.open("a", encoding="utf-8") as handle:
        for event in events:
            handle.write(json.dumps(event, ensure_ascii=False, sort_keys=True) + "\n")
```

- [ ] **Step 4: Run anomaly tests and verify they pass**

Run:

```bash
cd /Users/hanlianlyu/.nanobot-wife/workspace/skills/market-metrics-monitor
PYTHONPATH=scripts uv run --with pytest --with pandas python -m pytest scripts/tests/test_anomalies.py -q
```

Expected: PASS.

- [ ] **Step 5: Commit**

Run:

```bash
cd /Users/hanlianlyu/.nanobot-wife/workspace
git add skills/market-metrics-monitor
git commit -m "feat: detect market monitor anomalies"
```

Expected: commit succeeds with no AI attribution or co-author trailer.

## Task 8: Snapshot Builder

**Files:**
- Create: `/Users/hanlianlyu/.nanobot-wife/workspace/skills/market-metrics-monitor/scripts/market_metrics_monitor/snapshots.py`
- Create: `/Users/hanlianlyu/.nanobot-wife/workspace/skills/market-metrics-monitor/scripts/tests/test_snapshots.py`

- [ ] **Step 1: Write failing snapshot tests**

Create `scripts/tests/test_snapshots.py`:

```python
import pandas as pd

from market_metrics_monitor.snapshots import build_ticker_snapshot, build_watchlist_snapshot


def test_build_ticker_snapshot_contains_headline_and_explanations():
    metrics = pd.DataFrame(
        [
            {
                "date": "2026-05-06",
                "symbol": "AAPL",
                "close": 100.0,
                "price_action_state": "uptrend",
                "rsi_14": 58.0,
                "volatility_20d": 0.22,
                "volume_ratio_20d": 1.4,
                "trend_slope_20d": 0.5,
                "trend_slope_60d": 0.3,
                "quality_flags": [],
            }
        ]
    )
    snap = build_ticker_snapshot("AAPL", metrics, [])
    assert snap["symbol"] == "AAPL"
    assert snap["as_of"] == "2026-05-06"
    assert "趋势" in snap["headline"]
    assert "rsi_14" in snap["metric_explanations"]


def test_build_watchlist_snapshot_summarizes_anomalies():
    ticker_snaps = [
        {"symbol": "AAPL", "as_of": "2026-05-06", "state": "uptrend", "recent_anomalies": [{"severity": "medium"}], "quality_flags": []},
        {"symbol": "MSFT", "as_of": "2026-05-06", "state": "range_bound", "recent_anomalies": [], "quality_flags": ["stale_data"]},
    ]
    snap = build_watchlist_snapshot("wife_core", ticker_snaps)
    assert snap["watchlist"] == "wife_core"
    assert snap["symbols_total"] == 2
    assert snap["anomalies_total"] == 1
    assert snap["quality_flags"] == ["stale_data"]
```

- [ ] **Step 2: Run snapshot tests and verify they fail**

Run:

```bash
cd /Users/hanlianlyu/.nanobot-wife/workspace/skills/market-metrics-monitor
PYTHONPATH=scripts uv run --with pytest --with pandas python -m pytest scripts/tests/test_snapshots.py -q
```

Expected: FAIL because `snapshots.py` is missing.

- [ ] **Step 3: Implement snapshots**

Create `scripts/market_metrics_monitor/snapshots.py`:

```python
from __future__ import annotations

import pandas as pd


METRIC_EXPLANATIONS = {
    "rsi_14": "RSI衡量短线动能压力，70以上偏热，30以下偏冷。",
    "macd": "MACD观察动能加速或减速。",
    "volatility_20d": "20日波动率衡量最近价格摆动强度。",
    "volume_ratio_20d": "成交量比率显示当前成交量相对20日均量是否异常。",
    "ma_stack_state": "均线结构显示短中长期趋势排列。",
}


def _trend_word(value: float) -> str:
    if pd.isna(value):
        return "不明确"
    if value > 0:
        return "上行"
    if value < 0:
        return "下行"
    return "走平"


def build_ticker_snapshot(symbol: str, metrics: pd.DataFrame, anomalies: list[dict]) -> dict:
    if metrics.empty:
        return {
            "symbol": symbol.upper(),
            "as_of": None,
            "state": "no_data",
            "headline": "暂无足够数据生成趋势监控。",
            "key_metrics": [],
            "trendlines": [],
            "recent_anomalies": [],
            "metric_explanations": METRIC_EXPLANATIONS,
            "quality_flags": ["no_data"],
        }
    latest = metrics.iloc[-1]
    quality_flags = list(latest.get("quality_flags") or [])
    state = latest.get("price_action_state", "uncertain")
    trend20 = _trend_word(float(latest.get("trend_slope_20d", 0)))
    trend60 = _trend_word(float(latest.get("trend_slope_60d", 0)))
    headline = f"当前状态为{state}，20日趋势{trend20}，60日趋势{trend60}。"
    return {
        "symbol": symbol.upper(),
        "as_of": latest["date"],
        "state": state,
        "headline": headline,
        "key_metrics": [
            {"name": "close", "value": float(latest.get("close"))},
            {"name": "rsi_14", "value": float(latest.get("rsi_14"))},
            {"name": "volatility_20d", "value": float(latest.get("volatility_20d"))},
            {"name": "volume_ratio_20d", "value": float(latest.get("volume_ratio_20d"))},
        ],
        "trendlines": [
            {"name": "trend_slope_20d", "direction": trend20},
            {"name": "trend_slope_60d", "direction": trend60},
        ],
        "recent_anomalies": anomalies[-5:],
        "metric_explanations": METRIC_EXPLANATIONS,
        "quality_flags": quality_flags,
    }


def build_watchlist_snapshot(name: str, ticker_snapshots: list[dict]) -> dict:
    flags = sorted(
        {
            flag
            for snap in ticker_snapshots
            for flag in snap.get("quality_flags", [])
        }
    )
    anomalies_total = sum(len(snap.get("recent_anomalies", [])) for snap in ticker_snapshots)
    as_of = max((snap.get("as_of") or "" for snap in ticker_snapshots), default=None)
    return {
        "watchlist": name,
        "as_of": as_of,
        "symbols_total": len(ticker_snapshots),
        "anomalies_total": anomalies_total,
        "symbols": ticker_snapshots,
        "quality_flags": flags,
    }
```

- [ ] **Step 4: Run snapshot tests and verify they pass**

Run:

```bash
cd /Users/hanlianlyu/.nanobot-wife/workspace/skills/market-metrics-monitor
PYTHONPATH=scripts uv run --with pytest --with pandas python -m pytest scripts/tests/test_snapshots.py -q
```

Expected: PASS.

- [ ] **Step 5: Commit**

Run:

```bash
cd /Users/hanlianlyu/.nanobot-wife/workspace
git add skills/market-metrics-monitor
git commit -m "feat: build market monitor snapshots"
```

Expected: commit succeeds with no AI attribution or co-author trailer.

## Task 9: Integrated HTML and PNG Rendering

**Files:**
- Create: `/Users/hanlianlyu/.nanobot-wife/workspace/skills/market-metrics-monitor/scripts/market_metrics_monitor/render.py`
- Create: `/Users/hanlianlyu/.nanobot-wife/workspace/skills/market-metrics-monitor/scripts/tests/test_render.py`

- [ ] **Step 1: Write failing renderer tests**

Create `scripts/tests/test_render.py`:

```python
from pathlib import Path

from market_metrics_monitor.render import render_ticker_html, render_watchlist_html, write_html_card


def test_render_ticker_html_contains_integrated_sections():
    snapshot = {
        "symbol": "AAPL",
        "as_of": "2026-05-06",
        "headline": "当前状态为uptrend，20日趋势上行，60日趋势上行。",
        "quality_flags": [],
        "key_metrics": [{"name": "rsi_14", "value": 58.2}],
        "trendlines": [{"name": "trend_slope_20d", "direction": "上行"}],
        "recent_anomalies": [{"type": "volume_spike", "severity": "medium", "explanation": "成交量放大"}],
        "metric_explanations": {"rsi_14": "RSI衡量短线动能压力。"},
    }
    html = render_ticker_html(snapshot)
    assert "AAPL" in html
    assert "指标监控" in html
    assert "成交量放大" in html
    assert "RSI衡量短线动能压力" in html


def test_write_html_card_creates_file(tmp_path: Path):
    path = tmp_path / "card.html"
    write_html_card(path, "<html><body>ok</body></html>")
    assert path.read_text(encoding="utf-8") == "<html><body>ok</body></html>"


def test_render_watchlist_html_contains_rows_and_quality_notes():
    snapshot = {
        "watchlist": "wife_core",
        "as_of": "2026-05-06",
        "symbols_total": 2,
        "anomalies_total": 1,
        "quality_flags": ["stale_data"],
        "symbols": [
            {"symbol": "AAPL", "state": "uptrend", "headline": "AAPL趋势上行", "recent_anomalies": [{"severity": "medium"}], "quality_flags": []},
            {"symbol": "MSFT", "state": "range_bound", "headline": "MSFT震荡", "recent_anomalies": [], "quality_flags": ["stale_data"]},
        ],
    }
    html = render_watchlist_html(snapshot)
    assert "wife_core" in html
    assert "AAPL" in html
    assert "MSFT" in html
    assert "stale_data" in html
```

- [ ] **Step 2: Run renderer tests and verify they fail**

Run:

```bash
cd /Users/hanlianlyu/.nanobot-wife/workspace/skills/market-metrics-monitor
PYTHONPATH=scripts uv run --with pytest --with jinja2 python -m pytest scripts/tests/test_render.py -q
```

Expected: FAIL because `render.py` is missing.

- [ ] **Step 3: Implement renderer**

Create `scripts/market_metrics_monitor/render.py`:

```python
from __future__ import annotations

import asyncio
from pathlib import Path

from jinja2 import Template


CARD_TEMPLATE = Template(
    """<!doctype html>
<html lang="zh">
<head>
<meta charset="utf-8">
<meta name="viewport" content="width=device-width, initial-scale=1">
<title>{{ title }}</title>
<style>
body { margin: 0; font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", sans-serif; background: #f4f7fb; color: #15202b; }
.card { width: 1180px; min-height: 760px; padding: 28px; box-sizing: border-box; background: #f8fbff; }
.header { display: flex; justify-content: space-between; border-bottom: 2px solid #d7e0ea; padding-bottom: 14px; }
.symbol { font-size: 38px; font-weight: 750; }
.asof { font-size: 18px; color: #536471; }
.headline { margin-top: 18px; font-size: 26px; font-weight: 650; }
.grid { display: grid; grid-template-columns: 1.1fr 0.9fr; gap: 18px; margin-top: 20px; }
.panel { background: white; border: 1px solid #d8e2ec; border-radius: 8px; padding: 18px; }
.panel h2 { margin: 0 0 12px; font-size: 20px; }
.metric { display: flex; justify-content: space-between; border-bottom: 1px solid #eef2f6; padding: 8px 0; font-size: 17px; }
.anomaly { padding: 8px 0; border-bottom: 1px solid #eef2f6; }
.severity { font-weight: 700; color: #b42318; }
.footer { margin-top: 16px; color: #536471; font-size: 15px; }
</style>
</head>
<body>
<main class="card">
  <section class="header">
    <div>
      <div class="symbol">{{ symbol }} 指标监控</div>
      <div class="asof">截至 {{ as_of }} 收盘</div>
    </div>
    <div class="asof">Quality: {{ quality }}</div>
  </section>
  <section class="headline">{{ headline }}</section>
  <section class="grid">
    <div class="panel">
      <h2>关键指标</h2>
      {% for item in key_metrics %}
      <div class="metric"><span>{{ item.name }}</span><strong>{{ "%.4g"|format(item.value) }}</strong></div>
      {% endfor %}
    </div>
    <div class="panel">
      <h2>趋势线</h2>
      {% for item in trendlines %}
      <div class="metric"><span>{{ item.name }}</span><strong>{{ item.direction }}</strong></div>
      {% endfor %}
    </div>
    <div class="panel">
      <h2>近期异常</h2>
      {% for item in recent_anomalies %}
      <div class="anomaly"><span class="severity">{{ item.severity }}</span> {{ item.type }}: {{ item.explanation }}</div>
      {% else %}
      <div class="anomaly">最近没有高优先级异常。</div>
      {% endfor %}
    </div>
    <div class="panel">
      <h2>指标解释</h2>
      {% for name, text in metric_explanations.items() %}
      <div class="anomaly"><strong>{{ name }}</strong>: {{ text }}</div>
      {% endfor %}
    </div>
  </section>
  <div class="footer">这是指标监控，不是 finvesto 模型建议。</div>
</main>
</body>
</html>"""
)

WATCHLIST_TEMPLATE = Template(
    """<!doctype html>
<html lang="zh">
<head>
<meta charset="utf-8">
<meta name="viewport" content="width=device-width, initial-scale=1">
<title>{{ watchlist }} monitor</title>
<style>
body { margin: 0; font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", sans-serif; background: #f4f7fb; color: #15202b; }
.card { width: 1180px; min-height: 760px; padding: 28px; box-sizing: border-box; background: #f8fbff; }
.header { display: flex; justify-content: space-between; border-bottom: 2px solid #d7e0ea; padding-bottom: 14px; }
.title { font-size: 34px; font-weight: 750; }
.asof { font-size: 18px; color: #536471; }
table { width: 100%; border-collapse: collapse; margin-top: 22px; background: white; border: 1px solid #d8e2ec; }
th, td { padding: 12px; border-bottom: 1px solid #edf2f7; text-align: left; font-size: 16px; }
th { background: #eaf1f8; font-weight: 700; }
.flag { color: #b42318; font-weight: 700; }
.footer { margin-top: 16px; color: #536471; font-size: 15px; }
</style>
</head>
<body>
<main class="card">
  <section class="header">
    <div>
      <div class="title">{{ watchlist }} 组合指标监控</div>
      <div class="asof">截至 {{ as_of }} 收盘 · {{ symbols_total }} 只股票 · {{ anomalies_total }} 个近期异常</div>
    </div>
    <div class="asof">Quality: {{ quality }}</div>
  </section>
  <table>
    <thead>
      <tr><th>Symbol</th><th>状态</th><th>摘要</th><th>异常数</th><th>质量</th></tr>
    </thead>
    <tbody>
    {% for item in symbols %}
      <tr>
        <td><strong>{{ item.symbol }}</strong></td>
        <td>{{ item.state }}</td>
        <td>{{ item.headline }}</td>
        <td>{{ item.recent_anomalies|length }}</td>
        <td class="flag">{{ item.quality_flags|join(", ") if item.quality_flags else "OK" }}</td>
      </tr>
    {% endfor %}
    </tbody>
  </table>
  <div class="footer">这是指标监控，不是 finvesto 模型建议。</div>
</main>
</body>
</html>"""
)


def render_ticker_html(snapshot: dict) -> str:
    flags = snapshot.get("quality_flags") or []
    return CARD_TEMPLATE.render(
        title=f"{snapshot.get('symbol')} monitor",
        symbol=snapshot.get("symbol", ""),
        as_of=snapshot.get("as_of", ""),
        quality=", ".join(flags) if flags else "OK",
        headline=snapshot.get("headline", ""),
        key_metrics=snapshot.get("key_metrics", []),
        trendlines=snapshot.get("trendlines", []),
        recent_anomalies=snapshot.get("recent_anomalies", []),
        metric_explanations=snapshot.get("metric_explanations", {}),
    )


def render_watchlist_html(snapshot: dict) -> str:
    flags = snapshot.get("quality_flags") or []
    return WATCHLIST_TEMPLATE.render(
        watchlist=snapshot.get("watchlist", ""),
        as_of=snapshot.get("as_of", ""),
        symbols_total=snapshot.get("symbols_total", 0),
        anomalies_total=snapshot.get("anomalies_total", 0),
        quality=", ".join(flags) if flags else "OK",
        symbols=snapshot.get("symbols", []),
    )


def write_html_card(path: Path, html: str) -> None:
    path.parent.mkdir(parents=True, exist_ok=True)
    path.write_text(html, encoding="utf-8")


async def _html_to_png_async(html_path: Path, png_path: Path) -> None:
    from playwright.async_api import async_playwright

    async with async_playwright() as p:
        browser = await p.chromium.launch()
        page = await browser.new_page(viewport={"width": 1180, "height": 760}, device_scale_factor=2)
        await page.goto(html_path.resolve().as_uri())
        await page.screenshot(path=str(png_path), full_page=True)
        await browser.close()


def html_to_png(html_path: Path, png_path: Path) -> None:
    png_path.parent.mkdir(parents=True, exist_ok=True)
    asyncio.run(_html_to_png_async(html_path, png_path))
```

- [ ] **Step 4: Run renderer tests and verify they pass**

Run:

```bash
cd /Users/hanlianlyu/.nanobot-wife/workspace/skills/market-metrics-monitor
PYTHONPATH=scripts uv run --with pytest --with jinja2 python -m pytest scripts/tests/test_render.py -q
```

Expected: PASS.

- [ ] **Step 5: Verify PNG rendering on this Mac**

Run:

```bash
cd /Users/hanlianlyu/.nanobot-wife/workspace/skills/market-metrics-monitor
uv run --with playwright python -m playwright install chromium
PYTHONPATH=scripts uv run --with playwright --with jinja2 python - <<'PY'
from pathlib import Path
from market_metrics_monitor.render import html_to_png, render_ticker_html, render_watchlist_html, write_html_card

snapshot = {
    "symbol": "AAPL",
    "as_of": "2026-05-06",
    "headline": "当前状态为uptrend，20日趋势上行，60日趋势上行。",
    "quality_flags": [],
    "key_metrics": [{"name": "rsi_14", "value": 58.2}],
    "trendlines": [{"name": "trend_slope_20d", "direction": "上行"}],
    "recent_anomalies": [{"type": "volume_spike", "severity": "medium", "explanation": "成交量放大"}],
    "metric_explanations": {"rsi_14": "RSI衡量短线动能压力。"},
}
html = Path("/tmp/aapl_monitor.html")
png = Path("/tmp/aapl_monitor.png")
write_html_card(html, render_ticker_html(snapshot))
html_to_png(html, png)
print(png.exists(), png.stat().st_size)
PY
```

Expected: prints `True` and a positive PNG byte size.

- [ ] **Step 6: Commit**

Run:

```bash
cd /Users/hanlianlyu/.nanobot-wife/workspace
git add skills/market-metrics-monitor
git commit -m "feat: render integrated monitor cards"
```

Expected: commit succeeds with no AI attribution or co-author trailer.

## Task 10: CLI and Update Orchestration

**Files:**
- Create: `/Users/hanlianlyu/.nanobot-wife/workspace/skills/market-metrics-monitor/scripts/market_metrics_monitor/cli.py`
- Create: `/Users/hanlianlyu/.nanobot-wife/workspace/skills/market-metrics-monitor/scripts/tests/test_cli.py`

- [ ] **Step 1: Write failing CLI tests**

Create `scripts/tests/test_cli.py`:

```python
import json

from market_metrics_monitor.cli import main


def test_add_watchlist_cli_creates_registry(temp_monitor_root, monkeypatch):
    monkeypatch.setenv("MARKET_METRICS_ROOT", str(temp_monitor_root))
    code = main(["add-watchlist", "wife_core", "AAPL", "MSFT"])
    assert code == 0
    watchlists = json.loads((temp_monitor_root / "registry/watchlists.json").read_text())
    assert watchlists["wife_core"]["symbols"] == ["AAPL", "MSFT"]


def test_status_cli_reports_empty_registry(temp_monitor_root, monkeypatch, capsys):
    monkeypatch.setenv("MARKET_METRICS_ROOT", str(temp_monitor_root))
    code = main(["status"])
    assert code == 0
    assert "active_symbols=0" in capsys.readouterr().out


def test_render_watchlist_cli_uses_latest_snapshot(temp_monitor_root, monkeypatch, capsys):
    monkeypatch.setenv("MARKET_METRICS_ROOT", str(temp_monitor_root))
    snapshot_dir = temp_monitor_root / "us/snapshots"
    snapshot_dir.mkdir(parents=True)
    (snapshot_dir / "wife_core.latest.json").write_text(
        json.dumps(
            {
                "watchlist": "wife_core",
                "as_of": "2026-05-06",
                "symbols_total": 1,
                "anomalies_total": 0,
                "quality_flags": [],
                "symbols": [{"symbol": "AAPL", "state": "uptrend", "headline": "趋势上行", "recent_anomalies": [], "quality_flags": []}],
            }
        ),
        encoding="utf-8",
    )

    def fake_png(html_path, png_path):
        png_path.write_bytes(b"png")

    monkeypatch.setattr("market_metrics_monitor.cli.html_to_png", fake_png)
    code = main(["render", "wife_core", "--kind", "watchlist"])
    assert code == 0
    assert "wife_core_monitor_2026-05-06.png" in capsys.readouterr().out
```

- [ ] **Step 2: Run CLI tests and verify they fail**

Run:

```bash
cd /Users/hanlianlyu/.nanobot-wife/workspace/skills/market-metrics-monitor
PYTHONPATH=scripts uv run --with pytest --with pandas --with pyarrow python -m pytest scripts/tests/test_cli.py -q
```

Expected: FAIL because `cli.py` is missing.

- [ ] **Step 3: Implement CLI**

Create `scripts/market_metrics_monitor/cli.py`:

```python
from __future__ import annotations

import argparse
import json
import os
from datetime import date, timedelta
from pathlib import Path

import pandas as pd

from market_metrics_monitor.anomalies import append_events, detect_anomalies
from market_metrics_monitor.atomic import atomic_write_json
from market_metrics_monitor.config import load_provider_config
from market_metrics_monitor.metrics import compute_metrics
from market_metrics_monitor.paths import MonitorPaths
from market_metrics_monitor.providers import (
    AlpacaProvider,
    FinvestoRawCacheProvider,
    ProviderChain,
    StooqProvider,
    parse_env_file,
)
from market_metrics_monitor.registry import Registry
from market_metrics_monitor.render import html_to_png, render_ticker_html, write_html_card
from market_metrics_monitor.snapshots import build_ticker_snapshot, build_watchlist_snapshot
from market_metrics_monitor.storage import PriceStore


def _paths() -> MonitorPaths:
    root = Path(os.environ.get("MARKET_METRICS_ROOT", MonitorPaths().root))
    paths = MonitorPaths(root)
    paths.ensure()
    return paths


def _provider_chain(paths: MonitorPaths) -> ProviderChain:
    cfg = load_provider_config(paths.registry_dir / "providers.json")
    alpaca_ref = cfg.credential_refs.get("alpaca", {})
    env_file = Path(alpaca_ref.get("env_file", "/Users/hanlianlyu/Github/finvesto/.env"))
    env = parse_env_file(env_file)
    raw_ref = cfg.credential_refs.get("finvesto_raw_cache", {})
    raw_dir = Path(raw_ref["raw_dir"]) if raw_ref.get("raw_dir") else None
    providers = {
        "alpaca": AlpacaProvider(env),
        "stooq": StooqProvider(),
        "finvesto_raw_cache": FinvestoRawCacheProvider(raw_dir),
    }
    return ProviderChain([providers[name] for name in cfg.priority if name in providers])


def _update_symbol(symbol: str, paths: MonitorPaths, chain: ProviderChain) -> dict:
    store = PriceStore(paths)
    existing = store.read_prices(symbol)
    end = date.today()
    if existing.empty:
        start = end - timedelta(days=730)
    else:
        start = end - timedelta(days=16)
    result = chain.fetch_daily(symbol, start=start, end=end)
    if result.frame.empty:
        return {"symbol": symbol, "status": "failed", "provider": "", "rows": 0}
    prices = store.upsert_prices(symbol, result.frame)
    metrics = compute_metrics(prices)
    store.write_metrics(symbol, metrics)
    events = detect_anomalies(metrics)
    append_events(paths.anomalies_dir / f"{symbol}.jsonl", events)
    snapshot = build_ticker_snapshot(symbol, metrics, events)
    atomic_write_json(paths.snapshots_dir / f"{symbol}.latest.json", snapshot)
    return {"symbol": symbol, "status": "updated", "provider": result.provider, "rows": len(result.frame)}


def _cmd_add_watchlist(args) -> int:
    reg = Registry(_paths())
    reg.add_watchlist(args.name, args.symbols)
    print(f"watchlist={args.name} symbols={','.join(reg.load_watchlists()[args.name]['symbols'])}")
    return 0


def _cmd_status(args) -> int:
    del args
    reg = Registry(_paths())
    print(f"active_symbols={len(reg.active_symbols())}")
    return 0


def _cmd_update_all(args) -> int:
    del args
    paths = _paths()
    reg = Registry(paths)
    chain = _provider_chain(paths)
    results = [_update_symbol(symbol, paths, chain) for symbol in reg.active_symbols()]
    watchlists = reg.load_watchlists()
    for name, entry in watchlists.items():
        ticker_snaps = []
        for symbol in entry.get("symbols", []):
            snap_path = paths.snapshots_dir / f"{symbol}.latest.json"
            if snap_path.exists():
                ticker_snaps.append(json.loads(snap_path.read_text(encoding="utf-8")))
        atomic_write_json(paths.snapshots_dir / f"{name}.latest.json", build_watchlist_snapshot(name, ticker_snaps))
    run = {
        "date": date.today().isoformat(),
        "tickers_total": len(results),
        "tickers_updated": sum(1 for r in results if r["status"] == "updated"),
        "tickers_failed": [r["symbol"] for r in results if r["status"] != "updated"],
        "results": results,
    }
    with (paths.runs_dir / "update_runs.jsonl").open("a", encoding="utf-8") as handle:
        handle.write(json.dumps(run, ensure_ascii=False, sort_keys=True) + "\n")
    with (paths.runs_dir / "data_quality.jsonl").open("a", encoding="utf-8") as handle:
        for result in results:
            if result["status"] != "updated":
                handle.write(json.dumps({"date": run["date"], "symbol": result["symbol"], "flag": "update_failed"}, sort_keys=True) + "\n")
    print(json.dumps(run, ensure_ascii=False, sort_keys=True))
    return 0 if not run["tickers_failed"] else 2


def _cmd_render(args) -> int:
    paths = _paths()
    if args.kind == "ticker":
        snap_path = paths.snapshots_dir / f"{args.name.upper()}.latest.json"
        snapshot = json.loads(snap_path.read_text(encoding="utf-8"))
        html = render_ticker_html(snapshot)
        stem = f"{args.name.upper()}_monitor_{snapshot['as_of']}"
        html_path = paths.cards_dir / f"{stem}.html"
        png_path = paths.cards_dir / f"{stem}.png"
        write_html_card(html_path, html)
        html_to_png(html_path, png_path)
        print(str(png_path))
        return 0
    if args.kind == "watchlist":
        snap_path = paths.snapshots_dir / f"{args.name}.latest.json"
        snapshot = json.loads(snap_path.read_text(encoding="utf-8"))
        html = render_watchlist_html(snapshot)
        stem = f"{args.name}_monitor_{snapshot['as_of']}"
        html_path = paths.cards_dir / f"{stem}.html"
        png_path = paths.cards_dir / f"{stem}.png"
        write_html_card(html_path, html)
        html_to_png(html_path, png_path)
        print(str(png_path))
        return 0
    raise ValueError(f"unsupported render kind: {args.kind}")


def build_parser() -> argparse.ArgumentParser:
    parser = argparse.ArgumentParser(prog="update_metrics.py")
    sub = parser.add_subparsers(dest="command", required=True)
    add = sub.add_parser("add-watchlist")
    add.add_argument("name")
    add.add_argument("symbols", nargs="+")
    add.set_defaults(func=_cmd_add_watchlist)
    status = sub.add_parser("status")
    status.set_defaults(func=_cmd_status)
    update = sub.add_parser("update-all")
    update.set_defaults(func=_cmd_update_all)
    render = sub.add_parser("render")
    render.add_argument("name")
    render.add_argument("--kind", choices=["ticker", "watchlist"], default="ticker")
    render.set_defaults(func=_cmd_render)
    return parser


def main(argv: list[str] | None = None) -> int:
    parser = build_parser()
    args = parser.parse_args(argv)
    return int(args.func(args))
```

- [ ] **Step 4: Run CLI tests and verify they pass**

Run:

```bash
cd /Users/hanlianlyu/.nanobot-wife/workspace/skills/market-metrics-monitor
PYTHONPATH=scripts uv run --with pytest --with pandas --with pyarrow --with httpx --with jinja2 --with playwright python -m pytest scripts/tests/test_cli.py -q
```

Expected: PASS.

- [ ] **Step 5: Run full unit suite**

Run:

```bash
cd /Users/hanlianlyu/.nanobot-wife/workspace/skills/market-metrics-monitor
PYTHONPATH=scripts uv run \
  --with pytest \
  --with pandas \
  --with numpy \
  --with pyarrow \
  --with httpx \
  --with jinja2 \
  --with playwright \
  --with python-dotenv \
  --with pyyaml \
  python -m pytest scripts/tests -q
```

Expected: PASS.

- [ ] **Step 6: Commit**

Run:

```bash
cd /Users/hanlianlyu/.nanobot-wife/workspace
git add skills/market-metrics-monitor
git commit -m "feat: add market monitor CLI"
```

Expected: commit succeeds with no AI attribution or co-author trailer.

## Task 11: launchd Installer

**Files:**
- Create: `/Users/hanlianlyu/.nanobot-wife/workspace/skills/market-metrics-monitor/scripts/install_launchd.py`
- Create: `/Users/hanlianlyu/.nanobot-wife/workspace/skills/market-metrics-monitor/scripts/tests/test_launchd.py`

- [ ] **Step 1: Write failing launchd test**

Create `scripts/tests/test_launchd.py`:

```python
from pathlib import Path

from install_launchd import build_plist


def test_build_plist_runs_update_metrics_with_retry_schedule():
    script = Path("/Users/hanlianlyu/.nanobot-wife/workspace/skills/market-metrics-monitor/scripts/update_metrics.py")
    text = build_plist(script)
    assert "com.nanobot-wife.market-metrics-monitor" in text
    assert "update-all" in text
    assert "<integer>23</integer>" in text
    assert "<integer>2</integer>" in text
    assert "<integer>5</integer>" in text
```

- [ ] **Step 2: Run launchd test and verify it fails**

Run:

```bash
cd /Users/hanlianlyu/.nanobot-wife/workspace/skills/market-metrics-monitor
PYTHONPATH=scripts uv run --with pytest python -m pytest scripts/tests/test_launchd.py -q
```

Expected: FAIL because `install_launchd.py` is missing.

- [ ] **Step 3: Implement launchd installer**

Create `scripts/install_launchd.py`:

```python
from __future__ import annotations

import os
import subprocess
from pathlib import Path


LABEL = "com.nanobot-wife.market-metrics-monitor"
PLIST_PATH = Path.home() / "Library/LaunchAgents" / f"{LABEL}.plist"
LOG_DIR = Path.home() / ".nanobot-wife/workspace/data/market-metrics-monitor/runs"
SCRIPT_PATH = Path.home() / ".nanobot-wife/workspace/skills/market-metrics-monitor/scripts/update_metrics.py"


def build_plist(script_path: Path = SCRIPT_PATH) -> str:
    return f"""<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
  <key>Label</key>
  <string>{LABEL}</string>
  <key>ProgramArguments</key>
  <array>
    <string>/opt/homebrew/bin/uv</string>
    <string>run</string>
    <string>{script_path}</string>
    <string>update-all</string>
  </array>
  <key>StartCalendarInterval</key>
  <array>
    <dict><key>Hour</key><integer>23</integer><key>Minute</key><integer>30</integer></dict>
    <dict><key>Hour</key><integer>2</integer><key>Minute</key><integer>30</integer></dict>
    <dict><key>Hour</key><integer>5</integer><key>Minute</key><integer>30</integer></dict>
  </array>
  <key>StandardOutPath</key>
  <string>{LOG_DIR / "launchd.log"}</string>
  <key>StandardErrorPath</key>
  <string>{LOG_DIR / "launchd.err"}</string>
  <key>EnvironmentVariables</key>
  <dict>
    <key>PATH</key>
    <string>/opt/homebrew/bin:/usr/local/bin:/usr/bin:/bin</string>
  </dict>
</dict>
</plist>
"""


def install() -> None:
    LOG_DIR.mkdir(parents=True, exist_ok=True)
    PLIST_PATH.parent.mkdir(parents=True, exist_ok=True)
    PLIST_PATH.write_text(build_plist(), encoding="utf-8")
    subprocess.run(["launchctl", "bootout", f"gui/{os.getuid()}", str(PLIST_PATH)], capture_output=True)
    subprocess.run(["launchctl", "bootstrap", f"gui/{os.getuid()}", str(PLIST_PATH)], check=True)


if __name__ == "__main__":
    install()
    print(f"installed {PLIST_PATH}")
```

- [ ] **Step 4: Run launchd test and verify it passes**

Run:

```bash
cd /Users/hanlianlyu/.nanobot-wife/workspace/skills/market-metrics-monitor
PYTHONPATH=scripts uv run --with pytest python -m pytest scripts/tests/test_launchd.py -q
```

Expected: PASS.

- [ ] **Step 5: Commit**

Run:

```bash
cd /Users/hanlianlyu/.nanobot-wife/workspace
git add skills/market-metrics-monitor
git commit -m "feat: add market monitor launchd installer"
```

Expected: commit succeeds with no AI attribution or co-author trailer.

## Task 12: End-to-End Verification and Nanobot Restart

**Files:**
- Modify: `/Users/hanlianlyu/.nanobot-wife/workspace/skills/market-metrics-monitor/SKILL.md`
- Runtime output: `/Users/hanlianlyu/.nanobot-wife/workspace/data/market-metrics-monitor/`
- Runtime output: `/Users/hanlianlyu/Library/LaunchAgents/com.nanobot-wife.market-metrics-monitor.plist`

- [ ] **Step 1: Run full test suite**

Run:

```bash
cd /Users/hanlianlyu/.nanobot-wife/workspace/skills/market-metrics-monitor
PYTHONPATH=scripts uv run \
  --with pytest \
  --with pandas \
  --with numpy \
  --with pyarrow \
  --with httpx \
  --with jinja2 \
  --with playwright \
  --with python-dotenv \
  --with pyyaml \
  python -m pytest scripts/tests -q
```

Expected: all tests pass.

- [ ] **Step 2: Add a smoke-test watchlist**

Run:

```bash
cd /Users/hanlianlyu/.nanobot-wife/workspace/skills/market-metrics-monitor
uv run scripts/update_metrics.py add-watchlist smoke_test AAPL MSFT
```

Expected: output contains `watchlist=smoke_test symbols=AAPL,MSFT`.

- [ ] **Step 3: Run one manual update**

Run:

```bash
cd /Users/hanlianlyu/.nanobot-wife/workspace/skills/market-metrics-monitor
uv run scripts/update_metrics.py update-all
```

Expected: JSON output shows `tickers_total` at least 2. Exit code is 0 if both update or 2 if one provider fails; if exit code is 2, inspect `tickers_failed` and provider network errors before proceeding.

- [ ] **Step 4: Render a ticker card**

Run:

```bash
cd /Users/hanlianlyu/.nanobot-wife/workspace/skills/market-metrics-monitor
uv run --with playwright python -m playwright install chromium
uv run scripts/update_metrics.py render AAPL --kind ticker
```

Expected: output is a PNG path under `/Users/hanlianlyu/.nanobot-wife/workspace/data/market-metrics-monitor/us/cards/` and the file exists with positive size.

- [ ] **Step 5: Render a watchlist card**

Run:

```bash
cd /Users/hanlianlyu/.nanobot-wife/workspace/skills/market-metrics-monitor
uv run scripts/update_metrics.py render smoke_test --kind watchlist
```

Expected: output is a PNG path under `/Users/hanlianlyu/.nanobot-wife/workspace/data/market-metrics-monitor/us/cards/` and the file exists with positive size.

- [ ] **Step 6: Install launchd schedule**

Run:

```bash
cd /Users/hanlianlyu/.nanobot-wife/workspace/skills/market-metrics-monitor
uv run scripts/install_launchd.py
launchctl list | rg 'com.nanobot-wife.market-metrics-monitor'
```

Expected: `launchctl list` shows `com.nanobot-wife.market-metrics-monitor`.

- [ ] **Step 7: Restart wife gateway so the new skill is discoverable**

Run:

```bash
launchctl kickstart -k gui/$(id -u)/com.nanobot-wife.gateway
sleep 3
launchctl list | rg 'com.nanobot-wife.gateway'
```

Expected: wife gateway has a running PID.

- [ ] **Step 8: Verify skill summary through nanobot SDK**

Run:

```bash
cd /Users/hanlianlyu/Github/nanobot
uv run python - <<'PY'
from pathlib import Path
from nanobot.agent.skills import SkillsLoader

skills = SkillsLoader(Path("/Users/hanlianlyu/.nanobot-wife/workspace")).list_skills(filter_unavailable=False)
names = {item["name"] for item in skills}
print("market-metrics-monitor" in names)
PY
```

Expected: prints `True`.

- [ ] **Step 9: Commit final verification adjustments**

Run:

```bash
cd /Users/hanlianlyu/.nanobot-wife/workspace
git status --short
git add skills/market-metrics-monitor
git commit -m "chore: verify market metrics monitor deployment"
```

Expected: commit succeeds if files changed during verification. If no files changed, skip this commit.

## Self-Review Checklist

- Spec coverage: wife-only workspace, independent persistence, US-only watchlist-first data, launchd primary update, full-history metrics, integrated PNG card, optional HTML file, strict finvesto/finance isolation.
- Placeholder scan: no task uses unresolved markers, empty test instructions, or undefined implementation handoffs.
- Type consistency: paths use `MonitorPaths`; registry uses uppercase US symbols; storage upserts by `date + symbol`; metrics emit fields consumed by snapshots and rendering.
- Verification: each task has a failing test, a pass command, and a commit point.
