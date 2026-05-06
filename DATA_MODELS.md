# DATA MODELS
## Marketing Intelligence Platform - Entity Definitions & Relationships

**Last Updated:** 2026-05-06  
**Status:** Schema Version 1.0  
**Critical For:** System scalability and data integrity

---

## PURPOSE

This document defines the canonical data model for all entities, relationships, and constraints. All services reference this contract.

---

## CORE ENTITIES

### 1. ORGANIZATION

**Owner:** Org that provisioned the account  
**Lifecycle:** Active → Archived  
**Indexes:** `org_id` (primary), `created_at`

```sql
CREATE TABLE organizations (
    org_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL,
    industry VARCHAR(100),
    company_size VARCHAR(50),  -- startup, scale, enterprise
    plan VARCHAR(50),          -- free, starter, pro, enterprise
    status VARCHAR(50) DEFAULT 'active',  -- active, archived, suspended
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW(),
    
    UNIQUE(name),
    INDEX(created_at),
    INDEX(status)
);
```

**Data Isolation:** All queries filtered by `org_id`

---

### 2. USER

**Owner:** Organization  
**Lifecycle:** Active → Archived  
**Indexes:** `org_id`, `email`

```sql
CREATE TABLE users (
    user_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id UUID NOT NULL REFERENCES organizations(org_id),
    email VARCHAR(255) NOT NULL,
    full_name VARCHAR(255),
    role VARCHAR(50) DEFAULT 'member',  -- admin, member, viewer
    status VARCHAR(50) DEFAULT 'active',
    created_at TIMESTAMP DEFAULT NOW(),
    
    UNIQUE(org_id, email),
    INDEX(org_id),
    INDEX(email)
);
```

---

### 3. BRAND

**Owner:** Organization  
**Lifecycle:** Draft → Active → Archived  
**Relationship:** 1 Organization → Many Brands

```sql
CREATE TABLE brands (
    brand_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id UUID NOT NULL REFERENCES organizations(org_id),
    name VARCHAR(255) NOT NULL,
    slug VARCHAR(100),
    
    -- Brand essence
    core_values JSONB,  -- ["value1", "value2", ...]
    tone_voice JSONB,   -- {"primary": "professional", "secondary": "friendly"}
    target_audience JSONB,  -- {"description": "...", "personas": [...]}
    key_differentiators JSONB,
    brand_promise VARCHAR(500),
    
    -- Assets
    logo_url VARCHAR(500),
    color_palette JSONB,  -- {"primary": "#000000", ...}
    font_family VARCHAR(100),
    
    status VARCHAR(50) DEFAULT 'draft',  -- draft, active, archived
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW(),
    
    UNIQUE(org_id, slug),
    INDEX(org_id),
    INDEX(status)
);
```

**Data Contract:**
- `core_values`: Array of 3-5 strings
- `tone_voice`: JSON with descriptors
- `target_audience`: Rich structure with personas

---

### 4. STRATEGY

**Owner:** Organization (via Brand)  
**Lifecycle:** Draft → In Review → Approved → Active → Archived  
**Relationships:**
- 1 Brand → Many Strategies
- 1 Strategy → Many Versions

```sql
CREATE TABLE strategies (
    strategy_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id UUID NOT NULL REFERENCES organizations(org_id),
    brand_id UUID NOT NULL REFERENCES brands(brand_id),
    
    title VARCHAR(255) NOT NULL,
    market_id VARCHAR(100),
    
    -- Strategy content
    executive_summary TEXT,
    market_opportunity_score FLOAT CHECK (market_opportunity_score BETWEEN 0 AND 100),
    competitive_positioning TEXT,
    target_segments JSONB,  -- ["segment1", "segment2", ...]
    key_messages JSONB,     -- ["message1", "message2", ...]
    
    -- Tactics (JSON for flexibility)
    tactics JSONB,  -- Array of tactic objects
    success_metrics JSONB,
    risks JSONB,
    
    -- Confidence & tracking
    confidence_score FLOAT CHECK (confidence_score BETWEEN 0 AND 1),
    generated_by VARCHAR(50),  -- "gpt-4", "claude-3", etc
    
    -- Lifecycle
    status VARCHAR(50) DEFAULT 'draft',  -- draft, in_review, approved, active, archived
    approved_by UUID REFERENCES users(user_id),
    approved_at TIMESTAMP,
    
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW(),
    version INT DEFAULT 1,
    
    INDEX(org_id),
    INDEX(brand_id),
    INDEX(status),
    INDEX(created_at),
    INDEX(market_id)
);
```

**Tactic Structure (within tactics JSONB):**
```json
[
  {
    "name": "Sales enablement",
    "description": "...",
    "timeline_weeks": 8,
    "resource_level": "high",
    "estimated_cost_dollars": 45000,
    "confidence_score": 0.92,
    "expected_outcome": "..."
  }
]
```

---

### 5. WORKSHOP

**Owner:** Organization  
**Lifecycle:** Planned → In Progress → Complete → Archived  
**Relationships:**
- 1 Organization → Many Workshops
- 1 Workshop → Many Responses
- 1 Workshop → Many Insights

```sql
CREATE TABLE workshops (
    workshop_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id UUID NOT NULL REFERENCES organizations(org_id),
    brand_id UUID REFERENCES brands(brand_id),
    
    title VARCHAR(255),
    type VARCHAR(100),  -- "brand_essence", "market_strategy", etc
    
    -- Workshop flow
    round INT DEFAULT 1,
    total_rounds INT DEFAULT 3,
    
    -- Participants
    participants JSONB,  -- [{"user_id": "...", "email": "..."}, ...]
    facilitator_notes TEXT,
    
    -- Results
    insights JSONB,  -- Synthesized insights
    recommendations JSONB,
    
    status VARCHAR(50) DEFAULT 'planned',  -- planned, in_progress, complete
    created_at TIMESTAMP DEFAULT NOW(),
    started_at TIMESTAMP,
    completed_at TIMESTAMP,
    
    INDEX(org_id),
    INDEX(status),
    INDEX(created_at)
);
```

---

### 6. WORKSHOP_RESPONSE

**Owner:** Workshop  
**Lifecycle:** Created → Used for synthesis  
**Relationship:** 1 Workshop → Many Responses

```sql
CREATE TABLE workshop_responses (
    response_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    workshop_id UUID NOT NULL REFERENCES workshops(workshop_id),
    user_id UUID NOT NULL REFERENCES users(user_id),
    
    round INT,
    question_text TEXT,
    response_text TEXT,
    sentiment VARCHAR(50),  -- positive, neutral, negative
    confidence INT,  -- 1-5 scale
    
    created_at TIMESTAMP DEFAULT NOW(),
    
    INDEX(workshop_id),
    INDEX(user_id),
    INDEX(created_at)
);
```

---

### 7. PRESENTATION

**Owner:** Strategy  
**Lifecycle:** Draft → Generated → Ready → Shared → Archived  
**Relationship:** 1 Strategy → Many Presentations

```sql
CREATE TABLE presentations (
    presentation_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id UUID NOT NULL REFERENCES organizations(org_id),
    strategy_id UUID NOT NULL REFERENCES strategies(strategy_id),
    
    title VARCHAR(255),
    
    -- Export info
    format VARCHAR(50),  -- pptx, pdf, html
    file_path VARCHAR(500),
    file_size_bytes INT,
    download_url VARCHAR(500),
    
    -- Generation metadata
    generated_by VARCHAR(50),  -- "presentation_engine_v1"
    generation_time_seconds FLOAT,
    
    -- Usage
    view_count INT DEFAULT 0,
    download_count INT DEFAULT 0,
    last_viewed_at TIMESTAMP,
    
    status VARCHAR(50) DEFAULT 'draft',
    created_at TIMESTAMP DEFAULT NOW(),
    
    INDEX(org_id),
    INDEX(strategy_id),
    INDEX(created_at)
);
```

---

### 8. MARKET_DATA

**Owner:** Organization  
**Lifecycle:** Created → Indexed → Archived  
**Relationship:** 1 Organization → Many Market Data Points

```sql
CREATE TABLE market_data (
    data_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id UUID NOT NULL REFERENCES organizations(org_id),
    
    market_id VARCHAR(100),
    topic VARCHAR(255),
    source_name VARCHAR(255),
    source_url VARCHAR(500),
    
    -- Content
    content TEXT,
    summary TEXT,
    
    -- Metadata
    data_type VARCHAR(50),  -- report, news, trend, competitor_intel
    relevance_score FLOAT CHECK (relevance_score BETWEEN 0 AND 1),
    
    -- Embedding for semantic search
    content_embedding vector(1536),  -- OpenAI embedding
    
    created_at TIMESTAMP DEFAULT NOW(),
    expires_at TIMESTAMP,  -- When data becomes stale
    
    INDEX(org_id),
    INDEX(market_id),
    INDEX(data_type),
    INDEX(created_at),
    INDEX USING ivfflat (content_embedding)
);
```

---

### 9. AI_JOB

**Owner:** Organization  
**Lifecycle:** Queued → Running → Complete/Failed → Archived  
**Relationship:** 1 Organization → Many Jobs

```sql
CREATE TABLE ai_jobs (
    job_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id UUID NOT NULL REFERENCES organizations(org_id),
    
    job_type VARCHAR(100),  -- "strategy_generation", "market_analysis", etc
    status VARCHAR(50) DEFAULT 'queued',  -- queued, running, complete, failed
    
    -- Input
    input_params JSONB,
    
    -- Progress
    progress_percent INT DEFAULT 0,
    estimated_completion_seconds INT,
    
    -- Output
    result JSONB,
    error_message TEXT,
    
    -- Tracking
    created_at TIMESTAMP DEFAULT NOW(),
    started_at TIMESTAMP,
    completed_at TIMESTAMP,
    execution_time_seconds FLOAT,
    
    -- Resources
    tokens_used INT,
    cost_dollars FLOAT,
    
    INDEX(org_id),
    INDEX(job_type),
    INDEX(status),
    INDEX(created_at)
);
```

---

### 10. VECTOR_MEMORY

**Owner:** Organization  
**Lifecycle:** Created → Indexed → Archived  
**Relationship:** 1 Organization → Many Memories

```sql
CREATE TABLE vector_memories (
    memory_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id UUID NOT NULL REFERENCES organizations(org_id),
    
    content_type VARCHAR(50),  -- "strategy", "insight", "market_data"
    reference_id UUID,  -- Links back to source (strategy_id, etc)
    
    text_content TEXT,
    embedding vector(1536),  -- OpenAI embedding
    
    metadata JSONB,  -- Any additional context
    
    created_at TIMESTAMP DEFAULT NOW(),
    
    INDEX(org_id),
    INDEX(content_type),
    INDEX USING ivfflat (embedding)
);
```

---

## RELATIONSHIPS DIAGRAM

```
Organization (1) ──┬─→ (Many) Brand
                   ├─→ (Many) User
                   ├─→ (Many) Strategy ──→ (Many) Presentation
                   ├─→ (Many) Workshop ──→ (Many) WorkshopResponse
                   ├─→ (Many) MarketData
                   ├─→ (Many) AIJob
                   └─→ (Many) VectorMemory

Brand (1) ──→ (Many) Strategy
Workshop (1) ──→ (Many) WorkshopResponse
```

---

## INDEXES & QUERY PATTERNS

### Critical Queries

```sql
-- Get all strategies for org (with pagination)
SELECT * FROM strategies 
WHERE org_id = $1 AND status != 'archived'
ORDER BY created_at DESC
LIMIT $2 OFFSET $3;
-- Index: (org_id, status, created_at)

-- Get related market data for strategy
SELECT * FROM market_data 
WHERE org_id = $1 AND market_id = $2
ORDER BY relevance_score DESC;
-- Index: (org_id, market_id, relevance_score)

-- Semantic search for market data
SELECT * FROM market_data 
WHERE org_id = $1
ORDER BY content_embedding <-> $2
LIMIT $3;
-- Index: USING ivfflat (content_embedding)
```

---

## CACHING STRATEGY

### Cache Keys

```
cache:org:{org_id}:brand:{brand_id}
cache:org:{org_id}:strategies:{status}
cache:market_data:{org_id}:{market_id}
cache:org:{org_id}:user_count
```

### TTL Strategy

| Data Type | TTL | Reasoning |
|-----------|-----|-----------|
| Brand profile | 1 hour | Rarely changes |
| Strategies (list) | 30 min | Moderate changes |
| Market data | 24 hours | External source |
| Vector search results | 5 min | Very dynamic |
| User sessions | 24 hours | Standard auth |

---

## LIFECYCLE STATES

### Strategy Lifecycle

```
Draft
  ↓
In Review (optional human review)
  ↓
Approved
  ↓
Active (in use)
  ↓
Archived (historical)
```

### Workshop Lifecycle

```
Planned
  ↓
In Progress
  ↓
Complete
  ↓
Archived
```

---

## CONSTRAINTS & VALIDATION

### Data Quality Rules

```python
class StrategyValidator:
    def validate(self, strategy: dict) -> bool:
        # Business logic constraints
        assert 0 <= strategy["market_opportunity_score"] <= 100
        assert 0 <= strategy["confidence_score"] <= 1
        assert len(strategy["key_messages"]) >= 3
        assert len(strategy["tactics"]) >= 1
        
        # Referential integrity
        assert await Brand.exists(strategy["brand_id"])
        assert strategy["org_id"] == brand.org_id
        
        return True
```

---

## FUTURE EXTENSIBILITY

### Ready for:

✅ Multi-tenancy (org_id isolation)  
✅ Vector embeddings (pgvector support)  
✅ Job queues (ai_jobs tracking)  
✅ Analytics (all created_at timestamps)  
✅ Versioning (version column in strategies)  
✅ Archiving (status = 'archived')  
✅ Soft deletes (status column approach)  
✅ Webhooks (job status tracking)  

---

**Data Model Owner:** Data Engineering  
**Last Updated:** 2026-05-06  
**Next Review:** 2026-09-06

