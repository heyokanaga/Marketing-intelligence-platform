# BACKEND PHASES
## Marketing Intelligence Platform - Backend Development Strategy

**Last Updated:** 2026-05-06  
**Current Phase:** 1 - Foundation  
**Duration:** 6 months  
**Status:** Initiation

---

## BACKEND PHILOSOPHY

### Core Principles
1. **Backend Intelligence First** - UI is presentation layer, intelligence is the foundation
2. **Modular Services** - Each service is independent, testable, and scalable
3. **Async by Default** - Non-blocking operations wherever possible
4. **Structured Data** - All AI outputs must be typed and validated
5. **Enterprise Ready** - Built for security, compliance, and scale from day one

### Development Approach
- Test-driven development (TDD)
- Continuous integration/deployment
- Incremental feature delivery
- Regular architecture reviews
- Performance profiling

---

## PHASE 1: FOUNDATION (Months 1-6)
### Goal: Build robust, scalable backend infrastructure

### Key Objectives
✓ Establish AI orchestration layer  
✓ Implement async task processing  
✓ Build vector memory system  
✓ Create authentication & multi-tenancy  
✓ Launch market data pipeline  
✓ Establish monitoring and observability  

### Success Criteria
- [ ] 99.5% backend uptime
- [ ] <200ms API response time (p95)
- [ ] 10,000+ requests/minute capacity
- [ ] Zero data isolation breaches
- [ ] All code with >80% test coverage
- [ ] Production-ready deployment pipeline

---

## SPRINT 1.1: AI ORCHESTRATION INFRASTRUCTURE
**Duration:** Weeks 1-2  
**Owner:** Senior Backend Lead  
**Team Size:** 2 engineers

### Objectives
1. Create unified LLM provider abstraction
2. Implement structured output parsing
3. Build prompt versioning system
4. Establish cost tracking
5. Create error handling and retry logic

### Deliverables

#### 1. LLM Provider Abstraction Layer
```python
# /app/ai/llm_provider.py

from enum import Enum
from typing import Optional, Type
import openai
import anthropic
from pydantic import BaseModel

class LLMProvider(Enum):
    OPENAI = "openai"
    CLAUDE = "claude"
    FALLBACK = "fallback"

class LLMConfig:
    def __init__(self, provider: LLMProvider, model: str, temperature: float):
        self.provider = provider
        self.model = model
        self.temperature = temperature

class LLMRequest(BaseModel):
    prompt: str
    output_schema: Optional[Type[BaseModel]] = None
    provider_preference: LLMProvider = LLMProvider.OPENAI
    max_tokens: int = 2000
    temperature: float = 0.7

class LLMOrchestrator:
    def __init__(self):
        self.openai_client = openai.AsyncOpenAI()
        self.claude_client = anthropic.Anthropic()
        self.cost_tracker = CostTracker()
        
    async def generate(self, request: LLMRequest):
        """Main orchestration method"""
        try:
            # Try preferred provider
            response = await self._call_provider(
                request.provider_preference, 
                request
            )
            
            # Parse and validate response
            parsed = self._parse_response(response, request.output_schema)
            
            # Track costs
            self.cost_tracker.log(
                provider=request.provider_preference,
                model=self.model,
                input_tokens=response.usage.prompt_tokens,
                output_tokens=response.usage.completion_tokens
            )
            
            return parsed
            
        except Exception as e:
            # Fallback logic
            return await self._fallback_generation(request, e)
    
    async def _call_provider(self, provider: LLMProvider, request: LLMRequest):
        if provider == LLMProvider.OPENAI:
            return await self._call_openai(request)
        elif provider == LLMProvider.CLAUDE:
            return await self._call_claude(request)
        else:
            raise ValueError(f"Unknown provider: {provider}")
    
    async def _call_openai(self, request: LLMRequest):
        response = await self.openai_client.chat.completions.create(
            model="gpt-4",
            messages=[{"role": "user", "content": request.prompt}],
            temperature=request.temperature,
            max_tokens=request.max_tokens,
            response_format={"type": "json_object"} if request.output_schema else None
        )
        return response
    
    def _parse_response(self, response, schema: Optional[Type[BaseModel]]):
        """Parse and validate response"""
        if schema is None:
            return {"content": response.choices[0].message.content}
        
        try:
            import json
            content = response.choices[0].message.content
            parsed_data = json.loads(content)
            return schema(**parsed_data)
        except json.JSONDecodeError:
            raise ValueError(f"Invalid JSON response: {content}")
        except Exception as e:
            raise ValueError(f"Schema validation failed: {e}")

class CostTracker:
    def __init__(self):
        self.costs = {}
    
    def log(self, provider: str, model: str, input_tokens: int, output_tokens: int):
        key = f"{provider}:{model}"
        cost = self._calculate_cost(provider, model, input_tokens, output_tokens)
        if key not in self.costs:
            self.costs[key] = {"calls": 0, "tokens": 0, "cost_usd": 0}
        self.costs[key]["calls"] += 1
        self.costs[key]["tokens"] += input_tokens + output_tokens
        self.costs[key]["cost_usd"] += cost
    
    def _calculate_cost(self, provider: str, model: str, input_tokens: int, output_tokens: int):
        # Pricing as of 2026 (update as needed)
        pricing = {
            "openai:gpt-4": {"input": 0.00003, "output": 0.00006},
            "openai:gpt-3.5-turbo": {"input": 0.0000005, "output": 0.0000015},
            "claude:opus": {"input": 0.000015, "output": 0.00075},
        }
        if f"{provider}:{model}" not in pricing:
            return 0
        
        rates = pricing[f"{provider}:{model}"]
        return (input_tokens * rates["input"]) + (output_tokens * rates["output"])
```

#### 2. Output Schema Validation
```python
# /app/ai/schemas.py

from pydantic import BaseModel, Field
from typing import List, Dict

class StrategyOutput(BaseModel):
    """Validated strategy generation output"""
    title: str = Field(..., description="Strategy title")
    executive_summary: str = Field(..., max_length=500)
    market_opportunity: float = Field(..., ge=0, le=100, description="Market opportunity score 0-100")
    competitive_positioning: str = Field(...)
    target_segments: List[str] = Field(...)
    key_messages: List[str] = Field(..., min_items=3, max_items=5)
    tactics: List[Dict[str, str]] = Field(...)
    success_metrics: List[Dict[str, str]] = Field(...)
    risks: List[str] = Field(...)
    implementation_timeline_weeks: int = Field(..., ge=1, le=52)

class WorkshopInsight(BaseModel):
    """Validated workshop synthesis"""
    theme: str
    description: str
    related_responses: List[str]
    confidence: float = Field(..., ge=0, le=1)

class BrandAnalysis(BaseModel):
    """Validated brand essence extraction"""
    core_values: List[str] = Field(..., min_items=3, max_items=5)
    brand_personality: Dict[str, str]
    tone_descriptors: List[str]
    key_differentiators: List[str]
    target_audience_description: str
    brand_promise: str
```

#### 3. Structured Prompts with Versioning
```python
# /app/ai/prompts.py

from enum import Enum
from datetime import datetime

class PromptVersion(Enum):
    V1_0 = "1.0"
    V1_1 = "1.1"
    V2_0 = "2.0"

STRATEGY_GENERATION_PROMPT_V2 = """
You are a world-class marketing strategist with 20+ years of experience.

BRAND CONTEXT:
{brand_profile}

MARKET DATA:
{market_data}

COMPETITIVE ANALYSIS:
{competitive_analysis}

Your task is to generate a comprehensive marketing strategy.

Requirements:
1. Strategy must be grounded in the provided data
2. Recommendations should be specific and actionable
3. Include confidence scores for key claims
4. Identify potential risks and mitigation strategies
5. Provide implementation timeline

Response must be valid JSON matching this schema:
{output_schema}
"""

class PromptManager:
    def __init__(self):
        self.prompts = {
            "strategy_generation": {
                "1.0": STRATEGY_GENERATION_PROMPT_V1,
                "2.0": STRATEGY_GENERATION_PROMPT_V2,
            }
        }
        self.default_versions = {
            "strategy_generation": "2.0",
        }
    
    def get_prompt(self, prompt_name: str, version: Optional[str] = None) -> str:
        if version is None:
            version = self.default_versions[prompt_name]
        
        return self.prompts[prompt_name][version]
    
    def log_prompt_usage(self, prompt_name: str, version: str, provider: str, cost: float):
        # Log for A/B testing and cost analysis
        pass
```

#### 4. Error Handling & Retry Logic
```python
# /app/ai/exceptions.py

import asyncio
from typing import Callable, Any

class LLMException(Exception):
    """Base exception for LLM operations"""
    pass

class RateLimitException(LLMException):
    """Rate limit exceeded"""
    pass

class InvalidOutputException(LLMException):
    """LLM output doesn't match schema"""
    pass

async def retry_with_backoff(
    func: Callable,
    max_retries: int = 3,
    initial_delay: int = 1,
    backoff_factor: float = 2.0,
    jitter: bool = True
):
    """Retry function with exponential backoff"""
    import random
    
    delay = initial_delay
    last_exception = None
    
    for attempt in range(max_retries):
        try:
            return await func()
        except RateLimitException as e:
            last_exception = e
            if attempt == max_retries - 1:
                raise
            
            wait_time = delay + (random.random() if jitter else 0)
            print(f"Rate limited. Retrying in {wait_time}s...")
            await asyncio.sleep(wait_time)
            delay *= backoff_factor
        
        except InvalidOutputException as e:
            last_exception = e
            if attempt == max_retries - 1:
                raise
            
            print(f"Invalid output. Retrying attempt {attempt + 1}/{max_retries}...")
            await asyncio.sleep(delay)
            delay *= backoff_factor
    
    raise last_exception
```

### Testing Requirements
```python
# /tests/test_llm_orchestrator.py

import pytest
from unittest.mock import AsyncMock, patch

@pytest.mark.asyncio
async def test_openai_generation_success():
    orchestrator = LLMOrchestrator()
    
    request = LLMRequest(
        prompt="Generate strategy",
        output_schema=StrategyOutput,
        provider_preference=LLMProvider.OPENAI
    )
    
    # Mock the OpenAI response
    mock_response = AsyncMock()
    mock_response.choices[0].message.content = '{"title": "Test", ...}'
    
    with patch.object(orchestrator.openai_client.chat.completions, 'create', return_value=mock_response):
        result = await orchestrator.generate(request)
        assert isinstance(result, StrategyOutput)

@pytest.mark.asyncio
async def test_fallback_to_claude():
    orchestrator = LLMOrchestrator()
    
    # Mock OpenAI failure
    with patch.object(orchestrator.openai_client.chat.completions, 'create', side_effect=Exception("API Error")):
        with patch.object(orchestrator, '_call_claude', return_value=AsyncMock()):
            # Should fallback to Claude
            pass

@pytest.mark.asyncio
async def test_retry_on_rate_limit():
    # Test exponential backoff
    pass
```

### Deliverables Checklist
- [ ] LLMOrchestrator class with provider support
- [ ] Structured output validation with Pydantic
- [ ] Cost tracking system
- [ ] Prompt versioning and A/B testing framework
- [ ] Error handling and retry logic
- [ ] 90%+ test coverage
- [ ] Documentation and usage examples

### Acceptance Criteria
- All LLM calls return validated structured data
- Cost tracking accurate to within 1%
- Retry logic prevents 95% of transient failures
- Response time <2 seconds for typical requests

---

## SPRINT 1.2: ASYNC TASK QUEUE
**Duration:** Weeks 3-4  
**Owner:** Infrastructure Engineer  
**Team Size:** 1-2 engineers

### Objectives
1. Set up Redis cluster
2. Implement Celery task queue
3. Create task monitoring
4. Build dead-letter queue
5. Establish task priorities

### Key Deliverables
- Redis configuration (3-node cluster for HA)
- Celery worker pool
- Task monitoring dashboard (Flower)
- Dead-letter queue handling
- Task retry and scheduling

### Sample Task
```python
# /app/tasks/market_research.py

from celery import current_app as celery_app
from app.services import ResearchService

@celery_app.task(
    bind=True,
    max_retries=3,
    time_limit=3600,  # 1 hour max
    priority=5
)
def analyze_market_async(self, org_id: str, market_id: str):
    """Long-running market analysis task"""
    try:
        service = ResearchService()
        result = service.deep_market_analysis(org_id, market_id)
        return {
            "status": "complete",
            "org_id": org_id,
            "market_id": market_id,
            "result": result
        }
    except Exception as exc:
        # Retry with exponential backoff
        raise self.retry(exc=exc, countdown=60 * (2 ** self.request.retries))
```

---

## SPRINT 1.3: VECTOR MEMORY SYSTEM
**Duration:** Weeks 5-6  
**Owner:** Data Engineering Lead  
**Team Size:** 1-2 engineers

### Objectives
1. PostgreSQL pgvector setup
2. Embedding generation pipeline
3. Vector search implementation
4. Semantic memory retrieval

### Sample Implementation
```python
# /app/db/vector_memory.py

from sqlalchemy import Column, String, Float, DateTime, func, Index
from pgvector.sqlalchemy import Vector
import datetime

class VectorMemory(Base):
    __tablename__ = "vector_memories"
    
    id = Column(UUID, primary_key=True)
    org_id = Column(UUID, ForeignKey("organizations.org_id"), index=True)
    text_content = Column(String(5000))
    embedding = Column(Vector(1536))  # OpenAI embedding size
    metadata = Column(JSON)
    created_at = Column(DateTime, default=datetime.datetime.utcnow, index=True)
    
    __table_args__ = (
        Index('ix_vector_memories_embedding', embedding, postgresql_using='ivfflat'),
    )

class MemoryService:
    async def store_memory(self, org_id: str, content: str, metadata: dict):
        """Store text with embedding"""
        embedding = await self.generate_embedding(content)
        memory = VectorMemory(
            org_id=org_id,
            text_content=content,
            embedding=embedding,
            metadata=metadata
        )
        db.session.add(memory)
        await db.session.commit()
    
    async def semantic_search(self, org_id: str, query: str, top_k: int = 10):
        """Search by semantic similarity"""
        query_embedding = await self.generate_embedding(query)
        results = await db.session.execute(
            select(VectorMemory).where(
                VectorMemory.org_id == org_id
            ).order_by(
                VectorMemory.embedding.cosine_distance(query_embedding)
            ).limit(top_k)
        )
        return results.scalars().all()
```

---

## SPRINT 1.4-1.6: AUTHENTICATION, MULTI-TENANCY, RESEARCH PIPELINE
**Duration:** Weeks 7-12  
**Total Team:** 5 engineers (parallel sprints)

### Sprint 1.4: Authentication & Authorization
```python
# /app/auth/jwt.py

from datetime import datetime, timedelta
import jwt

class TokenManager:
    def create_token(self, user_id: str, org_id: str, expires_delta: timedelta = None):
        if expires_delta is None:
            expires_delta = timedelta(hours=24)
        
        expire = datetime.utcnow() + expires_delta
        payload = {
            "user_id": user_id,
            "org_id": org_id,
            "exp": expire,
            "iat": datetime.utcnow()
        }
        
        token = jwt.encode(payload, os.getenv("JWT_SECRET"), algorithm="HS256")
        return token
    
    def verify_token(self, token: str):
        try:
            payload = jwt.decode(token, os.getenv("JWT_SECRET"), algorithms=["HS256"])
            return payload
        except jwt.ExpiredSignatureError:
            raise UnauthorizedException("Token expired")
        except jwt.InvalidTokenError:
            raise UnauthorizedException("Invalid token")
```

### Sprint 1.5: Multi-Tenancy
```python
# /app/models/organization.py

class Organization(Base):
    __tablename__ = "organizations"
    
    org_id = Column(UUID, primary_key=True, default=uuid4)
    name = Column(String(255), index=True)
    industry = Column(String(100))
    size = Column(String(50))  # startup, scale, enterprise
    data_isolation_key = Column(String(100), unique=True)
    created_at = Column(DateTime, default=datetime.utcnow)

# Tenant context in every request
class TenantMiddleware:
    async def __call__(self, scope, receive, send):
        token = self.extract_token(scope)
        payload = verify_token(token)
        
        # Add org_id to request scope
        scope["org_id"] = payload["org_id"]
        
        await self.app(scope, receive, send)
```

### Sprint 1.6: Market Research Pipeline
```python
# /app/services/research_service.py

class ResearchService:
    async def start_market_monitoring(self, org_id: str, keywords: list[str]):
        """Initiate continuous market monitoring"""
        for keyword in keywords:
            await analyze_market_async.delay(org_id, keyword)
    
    async def get_latest_market_data(self, org_id: str, topic: str):
        """Retrieve latest market analysis"""
        # Query cache first
        cache_key = f"market:{org_id}:{topic}"
        cached = await redis.get(cache_key)
        if cached:
            return cached
        
        # Query database
        data = await db.fetch(
            "SELECT * FROM market_data WHERE org_id=$1 AND topic=$2 ORDER BY created_at DESC LIMIT 1",
            org_id, topic
        )
        return data
```

---

## PHASE 1 SUCCESS METRICS

### Operational Metrics
- [ ] 99.5% backend uptime (monitored continuously)
- [ ] <200ms p95 API response time
- [ ] 10,000+ requests/minute sustained capacity
- [ ] Zero data isolation incidents

### Engineering Metrics
- [ ] >80% code coverage
- [ ] 100% of critical paths tested
- [ ] <1 minute deployment time
- [ ] Zero production incidents from code

### Cost Metrics
- [ ] LLM API costs <$50K/month
- [ ] Infrastructure costs <$20K/month
- [ ] Cost tracking accurate within 1%

### Data Quality
- [ ] All outputs validated against schemas
- [ ] 99.9% output validity
- [ ] Zero unhandled exceptions in production

---

## PHASE 1 COMPLETION CHECKLIST

### Infrastructure Ready
- [ ] PostgreSQL with pgvector operational
- [ ] Redis cluster (3 nodes) in HA
- [ ] Celery worker pool (5+ workers)
- [ ] S3 bucket for assets
- [ ] Monitoring and alerting configured

### Services Operational
- [ ] LLM Orchestrator tested
- [ ] Authentication system live
- [ ] Organization/multi-tenancy working
- [ ] Task queue functioning
- [ ] API documentation complete

### Quality Standards
- [ ] All code reviewed
- [ ] >80% test coverage
- [ ] Performance baselines established
- [ ] Security audit passed
- [ ] Documentation complete

### Deployment Ready
- [ ] CI/CD pipeline automated
- [ ] Staging environment validates releases
- [ ] Rollback procedures tested
- [ ] On-call procedures established
- [ ] Runbooks written

---

## PHASE 2 PREVIEW (Months 7-12)

**Focus:** Advanced AI features and strategic intelligence

### Key Initiatives
1. **Strategy Engine Enhancements** - Multi-step reasoning, confidence scoring
2. **Workshop Optimization** - AI facilitation, real-time synthesis
3. **Advanced Analytics** - Predictive models, ROI forecasting
4. **Competitive Intelligence** - Automated monitoring, alerts

---

## CODING STANDARDS

### Python Style
- Black formatter (line length: 100)
- Flake8 linting
- Type hints on all functions
- Docstrings for all classes/methods

### Testing
- Pytest framework
- >80% coverage requirement
- Unit tests for all business logic
- Integration tests for API endpoints
- Mock external services

### Async/Await
- All I/O operations async
- No blocking calls in endpoints
- Proper asyncio context management

### Error Handling
```python
# Good
try:
    result = await risky_operation()
except SpecificError as e:
    logger.error(f"Operation failed: {e}", extra={"org_id": org_id})
    raise HTTPException(status_code=500, detail="Operation failed")

# Bad
try:
    result = await risky_operation()
except Exception:
    pass  # Silent failure
```

---

## DEPLOYMENT & OPERATIONS

### Deployment Process
1. Create feature branch
2. Push code, triggers CI
3. Tests run automatically
4. Code review required
5. Merge to main (auto-deploys to staging)
6. Manual approval for production
7. Blue-green deployment
8. Health checks validate deployment

### Monitoring Alerts
- API error rate >1%
- Response time p95 >500ms
- Database connection errors
- Memory usage >80%
- Queue depth >10,000

### On-Call Procedures
- 24/7 rotation
- Page on critical alerts
- <5 minute response time SLA
- Runbooks for common issues

---

**Backend Development Lead:** [To Be Assigned]  
**Last Updated:** 2026-05-06  
**Next Review:** 2026-06-06
