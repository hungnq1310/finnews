## 📋 Thông tin hệ thống hiện tại

```
Current Stack:
- FastAPI
- ClickHouse (time-series: OHLCV, orderbook)
- MongoDB (news, symbols, bctc)
- Python 3.11
- Docker Compose

Target Stack:
- Elasticsearch (document storage: news, financials, metadata)
```

**Files quan trọng cần đọc:**
- [PLAN_V2.md](./PLAN_V2.md) - Migration plan
- [ARCHITECT.md](./ARCHITECT.md) - Current architecture
- [src/finhouse/db.py](./src/finhouse/db.py) - Database connections
- [src/finhouse/schema/core.py](./src/finhouse/schema/core.py) - Data models


## Implementation Plan
- Migrate hệ thống sang DB Elastic Search
    + Data
    + API
    + Logic Flow
- Thêm API /search 
    + Note: Dùng repo [agentsearch](https://github.com/hungnq1310/elastic-agent/tree/dev) để kết nối với current DB

- Chỉnh Windmill flow trên server để insert data vào DB Elastic 