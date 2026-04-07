# README: Database Schema Documentation

## Tổng quan (Overview)

Tài liệu này mô tả kiến trúc cơ sở dữ liệu cho hệ thống giao dịch tài chính đa tài sản (multi-asset), sử dụng **MongoDB** cho dữ liệu thông tin tĩnh và **ClickHouse** cho dữ liệu chuỗi thời gian (time-series).

> **Note:** Tất cả timestamps trong hệ thống được lưu dạng **Int64 (milliseconds since epoch)** để đảm bảo tính nhất quán và dễ dàng xử lý cross-platform.

## Kiến trúc Database (Database Architecture)

### MongoDB - Information Storage
Lưu trữ thông tin tham chiếu, cấu hình và metadata ít thay đổi.

### ClickHouse - Time-Series Data
Lưu trữ dữ liệu giá, volume và snapshot theo thời gian thực.

---

## Enums & Constants

```python
class AssetType(str, Enum):
    FOREX = "FOREX"
    STOCK = "STOCK"
    CRYPTO = "CRYPTO"
    INDEX = "INDEX"
    ETF = "ETF"
    BOND = "BOND"
    FUTURE = "FUTURE"
    COMMODITY = "COMMODITY"
    MUTUAL_FUND = "MUTUAL_FUND"

class Timeframe(str, Enum):
    S1 = "1s"
    M1 = "m1"
    M5 = "m5"
    M15 = "m15"
    M30 = "m30"
    H1 = "h1"
    H4 = "h4"
    D1 = "d1"
    W1 = "w1"
    MN1 = "mn1"
```

---

## I. MongoDB Collections - Thông tin tĩnh

### 1. `market_exchange` - Thông tin Sàn Giao dịch

```javascript
{
  "_id": ObjectId("..."),
  "exchange_id": String,          // UUID format
  "exchange_code": String,        // Mã sàn: 'HOSE', 'NYSE', 'HNX', 'BINANCE'
  "exchange_name": String,        // Tên sàn, unique
  "country_code": String,         // ISO 3166-1 alpha-2: 'VN', 'US'
  "timezone": String,             // 'Asia/Ho_Chi_Minh', 'America/New_York'
  "currency": String,             // ISO 4217: 'VND', 'USD'
  "last_updated": Int64,          // Timestamp in milliseconds
  
  // Additional metadata (optional)
  "metadata": {
    "regular_open_time": String,      // "09:00:00+07:00"
    "regular_close_time": String,     // "15:00:00+07:00"
    "pre_market_open": String,        // Optional
    "after_market_close": String,     // Optional
    "settlement_period_days": Number, // T+2, T+3
    "margin_trading_allowed": Boolean,
    "short_selling_allowed": Boolean,
    "data_source": String             // e.g., "MT5", "Bloomberg"
  }
}
```

**Indexes:**
```javascript
db.market_exchange.createIndex({ "exchange_id": 1 }, { unique: true })
db.market_exchange.createIndex({ "exchange_code": 1 }, { unique: true })
db.market_exchange.createIndex({ "exchange_name": 1 }, { unique: true })
db.market_exchange.createIndex({ "country_code": 1 })
```

---

### 2. `company_information` - Thông tin Công ty

```javascript
{
  "_id": ObjectId("..."),
  "company_id": String,               // UUID format
  "company_name": String,             // Tên công ty tiếng Anh
  "company_name_native": String,      // Tên công ty bản địa
  "last_updated": Int64,              // Timestamp in milliseconds
  
  // Thông tin bổ sung (optional)
  "address": String,
  "website": String,
  "founding_date": Int64,             // Timestamp in milliseconds
  "phone": String,
  "email": String
}
```

**Indexes:**
```javascript
db.company_information.createIndex({ "company_id": 1 }, { unique: true })
db.company_information.createIndex({ "company_name_native": "text", "company_name": "text" })
```

---

### 3. `icb_industry` - Phân loại Ngành (Industry Classification Benchmark)

```javascript
{
  "_id": ObjectId("..."),
  "icb_id": String,                   // UUID format
  "icb_code": String,                 // Mã ngành unique: "1000", "1100", "1110"
  "icb_name": String,              // Tên ngành: "Technology", "Software"
  "icb_name_native": String,       // Tên ngành bản địa: "Công nghệ Thông tin"
  "icb_level": Number,                // 1=Industry, 2=Supersector, 3=Sector, 4=Subsector
  "parent_icb_id": String,            // Reference đến parent UUID, null nếu là root
  "path": String,                     // Materialized path: "/1000/1100/1110"
  "last_updated": Int64               // Timestamp in milliseconds
}
```

**Indexes:**
```jvascript
db.icb_industry.createIndex({ "icb_id": 1 }, { unique: true })
db.icb_industry.createIndex({ "icb_code": 1 }, { unique: true })
db.icb_industry.createIndex({ "parent_icb_id": 1 })
db.icb_industry.createIndex({ "icb_level": 1 })
db.icb_industry.createIndex({ "path": 1 })
```

---
### 4. `symbol_icb_mapping` - Mapping Symbol với ICB Industry

```javascript
{
  "_id": ObjectId("..."),
  "symbol_id": String,                // UUID format, reference to symbol_information
  "icb_id": String,                   // UUID format, reference to icb_industry
  "created_at": Int64,                // Assignment timestamp in milliseconds
  "last_updated": Int64               // Last updated timestamp in milliseconds
}
```

**Indexes:**
```javascript
db.symbol_icb_mapping.createIndex({ "symbol_id": 1, "icb_id": 1 }, { unique: true })
db.symbol_icb_mapping.createIndex({ "icb_id": 1 })
```

---


### 5. `symbol_information` - Thông tin Mã chứng khoán/Tài sản

```javascript
{
  "_id": ObjectId("..."),
  "symbol_id": String,                // Format: {exchange:symbol_code} -> elastic text search
  "symbol_code": String,              // VNM, BTC-USDT, EUR/USD, VN30F2025M03
  "exchange_id": String,              // UUID, Reference to market_exchange
  "is_active": Boolean,               // Whether symbol is active
  "asset_type": String,               // 'FOREX', 'STOCK', 'CRYPTO', 'INDEX', 'ETF', 'BOND', 'FUTURE', 'COMMODITY', 'MUTUAL_FUND'
  
  // Optional references
  "company_id": String,               // UUID, Reference to company_information (nullable)
  "icb_id": String,                   // UUID, Reference to icb_industry (nullable)
  // Important dates
  "created_at": Int64,              // Timestamp in milliseconds
  "last_updated": Int64,              // Timestamp in milliseconds
  
  // Asset-specific metadata (flexible subdocument, optional)
  "asset_metadata": {
    // For STOCK
    "stock_type": String,             // 'Common Stock', 'Preferred Stock'
    "par_value": Number,
    "trading_unit": Number,
    
    // For CRYPTO
    "blockchain": String,             // 'Ethereum', 'BSC'
    "contract_address": String,
    "max_supply": Number,
    
    // For FOREX
    "base_currency": String,          // 'USD'
    "quote_currency": String,         // 'VND'
    "pip_value": Number,
    
    // For FUTURE
    "underlying_asset": String,
    "contract_month": String,
    "contract_multiplier": Number,
    
    // For BOND
    "issuer": String,
    "coupon_rate": Number,
    "maturity_date": Int64,           // Timestamp in milliseconds
    
    // For ETF/MUTUAL_FUND
    "fund_name": String,
    "expense_ratio": Number,
    "benchmark_index": String
  },

}
```

**Indexes:**
```javascript
db.symbol_information.createIndex({ "symbol_id": 1 }, { unique: true })
db.symbol_information.createIndex({ "symbol_code": 1, "exchange_id": 1 }, { unique: true })
db.symbol_information.createIndex({ "asset_type": 1 })
db.symbol_information.createIndex({ "exchange_id": 1 })
db.symbol_information.createIndex({ "company_id": 1 })
db.symbol_information.createIndex({ "icb_id": 1 })
db.symbol_information.createIndex({ "is_active": 1 })
```

---

### 6. `index_information` - Thông tin Chỉ số

```javascript
{
  "_id": ObjectId("..."),
  "index_id": String,                 // UUID format
  "index_name": String,               // VN-Index, S&P 500, VN30
  "symbol_code": String,              // 'MXX', 'SPX', 'VN30'
  "exchange_id": String,              // UUID, Reference to market_exchange
  "created_at": Int64,                // Timestamp in milliseconds
  "last_updated": Int64,              // Timestamp in milliseconds

  // Additional metadata (optional)
  "metadata": {
    "calculation_method": String,     // 'Market-cap weighted', 'Price-weighted', 'Equal-weighted'
    "base_date": Int64,               // Timestamp in milliseconds
    "base_value": Number,             // Giá trị cơ sở
    "is_active": Boolean
  }
}
```

**Indexes:**
```javascript
db.index_information.createIndex({ "index_id": 1 }, { unique: true })
db.index_information.createIndex({ "index_code": 1, "exchange_id": 1 }, { unique: true })
```

---

### 7. `daily_market_metrics` - Daily Market Metrics Collection

**Mục đích:** Lưu trữ metrics thị trường hàng ngày từ scraping dữ liệu, sử dụng cho báo cáo ngành và thị trường.

**Enums:**
```python
class MarketSource(str, Enum):
    """Market exchanges"""
    HSX = "HSX"      # Ho Chi Minh City Stock Exchange
    HNX = "HNX"      # Hanoi Stock Exchange
    UPCOM = "UPCOM"  # Unlisted Public Company Market
    MT5 = "MT5"      # MetaTrader 5
    NASDAQ = "NASDAQ"
    NYSE = "NYSE"

class PeriodType(str, Enum):
    """Report period types"""
    MORNING = "morning"    # Before 12:00
    MIDDAY = "midday"      # 12:00 - 16:00
    AFTERNOON = "afternoon" # After 16:00
```

```javascript
{
  "_id": ObjectId("..."),
  "timestamp": ISODate("..."),           // ISO datetime string
  "trading_date": String,                // YYYY-MM-DD format
  "exchange_code": String,               // 'HSX', 'HNX', 'UPCOM', 'MT5' (default: 'HSX')
  "market": String,                      // exchange_name alias
  "period": String,                      // 'morning', 'midday', 'afternoon'
  "created_at": ISODate("..."),          // Document creation timestamp

  // Index data - List of IndexMetric
  "indexes": [
    {
      "index_id": String,                // UUID reference to index_information
      "index_value": Number,             // close_price reference to symbol_trade_snapshot
      "change": Number,                  // reference to symbol_trade_snapshot
      "change_percent": Number,          // reference to symbol_trade_snapshot
      "volume": Number,                  // total_volume - refers to symbol_trade_snapshot
      "net_value": Number,               // Optional - KLGD = total volume * price
      "advances": Number,                // Optional
      "declines": Number,                // Optional
      "no_change": Number,               // Optional
      "prev_volume": Number,             // KLGD from previous day
      "prev_net_value": Number           // GTGD from previous day
    }
  ],

  // Sector data - List of SectorMetric
  "top_sectors": [
    {
      "icb_code": String,                // ICB industry code: "6000", "7000"
      "icb_name": String,                // ICB industry name: "Financials", "Energy"
      "change_percent_return": {
        "1d": Number,                    // 1-day return percentage
        "1w": Number,                    // 1-week return percentage
        "1m": Number,                    // 1-month return percentage
        "3m": Number,                    // 3-month return percentage
        "6m": Number,                    // 6-month return percentage
        "1y": Number                     // 1-year return percentage
      },
      "market_cap": Number,              // Total market capitalization
      "market_weight_percentage": Number, // Weight relative to total market
      "last_updated": Int64,             // Unix timestamp in milliseconds
      "created_at": Int64                // Unix timestamp in milliseconds
    }
  ],

  // Top interested stocks
  "top_interested_stocks": [String],     // List of stock symbols: ["VCB", "VIC", "VHM"]

  // Exchange fluctuation - FluctuationMetric
  "fluctuation_metric": {
    "impact_up_stocks": [String],        // Stocks with positive impact
    "impact_up_total": Number,           // Total positive impact
    "impact_down_stocks": [String],      // Stocks with negative impact
    "impact_down_total": Number          // Total negative impact
  },

  // Foreign trading - ForeignTradingMetric
  "foreign_metric": {
    "top_buy": [
      {
        "symbol_code": String,
        "value": Number,                 // Trading value
        "vol": Number,                   // Trading volume
        "change_percent": Number         // Percentage change
      }
    ],
    "top_sell": [
      {
        "symbol_code": String,
        "value": Number,
        "vol": Number,
        "change_percent": Number
      }
    ]
  },

  // Investor sectors - InvestorMetric
  "investor_metric": {
    "proprietary_group": Number,         // Khối tự doanh ( proprietary trading)
    "foreign_volume": Number,            // Khối ngoại - trading volume
    "foreign_net_value": Number          // Khối ngoại - net trading value
  }
}
```

**Indexes:**
```javascript
db.daily_market_metrics.createIndex({ "trading_date": 1, "market": 1, "period": 1 })
db.daily_market_metrics.createIndex({ "timestamp": -1 })
db.daily_market_metrics.createIndex({ "market": 1 })
db.daily_market_metrics.createIndex({ "period": 1 })
db.daily_market_metrics.createIndex({ "created_at": -1 })

// Compound index for time-series queries
db.daily_market_metrics.createIndex(
  { "trading_date": -1, "market": 1, "period": 1 },
  { name: "market_metrics_lookup" }
)

// TTL index - auto-delete documents older than 90 days
db.daily_market_metrics.createIndex(
  { "created_at": 1 },
  { expireAfterSeconds: 7776000, name: "ttl_90_days" }  // 90 * 24 * 60 * 60
)
```

**Field Mapping Notes:**
- `indexes[].volume`: Corresponds to KLGD (Khối lượng giao dịch)
- `indexes[].net_value`: Corresponds to GTGD (Giá trị giao dịch)
- `investor_metric.proprietary_group`: Maps from API field `tu_doanh_value`
- `investor_metric.foreign_volume`: Maps from API field `ngoai_volume`
- `investor_metric.foreign_net_value`: Maps from API field `ngoai_net_value`

**Usage:**
- Collection được populate bởi Windmill flow thông qua FastAPI endpoints
- API endpoints: `/api/v1/metrics/market` (POST), `/api/v1/metrics/daily/{date}` (GET)
- Dữ liệu được sử dụng để generate sector reports và market reports

---

### 8. `sector_metrics` - Sector Performance Metrics Collection

**Mục đích:** Lưu trữ metrics hiệu suất ngành (ICB sector) từ dữ liệu thị trường, được update định kỳ để theo dõi diễn biến ngành.

```javascript
{
  "_id": ObjectId("..."),
  "icb_code": String,                // ICB industry code: "0001", "1000", "6000", "7000"
  "icb_name": String,                // ICB industry name: "Financials", "Energy", "Technology"
  "exchange_code": String,           // Reference to market_exchange code (default: 'HSX')

  // Percentage returns for different time periods
  "change_percent_return": {
    "1d": Number,                    // 1-day return percentage
    "1w": Number,                    // 1-week return percentage
    "1m": Number,                    // 1-month return percentage
    "3m": Number,                    // 3-month return percentage
    "6m": Number,                    // 6-month return percentage
    "1y": Number                     // 1-year return percentage
  },

  // Market capitalization and weight
  "market_cap": Number,              // Total market capitalization of sector (in billions)
  "market_weight_percentage": Number, // Weight of sector relative to total market

  // Timestamps
  "last_updated": Int64,             // Unix timestamp in milliseconds
  "created_at": Int64                // Unix timestamp in milliseconds
}
```

**Indexes:**
```javascript
db.sector_metrics.createIndex({ "icb_code": 1, "exchange_code": 1 }, { unique: true })
db.sector_metrics.createIndex({ "icb_code": 1 })
db.sector_metrics.createIndex({ "exchange_code": 1 })
db.sector_metrics.createIndex({ "change_percent_return.1d": -1 })
db.sector_metrics.createIndex({ "change_percent_return.1w": -1 })
db.sector_metrics.createIndex({ "market_cap": -1 })
db.sector_metrics.createIndex({ "last_updated": 1 })

// TTL index - cache expires after 24 hours
db.sector_metrics.createIndex(
  { "last_updated": 1 },
  { expireAfterSeconds: 86400, name: "ttl_24h" }  // 24 * 60 * 60
)
```

**ICB Code Reference (Vietnam Market):**
```javascript
// Major ICB Industry Codes
{
  "0001": "All Industries",
  "1000": "Oil & Gas",
  "2000": "Basic Materials",
  "3000": "Industrials",
  "4000": "Consumer Goods",
  "5000": "Health Care",
  "6000": "Consumer Services",
  "7000": "Telecommunications",
  "8000": "Utilities",
  "9000": "Financials",
  "10000": "Technology"
}
```

**Field Mapping Notes:**
- `change_percent_return`: Dictionary mapping time periods to return percentages
- `market_cap`: Total capitalization of all companies in the sector
- `market_weight_percentage`: (sector_market_cap / total_market_cap) * 100
- `last_updated`: Timestamp of last calculation/update
- `created_at`: Timestamp when the document was first created

**Usage:**
- Collection được update định kỳ (ví dụ: mỗi 30 phút hoặc cuối ngày)
- Dùng cho API endpoint: `/api/v1/sectors/top`, `/api/v1/sectors/{icb_code}`
- Dữ liệu được sử dụng để:
  - Hiển thị top gaining/losing sectors
  - So sánh hiệu suất ngành
  - Generate sector performance reports
  - Phân tích xu hướng dòng tiền giữa các ngành

**Example Query - Top 5 Sectors by 1D Return:**
```javascript
db.sector_metrics.find(
  { "exchange_code": "HSX" },
  { "icb_code": 1, "icb_name": 1, "change_percent_return.1d": 1 }
).sort({ "change_percent_return.1d": -1 }).limit(5)
```

---

## II. ClickHouse Tables - Time-Series Data

### Core Data Models (Python Pydantic)

```python
class TickData(BaseModel):
    """Real-time tick data for agg_symbol_snapshot_1s"""
    symbol_id: str               # {scode_ex-id_uuid} -> elastic text search
    symbol_code: str
    open_price: float
    high_price: float
    low_price: float
    close_price: float
    volume: float
    created_at: int              # Timestamp in milliseconds 


class OrderBookData(BaseModel):
    """Order book data for symbol_order_book - Vietnam stock market format"""
    symbol_id: str               # {scode_ex-id_uuid} -> elastic text search
    symbol_code: str
    created_at: int              # Timestamp in milliseconds 
    
    # Reference prices
    high_price: float            # Highest
    low_price: float             # Lowest
    prev_close_price: float      # prev_close
    
    # Bid levels 
    bid_price_3: Optional[float] = 0
    bid_volume_3: Optional[float] = 0
    bid_price_2: Optional[float] = 0
    bid_volume_2: Optional[float] = 0
    bid_price_1: float           # Best bid
    bid_volume_1: float
    
    # Matched order 
    close_price: float           # Close price
    volume: float                # Close volume
    price_change: float          # +/-
    price_change_percent: float  # +/- (%)
    
    # Ask levels
    ask_price_1: float           # Best ask
    ask_volume_1: float
    ask_price_2: Optional[float] = 0
    ask_volume_2: Optional[float] = 0
    ask_price_3: Optional[float] = 0
    ask_volume_3: Optional[float] = 0
    
    # Summary
    total_volume: float          # Total volume


class SnapshotData(BaseModel):
    """K-timeframe snapshot for symbol_trade_snapshot"""
    symbol_id: str               # {ex-name:scode} -> elastic text search
    symbol_code: str
    timeframe: str
    created_at: int              # Timestamp in milliseconds 
    open_price: float
    high_price: float
    low_price: float
    close_price: float
    volume: float
    price_change: float
    price_change_percent: float
```


---

## Contact & Support

For questions or issues, please contact the development team.

**Version:** 3.0  
**Last Updated:** 2026-02-03
**Pydantic Schema:** `src/finhouse/schema/core.py`