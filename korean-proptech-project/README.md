# Rebuild Analytics

> **Urban Redevelopment Feasibility Analysis Platform** - Supporting investment decisions for urban renewal projects through map-based polygon selection

---

## Project Overview

### Overview

Rebuild Analytics is a real estate data analysis service that allows users to intuitively define redevelopment/reconstruction target areas or areas of interest through map-based polygon selection, and instantly view feasibility analysis results for those areas.

Users simply click on the map to define boundaries, and land registers and feasibility analysis results for that area are automatically generated. The platform provides comprehensive insights based on aging conditions and real estate indices (transaction volume, average age, average sale price) applicable to each region, allowing users to run various profit/loss simulations by adjusting different parameters.

### Core Value Propositions

- **Map-based Polygon Search**: Draw polygons on the map to instantly define redevelopment/reconstruction target areas
- **Real-time Feasibility Analysis**: Automatically generate comprehensive feasibility analysis results based on land registers, aging conditions, and real estate indices
- **Profit/Loss Simulation**: Combine various conditions to instantly view profit/loss simulations
- **Analysis Report Download**: Instantly generate Excel-format analysis reports for easy sharing with third parties
- **AI-powered Data Interpretation**: LLM automatically explains visualization graphs in natural language, making it easy for non-experts to understand
- **Integrated Urban Renewal Zone Data**: Reflects Seoul's agenda-processed urban renewal zones, redevelopment promotion districts, and urban development zone lists

### Product/Service Competitiveness

Rebuild Analytics is a data analysis tool specialized for urban renewal projects. Users can freely designate areas of interest in polygon form on the map and instantly view feasibility analysis results for those zones. The platform provides the underlying data (land information, aging status, actual transaction prices, sale prices, etc.) used in the analysis, and results can be downloaded as Excel files, enabling users to simulate various conditions independently or easily share with external experts.

**Future Value-Oriented Decision Support**

Rebuild Analytics differentiates itself by supporting decision-making centered on future value rather than current market price-based evaluation. As urban aging is expected to accelerate, simply relying on current transaction prices or official land prices is insufficient for assessing regional value. More precise valuation becomes possible only when considering future business potential, including renewal project feasibility, post-redevelopment asset value, and proportional ratios.

Reflecting this perspective, Rebuild Analytics provides not only current baseline data but also future development potential and profit/loss simulations based on various scenarios, helping users make forward-looking decisions.

**Target Users**

Rebuild Analytics is a feasibility review tool for professionals, corporate investors, and individual investors alike, aiming to improve decision-making accuracy and efficiency in the information-asymmetric urban renewal sector.

### Ongoing Experiments

- **LLM-based Automatic Graph Explanation**: Experimenting with a feature that automatically explains visualization graphs from analysis results using LLM. Providing natural language interpretation for metrics and data that users find difficult to understand
- **MCP (Model Context Protocol) Integration**: Experimenting with connecting self-developed APIs via MCP for Claude client integration. Exploring the potential for users to more easily understand analysis results for specific project sites and expand into chatbot investment consulting

---

## Tech Stack

```
Backend       Kotlin, Java 17, Spring Boot 3.3.2, FastAPI
Data          Elasticsearch 8.8, Redis, PostgreSQL
AI/ML         LangChain, OpenAI GPT-4, MCP (Model Context Protocol)
Auth          Keycloak 23.0, OAuth2, JWT, RBAC
DevOps        Docker, GitHub Actions, Self-hosted Runner
Document      Apache POI (Excel), OpenHTMLtoPDF, iText7
```

---

## Key Challenges and Solutions

### 1. Processing Unstructured/Large-scale Government Public Data

**Problem:**
- Government-provided CSV files contain hundreds of thousands to millions of records
- Each file has different encoding, column structure, and data format
- Simple parsing causes memory overflow

**Solution:**
- Built custom parser based on OpenCSV with streaming for memory efficiency
- Designed dedicated mapper classes for each data type (D003, D006, D151, etc.)
- Batch indexing with Elasticsearch Bulk API for optimized processing speed
- Added data integrity validation logic to filter missing/erroneous data

### 2. Building Complex Authentication/Authorization System

**Problem:**
- Multiple roles needed: guest, regular user, premium user, admin
- Authentication sharing required between AI service (Python) and main API (Kotlin)
- Same authentication system needed for MCP server

**Solution:**
- Introduced Keycloak as IdP, compliant with OAuth 2.0 + OIDC standards
- Implemented RBAC through Spring Security and Keycloak integration
- Built custom JWT validation middleware for FastAPI
- Minimized repeated authentication with session token caching on MCP server

**RBAC Structure:**
```
guest      → Basic API Access
user       → Standard Features
super_user → Premium (Excel, PDF Export)
admin      → Administrative APIs
```

### 3. Unified Log Management Across Multiple Environments

**Problem:**
- Logs scattered across Dev, Stage, Prod environments
- Required accessing multiple servers to check logs during incidents
- Text logs difficult to search/analyze

**Solution:**
- Self-built ELK Stack (Elasticsearch + Logstash + Kibana)
- JSON structured log transmission with Logstash Logback Encoder
- Environment-specific TCP port separation (dev:4560, prod:4561, stage:4562)
- Environment-specific index patterns enabling unified search in Kibana

### 4. AI Service Integration with Existing System

**Problem:**
- Needed to add GPT-powered land analysis functionality
- Required seamless integration with existing Spring Boot API
- Wanted to enable direct data queries from Claude Desktop

**Solution:**
- Built separate AI service with LangChain + FastAPI
- Exposed Chains as REST API via LangServe
- Implemented MCP (Model Context Protocol) server for Claude Desktop integration
- LangSmith integration for tracking and debugging all LLM calls

### 5. Multi-Environment Deployment Automation

**Problem:**
- HTTPS service needed on personal server, not cloud
- Manual deployment has high error potential and is time-consuming

**Solution:**
- Manual issuance and renewal of free SSL certificates via Let's Encrypt
- Built CI/CD pipeline with GitHub Actions + Self-hosted Runner
- Automatic deployment trigger on Git tag creation
- PKCS12 Keystore conversion automation (using GitHub Secrets)

**Environment Configuration:**

| Env | Port | Protocol | Purpose |
|-----|------|----------|---------|
| Local | 8080 | HTTP | IDE Development |
| Dev | 8080 | HTTP | Docker Testing |
| Stage | 9442 | HTTPS | Pre-production Validation |
| Prod | 9443 | HTTPS | Production (Full Stack + AI) |

---

## Technology Selection Rationale

### Kotlin + Spring Boot
- **Reason**: Extensive experience with Kotlin + Spring combination
- **Result**: Virtually no runtime NullPointerExceptions

### Elasticsearch
- **Reason**: Korean morphological analysis support, location-based search via GeoShape queries
- **Result**: Millisecond-level search responses across millions of records

### Keycloak
- **Reason**: Full OAuth 2.0/OIDC support, built-in RBAC, management UI provided
- **Result**: Implemented complex role-based access control through configuration alone

### LangChain + MCP
- **Reason**: De facto standard for LLM applications, direct Claude integration via MCP
- **Result**: Prompt management, call tracking, diverse client support

---

## System Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                        Client Applications                          │
└─────────────────────────────────────────────────────────────────────┘
                                   │
                                   ▼
┌─────────────────────────────────────────────────────────────────────┐
│                       Application Layer                             │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────────┐ │
│  │  Spring Boot    │  │  FastAPI        │  │  MCP Server         │ │
│  │  REST API       │  │  AI Service     │  │  Claude Integration │ │
│  │  (:9443)        │  │  (:8123)        │  │                     │ │
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
│  │     (Cache)     │  │  OpenAI │ Map API │ Government Data   │    │
│  └─────────────────┘  └───────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Project Structure

```
├── application/          # Spring Boot REST API
├── batch/etl/            # Data Pipeline (Picocli CLI)
├── common/
│   ├── csv-reader/       # Government CSV Parser
│   ├── es/               # Elasticsearch Integration
│   ├── generator/        # Report Engine (PDF, Excel)
│   ├── model/            # Shared DTOs & Entities
│   └── redis/            # Cache Module
├── langchain/            # AI Service (FastAPI + LangChain)
└── mcp-server/           # Model Context Protocol Server
```

---

## Project Achievements

| Metric | Value |
|--------|-------|
| Total Modules | 8 (Multi-module Gradle) |
| API Endpoints | 29 |
| Data Processed | 10+ types of government public data |
| Docker Services | 6 (API, AI, Auth, ES, Redis, ELK) |
| Lines of Code | 30,000+ |
| Development/Operation Period | 1+ year (currently in production) |

---

## Applied Technologies Summary

| Category | Skills |
|----------|--------|
| **Backend** | Spring Boot, Kotlin, RESTful API, Multi-module Architecture |
| **Data** | Elasticsearch, ETL Pipeline, Data Modeling, GeoShape Query |
| **DevOps** | Docker, CI/CD, Multi-environment Management, SSL/TLS |
| **Security** | OAuth2, JWT, RBAC, Keycloak |
| **AI** | LangChain, OpenAI Integration, MCP, LangSmith |
| **Monitoring** | ELK Stack, Structured Logging, Kibana Dashboard |

---

## API Design Overview

### API Layer Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                        Client Layer                                  │
│          Web Client │ Mobile Client │ API Testing Tools             │
└─────────────────────────────────────────────────────────────────────┘
                                   │
                                   ▼
┌─────────────────────────────────────────────────────────────────────┐
│                       Security Layer                                 │
│              CORS Filter │ OAuth2 Resource Server │ JWT              │
└─────────────────────────────────────────────────────────────────────┘
                                   │
                                   ▼
┌─────────────────────────────────────────────────────────────────────┐
│                         API Layer                                    │
│  ┌───────────────┐  ┌───────────────┐  ┌───────────────────────┐   │
│  │  Auth API     │  │  Reports API  │  │  Download API         │   │
│  │  Token Issue  │  │  Land Analysis│  │  Excel/PDF Export     │   │
│  └───────────────┘  └───────────────┘  └───────────────────────┘   │
│  ┌───────────────┐  ┌───────────────┐  ┌───────────────────────┐   │
│  │  User API     │  │  Data API     │  │  Admin API            │   │
│  │  Search Hist. │  │  Urban Zones  │  │  Statistics/Mgmt      │   │
│  └───────────────┘  └───────────────┘  └───────────────────────┘   │
└─────────────────────────────────────────────────────────────────────┘
```

### Main API Functions

| API Group | Key Features | Description |
|-----------|--------------|-------------|
| **Reports API** | Regional land reports, feasibility analysis | Land data queries based on GeoShape queries |
| **Download API** | Excel/PDF download | Export analysis results as reports |
| **User API** | Search history, favorites | User-specific search record management |
| **Data API** | Urban planning zone queries | Seoul urban renewal zone data |
| **Admin API** | Usage statistics, system management | Admin dashboard data |

### Role-Based Access Control (RBAC)

| Role | Basic API | Premium Features | Admin Features |
|------|-----------|------------------|----------------|
| guest | O | X | X |
| user | O | X | X |
| super_user | O | O (Excel/PDF Export) | X |
| admin | X | X | O |

---

## Authentication System Design

### OAuth 2.0-based Authentication Architecture

```
┌─────────────┐     ┌─────────────────┐     ┌─────────────────┐
│   Client    │────▶│  Spring Boot    │────▶│    Keycloak     │
│             │     │  (Resource      │     │  (Identity      │
│             │◀────│   Server)       │◀────│   Provider)     │
└─────────────┘     └─────────────────┘     └─────────────────┘
      │                     │                       │
      │  1. API Request     │  2. JWT Validation    │
      │  + Bearer Token     │                       │
      │                     │  3. Role Extraction   │
      │                     │                       │
      │  4. Response        │                       │
      │◀────────────────────│                       │
```

### Authentication Flow

1. **Token Issuance**: Client requests Password Grant from Keycloak
2. **JWT Reception**: Receives Access Token + Refresh Token
3. **API Request**: API call with Bearer Token
4. **Validation**: Spring Security verifies JWT signature and expiration
5. **Authorization**: Role-based access control from token claims

### Security Features

- **OAuth 2.0 + OIDC** standard compliance
- **JWT-based** stateless authentication
- **Role-Based Access Control** (RBAC)
- **Cross-service authentication sharing** (Spring Boot ↔ FastAPI ↔ MCP Server)

---

## Data Analysis Model

### Regional Price Adjustment Model

Developed a model that adjusts the gap between official assessed prices and actual market prices using the Korea Real Estate Board's apartment transaction price index.

**Background:**
- Official prices are assessed annually, failing to reflect rapidly changing market conditions
- Limited ability to capture different price fluctuation rates by region
- Stark contrasts within Seoul: Seocho-gu (+17.3%) vs Dobong-gu (-14.0%)

**Core Algorithm:**
```
Adjustment Weight (W) = (1 ÷ Realization Rate) × (Current Price Index ÷ Assessment Date Price Index)
Adjusted Price = Official Price × W
```

**Model Characteristics:**
- Adjusts only for pure market changes after official price assessment
- Prevents weight overestimation from long-term cumulative effects
- Reflects region-specific market trends

**Application Areas:**
- Real estate asset portfolio valuation
- Collateral value assessment and LTV calculation
- Investment decision support

---

## Future Development Roadmap

- Kubernetes migration for scalability
- RAG (Retrieval-Augmented Generation) introduction for enhanced AI analysis
- Real-time data streaming pipeline construction
