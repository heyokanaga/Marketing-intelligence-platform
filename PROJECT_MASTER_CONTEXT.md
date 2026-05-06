# PROJECT MASTER CONTEXT
## Marketing Intelligence Platform - Central Reference

**Last Updated:** 2026-05-06  
**Status:** Core Architecture Definition  
**Audience:** GitHub AI Agents, Engineering Teams, Cursor, Claude Code, OpenAI Codex

---

## 1. PROJECT VISION

Build an **AI-powered marketing intelligence operating system** that transforms how organizations approach strategy, branding, and market analysis.

This is **NOT** simply an AI PowerPoint generator.

### The Platform Should:
- Perform strategic branding workshops
- Analyze competitive markets with depth
- Generate data-driven marketing strategies
- Create enterprise-grade presentations
- Maintain CI/CD brand consistency across all outputs
- Generate intelligent speaker notes and talking points
- Create comprehensive marketing strategies
- Support multi-client organizations
- Eventually support autonomous marketing workflows

### Frontend Positioning
Frontend remains **secondary priority** until backend intelligence layer is stable and scalable.

---

## 2. CORE PHILOSOPHY

- **Backend-First Architecture** - Intelligence layer is the foundation
- **Modular AI Orchestration** - Each AI system is independent and composable
- **Brand Memory Persistence** - Organizations retain strategic continuity
- **Strategic Intelligence Over Surface-Level Generation** - Depth over speed
- **Marketing & Customer Psychology Centered** - Decisions rooted in behavioral science
- **Enterprise-Ready Structure** - Built for scale, reliability, and governance
- **Scalable API-First Design** - Every feature exposed as a service

---

## 3. CURRENT TECH STACK

### Backend
- **FastAPI** - Async-first web framework
- **SQLAlchemy** - PostgreSQL-ready ORM models
- **Pydantic** - Type-safe data validation and schemas
- **Docker** - Containerization and deployment

### Frontend
- **React** - Component-based UI
- **Vite** - Fast build tooling

### Planned Infrastructure
- **Redis** - Caching and session management
- **Celery** - Async task queue
- **pgvector** - Vector database support for embeddings
- **OpenAI API** - LLM orchestration
- **Claude API** - Alternative LLM provider
- **Pinecone** - Vector search and semantic memory
- **Async Agents** - Autonomous workflow orchestration

---

## 4. CORE BACKEND MODULES

### 1. Brand Intelligence Engine
Analyzes and maintains organizational brand identity, values, tone, and messaging guidelines for consistency across all outputs.

**Responsibilities:**
- Brand profile management
- Tone and voice analysis
- Brand asset storage
- Messaging consistency enforcement
- Brand evolution tracking

### 2. Workshop Intelligence
Facilitates strategic branding workshops with AI-guided questioning, analysis, and synthesis.

**Responsibilities:**
- Workshop flow orchestration
- Intelligent questioning engine
- Real-time analysis and synthesis
- Participant feedback aggregation
- Workshop insights generation

### 3. Strategy Generation Engine
Generates data-driven marketing strategies based on brand analysis, market data, and competitive intelligence.

**Responsibilities:**
- Competitive analysis
- Market positioning strategies
- Go-to-market planning
- Campaign strategy generation
- Tactical recommendation synthesis

### 4. Presentation Engine
Converts strategic outputs into polished, branded presentations with consistent visual design.

**Responsibilities:**
- Slide deck generation
- Brand-compliant templating
- Visual asset integration
- Export to multiple formats (PowerPoint, PDF)
- Presentation optimization

### 5. CI/CD Brand Enforcement
Ensures all AI-generated outputs maintain brand consistency through automated validation and correction.

**Responsibilities:**
- Brand compliance checking
- Output validation pipelines
- Automatic correction workflows
- Consistency scoring
- Quality assurance gates

### 6. Asset Recommendation Engine
Suggests appropriate visual assets, templates, and designs based on brand guidelines and content.

**Responsibilities:**
- Asset library management
- Contextual recommendation
- Design template matching
- Brand compliance scoring
- Asset optimization

### 7. Speaker Notes Intelligence
Generates detailed, persuasive speaker notes from strategic insights and presentation content.

**Responsibilities:**
- Speaker note generation
- Talking point synthesis
- Audience engagement scripting
- Narrative flow optimization
- Key message emphasis

### 8. Research & Intelligence Scraping
Gathers market data, competitive intelligence, and industry trends for strategy generation.

**Responsibilities:**
- Market data aggregation
- Competitor monitoring
- Trend analysis
- Industry research synthesis
- Real-time data integration

---

## 5. CURRENT FOLDER STRUCTURE

```
/app
  /api
    - Main FastAPI application setup
  /models
    - SQLAlchemy ORM models (Brand, Workshop, Strategy, etc.)
  /schemas
    - Pydantic request/response schemas
  /services
    - Business logic layer (Brand, Workshop, Strategy services)
  /ai
    - AI orchestration and LLM integration
  /core
    - Core utilities (config, logging, exceptions)
  /db
    - Database setup and migrations
  /utils
    - Helper functions and utilities
/frontend
  /src
    - React components and pages
  /public
    - Static assets
/docker
  - Docker configurations
/docs
  - Documentation and guides
```

---

## 6. IMMEDIATE PRIORITIES

**These are ranked by impact and dependency:**

1. **Improve AI Orchestration Layer** - Robust, composable LLM integration
2. **Add Async Job Queue** - Non-blocking task processing with Celery + Redis
3. **Add OpenAI Integration** - Primary LLM provider integration
4. **Add Vector Memory System** - Semantic storage and retrieval with pgvector
5. **Add Research Scraping Engine** - Market data and competitive intelligence
6. **Improve PowerPoint Export Engine** - Professional presentation generation
7. **Add Authentication & Authorization** - User management and security
8. **Add Organization/Team Support** - Multi-tenant architecture
9. **Add Audit Logging** - Track all AI decisions and outputs
10. **Add Monitoring & Observability** - Production readiness

---

## 7. CODING RULES & STANDARDS

### Architecture
- Keep backend modular and independent
- Avoid moving frontend-heavy logic to backend
- Separate services cleanly with clear interfaces
- Build for horizontal scalability

### AI & Data
- All AI outputs must support structured JSON
- Use Pydantic for all request/response validation
- Store all decisions with timestamps and reasoning
- Maintain audit trail for compliance

### Code Quality
- Avoid monolithic files (max ~200 lines per file)
- Use type hints throughout
- Keep functions focused and testable
- Document complex business logic

### Deployment & Scalability
- Keep export systems independent from core logic
- Design for containerization (Docker)
- Use environment variables for configuration
- Plan for async scaling from day one

### Testing
- Unit tests for all services
- Integration tests for API endpoints
- Mock external API calls
- Maintain >80% code coverage

---

## 8. LONG-TERM VISION (12-24 Months)

The system should evolve into an autonomous AI-powered marketing ecosystem:

### Phase 1: Foundation (Current)
- Robust backend intelligence layer
- Professional presentation generation
- Multi-client support

### Phase 2: Autonomy (6-12 months)
- AI-driven strategy refinement
- Automated competitive monitoring
- Self-improving recommendation engines
- Autonomous campaign optimization

### Phase 3: Full Orchestration (12-24 months)
- End-to-end autonomous marketing workflows
- Real-time market adaptation
- AI-driven budget allocation
- Predictive customer behavior modeling
- Enterprise governance and compliance automation

---

## 9. DEVELOPMENT CONSTRAINTS

### Known Limitations
- Frontend is placeholder pending backend stability
- No multi-tenancy yet (planned)
- No user authentication yet (critical path)
- Vector search not yet implemented

### Technical Debt to Address
- Consolidate AI orchestration patterns
- Improve error handling consistency
- Add comprehensive logging
- Implement rate limiting

### Security Considerations
- All API endpoints require authentication
- Sensitive data encrypted at rest
- PII handling compliance
- API key rotation procedures

---

## 10. SUCCESS METRICS

Track these KPIs as the project evolves:

- **Backend API Response Time** - <200ms for standard requests
- **Strategy Generation Quality** - User satisfaction >4.5/5
- **Brand Consistency Score** - >95% compliance on all outputs
- **System Uptime** - >99.5% availability
- **LLM Accuracy** - Domain-specific benchmarks >90%
- **User Retention** - >80% month-over-month
- **Scalability** - Handle 10x traffic without code changes

---

## 11. HOW TO USE THIS DOCUMENT

### For GitHub AI Agents
Before implementing any feature:
1. Review the relevant module section
2. Check immediate priorities for context
3. Follow coding rules and standards
4. Suggest improvements before major architectural changes

### For Engineering Teams
- Treat this as the source of truth for architecture
- Reference specific modules when building features
- Keep long-term vision in mind for design decisions
- Propose updates to this document for major changes

### For Cursor & Claude Code
- Study the entire architecture before making suggestions
- Focus backend improvements first
- Propose changes before implementing
- Maintain modularity and separation of concerns

---

## 12. NEXT STEPS

1. Implement async job queue infrastructure
2. Build robust AI orchestration layer
3. Add comprehensive error handling
4. Set up monitoring and logging
5. Plan authentication system
6. Design multi-client data isolation

---

**Master Context Owner:** Backend Architecture Team  
**Last Review:** 2026-05-06  
**Next Review:** 2026-06-06
