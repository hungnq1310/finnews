# PLAN_V2: Kafka-Based Event-Driven Architecture Migration

> **Version:** 2.0  
> **Date:** 2026-04-03  
> **Status:** Feasibility Analysis & Implementation Plan

---

## Context

The current Finhouse architecture (v3.0) uses a **monolithic FastAPI** application with direct database writes to MongoDB and ClickHouse. The proposed architecture introduces an **event-driven microservices architecture** with Kafka as the central message bus.

This plan analyzes whether the proposed architecture can be applied and provides a phased migration roadmap.

---


## Executive Summary

| Aspect | Current (v3.0) | Proposed (Updated) | Feasibility |
|--------|----------------|-------------------|-------------|
| **API Pattern** | Monolithic FastAPI | Microservices + API Gateway | ✅ High |
| **Message Queue** | None (direct DB writes) | Kafka/Redpanda (market data only) | ✅ High |
| **Stream Processing** | None | ClickHouse native (+ optional Faust) | ✅ High |
| **Document Storage** | MongoDB | Elasticsearch only | ✅ High |
| **Data Ingestion** | Direct API calls | Windmill → ES direct, Kafka for ticks | ✅ High |
| **Deployment** | Single container | Multi-container orchestration | ✅ High |

**Conclusion**: The updated architecture is **technically feasible** with a **simplified implementation approach**. Key changes from original plan: Windmill writes directly to Elasticsearch, MongoDB is removed, and Kafka is used only for market data.


## Architecture Comparison

### Current Architecture (v3.0)

```
┌─────────────────────────────────────────────────────────────────────┐
│                         DATA SOURCES LAYER                          │
│  MT5 Terminal  │     VNStock     │  Financial Reports  │  Web APIs  │
└────────┬────────────────┬────────────────┬────────────────┬─────────┘
         │                │                │                │
         ▼                ▼                ▼                ▼
┌────────────────────────────────────────────────────────────────────┐
│                      PROCESSING LAYER                              │
│                    Windmill Workflows + MT5 Scripts                │
└────────┬────────────────────────────────────────┬──────────────────┘
         │                                        │
         ▼                                        ▼
┌─────────────────────────────┐    ┌────────────────────────────────┐
│     MONGODB (Documents)     │    │   CLICKHOUSE (Time-Series)     │
│ • Symbol metadata           │    │ • OHLCV data                   │
│ • News articles             │    │ • Order book                   │
│ • Financial reports         │    │ • Aggregations                 │
└─────────────────────────────┘    └────────────────────────────────┘
         │                                        │
         └────────────────┬───────────────────────┘
                          ▼
                 ┌────────────────────────────────────────┐
                 │      REST API V3 (FastAPI)             │
                 │  Single monolithic application        │
                 └────────────────────────────────────────┘
```

### Proposed Architecture (Updated)

```
┌─────────────────────────────────────────────────────────────────────┐
│                         DATA SOURCES                                │
│  MT5 Terminal  │  Windmill Jobs (News, BCTC)  │   VNStock/APIs     │
└────────┬────────────────┬────────────────────────┬─────────────────┘
         │                │                        │
         ▼                ▼                        ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    INGESTION LAYER                                  │
│  ┌──────────────────┐         ┌──────────────────────────────────┐ │
│  │   Kafka/Redpanda │         │   Windmill ES Writer             │ │
│  │   (market-ticks) │         │   (Direct to Elasticsearch)      │ │
│  └────────┬─────────┘         └────────────────┬─────────────────┘ │
└───────────┼────────────────────────────────────┼───────────────────┘
            │                                    │
            ▼                                    ▼
┌─────────────────────────┐      ┌──────────────────────────────────┐
│  Market Data Service    │      │      ELASTICSEARCH               │
│  (Consumes Kafka)       │      │   • News articles                │
└───────────┬─────────────┘      │   • Financial reports (BCTC)     │
            │                    │   • Symbol metadata               │
            ▼                    │   • Company information          │
┌─────────────────────────┐      │   • ICB industries               │
│     CLICKHOUSE          │      └──────────────────────────────────┘
│   (OHLCV time-series)   │
└─────────────────────────┘
            │
┌───────────┴─────────────────────────────────────────────────────────┐
│                      API GATEWAY                                    │
│  Routes /api/v1/market/* → Market Data Service                      │
│  Routes /api/v1/search/* → Elasticsearch (via search service)       │
└───────────────────────────────┬─────────────────────────────────────┘
                                │
                        ┌───────┴───────┐
                        │  Web / Mobile │
                        │    Clients    │
                        └───────────────┘
```

**Key Changes from Original Plan:**
- **Windmill writes directly to Elasticsearch** - No API Gateway hop for News/BCTC data
- **MongoDB removed** - All document storage moves to Elasticsearch
- **Single Kafka topic** - Only `market-ticks` for high-frequency market data
- **Simplified architecture** - Fewer moving parts, faster implementation