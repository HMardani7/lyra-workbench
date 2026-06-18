# Etsy Print Farm — Implementation Plan

> **For Hermes:** Use subagent-driven-development skill to implement this plan task-by-task.

**Goal:** Build a Python automation system that polls Etsy for new orders, maps them to pre-sliced GCODE files, sends them to a Bambu Lab P1S printer, and tracks orders + profit.

**Architecture:** Python 3.11+ daemon with modular components: Etsy poller, product mapper, print queue (SQLite), printer bridge (FTPS + MQTT). CLI via Typer. Tested with pytest.

**Tech Stack:** Python 3.11+, SQLite (SQLModel), requests, paho-mqtt, PyYAML, Typer, pytest, ruff

---

## What This Upgrade Does

See `00-overview.md` for full context.

---

## Project Setup

The project lives at a location of your choice (e.g., `~/etsy-store/`). The plan uses `$PROJECT` as a placeholder.

### Project Structure

```
$PROJECT/
├── pyproject.toml
├── .env.example
├── products.yaml          (mapping: listing → file paths)
├── etsy_store/
│   ├── __init__.py
│   ├── cli.py            (Typer CLI entry)
│   ├── config.py         (Pydantic Settings from .env)
│   ├── db.py             (SQLModel database layer)
│   ├── poller.py         (Etsy receipt polling)
│   ├── processor.py      (order → job creation)
│   ├── mapper.py         (listing + property → GCODE path)
│   ├── queue.py          (print queue management)
│   ├── dispatcher.py     (send to printer)
│   ├── monitor.py        (MQTT print monitoring)
│   └── metrics.py        (revenue/profit queries)
└── tests/
    ├── __init__.py
    ├── conftest.py
    ├── test_mapper.py
    ├── test_poller.py
    ├── test_processor.py
    ├── test_queue.py
    ├── test_dispatcher.py
    └── test_monitor.py
```

---

## Task Breakdown

---

### Task 1: Initialize project structure

**Objective:** Create the Python project scaffold with dependencies

**Files:**
- Create: `pyproject.toml`
- Create: `.env.example`
- Create: `etsy_store/__init__.py`
- Create: `tests/__init__.py`

**Step 1: Create `pyproject.toml`**

```toml
[project]
name = "etsy-store"
version = "0.1.0"
description = "Etsy print farm automation"
requires-python = ">=3.11"
dependencies = [
    "requests>=2.31",
    "paho-mqtt>=2.0",
    "sqlmodel>=0.0.16",
    "pyyaml>=6.0",
    "pydantic-settings>=2.0",
    "typer>=0.9",
]

[project.optional-dependencies]
dev = [
    "pytest>=8.0",
    "pytest-mock>=3.12",
    "ruff>=0.4",
]

[tool.ruff]
line-length = 100
target-version = "py311"

[tool.pytest.ini_options]
testpaths = ["tests"]
```

**Step 2: Create `.env.example`**

```bash
# Etsy API v3 credentials
ETSY_API_KEY=your_keystring_here
ETSY_SHARED_SECRET=your_shared_secret
ETSY_REFRESH_TOKEN=oauth_refresh_token
ETSY_SHOP_ID=12345678

# Polling interval in seconds (default: 300 = 5 min)
POLL_INTERVAL_SECONDS=300

# Printer (Bambu Lab P1S)
PRINTER_HOST=192.168.1.100
PRINTER_ACCESS_CODE=your_printer_access_code
PRINTER_SERIAL=your_printer_serial

# Product mapping file
PRODUCTS_YAML_PATH=./products.yaml

# Database
DATABASE_URL=sqlite:///etsy_store.db
```

**Step 3: Verify project structure**

Run: `find . -type f | sort`
Expected: See `pyproject.toml`, `.env.example`, `etsy_store/__init__.py`, `tests/__init__.py`

**Step 4: Commit**

```bash
git add pyproject.toml .env.example etsy_store/__init__.py tests/__init__.py
git commit -m "chore: initialize project structure with dependencies"
```

---

### Task 2: Configuration layer (Pydantic Settings)

**Objective:** Load Etsy credentials, printer config, and operational settings from `.env`

**Files:**
- Create: `etsy_store/config.py`

**Step 1: Write failing test**

```python
# tests/test_config.py
import os
import pytest
from etsy_store.config import Settings

def test_settings_loads_from_env():
    os.environ["ETSY_API_KEY"] = "test-key"
    os.environ["ETSY_SHARED_SECRET"] = "test-secret"
    os.environ["ETSY_REFRESH_TOKEN"] = "test-refresh"
    os.environ["ETSY_SHOP_ID"] = "12345678"
    os.environ["PRINTER_HOST"] = "192.168.1.100"
    os.environ["PRINTER_ACCESS_CODE"] = "abc123"
    os.environ["PRINTER_SERIAL"] = "SERIAL01"

    settings = Settings()

    assert settings.etsy_api_key == "test-key"
    assert settings.etsy_shop_id == 12345678
    assert settings.printer_host == "192.168.1.100"
    assert settings.poll_interval_seconds == 300  # default

def test_settings_defaults():
    # Must set required fields or they fail validation
    os.environ["ETSY_API_KEY"] = "k"
    os.environ["ETSY_REFRESH_TOKEN"] = "r"
    os.environ["ETSY_SHOP_ID"] = "1"
    os.environ["PRINTER_HOST"] = "localhost"
    os.environ["PRINTER_ACCESS_CODE"] = "x"
    os.environ["PRINTER_SERIAL"] = "s"

    settings = Settings()

    assert settings.poll_interval_seconds == 300
    assert settings.database_url == "sqlite:///etsy_store.db"
    assert settings.products_yaml_path == "./products.yaml"
```

**Step 2: Run test to verify failure**

Run: `pytest tests/test_config.py -v`
Expected: FAIL — module not found or Settings not defined

**Step 3: Implement `etsy_store/config.py`**

```python
from pydantic_settings import BaseSettings


class Settings(BaseSettings):
    # Etsy
    etsy_api_key: str
    etsy_shared_secret: str
    etsy_refresh_token: str
    etsy_shop_id: int

    # Polling
    poll_interval_seconds: int = 300

    # Printer
    printer_host: str
    printer_access_code: str
    printer_serial: str

    # Paths
    products_yaml_path: str = "./products.yaml"
    database_url: str = "sqlite:///etsy_store.db"

    model_config = {"env_file": ".env", "env_file_encoding": "utf-8"}
```

**Step 4: Run test to verify pass**

Run: `pytest tests/test_config.py -v`
Expected: PASS (2 tests)

**Step 5: Commit**

```bash
git add etsy_store/config.py tests/test_config.py
git commit -m "feat: add Pydantic Settings config layer"
```

---

### Task 3: Database layer (SQLModel + SQLite)

**Objective:** Define the Order model and database initialization

**Files:**
- Create: `etsy_store/db.py`

**Step 1: Write failing test**

```python
# tests/test_db.py
import tempfile
import os
from etsy_store.db import init_db, Order, OrderStatus, get_session


def test_init_db_creates_tables():
    with tempfile.TemporaryDirectory() as tmpdir:
        db_url = f"sqlite:///{tmpdir}/test.db"
        engine = init_db(db_url)

        # Verify tables exist by querying sqlite_master
        from sqlalchemy import inspect
        inspector = inspect(engine)
        tables = inspector.get_table_names()
        assert "order" in tables


def test_order_creation():
    with tempfile.TemporaryDirectory() as tmpdir:
        db_url = f"sqlite:///{tmpdir}/test.db"
        init_db(db_url)

        session = get_session(db_url)
        order = Order(
            etsy_receipt_id=1001,
            etsy_transaction_id=5001,
            listing_id=2001,
            product_name="iron_man_helmet",
            property_values='{"Size": "22 inch"}',
            gcode_file_path="/data/products/iron_man_helmet/helmet_22inch.gcode.3mf",
            status=OrderStatus.PENDING,
            revenue_amount=4999,
            etsy_fee_amount=500,
            material_cost_estimate=300,
        )
        session.add(order)
        session.commit()

        fetched = session.query(Order).filter(Order.etsy_receipt_id == 1001).first()
        assert fetched is not None
        assert fetched.product_name == "iron_man_helmet"
        assert fetched.status == OrderStatus.PENDING
        assert fetched.revenue_amount == 4999

        session.close()
```

**Step 2: Run test to verify failure**

Run: `pytest tests/test_db.py -v`
Expected: FAIL — module not found

**Step 3: Implement `etsy_store/db.py`**

```python
import enum
from datetime import datetime, timezone
from typing import Optional

from sqlalchemy import create_engine, Engine
from sqlalchemy.orm import sessionmaker, Session
from sqlmodel import Field, SQLModel


class OrderStatus(str, enum.Enum):
    PENDING = "pending"
    PRINTING = "printing"
    DONE = "done"
    FAILED = "failed"


class Order(SQLModel, table=True):
    id: Optional[int] = Field(default=None, primary_key=True)
    etsy_receipt_id: int = Field(index=True, unique=True)
    etsy_transaction_id: int
    listing_id: int
    product_name: str
    property_values: str  # JSON string
    gcode_file_path: str
    status: OrderStatus = Field(default=OrderStatus.PENDING)

    # Financial
    revenue_amount: int = 0  # in cents
    etsy_fee_amount: int = 0  # in cents
    material_cost_estimate: int = 0  # in cents

    # Print
    print_time_seconds: Optional[int] = None

    created_at: datetime = Field(default_factory=lambda: datetime.now(timezone.utc))
    updated_at: datetime = Field(default_factory=lambda: datetime.now(timezone.utc))


def init_db(database_url: str) -> Engine:
    engine = create_engine(database_url, echo=False)
    SQLModel.metadata.create_all(engine)
    return engine


def get_session(database_url: str) -> Session:
    engine = create_engine(database_url, echo=False)
    SessionLocal = sessionmaker(bind=engine)
    return SessionLocal()
```

**Step 4: Run test to verify pass**

Run: `pytest tests/test_db.py -v`
Expected: PASS (2 tests)

**Step 5: Commit**

```bash
git add etsy_store/db.py tests/test_db.py
git commit -m "feat: add database layer with Order model"
```

---

### Task 4: Product mapper (listing + properties → GCODE path)

**Objective:** Given a listing ID and property values from an Etsy order, resolve the correct GCODE file path using a YAML mapping config

**Files:**
- Create: `etsy_store/mapper.py`
- Create: `products.yaml` (example config)

**Step 1: Create example `products.yaml`**

```yaml
# Etsy product mapping configuration
# listing_id → folder and property-value → file mapping

listings:
  # Iron Man Helmet — Etsy Listing ID 2001
  2001:
    folder: iron_man_helmet
    property: Size          # The Etsy property name on the listing
    files:
      "22 inch": helmet_circumference_22inch.gcode.3mf
      "23 inch": helmet_circumference_23inch.gcode.3mf
      "24 inch": helmet_circumference_24inch.gcode.3mf

  # Mjolnir Hammer — Etsy Listing ID 2002
  2002:
    folder: mjolnir_hammer
    property: Version
    files:
      "Standard": full_size_standard.gcode.3mf
      "Mini Desk": mini_desk_version.gcode.3mf

  # Dragon Skull Decor — Etsy Listing ID 2003 (no variations)
  2003:
    folder: dragon_skull_decor
    property: null
    files:
      default: dragon_skull_one_size.gcode.3mf

# Base path where product folders live
base_path: /data/products
```

**Step 2: Write failing test**

```python
# tests/test_mapper.py
import tempfile
import os
import yaml
from etsy_store.mapper import ProductMapper


def test_mapper_resolves_with_property():
    yaml_content = {
        "base_path": "/data/products",
        "listings": {
            2001: {
                "folder": "iron_man_helmet",
                "property": "Size",
                "files": {
                    "22 inch": "helmet_22inch.gcode.3mf",
                    "23 inch": "helmet_23inch.gcode.3mf",
                },
            }
        },
    }
    with tempfile.NamedTemporaryFile(mode="w", suffix=".yaml") as f:
        yaml.dump(yaml_content, f)
        f.flush()

        mapper = ProductMapper(f.name)

        path = mapper.resolve(
            listing_id=2001,
            property_values={"Size": "22 inch"},
        )
        assert path == "/data/products/iron_man_helmet/helmet_22inch.gcode.3mf"


def test_mapper_resolves_without_variations():
    yaml_content = {
        "base_path": "/data/products",
        "listings": {
            3001: {
                "folder": "simple_decor",
                "property": None,
                "files": {"default": "decor.gcode.3mf"},
            }
        },
    }
    with tempfile.NamedTemporaryFile(mode="w", suffix=".yaml") as f:
        yaml.dump(yaml_content, f)
        f.flush()

        mapper = ProductMapper(f.name)

        # No property values needed when no variations
        path = mapper.resolve(listing_id=3001, property_values={})
        assert path == "/data/products/simple_decor/decor.gcode.3mf"


def test_mapper_raises_on_unknown_listing():
    yaml_content = {"base_path": "/data/products", "listings": {}}
    with tempfile.NamedTemporaryFile(mode="w", suffix=".yaml") as f:
        yaml.dump(yaml_content, f)
        f.flush()

        mapper = ProductMapper(f.name)

        import pytest
        with pytest.raises(ValueError, match="Listing 9999 not found"):
            mapper.resolve(listing_id=9999, property_values={})


def test_mapper_raises_on_unknown_property_value():
    yaml_content = {
        "base_path": "/data/products",
        "listings": {
            2001: {
                "folder": "helmet",
                "property": "Size",
                "files": {"22 inch": "small.gcode.3mf"},
            }
        },
    }
    with tempfile.NamedTemporaryFile(mode="w", suffix=".yaml") as f:
        yaml.dump(yaml_content, f)
        f.flush()

        mapper = ProductMapper(f.name)

        import pytest
        with pytest.raises(ValueError, match="Property value '99 inch' not found"):
            mapper.resolve(listing_id=2001, property_values={"Size": "99 inch"})
```

**Step 3: Run test to verify failure**

Run: `pytest tests/test_mapper.py -v`
Expected: FAIL — module not found

**Step 4: Implement `etsy_store/mapper.py`**

```python
from pathlib import Path
from typing import Any

import yaml


class ProductMapper:
    """Resolve Etsy listing + property values → local GCODE file path."""

    def __init__(self, yaml_path: str):
        self.yaml_path = Path(yaml_path)
        if not self.yaml_path.exists():
            raise FileNotFoundError(f"Product mapping file not found: {yaml_path}")

        with open(self.yaml_path) as f:
            self.config = yaml.safe_load(f)

        self.base_path = Path(self.config.get("base_path", "."))
        self.listings: dict = self.config.get("listings", {})

    def resolve(self, listing_id: int, property_values: dict[str, str]) -> str:
        listing_id_str = str(listing_id)
        listing_id_int = listing_id

        # Support both int and str keys in YAML
        listing = self.listings.get(listing_id_int) or self.listings.get(listing_id_str)
        if listing is None:
            raise ValueError(f"Listing {listing_id} not found in product mapping")

        folder = listing["folder"]
        prop_name = listing.get("property")
        files: dict[str, str] = listing["files"]

        if prop_name is None:
            # No variations — use default
            filename = files["default"]
        else:
            prop_value = property_values.get(prop_name)
            if prop_value is None:
                raise ValueError(
                    f"Property '{prop_name}' missing from order. "
                    f"Got: {property_values}"
                )
            filename = files.get(prop_value)
            if filename is None:
                raise ValueError(
                    f"Property value '{prop_value}' not found for listing {listing_id}. "
                    f"Available: {list(files.keys())}"
                )

        return str(self.base_path / folder / filename)
```

**Step 5: Run test to verify pass**

Run: `pytest tests/test_mapper.py -v`
Expected: PASS (4 tests)

**Step 6: Commit**

```bash
git add etsy_store/mapper.py products.yaml tests/test_mapper.py
git commit -m "feat: add product mapper (YAML config → GCODE path)"
```

---

### Task 5: Etsy API client

**Objective:** Wrap Etsy API v3 endpoints for authentication, receipt polling, and receipt detail

**Files:**
- Create: `etsy_store/etsy_client.py`

**Step 1: Write failing test**

```python
# tests/test_etsy_client.py
import pytest
from unittest.mock import patch, MagicMock
from etsy_store.etsy_client import EtsyClient


@pytest.fixture
def client():
    return EtsyClient(
        api_key="test-key",
        refresh_token="test-refresh",
        shop_id=12345678,
    )


@patch("etsy_store.etsy_client.requests.Session.post")
@patch("etsy_store.etsy_client.requests.Session.request")
def test_get_new_receipts(mock_request, mock_post, client):
    # Mock token refresh
    mock_post.return_value.status_code = 200
    mock_post.return_value.json.return_value = {"access_token": "fake-token"}

    # Mock receipt API response
    mock_request.return_value.status_code = 200
    mock_request.return_value.json.return_value = {
        "count": 2,
        "results": [
            {"receipt_id": 1001, "grandtotal": {"amount": 4999, "divisor": 100}},
            {"receipt_id": 1002, "grandtotal": {"amount": 2999, "divisor": 100}},
        ],
    }

    receipts = list(client.get_new_receipts(min_created_timestamp=1700000000))

    assert len(receipts) == 2
    assert receipts[0]["receipt_id"] == 1001
    assert receipts[1]["receipt_id"] == 1002


@patch("etsy_store.etsy_client.requests.Session.post")
@patch("etsy_store.etsy_client.requests.Session.request")
def test_get_receipt_detail(mock_request, mock_post, client):
    mock_post.return_value.status_code = 200
    mock_post.return_value.json.return_value = {"access_token": "fake-token"}

    mock_request.return_value.status_code = 200
    mock_request.return_value.json.return_value = {
        "receipt_id": 1001,
        "transactions": [
            {
                "transaction_id": 5001,
                "listing_id": 2001,
                "price": {"amount": 4999, "divisor": 100},
                "variations": [
                    {"formatted_name": "Size", "formatted_value": "22 inch"}
                ],
            }
        ],
    }

    detail = client.get_receipt_detail(receipt_id=1001)

    assert detail["receipt_id"] == 1001
    assert len(detail["transactions"]) == 1
    assert detail["transactions"][0]["listing_id"] == 2001


@patch("etsy_store.etsy_client.requests.Session.post")
@patch("etsy_store.etsy_client.requests.Session.request")
def test_receipt_already_processed_skipped(mock_request, mock_post, client):
    """Integration hint: the processor layer checks this, but the client returns raw data."""
    mock_post.return_value.status_code = 200
    mock_post.return_value.json.return_value = {"access_token": "fake-token"}

    mock_request.return_value.status_code = 200
    mock_request.return_value.json.return_value = {
        "count": 0,
        "results": [],
    }

    receipts = list(client.get_new_receipts(min_created_timestamp=1700000000))
    assert len(receipts) == 0
```

**Step 2: Run test to verify failure**

Run: `pytest tests/test_etsy_client.py -v`
Expected: FAIL — module not found

**Step 3: Implement `etsy_store/etsy_client.py`**

```python
"""Etsy API v3 client — auth, receipt polling, receipt detail."""

import logging
import time
from datetime import datetime, timedelta, timezone
from typing import Any, Iterator, Optional

import requests

logger = logging.getLogger(__name__)

TOKEN_URL = "https://api.etsy.com/v3/public/oauth/token"
BASE_URL = "https://openapi.etsy.com/v3/application/"
PAGE_SIZE = 100


class EtsyError(Exception):
    def __init__(self, message: str, status_code: Optional[int] = None):
        super().__init__(message)
        self.status_code = status_code


class EtsyClient:
    def __init__(self, api_key: str, refresh_token: str, shop_id: int, *, timeout: int = 60):
        self.api_key = api_key
        self.refresh_token = refresh_token
        self.shop_id = shop_id
        self.timeout = timeout
        self.session = requests.Session()
        self._access_token: Optional[str] = None
        self._token_expires_at: Optional[datetime] = None

    # ---- Auth ----

    def _get_access_token(self) -> str:
        now = datetime.now(timezone.utc)
        if self._access_token and self._token_expires_at and now < self._token_expires_at:
            return self._access_token

        resp = self.session.post(
            TOKEN_URL,
            data={
                "grant_type": "refresh_token",
                "client_id": self.api_key,
                "refresh_token": self.refresh_token,
            },
            headers={"Content-Type": "application/x-www-form-urlencoded"},
            timeout=self.timeout,
        )
        if resp.status_code != 200:
            raise EtsyError(f"Token refresh failed: HTTP {resp.status_code}", resp.status_code)

        data = resp.json()
        token = data.get("access_token")
        if not token:
            raise EtsyError("Token response missing access_token")

        self._access_token = token
        # Tokens live 3600s; refresh at 3300s (5 min buffer)
        self._token_expires_at = now + timedelta(seconds=3300)
        return token

    def _headers(self) -> dict:
        return {
            "x-api-key": self.api_key,
            "Authorization": f"Bearer {self._get_access_token()}",
            "Accept": "application/json",
        }

    # ---- Receipt Polling ----

    def get_new_receipts(
        self,
        min_created_timestamp: int,
        *,
        status: Optional[str] = None,
    ) -> Iterator[dict]:
        """Yield receipts created after the given Unix timestamp."""
        max_created = int(datetime.now(timezone.utc).timestamp())
        offset = 0

        while True:
            params: dict[str, Any] = {
                "min_created": min_created_timestamp,
                "max_created": max_created,
                "limit": PAGE_SIZE,
                "offset": offset,
            }
            if status:
                params["status"] = status

            data = self._request(
                "GET", f"shops/{self.shop_id}/receipts", params=params
            )
            results = data.get("results") or []
            count = data.get("count", 0)

            for receipt in results:
                yield receipt

            offset += len(results)
            if not results or offset >= count:
                return

    def get_receipt_detail(self, receipt_id: int) -> Optional[dict]:
        """Get full receipt with transactions array."""
        try:
            return self._request(
                "GET", f"shops/{self.shop_id}/receipts/{receipt_id}"
            )
        except EtsyError as e:
            if e.status_code == 404:
                return None
            raise

    # ---- Internal ----

    def _request(self, method: str, path: str, params: Optional[dict] = None) -> Any:
        url = f"{BASE_URL}{path.lstrip('/')}"
        for attempt in range(3):
            try:
                resp = self.session.request(
                    method, url, headers=self._headers(), params=params, timeout=self.timeout
                )
            except (requests.ConnectionError, requests.Timeout) as e:
                if attempt == 2:
                    raise EtsyError(f"Request failed after 3 attempts: {e}")
                time.sleep(2**attempt)
                continue

            if resp.status_code == 401:
                # Token expired — clear and retry once
                self._access_token = None
                resp = self.session.request(
                    method, url, headers=self._headers(), params=params, timeout=self.timeout
                )

            if resp.status_code in {429, 500, 502, 503, 504}:
                if attempt == 2:
                    raise EtsyError(f"Etsy unavailable: HTTP {resp.status_code}", resp.status_code)
                time.sleep(2**attempt)
                continue

            if not resp.ok:
                raise EtsyError(
                    f"Etsy API error: {method} {path} → HTTP {resp.status_code}",
                    resp.status_code,
                )

            return resp.json()

        raise EtsyError("Request failed for unknown reason")
```

**Step 4: Run test to verify pass**

Run: `pytest tests/test_etsy_client.py -v`
Expected: PASS (3 tests)

**Step 5: Commit**

```bash
git add etsy_store/etsy_client.py tests/test_etsy_client.py
git commit -m "feat: add Etsy API v3 client (auth, receipts)"
```

---

### Task 6: Order processor (receipt → database job)

**Objective:** Take raw Etsy receipts, extract transactions and property values, map to GCODE files, create Order records in the database

**Files:**
- Create: `etsy_store/processor.py`

**Step 1: Write failing test**

```python
# tests/test_processor.py
import pytest
import tempfile
import yaml
from unittest.mock import MagicMock
from etsy_store.processor import OrderProcessor
from etsy_store.mapper import ProductMapper
from etsy_store.db import init_db, get_session, Order, OrderStatus


@pytest.fixture
def processor():
    # Setup temp DB
    tmpdir = tempfile.mkdtemp()
    db_url = f"sqlite:///{tmpdir}/test.db"
    init_db(db_url)

    # Setup temp product mapping
    yaml_content = {
        "base_path": "/data/products",
        "listings": {
            2001: {
                "folder": "iron_man_helmet",
                "property": "Size",
                "files": {"22 inch": "helmet_22inch.gcode.3mf"},
            }
        },
    }
    yaml_path = f"{tmpdir}/products.yaml"
    with open(yaml_path, "w") as f:
        yaml.dump(yaml_content, f)

    mapper = ProductMapper(yaml_path)
    return OrderProcessor(db_url=db_url, mapper=mapper)


def test_process_new_receipt_creates_order(processor):
    receipt = {
        "receipt_id": 1001,
        "grandtotal": {"amount": 4999, "divisor": 100},
        "transactions": [
            {
                "transaction_id": 5001,
                "listing_id": 2001,
                "price": {"amount": 4999, "divisor": 100},
                "variations": [
                    {"formatted_name": "Size", "formatted_value": "22 inch"}
                ],
            }
        ],
    }

    orders = processor.process_receipt(receipt)
    assert len(orders) == 1
    order = orders[0]
    assert order.etsy_receipt_id == 1001
    assert order.product_name == "iron_man_helmet"
    assert order.status == OrderStatus.PENDING
    assert order.revenue_amount == 4999


def test_duplicate_receipt_skipped(processor):
    receipt = {
        "receipt_id": 1001,
        "grandtotal": {"amount": 4999, "divisor": 100},
        "transactions": [
            {
                "transaction_id": 5001,
                "listing_id": 2001,
                "price": {"amount": 4999, "divisor": 100},
                "variations": [
                    {"formatted_name": "Size", "formatted_value": "22 inch"}
                ],
            }
        ],
    }

    # First call should create
    orders1 = processor.process_receipt(receipt)
    assert len(orders1) == 1

    # Second call should skip (already exists)
    orders2 = processor.process_receipt(receipt)
    assert len(orders2) == 0


def test_process_receipt_without_variations(processor):
    """Receipt for a listing with no property variations."""
    receipt = {
        "receipt_id": 1002,
        "grandtotal": {"amount": 1999, "divisor": 100},
        "transactions": [
            {
                "transaction_id": 5002,
                "listing_id": 3001,  # Not in mapper — let's adjust
                "price": {"amount": 1999, "divisor": 100},
                "variations": [],
            }
        ],
    }
    # This listing exists in our mapper (no property)
    processor.mapper.config["listings"][3001] = {
        "folder": "simple",
        "property": None,
        "files": {"default": "simple.gcode.3mf"},
    }

    orders = processor.process_receipt(receipt)
    assert len(orders) == 1
    assert orders[0].product_name == "simple"
```

**Step 2: Run test to verify failure**

Run: `pytest tests/test_processor.py -v`
Expected: FAIL

**Step 3: Implement `etsy_store/processor.py`**

```python
"""Order processor: Etsy receipt → database Order records."""

import json
import logging
from typing import Optional

from etsy_store.db import Order, OrderStatus, get_session
from etsy_store.mapper import ProductMapper

logger = logging.getLogger(__name__)


class OrderProcessor:
    def __init__(self, db_url: str, mapper: ProductMapper):
        self.db_url = db_url
        self.mapper = mapper

    def process_receipt(self, receipt: dict) -> list[Order]:
        """Convert an Etsy receipt into Order records. Returns newly created orders."""
        session = get_session(self.db_url)
        try:
            receipt_id = receipt["receipt_id"]

            # Skip if already processed
            existing = session.query(Order).filter(
                Order.etsy_receipt_id == receipt_id
            ).first()
            if existing:
                logger.info(f"Receipt {receipt_id} already processed — skipping")
                return []

            new_orders = []
            transactions = receipt.get("transactions", [])

            for txn in transactions:
                listing_id = txn["listing_id"]
                property_values = self._extract_property_values(txn)
                gcode_path = self.mapper.resolve(listing_id, property_values)

                # Resolve product name from mapper config
                listing_config = self.mapper.get_listing_config(listing_id)
                product_name = listing_config["folder"]

                # Calculate revenue in cents
                price = txn.get("price", {})
                revenue_cents = self._amount_to_cents(price)

                # Estimate Etsy fee (rough — 6.5% transaction + $0.20 listing)
                etsy_fee_cents = int(revenue_cents * 0.065) + 20

                order = Order(
                    etsy_receipt_id=receipt_id,
                    etsy_transaction_id=txn["transaction_id"],
                    listing_id=listing_id,
                    product_name=product_name,
                    property_values=json.dumps(property_values),
                    gcode_file_path=gcode_path,
                    status=OrderStatus.PENDING,
                    revenue_amount=revenue_cents,
                    etsy_fee_amount=etsy_fee_cents,
                    material_cost_estimate=0,  # Set manually or via config later
                )
                session.add(order)
                new_orders.append(order)
                logger.info(
                    f"New order: {product_name} → {gcode_path} (${revenue_cents/100:.2f})"
                )

            session.commit()

            # Refresh to get DB-assigned IDs
            for o in new_orders:
                session.refresh(o)

            return new_orders
        finally:
            session.close()

    @staticmethod
    def _extract_property_values(transaction: dict) -> dict[str, str]:
        """Extract variation selections from a transaction."""
        variations = transaction.get("variations", [])
        return {v["formatted_name"]: v["formatted_value"] for v in variations}

    @staticmethod
    def _amount_to_cents(price: dict) -> int:
        """Convert Etsy price {amount, divisor} to integer cents."""
        amount = price.get("amount", 0)
        divisor = price.get("divisor", 100)
        return int(round(amount * 100 / divisor))
```

**Step 4: Update mapper.py** — add `get_listing_config` method:

Add this method to `ProductMapper` in `etsy_store/mapper.py`:

```python
    def get_listing_config(self, listing_id: int) -> dict:
        """Return the raw listing config dict for a given listing ID."""
        listing = self.listings.get(listing_id) or self.listings.get(str(listing_id))
        if listing is None:
            raise ValueError(f"Listing {listing_id} not found in product mapping")
        return listing
```

**Step 5: Run test to verify pass**

Run: `pytest tests/test_processor.py -v`
Expected: PASS (3 tests)

**Step 6: Commit**

```bash
git add etsy_store/processor.py etsy_store/mapper.py tests/test_processor.py
git commit -m "feat: add order processor (Etsy receipt → DB Order)"
```

---

### Task 7: Print queue manager

**Objective:** FIFO queue operations — enqueue, dequeue next job, update status

**Files:**
- Create: `etsy_store/queue.py`

**Step 1: Write failing test**

```python
# tests/test_queue.py
import pytest
import tempfile
from etsy_store.db import init_db, get_session, Order, OrderStatus
from etsy_store.queue import PrintQueue


@pytest.fixture
def queue():
    tmpdir = tempfile.mkdtemp()
    db_url = f"sqlite:///{tmpdir}/test.db"
    init_db(db_url)
    return PrintQueue(db_url)


def test_enqueue_and_dequeue_next(queue):
    # Add some orders
    session = get_session(queue.db_url)
    o1 = Order(
        etsy_receipt_id=1, etsy_transaction_id=10, listing_id=100,
        product_name="test", property_values="{}",
        gcode_file_path="/tmp/a.gcode.3mf", status=OrderStatus.PENDING,
    )
    o2 = Order(
        etsy_receipt_id=2, etsy_transaction_id=20, listing_id=200,
        product_name="test2", property_values="{}",
        gcode_file_path="/tmp/b.gcode.3mf", status=OrderStatus.PENDING,
    )
    session.add_all([o1, o2])
    session.commit()
    session.close()

    # Dequeue — should get the oldest PENDING
    next_job = queue.dequeue_next()
    assert next_job is not None
    assert next_job.etsy_receipt_id == 1
    assert next_job.status == OrderStatus.PRINTING  # Status changed on dequeue

    # Second dequeue — should get the other one
    next_job2 = queue.dequeue_next()
    assert next_job2 is not None
    assert next_job2.etsy_receipt_id == 2


def test_dequeue_empty_returns_none(queue):
    assert queue.dequeue_next() is None


def test_update_status(queue):
    session = get_session(queue.db_url)
    o = Order(
        etsy_receipt_id=1, etsy_transaction_id=10, listing_id=100,
        product_name="test", property_values="{}",
        gcode_file_path="/tmp/a.gcode.3mf", status=OrderStatus.PENDING,
    )
    session.add(o)
    session.commit()
    order_id = o.id
    session.close()

    queue.update_status(order_id, OrderStatus.DONE, print_time_seconds=3600)

    # Verify
    session = get_session(queue.db_url)
    updated = session.query(Order).filter(Order.id == order_id).first()
    assert updated.status == OrderStatus.DONE
    assert updated.print_time_seconds == 3600
    session.close()


def test_pending_count(queue):
    assert queue.pending_count() == 0

    session = get_session(queue.db_url)
    for i in range(3):
        session.add(Order(
            etsy_receipt_id=i, etsy_transaction_id=i*10, listing_id=100,
            product_name="test", property_values="{}",
            gcode_file_path="/tmp/a.gcode.3mf", status=OrderStatus.PENDING,
        ))
    session.commit()
    session.close()

    assert queue.pending_count() == 3
```

**Step 2: Run test to verify failure**

Run: `pytest tests/test_queue.py -v`
Expected: FAIL

**Step 3: Implement `etsy_store/queue.py`**

```python
"""Print queue — FIFO job management backed by SQLite."""

import logging
from datetime import datetime, timezone
from typing import Optional

from sqlalchemy import desc

from etsy_store.db import Order, OrderStatus, get_session

logger = logging.getLogger(__name__)


class PrintQueue:
    def __init__(self, db_url: str):
        self.db_url = db_url

    def dequeue_next(self) -> Optional[Order]:
        """Get the oldest PENDING order and mark it PRINTING. Returns None if empty."""
        session = get_session(self.db_url)
        try:
            order = (
                session.query(Order)
                .filter(Order.status == OrderStatus.PENDING)
                .order_by(Order.created_at.asc())
                .first()
            )
            if order is None:
                return None

            order.status = OrderStatus.PRINTING
            order.updated_at = datetime.now(timezone.utc)
            session.commit()
            session.refresh(order)
            logger.info(f"Dequeued order {order.id} ({order.product_name}) → printing")
            return order
        finally:
            session.close()

    def update_status(
        self,
        order_id: int,
        status: OrderStatus,
        *,
        print_time_seconds: Optional[int] = None,
    ) -> None:
        """Update the status (and optionally print time) of an order."""
        session = get_session(self.db_url)
        try:
            order = session.query(Order).filter(Order.id == order_id).first()
            if order is None:
                raise ValueError(f"Order {order_id} not found")

            order.status = status
            order.updated_at = datetime.now(timezone.utc)
            if print_time_seconds is not None:
                order.print_time_seconds = print_time_seconds

            session.commit()
            logger.info(f"Order {order_id} → {status.value}")
        finally:
            session.close()

    def pending_count(self) -> int:
        """Return number of jobs waiting in the queue."""
        session = get_session(self.db_url)
        try:
            return (
                session.query(Order)
                .filter(Order.status == OrderStatus.PENDING)
                .count()
            )
        finally:
            session.close()
```

**Step 4: Run test to verify pass**

Run: `pytest tests/test_queue.py -v`
Expected: PASS (4 tests)

**Step 5: Commit**

```bash
git add etsy_store/queue.py tests/test_queue.py
git commit -m "feat: add print queue manager (FIFO enqueue/dequeue)"
```

---

### Task 8: Printer dispatcher (FTPS upload + MQTT)

**Objective:** Take a queued job, upload GCODE to Bambu Lab P1S via FTPS, start the print, monitor via MQTT

**Files:**
- Create: `etsy_store/dispatcher.py`

**Step 1: Write failing test**

```python
# tests/test_dispatcher.py
import pytest
from unittest.mock import patch, MagicMock
from etsy_store.dispatcher import PrinterDispatcher


@pytest.fixture
def dispatcher():
    return PrinterDispatcher(
        printer_host="192.168.1.100",
        access_code="abc123",
        printer_serial="SERIAL01",
    )


@patch("etsy_store.dispatcher.ftplib.FTP_TLS")
def test_upload_gcode(mock_ftp_tls, dispatcher):
    mock_ftp = MagicMock()
    mock_ftp_tls.return_value = mock_ftp

    dispatcher.upload_gcode("/data/products/test.gcode.3mf")

    mock_ftp_tls.assert_called_once()
    mock_ftp.connect.assert_called_once_with("192.168.1.100", 990)
    mock_ftp.login.assert_called_once_with("bblp", "abc123")
    mock_ftp.prot_p.assert_called_once()
    mock_ftp.storbinary.assert_called_once()


@patch("etsy_store.dispatcher.requests.Session.post")
def test_send_gcode_file_command(mock_post, dispatcher):
    mock_post.return_value.status_code = 200
    mock_post.return_value.json.return_value = {}

    result = dispatcher.send_gcode_command(
        filename="test.gcode.3mf",
        plate_index=0,
        use_ams=False,
    )

    assert result is True
    mock_post.assert_called()
    call_args = mock_post.call_args
    # Verify the request went to the right endpoint
    assert f"/v1/iot/{dispatcher.printer_serial}" in call_args[0][0]


def test_file_exists_locally(dispatcher):
    # Non-existent file
    assert dispatcher._file_exists("/nonexistent/path.gcode.3mf") is False


@patch("etsy_store.dispatcher.PrinterDispatcher.upload_gcode")
@patch("etsy_store.dispatcher.PrinterDispatcher.send_gcode_command")
def test_dispatch_job_full_flow(mock_send, mock_upload, dispatcher):
    mock_upload.return_value = None
    mock_send.return_value = True

    from etsy_store.db import Order, OrderStatus
    order = Order(
        id=1,
        etsy_receipt_id=1001,
        etsy_transaction_id=5001,
        listing_id=2001,
        product_name="iron_man_helmet",
        property_values='{"Size": "22 inch"}',
        gcode_file_path="/data/products/iron_man_helmet/test.gcode.3mf",
        status=OrderStatus.PENDING,
        revenue_amount=4999,
        etsy_fee_amount=345,
    )

    result = dispatcher.dispatch(order)
    assert result is True
    mock_upload.assert_called_once_with(order.gcode_file_path)
    mock_send.assert_called_once()
```

**Step 2: Run test to verify failure**

Run: `pytest tests/test_dispatcher.py -v`
Expected: FAIL

**Step 3: Implement `etsy_store/dispatcher.py`**

```python
"""Printer bridge — upload GCODE to Bambu Lab P1S via FTPS, start print."""

import ftplib
import logging
import os
from pathlib import Path

import requests

from etsy_store.db import Order

logger = logging.getLogger(__name__)

# Bambu Lab P1S FTPS settings
FTP_PORT = 990
FTP_USER = "bblp"

# MQTT / HTTPS endpoint for sending GCODE commands (LAN mode)
# The Bambu Lab printer exposes a simple HTTPS API on port 80/443 in LAN mode
# and a cloud MQTT bridge. We use the LAN HTTPS API for direct control.


class PrinterDispatcher:
    def __init__(self, printer_host: str, access_code: str, printer_serial: str):
        self.printer_host = printer_host
        self.access_code = access_code
        self.printer_serial = printer_serial
        self.base_url = f"http://{printer_host}"

    # ---- FTPS Upload ----

    def upload_gcode(self, file_path: str) -> None:
        """Upload a GCODE/3MF file to the printer via FTPS."""
        if not self._file_exists(file_path):
            raise FileNotFoundError(f"GCODE file not found: {file_path}")

        filename = Path(file_path).name
        logger.info(f"Uploading {filename} to printer {self.printer_host}...")

        ftp = ftplib.FTP_TLS()
        try:
            ftp.connect(self.printer_host, FTP_PORT)
            ftp.login(FTP_USER, self.access_code)
            ftp.prot_p()  # Secure data channel

            with open(file_path, "rb") as f:
                ftp.storbinary(f"STOR {filename}", f)

            logger.info(f"Uploaded {filename} successfully")
        finally:
            try:
                ftp.quit()
            except Exception:
                pass

    # ---- Send GCODE Command ----

    def send_gcode_command(
        self,
        filename: str,
        plate_index: int = 0,
        use_ams: bool = False,
    ) -> bool:
        """Send a command to the printer to start printing the uploaded file.

        Uses the Bambu Lab LAN HTTP API.
        """
        url = f"{self.base_url}/v1/iot/{self.printer_serial}/command"
        payload = {
            "command": "gcode_file",
            "param": f"{filename}",
            "plate_index": plate_index,
            "use_ams": use_ams,
            "access_code": self.access_code,
        }

        try:
            resp = requests.post(url, json=payload, timeout=30)
            resp.raise_for_status()
            logger.info(f"Print command sent for {filename}: HTTP {resp.status_code}")
            return True
        except requests.RequestException as e:
            logger.error(f"Failed to send print command: {e}")
            return False

    # ---- Main Dispatch ----

    def dispatch(self, order: Order) -> bool:
        """Full dispatch: upload + start print for an order."""
        logger.info(
            f"Dispatching order {order.id}: {order.product_name} → {order.gcode_file_path}"
        )

        # 1. Upload
        try:
            self.upload_gcode(order.gcode_file_path)
        except Exception as e:
            logger.error(f"Upload failed for order {order.id}: {e}")
            return False

        # 2. Start print
        filename = Path(order.gcode_file_path).name
        success = self.send_gcode_command(filename=filename)

        if success:
            logger.info(f"Order {order.id} dispatched successfully")

        return success

    @staticmethod
    def _file_exists(path: str) -> bool:
        return os.path.isfile(path)
```

**Step 4: Run test to verify pass**

Run: `pytest tests/test_dispatcher.py -v`
Expected: PASS (4 tests)

**Step 5: Commit**

```bash
git add etsy_store/dispatcher.py tests/test_dispatcher.py
git commit -m "feat: add printer dispatcher (FTPS upload + LAN command)"
```

---

### Task 9: Metrics dashboard

**Objective:** Query the database for revenue, profit, order counts, print time summaries

**Files:**
- Create: `etsy_store/metrics.py`

**Step 1: Write failing test**

```python
# tests/test_metrics.py
import pytest
import tempfile
from datetime import datetime, timezone, timedelta
from etsy_store.db import init_db, get_session, Order, OrderStatus
from etsy_store.metrics import MetricsReporter


@pytest.fixture
def reporter():
    tmpdir = tempfile.mkdtemp()
    db_url = f"sqlite:///{tmpdir}/test.db"
    init_db(db_url)

    # Seed some orders
    session = get_session(db_url)
    orders = [
        Order(
            etsy_receipt_id=i, etsy_transaction_id=i*10, listing_id=100,
            product_name="test", property_values="{}",
            gcode_file_path="/tmp/a.gcode.3mf",
            status=OrderStatus.DONE,
            revenue_amount=5000, etsy_fee_amount=345,
            material_cost_estimate=300,
            print_time_seconds=3600,
        )
        for i in range(1, 6)  # 5 completed orders
    ]
    # Add one pending
    orders.append(Order(
        etsy_receipt_id=99, etsy_transaction_id=990, listing_id=100,
        product_name="test", property_values="{}",
        gcode_file_path="/tmp/a.gcode.3mf",
        status=OrderStatus.PENDING,
        revenue_amount=3000, etsy_fee_amount=215,
        material_cost_estimate=200,
    ))
    session.add_all(orders)
    session.commit()
    session.close()

    return MetricsReporter(db_url)


def test_total_revenue(reporter):
    summary = reporter.summary()
    # 5 completed + 1 pending = 6 orders
    assert summary["total_orders"] == 6
    assert summary["completed_orders"] == 5
    assert summary["total_revenue"] == 28000  # 5×5000 + 3000 = 28000 cents


def test_profit_calculation(reporter):
    summary = reporter.summary()
    total_revenue = summary["total_revenue"]
    total_fees = summary["total_fees"]  # 5×345 + 215 = 1940
    total_material = summary["total_material_cost"]  # 5×300 + 200 = 1700
    profit = summary["profit"]

    assert total_fees == 1940
    assert total_material == 1700
    assert profit == total_revenue - total_fees - total_material  # 28000 - 1940 - 1700 = 24360


def test_average_print_time(reporter):
    summary = reporter.summary()
    # 5 completed orders × 3600s = 18000s total, avg = 3600s
    assert summary["avg_print_time_seconds"] == 3600
    assert summary["total_print_time_seconds"] == 18000


def test_empty_database():
    tmpdir = tempfile.mkdtemp()
    db_url = f"sqlite:///{tmpdir}/test.db"
    init_db(db_url)
    reporter = MetricsReporter(db_url)

    summary = reporter.summary()
    assert summary["total_orders"] == 0
    assert summary["total_revenue"] == 0
    assert summary["profit"] == 0
```

**Step 2: Run test to verify failure**

Run: `pytest tests/test_metrics.py -v`
Expected: FAIL

**Step 3: Implement `etsy_store/metrics.py`**

```python
"""Metrics reporting — revenue, profit, print time summaries."""

import logging

from sqlalchemy import func

from etsy_store.db import Order, OrderStatus, get_session

logger = logging.getLogger(__name__)


class MetricsReporter:
    def __init__(self, db_url: str):
        self.db_url = db_url

    def summary(self) -> dict:
        """Return a summary dict with key metrics."""
        session = get_session(self.db_url)
        try:
            total = session.query(Order).count()
            completed = (
                session.query(Order)
                .filter(Order.status == OrderStatus.DONE)
                .count()
            )

            # Revenue and costs (all orders)
            rev_result = session.query(func.sum(Order.revenue_amount)).scalar() or 0
            fee_result = session.query(func.sum(Order.etsy_fee_amount)).scalar() or 0
            mat_result = (
                session.query(func.sum(Order.material_cost_estimate)).scalar() or 0
            )

            # Print time (completed only)
            pt_result = (
                session.query(func.sum(Order.print_time_seconds))
                .filter(Order.status == OrderStatus.DONE, Order.print_time_seconds.isnot(None))
                .scalar()
            ) or 0

            avg_print = (pt_result // completed) if completed > 0 else 0
            profit = rev_result - fee_result - mat_result

            return {
                "total_orders": total,
                "completed_orders": completed,
                "total_revenue": int(rev_result),
                "total_fees": int(fee_result),
                "total_material_cost": int(mat_result),
                "profit": int(profit),
                "total_print_time_seconds": int(pt_result),
                "avg_print_time_seconds": int(avg_print),
            }
        finally:
            session.close()
```

**Step 4: Run test to verify pass**

Run: `pytest tests/test_metrics.py -v`
Expected: PASS (4 tests)

**Step 5: Commit**

```bash
git add etsy_store/metrics.py tests/test_metrics.py
git commit -m "feat: add metrics reporter (revenue, profit, print time)"
```

---

### Task 10: CLI entry point (Typer)

**Objective:** Wire everything together into a runnable CLI

**Files:**
- Create: `etsy_store/cli.py`
- Modify: `etsy_store/mapper.py` (extract listing_name logic)

**Step 1: Write failing test**

```python
# tests/test_cli.py
import pytest
from unittest.mock import patch, MagicMock
from typer.testing import CliRunner
from etsy_store.cli import app

runner = CliRunner()


@patch("etsy_store.cli.Settings")
@patch("etsy_store.cli.EtsyClient")
@patch("etsy_store.cli.ProductMapper")
@patch("etsy_store.cli.OrderProcessor")
@patch("etsy_store.cli.PrintQueue")
@patch("etsy_store.cli.PrinterDispatcher")
@patch("etsy_store.cli.MetricsReporter")
@patch("etsy_store.cli.init_db")
def test_poll_command(
    mock_init_db, mock_metrics, mock_disp, mock_queue,
    mock_proc, mock_mapper, mock_client, mock_settings,
):
    # Setup mocks
    mock_settings.return_value.etsy_shop_id = 123
    mock_settings.return_value.poll_interval_seconds = 60
    mock_settings.return_value.database_url = "sqlite:///test.db"

    mock_client.return_value.get_new_receipts.return_value = iter([])

    mock_mapper.return_value.get_listing_config.return_value = {"folder": "test"}

    result = runner.invoke(app, ["poll", "--once"])
    assert result.exit_code == 0
    assert "Polling" in result.stdout


@patch("etsy_store.cli.MetricsReporter")
def test_metrics_command(mock_metrics):
    mock_metrics.return_value.summary.return_value = {
        "total_orders": 5,
        "completed_orders": 4,
        "total_revenue": 20000,
        "profit": 15000,
    }
    result = runner.invoke(app, ["metrics"])
    assert result.exit_code == 0
    assert "200.00" in result.stdout  # cents → dollars display
```

**Step 2: Run test to verify failure**

Run: `pytest tests/test_cli.py -v`
Expected: FAIL

**Step 3: Implement `etsy_store/cli.py`**

```python
"""CLI entry point — poll, metrics, queue management."""

import logging
import time
from datetime import datetime, timezone
from typing import Optional

import typer

from etsy_store.config import Settings
from etsy_store.db import init_db, OrderStatus
from etsy_store.etsy_client import EtsyClient
from etsy_store.mapper import ProductMapper
from etsy_store.processor import OrderProcessor
from etsy_store.queue import PrintQueue
from etsy_store.dispatcher import PrinterDispatcher
from etsy_store.metrics import MetricsReporter

logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s [%(levelname)s] %(name)s: %(message)s",
)
logger = logging.getLogger(__name__)

app = typer.Typer(help="Etsy Print Farm Automation")


def _build_components():
    """Build all components from settings."""
    settings = Settings()
    init_db(settings.database_url)

    etsy = EtsyClient(
        api_key=settings.etsy_api_key,
        refresh_token=settings.etsy_refresh_token,
        shop_id=settings.etsy_shop_id,
    )
    mapper = ProductMapper(settings.products_yaml_path)
    processor = OrderProcessor(db_url=settings.database_url, mapper=mapper)
    queue = PrintQueue(db_url=settings.database_url)
    dispatcher = PrinterDispatcher(
        printer_host=settings.printer_host,
        access_code=settings.printer_access_code,
        printer_serial=settings.printer_serial,
    )
    metrics = MetricsReporter(db_url=settings.database_url)

    return settings, etsy, mapper, processor, queue, dispatcher, metrics


@app.command()
def poll(
    once: bool = typer.Option(False, "--once", help="Poll once and exit"),
    interval: Optional[int] = typer.Option(None, "--interval", help="Override poll interval in seconds"),
):
    """Poll Etsy for new orders and queue them for printing."""
    settings, etsy, mapper, processor, queue, dispatcher, metrics = _build_components()
    poll_interval = interval or settings.poll_interval_seconds

    logger.info(f"Starting Etsy poller (interval: {poll_interval}s, shop: {settings.etsy_shop_id})")

    while True:
        logger.info("Polling for new orders...")
        last_poll = int(datetime.now(timezone.utc).timestamp()) - poll_interval - 10

        try:
            for receipt in etsy.get_new_receipts(min_created_timestamp=last_poll):
                new_orders = processor.process_receipt(receipt)
                for order in new_orders:
                    logger.info(f"  Queued: {order.product_name} (${order.revenue_amount/100:.2f})")

                    # Auto-dispatch if printer is idle
                    pending = queue.pending_count()
                    if pending == 1:  # Only the one we just added
                        dispatched = dispatcher.dispatch(order)
                        if dispatched:
                            queue.update_status(order.id, order.status)  # stays PRINTING
                        else:
                            logger.error(f"Dispatch failed for order {order.id}")
                    else:
                        # Queue it, dispatcher loop handles it separately
                        pass

        except Exception as e:
            logger.error(f"Poll error: {e}")

        if once:
            break

        logger.info(f"Sleeping {poll_interval}s...")
        time.sleep(poll_interval)


@app.command()
def metrics():
    """Show metrics dashboard."""
    _, _, _, _, _, _, reporter = _build_components()
    summary = reporter.summary()

    typer.echo("═══ Etsy Print Farm Metrics ═══")
    typer.echo(f"Total Orders:     {summary['total_orders']}")
    typer.echo(f"Completed:        {summary['completed_orders']}")
    typer.echo(f"")
    typer.echo(f"Revenue:          ${summary['total_revenue']/100:.2f}")
    typer.echo(f"Etsy Fees:        ${summary['total_fees']/100:.2f}")
    typer.echo(f"Material Cost:    ${summary['total_material_cost']/100:.2f}")
    typer.echo(f"─────────────────────────────")
    typer.echo(f"Profit:           ${summary['profit']/100:.2f}")
    typer.echo(f"")
    typer.echo(f"Total Print Time: {summary['total_print_time_seconds']//3600}h {(summary['total_print_time_seconds']%3600)//60}m")
    typer.echo(f"Avg Print Time:   {summary['avg_print_time_seconds']//60}m {summary['avg_print_time_seconds']%60}s")


@app.command()
def queue():
    """Show current print queue."""
    _, _, _, _, queue, _, _ = _build_components()
    pending = queue.pending_count()
    typer.echo(f"Pending jobs in queue: {pending}")


@app.command()
def dispatch():
    """Manually dispatch the next job in the queue."""
    _, _, _, _, queue, dispatcher, _ = _build_components()
    next_job = queue.dequeue_next()
    if next_job is None:
        typer.echo("No pending jobs in queue.")
        return

    typer.echo(f"Dispatching: {next_job.product_name} → {next_job.gcode_file_path}")
    success = dispatcher.dispatch(next_job)
    if success:
        typer.echo("✓ Dispatched successfully")
    else:
        typer.echo("✗ Dispatch failed")
        queue.update_status(next_job.id, OrderStatus.PENDING)
```

**Step 4: Run test to verify pass**

Run: `pytest tests/test_cli.py -v`
Expected: PASS (2 tests)

**Step 5: Commit**

```bash
git add etsy_store/cli.py tests/test_cli.py
git commit -m "feat: add CLI entry point (poll, metrics, queue, dispatch)"
```

---

### Task 11: Run all tests and verify

**Objective:** Ensure the full test suite passes

**Step 1: Run the full test suite**

Run: `pytest tests/ -v`
Expected: ALL TESTS PASS (~20+ tests)

**Step 2: Check for lint errors**

Run: `ruff check etsy_store/ tests/`
Expected: No errors (or only minor style warnings)

**Step 3: Commit**

```bash
git commit -m "test: full test suite passing, lint clean"
```

---

## Verification Checklist

- [ ] `Settings` loads all required fields from `.env`
- [ ] `init_db` creates the `order` table with correct schema
- [ ] `Order` model accepts PENDING, PRINTING, DONE, FAILED statuses
- [ ] `ProductMapper.resolve()` maps `listing_id + {"Size": "22 inch"}` to correct GCODE path
- [ ] `ProductMapper.resolve()` works for listings without property variations (`property: null`)
- [ ] `ProductMapper.resolve()` raises `ValueError` on unknown listing
- [ ] `ProductMapper.resolve()` raises `ValueError` on unknown property value
- [ ] `EtsyClient.get_new_receipts()` yields receipt dicts with pagination
- [ ] `EtsyClient.get_receipt_detail()` returns receipt with `transactions[]` array
- [ ] `OrderProcessor.process_receipt()` creates Order records from Etsy receipts
- [ ] `OrderProcessor.process_receipt()` skips already-processed receipts (idempotent)
- [ ] `OrderProcessor.process_receipt()` extracts property values from transaction variations
- [ ] `PrintQueue.dequeue_next()` returns oldest PENDING and marks it PRINTING
- [ ] `PrintQueue.dequeue_next()` returns `None` when queue is empty
- [ ] `PrintQueue.update_status()` correctly updates status and print time
- [ ] `PrinterDispatcher.upload_gcode()` calls FTPS with correct host/port/credentials
- [ ] `PrinterDispatcher.send_gcode_command()` sends correct HTTP payload to printer API
- [ ] `PrinterDispatcher.dispatch()` chains upload + send as a single flow
- [ ] `MetricsReporter.summary()` returns correct `total_revenue`, `total_fees`, `profit`
- [ ] `MetricsReporter.summary()` handles empty database gracefully
- [ ] `poll --once` exits cleanly without errors
- [ ] `metrics` command displays formatted output
- [ ] `dispatch` command processes next queue item
- [ ] All existing tests pass (`pytest tests/ -v`)
- [ ] No lint errors (`ruff check`)

---

## Design Decisions

| Decision | Rationale |
|----------|-----------|
| **SQLite over PostgreSQL** | Single-machine deployment, zero operational overhead. Scales fine for a print farm (thousands of orders). |
| **Polling over webhooks** | Etsy webhooks require a public HTTPS endpoint. Polling every 5 minutes is simpler for a home server and uses ~288 API calls/day (well within Etsy's rate limits). |
| **Pre-sliced GCODE, not STL** | Eliminates slicing compute at order time. Seller controls quality. Bambu Lab P1S accepts 3MF with embedded GCODE. |
| **YAML for product mapping** | Human-editable config. Easier for non-developers than JSON. Supports comments. |
| **FTPS for upload, not MQTT** | Bambu Lab P1S supports FTPS natively for file transfer. MQTT is better suited for status monitoring, not bulk file upload. |
| **Typer over argparse** | Cleaner CLI with auto-generated help, type validation. Same ecosystem as the rest of the stack. |
| **Etsy fee estimate at 6.5% + $0.20** | Close enough for profit estimation. Actual fees vary by region and payment method. Exact fee data is available in Etsy receipts for v2. |
| **Single printer in v1** | Bambu Lab P1S is one printer. Multi-printer scheduling is a v2 feature. |
| **Manual bed clearing** | No off-the-shelf automation for removing prints from a bed. Human step is the hard constraint. |

---

## Future Enhancements (not in this plan)

- **Multi-printer support:** Load balance across multiple P1S/A1 printers, respecting filament type and nozzle size
- **Etsy webhooks:** Replace polling with real-time order notifications (requires public endpoint or tunnel)
- **Auto-slicing:** Accept STL files and slice on-the-fly with Bambu Studio CLI, applying pre-configured profiles
- **Shipping integration:** Mark orders as fulfilled on Etsy, generate shipping labels
- **Filament tracking:** Track AMS filament usage, alert when spools are low
- **Failure detection:** Monitor MQTT for print failures (spaghetti detection on P1S), auto-retry or alert
- **Material cost auto-calculation:** Parse 3MF metadata for filament grams used, multiply by cost-per-gram from config
- **Dashboard web UI:** Simple Flask/FastAPI dashboard showing queue, printer status, metrics
- **Notification integration:** Telegram/Discord notifications for order received, print started, print done, errors
- **Listing template sync:** Keep Etsy listing variations in sync with local product folders

---

## Resource Impact

| Resource | Estimate | Notes |
|----------|----------|-------|
| RAM | ~50 MB | Python process + SQLite. Negligible for any modern system. |
| Storage | ~10 KB/order | SQLite row per order. 10K orders = ~100 MB. GCODE files stored separately. |
| CPU | Minimal | Polls every 5 min, negligible between polls. |
| API calls | ~12/hour | ~288 calls/day for 5-min polling. Etsy limit is thousands/hour. |
| Network | <1 MB/day | JSON API responses are small. GCODE upload uses LAN bandwidth. |

---

**Plan complete.** Ready to execute using `subagent-driven-development` — each task dispatched to a fresh subagent with two-stage review. Shall I proceed?
