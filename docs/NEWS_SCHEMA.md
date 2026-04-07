# Tài Liệu Kỹ Thuật: Hệ Thống Phân Tích Tin Tức Tài Chính Tự Động

**Version:** 3.0
**Ngày cập nhật:** 2026-01-22
**Tech Stack:** Windmill, MongoDB, LLM (GPT-4/Claude), Python
**Pydantic Schema:** `src/finhouse/schema/core.py`

> **Note:** Tất cả timestamps trong hệ thống được lưu dạng **Int64 (milliseconds since epoch)** để đảm bảo tính nhất quán và dễ dàng xử lý cross-platform.

---

## 📋 Mục Lục

1. [Tổng Quan Hệ Thống](#1-tổng-quan-hệ-thống)
2. [Kiến Trúc Workflow](#2-kiến-trúc-workflow)
3. [Schema MongoDB](#3-schema-mongodb)


---

## 1. Tổng Quan Hệ Thống

### 1.1 Mục Tiêu

Xây dựng hệ thống tự động hóa hoàn toàn quy trình:
- **Thu thập** tin tức tài chính từ nhiều nguồn
- **Trích xuất** thông tin có cấu trúc bằng LLM
- **Phân tích** và tạo báo cáo tự động theo lịch
- **Tổng hợp** từ cấp độ cổ phiếu → ngành → thị trường

### 1.2 Công Nghệ Sử Dụng

| Component | Technology | Lý Do Chọn |
|-----------|-----------|------------|
| Workflow Engine | Windmill | Visual workflow, schedule jobs, error handling |
| Database | MongoDB | Flexible schema, document-oriented, scale tốt |
| LLM | GPT-4 Turbo / Claude 3.5 Sonnet | Extraction & generation quality cao |
| Queue | Redis (optional) | Rate limiting, job queue |
| Storage | MongoDB GridFS | Lưu trữ HTML/PDF raw |

### 1.3 Data Flow Overview

```
┌───────────────────────────────────────────────────────────────────────────┐
│                    REALTIME CRAWLING WORKFLOW                             │
│                                                                           │
│  RSS Feeds/Web Sources  →  Crawler  →  Raw HTML    →     LLM Extract      │
│         ↓                                                        ↓        │
│    Schedule Check                                        MongoDB Insert   │
│    (Continuous)                                          (news_articles)  │
└───────────────────────────────────────────────────────────────────────────┘

┌───────────────────────────────────────────────────────────────────┐
│              SCHEDULED ANALYSIS WORKFLOW (3x/day)                 │
│                     8:30 AM | 12:00 PM | 4:00 PM                  │
│                                                                   │
│  Query News  →  Stock Analysis  →  Sector Analysis  →  Market     │
│ (Last Period)      (LLM)            (LLM)          Overview(LLM)  │
│       ↓              ↓                 ↓                 ↓        │
│   Filter by     Write Reports     Aggregate by     Final Report   │
│   Companies       per Ticker         Sector         Generation    │
│    & Days            ↓                 ↓                ↓         │
│                stock_reports     sector_reports   market_reports  │
└───────────────────────────────────────────────────────────────────┘
```

---
...
---

## 3. Schema MongoDB

### 3.1 Pydantic Model: NewsArticle

```python
class NewsArticle(BaseModel):
    """News article model with extracted information"""
    article_id: str = Field(..., description="Unique article identifier UUID")
    original_url: str = Field(..., description="URL of the article")
    source: dict = Field(..., description="Source information including name, domain, credibility_score")
    """
    {
      name: String,              // Source name (Bloomberg, Reuters, etc.)
      domain: String,            // e.g., bloomberg.com
      credibility_score: Float   // 0.0-1.0
    }
    """
    content: dict = Field(..., description="Article content including headline, subheadline, summary, body, author")
    """
    {
      headline: String,          // Main headline
      subheadline: String,       // Subheadline if exists
      summary: String,           // Brief 2-3 sentence summary
      body: String,              // Full article text, cleaned
      author: String             // Author name if available
    }
    """
    timing: dict = Field(..., description="Timing information including published_at, updated_at, market_session")
    """
    {
      published_at: Int64,       // Timestamp in milliseconds
      updated_at: Int64,         // Timestamp in milliseconds
      market_session: String     // pre_market, market_hours, after_hours, closed
    }
    """
    classification: dict = Field(..., description="Article classification including primary_category, sub_categories, topics")
    """
    {
      primary_category: String,   // economics, technology, politics, general
      sub_categories: List[String], // [finance, technology, healthcare, consumer_discretionary, industrials, energy, utilities, real_estate, materials, communication_services, consumer_staples, stock, bond, commodity, currency, crypto, etf, mutual_fund, banks, housing, macro_economy, global_markets, jobs_employment, taxes_rates, inflation, insurance, retail, automotive, semiconductors, software, biotech, oil_gas, mining, transportation]
      topics: List[String]        // [financial_performance, market_expansion, strategic_partnership, ...]
    }
    """
    companies_mentioned: List[dict] = Field(..., description="List of companies mentioned with relevance and sentiment")
    """
    [
      {
        symbol_code: String,      // e.g., VNM
        exchange_code: String,      // e.g., HOSE
        company_name: String,     // e.g., Vinamilk
        relevance_score: Float,   // 0.0-1.0 scale
        mention_type: String,     // primary_subject, secondary_subject, mentioned
        sentiment: Float,         // -1.0 to 1.0
        context: String           // Brief context of how company is mentioned
      },
      ...
    ]
    """
    events_extracted: List[dict] = Field(..., description="List of extracted events with details")
    """
    [
      {
        event_type: String,       // bankruptcy_liquidation, shareholder_reduction, equity_pledge, shareholder_increase, equity_freeze, senior_executive_death, major_asset_loss, major_external_compensation, major_safety_incident
        description: String,      // Brief description
        companies_affected: List[String], // ["VNM"]
        impact_magnitude: Float,  // 0.0-1.0
        confidence: Float         // 0.0-1.0
      },
      ...
    ]
    """
    sentiment: dict = Field(..., description="Overall sentiment analysis of the article")
    """
    {
      overall_sentiment: Float,    // -1.0 to 1.0
      sentiment_magnitude: Float,  // 0.0-1.0
      emotional_tone: {
        fear: Float,               // 0.0-1.0
        greed: Float,              // 0.0-1.0
        optimism: Float,           // 0.0-1.0
        pessimism: Float,          // 0.0-1.0
        urgency: Float             // 0.0-1.0
      },
      market_impact_score: Float   // 0.0-1.0
    }
    """
    confidence_score: float = Field(..., description="Overall extraction confidence score 0.0-1.0")
    metadata: dict = Field(..., description="Processing status and extraction metadata")
    """
    {
      model: String,               // e.g., "gpt-oss"
      token_count: Dict[String, Number],
      process_status: String,      // pending, processing, completed, failed
      processed_at: Int64,         // Timestamp in milliseconds
      error_message: String,
      retry_count: Number
    }
    """
    created_at: int = Field(..., description="Article creation timestamp in milliseconds")
```

---

### 3.3 Collection: news_articles

**Mục đích:** Tin tức đã được LLM trích xuất thành cấu trúc. Đồng bộ với `NewsArticle` Pydantic model.

```javascript
{
  _id: "uuid-v4",
  article_id: "uuid-v4",              // Unique article identifier
  original_url: "https://bloomberg.com/...",

  source: {
    name: "Bloomberg",
    domain: "bloomberg.com",
    credibility_score: 0.92           // 0.0-1.0
  },

  content: {
    headline: "Apple Announces Record Quarterly Revenue",
    subheadline: "iPhone sales drive growth amid market challenges",
    summary: "Apple Inc. reported record-breaking quarterly revenue...",
    body: "Full article text...",
    author: "Jane Smith"
  },

  timing: {
    published_at: Int64,              // Timestamp in milliseconds
    updated_at: Int64,                // Timestamp in milliseconds
    market_session: "market_hours"    // pre_market, market_hours, after_hours, closed
  },

  classification: {
    primary_category: "technology",   // economics, technology, politics, general
    sub_categories: ["technology", "consumer_electronics"],
    topics: ["financial_performance", "market_expansion"]
  },

  companies_mentioned: [
    {
      symbol_code: "AAPL",
      exchange_code: "NASDAQ",
      company_name: "Apple Inc.",
      relevance_score: 0.95,          // 0.0-1.0 scale
      mention_type: "primary_subject", // primary_subject, secondary_subject, mentioned
      sentiment: 0.65,                // -1.0 to 1.0
      context: "Apple's Q4 results exceeded expectations with 8% revenue growth"
    },
    {
      symbol_code: "GOOGL",
      exchange_code: "NASDAQ",
      company_name: "Alphabet Inc.",
      relevance_score: 0.25,
      mention_type: "mentioned",
      sentiment: 0.3,
      context: "Compared to Google's 5% growth in the same period"
    }
  ],

  events_extracted: [
    {
      event_type: "major_external_compensation",
      description: "Apple reported Q4 revenue of $89.5B, beating estimates of $87.2B",
      companies_affected: ["AAPL"],
      impact_magnitude: 0.89,         // 0.0-1.0
      confidence: 0.92                // 0.0-1.0
    }
  ],

  sentiment: {
    overall_sentiment: 0.68,          // -1.0 to 1.0
    sentiment_magnitude: 0.85,        // 0.0-1.0
    emotional_tone: {
      fear: 0.05,                     // 0.0-1.0
      greed: 0.15,                    // 0.0-1.0
      optimism: 0.75,                 // 0.0-1.0
      pessimism: 0.10,                // 0.0-1.0
      urgency: 0.45                   // 0.0-1.0
    },
    market_impact_score: 0.78         // 0.0-1.0
  },

  confidence_score: 0.92,             // 0.0-1.0

  metadata: {
    model: "gpt-4-turbo-2024-04-09",
    token_count: {
      input: 5000,
      output: 800,
      total: 5800
    },
    process_status: "completed",      // pending, processing, completed, failed
    processed_at: Int64,              // Timestamp in milliseconds
    error_message: null,
    retry_count: 0
  },

  created_at: Int64                   // Timestamp in milliseconds
}
```

**Indexes:**
```javascript
db.news_articles.createIndex({ "article_id": 1 }, { unique: true })
db.news_articles.createIndex({ "original_url": 1 }, { unique: true })
db.news_articles.createIndex({ "timing.published_at": -1 })
db.news_articles.createIndex({ "companies_mentioned.symbol_code": 1 })
db.news_articles.createIndex({ "companies_mentioned.exchange_code": 1 })
db.news_articles.createIndex({ "classification.primary_category": 1 })
db.news_articles.createIndex({ "sentiment.overall_sentiment": 1 })
db.news_articles.createIndex({ "created_at": -1 })
```

---
...
---

**Document Version:** 3.0
**Last Updated:** 2026-01-22
**Pydantic Schema:** `src/finhouse/schema/core.py`