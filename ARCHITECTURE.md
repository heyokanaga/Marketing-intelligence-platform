# ARCHITECTURE
## Marketing Intelligence Platform - System Design & Technical Architecture

**Last Updated:** 2026-05-06  
**Architecture Version:** 1.0  
**Status:** Foundation Phase

---

## ARCHITECTURAL OVERVIEW

The Marketing Intelligence Platform is built on a **backend-first, modular architecture** designed for scalability, maintainability, and autonomous AI orchestration.

```
┌─────────────────────────────────────────────────────────────────┐
│                     CLIENT LAYER (Secondary)                     │
│              React Frontend | Vite Build | Web UI                │
└──────────────────────────┬──────────────────────────────────────┘
                           │ HTTPS/WebSocket
┌──────────────────────────▼──────────────────────────────────────┐
│                    API GATEWAY LAYER                             │
│         FastAPI | Rate Limiting | Auth | Request Routing        │
└──────────────────────────┬──────────────────────────────────────┘
                           │
        ┌──────────────────┼──────────────────┐
        │                  │                  │
┌───────▼─────────┐ ┌──────▼──────┐ ┌───────▼──────────┐
│  REST Endpoints │ │ WebSocket   │ │  Async Tasks    │
│  /api/v1/*      │ │  Connections│ │  /tasks/*       │
└───────┬─────────┘ └──────┬──────┘ └───────┬──────────┘
        │                  │                │
┌───────▴──────────────────▴────────────────▴──────────────────────┐
│              SERVICE LAYER (Business Logic)                       │
│                                                                    │
│  ├─ Brand Intelligence Service                                   │
│  ├─ Workshop Orchestration Service                               │
│  ├─ Strategy Generation Service                                  │
│  ├─ Presentation Engine Service                                  │
│  ├─ Research & Market Data Service                               │
│  ├─ Analytics & Prediction Service                               │
│  └─ AI Orchestration Service                                     │
└───────┬─────────────────────────────────────────────────────────┘
        │
┌───────▴─────────────────────────────────────────────────────────┐
│              AI ORCHESTRATION LAYER                               │
│                                                                    │
│  ├─ LLM Provider Abstraction (OpenAI, Claude, Fallback)         │
│  ├─ Prompt Management & Versioning                               │
│  ├─ Structured Output Parsing                                    │
│  ├─ Error Handling & Retry Logic                                 │
│  └─ Cost Tracking & Usage Monitoring                             │
└───────┬─────────────────────────────────────────────────────────┘
        │
┌───────▴─────────────────────────────────────────────────────────┐
│            DATA ACCESS & PERSISTENCE LAYER                        │
│                                                                    │
│  ├─ PostgreSQL (Primary Data Store)                              │
│  ├─ pgvector (Vector Memory & Semantic Search)                   │
│  ├─ Redis (Cache, Sessions, Queue State)                         │
│  ├─ S3/Object Storage (Assets, Exports)                          │
│  └─ Celery Task Queue (Async Processing)                         │
└─────────────────────────────────────────────────────────────────┘
```

---

## CORE COMPONENTS

### 1. API Gateway & Request Routing

**Purpose:** Single entry point for all client requests  
**Technology:** FastAPI  
**Responsibility:** Authentication, rate limiting, request routing, response formatting

```python
# /app/api/main.py

from fastapi import FastAPI, Depends, HTTPException
from fastapi.middleware.cors import CORSMiddleware
from app.core.auth import verify_token
from app.api.routes import brands, workshops, strategies, presentations

app = FastAPI(
    title="Marketing Intelligence Platform API",
    version="1.0.0",
    docs_url="/api/docs"
)

# CORS configuration
app.add_middleware(
    CORSMiddleware,
    allow_origins=["https://app.markerintel.com"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Route registration
app.include_router(brands.router, prefix="/api/v1/brands", tags=["Brand"])
app.include_router(workshops.router, prefix="/api/v1/workshops", tags=["Workshop"])
app.include_router(strategies.router, prefix="/api/v1/strategies", tags=["Strategy"])
app.include_router(presentations.router, prefix="/api/v1/presentations", tags=["Presentation"])
```

### 2. Authentication & Authorization

**Purpose:** Secure API access, enforce multi-tenancy  
**Technology:** JWT tokens, role-based access control (RBAC)  
**Implementation:**

```python
# /app/core/auth.py

from fastapi import Depends, HTTPException, status
from app.models import User, Organization
import jwt

async def verify_token(token: str) -> dict:
    try:
        payload = jwt.decode(token, os.getenv("JWT_SECRET"), algorithms=["HS256"])
        org_id: str = payload.get("org_id")
        if org_id is None:
            raise HTTPException(status_code=401, detail="Invalid token")
        return payload
    except jwt.ExpiredSignatureError:
        raise HTTPException(status_code=401, detail="Token expired")
    except jwt.InvalidTokenError:
        raise HTTPException(status_code=401, detail="Invalid token")

async def get_current_org(token: str = Depends(verify_token)) -> Organization:
    org_id = token.get("org_id")
    org = await Organization.get(org_id)
    if org is None:
        raise HTTPException(status_code=404, detail="Organization not found")
    return org

async def check_permission(
    required_role: str,
    org: Organization = Depends(get_current_org)
):
    # RBAC check
    pass
```

### 3. Brand Intelligence Service

**Purpose:** Manage and analyze brand profiles  
**Functionality:**
- Store brand guidelines, values, tone
- Analyze brand consistency
- Enforce brand compliance on all outputs

```python
# /app/services/brand_service.py

from app.models import Brand, Organization
from app.schemas import BrandProfileSchema

class BrandService:
    async def create_brand_profile(self, org_id: str, profile: BrandProfileSchema):
        """Create organization brand profile"""
        brand = Brand(
            org_id=org_id,
            name=profile.name,
            core_values=profile.core_values,
            tone_voice=profile.tone_voice,
            key_differentiators=profile.key_differentiators,
            target_audience=profile.target_audience,
            brand_promise=profile.brand_promise
        )
        await brand.save()
        return brand
    
    async def get_brand_profile(self, org_id: str) -> Brand:
        """Retrieve brand profile"""
        brand = await Brand.filter(org_id=org_id).first()
        return brand
    
    async def check_brand_compliance(self, org_id: str, content: str) -> dict:
        """Analyze content for brand compliance"""
        brand = await self.get_brand_profile(org_id)
        
        # Use LLM to analyze compliance
        from app.ai.llm_orchestrator import LLMOrchestrator
        llm = LLMOrchestrator()
        
        compliance_check = await llm.generate(
            prompt=f"""
            Analyze this content for brand compliance against these guidelines:
            {brand.to_dict()}
            
            Content: {content}
            
            Return compliance score 0-100 and specific issues.
            """
        )
        
        return compliance_check
```

### 4. Workshop Orchestration Service

**Purpose:** Facilitate AI-guided strategic workshops  
**Workflow:**
1. Initialize workshop with context
2. Generate intelligent questions
3. Record participant responses
4. Synthesize insights in real-time
5. Generate actionable recommendations

```python
# /app/services/workshop_service.py

from app.models import Workshop, WorkshopResponse
from app.ai.llm_orchestrator import LLMOrchestrator
from app.schemas import WorkshopInsight

class WorkshopService:
    async def start_workshop(self, org_id: str, workshop_type: str):
        """Initialize new workshop"""
        workshop = Workshop(
            org_id=org_id,
            type=workshop_type,
            status="in_progress"
        )
        await workshop.save()
        
        # Generate opening questions
        questions = await self.generate_questions(org_id, workshop_type, round=1)
        
        return {
            "workshop_id": workshop.id,
            "questions": questions
        }
    
    async def generate_questions(self, org_id: str, workshop_type: str, round: int, previous_responses: list = None):
        """Generate contextual workshop questions"""
        llm = LLMOrchestrator()
        
        context = await self._build_workshop_context(org_id)
        
        prompt = f"""
        You are facilitating a {workshop_type} workshop.
        
        Context:
        {context}
        
        Previous responses:
        {previous_responses}
        
        Generate 5 insightful questions for round {round}.
        Questions should build on previous responses and dig deeper.
        """
        
        questions = await llm.generate(prompt=prompt)
        return questions
    
    async def process_workshop_response(self, workshop_id: str, responses: dict):
        """Process participant responses and synthesize insights"""
        workshop = await Workshop.get(workshop_id)
        
        # Store responses
        for question_id, response in responses.items():
            wr = WorkshopResponse(
                workshop_id=workshop_id,
                question_id=question_id,
                response=response
            )
            await wr.save()
        
        # AI synthesis
        insights = await self._synthesize_insights(workshop)
        return insights
    
    async def _synthesize_insights(self, workshop: Workshop) -> List[WorkshopInsight]:
        """Use LLM to synthesize workshop insights"""
        llm = LLMOrchestrator()
        
        all_responses = await WorkshopResponse.filter(workshop_id=workshop.id).all()
        
        prompt = f"""
        Synthesize these workshop responses into key insights:
        
        {all_responses}
        
        Identify themes, opportunities, and strategic recommendations.
        """
        
        insights = await llm.generate(prompt=prompt, output_schema=List[WorkshopInsight])
        
        return insights
```

### 5. Strategy Generation Service

**Purpose:** Generate data-driven marketing strategies  
**Inputs:**
- Brand profile
- Market data
- Competitive intelligence
- Workshop insights

```python
# /app/services/strategy_service.py

from app.schemas import StrategyOutput

class StrategyService:
    async def generate_strategy(self, org_id: str, market_id: str) -> StrategyOutput:
        """Generate comprehensive marketing strategy"""
        
        # Gather context
        brand = await BrandService().get_brand_profile(org_id)
        market_data = await MarketDataService().get_latest_data(org_id, market_id)
        competitive = await CompetitiveAnalysisService().analyze_competitors(org_id, market_id)
        
        # LLM generation
        llm = LLMOrchestrator()
        strategy = await llm.generate(
            prompt=f"""
            Generate a comprehensive marketing strategy with these inputs:
            
            Brand Profile:
            {brand}
            
            Market Data:
            {market_data}
            
            Competitive Analysis:
            {competitive}
            
            Create a detailed, actionable marketing strategy.
            """,
            output_schema=StrategyOutput
        )
        
        # Store in database
        db_strategy = Strategy(
            org_id=org_id,
            market_id=market_id,
            content=strategy.dict()
        )
        await db_strategy.save()
        
        return strategy
    
    async def get_strategy(self, org_id: str, strategy_id: str) -> StrategyOutput:
        """Retrieve cached strategy"""
        strategy = await Strategy.filter(
            org_id=org_id,
            id=strategy_id
        ).first()
        return strategy
```

### 6. Presentation Engine Service

**Purpose:** Convert strategies into professional presentations  
**Capabilities:**
- Brand-compliant slide generation
- Dynamic content insertion
- PowerPoint/PDF export
- Presentation analytics

```python
# /app/services/presentation_service.py

from pptx import Presentation
from app.models import Strategy

class PresentationService:
    async def generate_presentation(self, org_id: str, strategy_id: str) -> str:
        """Generate PowerPoint presentation from strategy"""
        
        strategy = await Strategy.get(strategy_id)
        brand = await BrandService().get_brand_profile(org_id)
        
        # Create presentation
        prs = Presentation()
        prs.slide_width = Inches(10)
        prs.slide_height = Inches(7.5)
        
        # Title slide
        title_slide_layout = prs.slide_layouts[6]  # Blank layout
        slide = prs.slides.add_slide(title_slide_layout)
        self._add_brand_header(slide, brand)
        self._add_title_text(slide, strategy.title)
        
        # Executive summary
        self._add_content_slide(prs, "Executive Summary", strategy.executive_summary, brand)
        
        # Market opportunity
        self._add_content_slide(prs, "Market Opportunity", strategy.market_opportunity_text, brand)
        
        # Competitive positioning
        self._add_content_slide(prs, "Competitive Positioning", strategy.competitive_positioning, brand)
        
        # Key messages
        self._add_bullets_slide(prs, "Key Messages", strategy.key_messages, brand)
        
        # Tactics
        self._add_tactics_slide(prs, "Strategic Tactics", strategy.tactics, brand)
        
        # Success metrics
        self._add_metrics_slide(prs, "Success Metrics", strategy.success_metrics, brand)
        
        # Save and upload
        filename = f"strategy_{strategy_id}.pptx"
        filepath = f"/tmp/{filename}"
        prs.save(filepath)
        
        # Upload to S3
        s3_url = await self._upload_to_s3(filepath, org_id, filename)
        
        return s3_url
    
    def _add_brand_header(self, slide, brand):
        """Add brand-compliant header"""
        # Apply brand colors, logo, fonts
        pass
    
    def _add_content_slide(self, prs, title, content, brand):
        """Add content slide with brand styling"""
        pass
```

### 7. Research & Market Data Service

**Purpose:** Aggregate and analyze market data  
**Sources:**
- Public market data APIs
- Industry reports
- Competitor monitoring
- Trend analysis

```python
# /app/services/research_service.py

from app.tasks import market_research_tasks

class ResearchService:
    async def start_market_monitoring(self, org_id: str, topics: List[str]):
        """Initiate continuous market monitoring"""
        for topic in topics:
            # Queue async task
            await market_research_tasks.analyze_topic.delay(org_id, topic)
    
    async def get_market_summary(self, org_id: str, topic: str) -> dict:
        """Get latest market summary for topic"""
        # Query vector memory for relevant insights
        memory_service = VectorMemoryService()
        relevant_docs = await memory_service.semantic_search(
            org_id=org_id,
            query=f"market analysis for {topic}",
            top_k=10
        )
        
        return {
            "topic": topic,
            "summary": relevant_docs,
            "last_updated": datetime.utcnow()
        }
    
    async def analyze_competitor(self, org_id: str, competitor_name: str) -> dict:
        """Deep analysis of competitor"""
        # Use research scraping to gather data
        # Analyze with LLM
        pass
```

### 8. Analytics & Prediction Service

**Purpose:** Provide strategic insights and predictions  
**Capabilities:**
- Campaign ROI prediction
- Customer behavior modeling
- Budget optimization
- Performance forecasting

---

## DATA MODELS

### Core Entities

```python
# /app/models/core.py

from sqlalchemy import Column, String, UUID, DateTime, JSON, ForeignKey
from sqlalchemy.orm import relationship
import datetime

class Organization(Base):
    __tablename__ = "organizations"
    
    org_id = Column(UUID, primary_key=True, default=uuid4)
    name = Column(String(255), index=True)
    industry = Column(String(100))
    size = Column(String(50))
    created_at = Column(DateTime, default=datetime.datetime.utcnow)
    
    # Relationships
    brands = relationship("Brand", back_populates="organization")
    strategies = relationship("Strategy", back_populates="organization")
    workshops = relationship("Workshop", back_populates="organization")

class Brand(Base):
    __tablename__ = "brands"
    
    brand_id = Column(UUID, primary_key=True, default=uuid4)
    org_id = Column(UUID, ForeignKey("organizations.org_id"), index=True)
    name = Column(String(255))
    core_values = Column(JSON)
    tone_voice = Column(JSON)
    key_differentiators = Column(JSON)
    target_audience = Column(JSON)
    brand_promise = Column(String(500))
    created_at = Column(DateTime, default=datetime.datetime.utcnow)

class Strategy(Base):
    __tablename__ = "strategies"
    
    strategy_id = Column(UUID, primary_key=True, default=uuid4)
    org_id = Column(UUID, ForeignKey("organizations.org_id"), index=True)
    market_id = Column(String(100))
    title = Column(String(255))
    executive_summary = Column(String(2000))
    market_opportunity = Column(Float)
    competitive_positioning = Column(String(2000))
    key_messages = Column(JSON)
    tactics = Column(JSON)
    success_metrics = Column(JSON)
    created_at = Column(DateTime, default=datetime.datetime.utcnow)
    status = Column(String(50), default="draft")

class Workshop(Base):
    __tablename__ = "workshops"
    
    workshop_id = Column(UUID, primary_key=True, default=uuid4)
    org_id = Column(UUID, ForeignKey("organizations.org_id"), index=True)
    type = Column(String(100))
    status = Column(String(50), default="in_progress")
    insights = Column(JSON)
    created_at = Column(DateTime, default=datetime.datetime.utcnow)
    completed_at = Column(DateTime)
```

---

## DATA FLOW PATTERNS

### Strategy Generation Flow
```
1. Client requests strategy generation
   ↓
2. API validates request and authenticates
   ↓
3. Strategy service gathers context
   - Brand profile from database
   - Market data from cache/async tasks
   - Competitive intelligence from vector memory
   ↓
4. LLM orchestration generates strategy
   - Structured output validated
   - Cost tracked
   ↓
5. Generated strategy stored in database
   ↓
6. Response returned to client
   ↓
7. Async job: Index strategy in vector memory for future retrieval
```

### Workshop Flow
```
1. Workshop initiated with context
   ↓
2. LLM generates intelligent opening questions
   ↓
3. Questions sent to participants
   ↓
4. Responses collected and stored
   ↓
5. LLM synthesizes insights in real-time
   ↓
6. Next round of questions generated based on insights
   ↓
7. Process repeats for N rounds
   ↓
8. Final insights synthesized and stored
   ↓
9. Actionable recommendations generated
```

---

## SCALABILITY & PERFORMANCE

### Caching Strategy
- **Redis** for session cache, LLM response cache
- **CDN** for static assets
- **Database query cache** for frequently accessed data

### Database Optimization
- Indexes on `org_id` for all tenant-aware queries
- Partial indexes on status fields
- Vector indexes on pgvector embeddings (IVFFLAT)

### Async Processing
- Long-running tasks queued with Celery
- Background workers process in parallel
- WebSocket connections for real-time updates

### Rate Limiting
```python
from fastapi_limiter import FastAPILimiter

# API endpoints limited to 100 requests/minute per org
# LLM API calls limited by rate limiting service
```

---

## SECURITY ARCHITECTURE

### Data Isolation
- All queries filtered by `org_id`
- Database-level row security not yet implemented but planned
- Separate S3 bucket policies per organization

### Encryption
- HTTPS for all connections
- At-rest encryption for sensitive data in transit
- Database encryption planned

### API Security
- Rate limiting on all endpoints
- API key rotation every 90 days
- OAuth2 integration planned

---

## MONITORING & OBSERVABILITY

### Metrics Collected
- API response times
- LLM API costs and usage
- Task queue depth
- Database query performance
- Cache hit rates

### Logging Strategy
```python
import logging

logger = logging.getLogger(__name__)

# Structured logging with context
logger.info(
    "Strategy generated",
    extra={
        "org_id": org_id,
        "strategy_id": strategy_id,
        "llm_cost": cost,
        "response_time_ms": elapsed
    }
)
```

### Alerting Thresholds
- API error rate >1%
- Response time p95 >500ms
- LLM API cost >$500/day
- Queue depth >10,000 tasks

---

## DEPLOYMENT ARCHITECTURE

### Environment Configuration
```yaml
environments:
  development:
    database: PostgreSQL (local)
    cache: Redis (local)
    llm_provider: openai-dev
    
  staging:
    database: PostgreSQL (AWS RDS)
    cache: Redis (AWS ElastiCache)
    llm_provider: openai
    
  production:
    database: PostgreSQL (AWS RDS, Multi-AZ)
    cache: Redis (AWS ElastiCache, 3 nodes)
    llm_provider: openai + claude fallback
```

### Container Orchestration
- Docker for containerization
- Kubernetes for orchestration (planned)
- Auto-scaling based on queue depth

---

## EXTENSIBILITY & FUTURE

### Plugin System (Planned)
- Third-party integrations via webhooks
- Custom LLM provider support
- Custom service extensions

### API Versioning
- `/api/v1/*` - Current stable API
- `/api/v2/*` - In development

---

**Architecture Owner:** Platform Engineering Team  
**Last Updated:** 2026-05-06  
**Next Review:** 2026-08-06
