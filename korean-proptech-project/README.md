# Rebuild Analytics

> **Urban Redevelopment Feasibility Analysis Platform** - Map-based polygon selection for redevelopment investment decision support

---

## Project Overview

### Overview

Rebuild Analytics is a real estate data analysis service that allows users to intuitively define redevelopment/reconstruction target areas or regions of interest through map-based polygon selection, and instantly view feasibility analysis results for those areas.

Users simply click on the map to define boundaries, and the system automatically generates land registers and feasibility analysis results for the selected area. Based on aging conditions and real estate indices (transaction volume, average age, average sale price) applicable to each region, it provides comprehensive insights and enables users to run various profit/loss simulations by adjusting conditions.

### Core Value

- **Map-based Polygon Search**: Draw polygons on the map to instantly define redevelopment/reconstruction target areas
- **Real-time Feasibility Analysis**: Automatically generate comprehensive feasibility analysis based on land registers, aging conditions, and real estate indices
- **Profit/Loss Simulation**: Run instant profit/loss simulations by combining various conditions
- **Report Download**: Generate Excel and PDF analysis reports for easy sharing with third parties
- **AI Real Estate Consultation**: Query regional data in natural language and gain insights through LangGraph-based chatbot
- **Redevelopment Zone Integration**: Includes Seoul's officially designated redevelopment zones, regeneration promotion districts, and urban development areas

### Competitive Advantage

Rebuild Analytics is a data analysis tool specialized for redevelopment projects. Users can freely designate areas of interest in polygon form on the map and instantly view feasibility analysis results. The underlying data (land information, aging status, actual transaction prices, sale prices, etc.) is provided alongside the analysis, and results can be downloaded as Excel files, enabling users to run their own simulations or easily share with external experts.

**Future Value-Centered Decision Support**

Rebuild Analytics differentiates itself by supporting future value-centered decision-making, moving beyond traditional market price-based evaluations. As urban aging is expected to accelerate, evaluating regional value based solely on current market prices or officially assessed land prices is insufficient. More precise valuations require consideration of redevelopment potential, post-redevelopment asset values, and contribution ratios.

Reflecting this perspective, Rebuild Analytics provides not only current baseline data but also future development potential and profit/loss simulations across various scenarios, helping users make forward-looking decisions.

**Target Users**

Rebuild Analytics serves as a feasibility review tool for professionals, corporate investors, and individual investors alike, aiming to improve decision-making accuracy and efficiency in the information-asymmetric redevelopment sector.

---

## Tech Stack

```
Backend       Kotlin, Java 17, Spring Boot 3.3.2, FastAPI
AI/Agent      LangGraph, LangChain, OpenAI GPT-4o, MCP (Model Context Protocol)
Data          Elasticsearch 8.8, Redis, PostgreSQL
Auth          Keycloak 23.0, OAuth2, JWT, RBAC
DevOps        Docker, GitHub Actions, Self-hosted Runner
Document      Apache POI (Excel), OpenHTMLtoPDF, iText7
```

---

## AI Consultation Service (LangGraph)

### Overview

A LangGraph-based real estate consultation chatbot that responds to natural language queries based on land/building data for user-selected map regions.

### Key Features

- **Context-based Consultation**: Maintains selected polygon region data in session for multi-turn conversation support
- **Tool-based Data Retrieval**: LLM determines required information and calls Spring APIs to interpret results
- **Real-time Streaming**: SSE (Server-Sent Events) based response streaming for fast user experience
- **Follow-up Suggestions**: Automatically recommends relevant questions based on conversation context

### Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        LangGraph Agent                          │
│  ┌───────────────┐  ┌───────────────┐  ┌───────────────────┐   │
│  │  Agent Node   │──│  Tool Node    │──│  Checkpointer     │   │
│  │  (GPT-4o)     │  │  (API Calls)  │  │  (Redis/Memory)   │   │
│  └───────────────┘  └───────────────┘  └───────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
         │                    │
         ▼                    ▼
┌─────────────────┐  ┌─────────────────────────────────────────┐
│  FastAPI Server │  │            Spring Boot API              │
│  (SSE Streaming)│  │  Land Summary, Building Info, Aging,    │
│                 │  │  Feasibility Analysis                   │
└─────────────────┘  └─────────────────────────────────────────┘
```

### Available Tools

| Tool | Description |
|------|-------------|
| `get_land_summary` | Retrieve land summary for selected area |
| `get_land_summary_analysis` | Retrieve feasibility analysis results |
| `get_building_info` | Retrieve building detail information |
| `check_aging_building` | Check aging building status |
| `get_jibun_report` | Retrieve lot-by-lot detailed report |
| `convert_units` | Convert area/price units |

### MCP (Model Context Protocol) Integration

Provides an MCP server enabling direct land data queries from Claude Desktop and other MCP clients. Shares the same API client code to ensure consistent data access.

---

## System Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                        Client Applications                          │
│            Web App (Next.js)  │  Claude Desktop (MCP)               │
└─────────────────────────────────────────────────────────────────────┘
                                   │
                                   ▼
┌─────────────────────────────────────────────────────────────────────┐
│                       Application Layer                             │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────────┐ │
│  │  Spring Boot    │  │  LangGraph      │  │  MCP Server         │ │
│  │  REST API       │◄─│  AI Chatbot     │  │  Claude Integration │ │
│  └─────────────────┘  └─────────────────┘  └─────────────────────┘ │
└─────────────────────────────────────────────────────────────────────┘
                                   │
                                   ▼
┌─────────────────────────────────────────────────────────────────────┐
│                        Common Modules                               │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐  │
│  │csv-reader│ │    es    │ │generator │ │  model   │ │  redis   │  │
│  └──────────┘ └──────────┘ └──────────┘ └──────────┘ └──────────┘  │
└─────────────────────────────────────────────────────────────────────┘
                                   │
                                   ▼
┌─────────────────────────────────────────────────────────────────────┐
│                       Infrastructure Layer                          │
│  ┌───────────────────────────┐  ┌───────────────────────────────┐  │
│  │       ELK Stack           │  │       Auth Stack              │  │
│  │  ES │ Logstash │ Kibana   │  │  Keycloak │ PostgreSQL        │  │
│  └───────────────────────────┘  └───────────────────────────────┘  │
│  ┌─────────────────┐  ┌───────────────────────────────────────┐    │
│  │      Redis      │  │       External APIs                   │    │
│  │  (Cache/Session)│  │  OpenAI │ Map API │ Government Data   │    │
│  └─────────────────┘  └───────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Report Generation

Analysis results can be downloaded in various formats.

### Excel Reports

| Sheet | Contents |
|-------|----------|
| Land Summary | Comprehensive info: lot count, area, floor area ratio, aging rate |
| Building Info | Building details (structure, floors, approval date, etc.) |
| Feasibility Analysis | Development revenue, total cost, net profit, contribution ratio simulation |
| Aging Conditions | Aging building status and determination results |

### PDF Reports

- High-quality document generation based on HTML templates
- Includes charts and graphs (pie, bar, line charts)
- QR code insertion for original data access

---

## User Features

### Search History

- Automatic per-user search history storage
- Re-query support for previous search regions
- Search condition restoration (polygon, relation type)

### Favorites

- Save and manage regions of interest
- Quick access to saved regions

### Usage Statistics

- Daily/periodic service usage aggregation
- Statistics viewing from admin dashboard

---

## Project Metrics

| Item | Value |
|------|-------|
| Total Modules | 8 (Multi-module Gradle) |
| API Endpoints | 30+ |
| Processed Data | 10+ types of government public data |
| Docker Services | 7 (API, AI, Auth, ES, Redis, ELK, MCP) |
| Lines of Code | 35,000+ |
| Development/Operation | 1+ years (currently in production) |

---

## Applied Technologies

### Backend

| Technology | Application |
|------------|-------------|
| **Spring Boot 3.3** | REST API server, multi-module architecture |
| **Kotlin** | Null Safety, Data Class, Coroutines |
| **Spring Security** | OAuth2 Resource Server, JWT validation |
| **Spring Data Elasticsearch** | Repository pattern, Bulk Upsert |

### AI / Agent

| Technology | Application |
|------------|-------------|
| **LangGraph** | State-based agent, Tool Calling, Checkpointer |
| **LangChain** | Prompt management, LLM abstraction |
| **OpenAI GPT-4o** | Real estate consultation response generation |
| **MCP** | Claude Desktop integration, external client support |
| **LangSmith** | LLM call tracking, debugging |

### Data

| Technology | Application |
|------------|-------------|
| **Elasticsearch 8.8** | Document storage, full-text search, GeoShape Query |
| **GeoShape Query** | Lot/building boundary search with WKT polygons |
| **ETL Pipeline** | Government CSV/SHP data parsing and bulk indexing |
| **Redis** | API response caching, AI session state management |

### Auth / Security

| Technology | Application |
|------------|-------------|
| **Keycloak** | IdP, OAuth2/OIDC, user management UI |
| **RBAC** | 4-tier roles: guest, user, super_user, admin |
| **JWT** | Token-based auth, shared between Python/Kotlin |

### DevOps

| Technology | Application |
|------------|-------------|
| **Docker Compose** | Multi-container orchestration |
| **GitHub Actions** | CI/CD, tag-based auto deployment |
| **Self-hosted Runner** | Personal server deployment automation |
| **SSL/TLS** | Let's Encrypt certificates, HTTPS |

### Monitoring

| Technology | Application |
|------------|-------------|
| **ELK Stack** | Log collection/storage/visualization |
| **Structured Logging** | Logstash Logback Encoder, JSON logs |
| **Kibana** | Environment-specific index patterns, dashboards |

### Document Generation

| Technology | Application |
|------------|-------------|
| **Apache POI** | Excel report generation |
| **OpenHTMLtoPDF** | HTML → PDF conversion |
| **QuickChart** | Chart image generation API |

---

## Data Analysis Models

### Regional Price Adjustment Model

Developed a model that adjusts the gap between officially assessed prices and actual market prices using Korea Real Estate Board's apartment sale price index.

**Background:**
- Officially assessed prices are calculated annually and fail to reflect rapidly changing market conditions
- Limited ability to capture varying price fluctuation rates by region
- Stark differences exist even within Seoul: Seocho-gu (+17.3%) vs Dobong-gu (-14.0%)

**Core Algorithm:**
```
Adjustment Weight (W) = (1 ÷ Realization Rate) × (Current Sale Index ÷ Assessment Date Sale Index)
Adjusted Price = Officially Assessed Price × W
```

**Model Characteristics:**
- Adjusts only for pure market fluctuations after official assessment
- Prevents weight overestimation due to long-term cumulative effects
- Reflects region-specific market trends

### Outlier Detection and Linear Interpolation

Implemented algorithms to handle missing values and outliers in time-series data (price per pyeong, price history, etc.).

**Outlier Detection (Moving Average-based):**
```
1. Add padding to data edges (boundary handling)
2. Calculate moving average over window size
3. Mark values deviating 80%+ from moving average as outliers
4. Mark outliers as null
```

**Linear Interpolation (Polynomial Spline):**
```
1. Extract valid values as (index, value) pairs
2. Generate polynomial spline interpolation function
3. Fill null values using interpolation function estimates
4. Replace out-of-range values with boundary values
```

**Applications:**
- Fill missing data in yearly price-per-pyeong history
- Clean apartment transaction time-series data
- Generate continuous data for chart visualization

---

## Future Development

- Kubernetes migration for scalability
- RAG (Retrieval-Augmented Generation) integration for enhanced AI analysis
- Real-time data streaming pipeline construction
- Multi-language support (Korean/English)
