# HIRESIGHT - Implementation Overview

## Project Vision
HIRESIGHT is an AI-powered recruitment platform that automates candidate evaluation through resume parsing, GitHub analysis, LinkedIn data extraction, and intelligent scoring algorithms.

## Architecture Overview

### Tech Stack

#### Frontend
- **Framework**: React 18.x with TypeScript
- **Build Tool**: Vite
- **Routing**: React Router v6
- **State Management**: Zustand + React Query
- **UI Framework**: Tailwind CSS + Shadcn/ui
- **Forms**: React Hook Form + Zod validation
- **Charts**: Recharts + Chart.js
- **Tables**: TanStack Table (React Table v8)
- **File Upload**: React Dropzone
- **PDF Viewer**: react-pdf
- **Authentication**: React Context + JWT tokens

#### Backend
- **Framework**: FastAPI (Python 3.11+)
- **API Documentation**: FastAPI built-in (Swagger/OpenAPI)
- **Authentication**: FastAPI-Users + JWT
- **Database ORM**: SQLAlchemy 2.0 (async)
- **Migrations**: Alembic
- **Background Tasks**: Celery + Redis
- **Validation**: Pydantic v2
- **Testing**: Pytest + pytest-asyncio

#### Database & Storage
- **Primary Database**: PostgreSQL 15+
- **Cache/Queue**: Redis 7+
- **File Storage**: AWS S3 / MinIO (S3-compatible)
- **Full-Text Search**: PostgreSQL FTS + pg_trgm extension

#### External Services
- **Authentication**: OAuth (Google, GitHub)
- **Email**: SendGrid / Resend
- **Payments**: Stripe
- **Monitoring**: Sentry (errors) + Posthog (analytics)
- **File Processing**: python-docx, PyPDF2, pytesseract (OCR)

#### Infrastructure
- **Containerization**: Docker + Docker Compose
- **Frontend Hosting**: Vercel / Netlify
- **Backend Hosting**: Railway / Render / AWS ECS
- **CI/CD**: GitHub Actions
- **Reverse Proxy**: Nginx
- **SSL**: Let's Encrypt (Certbot)

---

## System Architecture

### High-Level Architecture

```text
┌─────────────────────────────────────────────────────────────┐
│                         FRONTEND                            │
│  React + TypeScript + Tailwind + Vite                       │
│  - Candidate Dashboard                                      │
│  - Developer Portal                                         │
│  - Analytics Views                                          │
│  - Settings & Admin                                         │
└─────────────────┬───────────────────────────────────────────┘
                  │ HTTPS/REST API
                  │
┌─────────────────▼───────────────────────────────────────────┐
│                    API GATEWAY / NGINX                       │
│  - Rate Limiting                                            │
│  - CORS                                                     │
│  - SSL Termination                                          │
└─────────────────┬───────────────────────────────────────────┘
                  │
┌─────────────────▼───────────────────────────────────────────┐
│                    FASTAPI BACKEND                           │
│  ┌──────────────────────────────────────────────────────┐  │
│  │  API Routes Layer                                     │  │
│  │  /api/v1/auth /candidates /resumes /analytics        │  │
│  └────────────────────┬─────────────────────────────────┘  │
│  ┌────────────────────▼─────────────────────────────────┐  │
│  │  Business Logic Layer                                │  │
│  │  - Resume Parser                                     │  │
│  │  - GitHub Analyzer                                   │  │
│  │  - AI Scoring Engine                                 │  │
│  │  - Authentication Service                            │  │
│  └────────────────────┬─────────────────────────────────┘  │
│  ┌────────────────────▼─────────────────────────────────┐  │
│  │  Data Access Layer (SQLAlchemy)                      │  │
│  └──────────────────────────────────────────────────────┘  │
└─────────────────┬───────────────────────────────────────────┘
                  │
      ┌───────────┼───────────┬──────────────┐
      │           │           │              │
┌─────▼─────┐ ┌──▼────┐ ┌────▼──────┐ ┌────▼────────┐
│PostgreSQL │ │ Redis │ │   AWS S3   │ │   Celery    │
│           │ │       │ │  (Files)   │ │  Workers    │
│- Users    │ │-Cache │ │            │ │             │
│-Candidates│ │-Queue │ │- Resumes   │ │- Async Jobs │
│- Scores   │ │-Session│ │- Exports  │ │- Email Send │
│- Analytics│ │       │ │            │ │- Data Proc  │
└───────────┘ └───────┘ └────────────┘ └─────────────┘
```

---

## Project Structure

### Backend Structure
```text
backend/
├── alembic/                 # Database migrations
│   └── versions/
├── app/
│   ├── __init__.py
│   ├── main.py             # FastAPI app entry point
│   ├── core/               # Core configuration
│   │   ├── config.py       # Settings management
│   │   ├── security.py     # JWT, password hashing
│   │   └── database.py     # DB connection
│   ├── models/             # SQLAlchemy models
│   │   ├── user.py
│   │   ├── candidate.py
│   │   ├── resume.py
│   │   └── score.py
│   ├── schemas/            # Pydantic schemas
│   │   ├── user.py
│   │   ├── candidate.py
│   │   └── api_responses.py
│   ├── api/                # API routes
│   │   └── v1/
│   │       ├── auth.py
│   │       ├── candidates.py
│   │       ├── resumes.py
│   │       ├── analytics.py
│   │       └── github.py
│   ├── services/           # Business logic
│   │   ├── resume_parser.py
│   │   ├── github_analyzer.py
│   │   ├── scoring_engine.py
│   │   ├── linkedin_service.py
│   │   └── email_service.py
│   ├── workers/            # Celery tasks
│   │   ├── resume_tasks.py
│   │   ├── github_tasks.py
│   │   └── email_tasks.py
│   └── utils/              # Utilities
│       ├── parsers.py
│       ├── validators.py
│       └── extractors.py
├── tests/
├── requirements.txt
└── Dockerfile
```

### Frontend Structure
```text
frontend/
├── public/
├── src/
│   ├── main.tsx            # App entry point
│   ├── App.tsx             # Root component
│   ├── routes/             # Route components
│   │   ├── DashboardLayout.tsx
│   │   ├── CandidatesPage.tsx
│   │   ├── CandidateDetailPage.tsx
│   │   ├── AnalyticsPage.tsx
│   │   └── SettingsPage.tsx
│   ├── components/         # Reusable components
│   │   ├── ui/             # Base UI components
│   │   ├── candidates/     # Candidate-specific
│   │   ├── analytics/      # Charts & metrics
│   │   └── forms/          # Form components
│   ├── features/           # Feature modules
│   │   ├── auth/
│   │   ├── candidates/
│   │   ├── analytics/
│   │   └── settings/
│   ├── services/           # API clients
│   │   ├── api.ts          # Axios instance
│   │   ├── candidateService.ts
│   │   └── authService.ts
│   ├── stores/             # Zustand stores
│   │   ├── authStore.ts
│   │   └── uiStore.ts
│   ├── hooks/              # Custom hooks
│   │   ├── useAuth.ts
│   │   └── useCandidates.ts
│   ├── types/              # TypeScript types
│   │   └── index.ts
│   └── utils/              # Helper functions
├── package.json
├── vite.config.ts
├── tailwind.config.js
└── Dockerfile
```

---

## Development Phases

### Phase 1: Foundation (Weeks 1-2)
- Set up development environment
- Initialize frontend and backend projects
- Configure database and migrations
- Implement authentication system
- Create basic UI layout

### Phase 2: Core Platform (Weeks 3-5)
- Resume parsing (PDF/DOCX)
- Candidate database CRUD
- File upload system
- Basic candidate dashboard
- Search and filtering

### Phase 3: GitHub Integration (Week 6)
- GitHub OAuth integration
- Repository analysis
- Commit statistics
- Language breakdown
- Activity scoring

### Phase 4: AI Scoring (Week 7)
- Keyword extraction
- Experience calculator
- Skills matching
- Score aggregation
- Ranking system

### Phase 5: Analytics (Week 8)
- Team metrics
- Hiring funnel
- Dashboard charts
- Export functionality

### Phase 6: Advanced Features (Weeks 9-10)
- LinkedIn Chrome extension
- Email notifications
- Webhooks
- Slack/Discord integration
- Background jobs

### Phase 7: Security & Compliance (Week 11)
- Security hardening
- Audit logging
- Data export (GDPR)
- Rate limiting
- Input validation

### Phase 8: Deployment & Polish (Week 12)
- Production deployment
- Performance optimization
- Documentation
- Testing coverage
- Monitoring setup

---

## Key Technical Decisions

### Why FastAPI?
- **Performance**: ASGI server with async/await support
- **Developer Experience**: Automatic OpenAPI docs, type hints
- **Modern Python**: Pydantic validation, dependency injection
- **Ecosystem**: Excellent integration with SQLAlchemy, Celery

### Why React + Vite?
- **Fast Development**: Hot module replacement, instant updates
- **Modern Tooling**: ESBuild for speed, TypeScript support
- **Ecosystem**: Vast component library, mature tooling
- **Performance**: Code splitting, tree shaking built-in

### Why PostgreSQL?
- **Reliability**: ACID compliance, mature ecosystem
- **Features**: Full-text search, JSON support, extensions
- **Scalability**: Handles millions of records efficiently
- **Cost**: Open source, no licensing fees

### Why Redis?
- **Speed**: In-memory operations for caching
- **Versatility**: Cache, session store, message broker
- **Celery Integration**: Native support for task queues

---

## API Design Principles

1. **RESTful**: Follow REST conventions (GET, POST, PUT, DELETE)
2. **Versioned**: All routes under `/api/v1/`
3. **Consistent**: Standard response format
4. **Documented**: Auto-generated OpenAPI specs
5. **Secure**: JWT authentication, rate limiting
6. **Paginated**: All list endpoints support pagination
7. **Filtered**: Support for query parameters

### Standard Response Format
```json
{
  "success": true,
  "data": { ... },
  "message": "Operation successful",
  "metadata": {
    "page": 1,
    "per_page": 20,
    "total": 150
  }
}
```

### Error Response Format
```json
{
  "success": false,
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Invalid input data",
    "details": [
      {
        "field": "email",
        "message": "Invalid email format"
      }
    ]
  }
}
```

---

## Security Considerations

1. **Authentication**: JWT tokens with refresh mechanism
2. **Authorization**: Role-based access control (RBAC)
3. **Input Validation**: Pydantic schemas, Zod on frontend
4. **SQL Injection**: SQLAlchemy ORM (parameterized queries)
5. **XSS Protection**: React auto-escaping, Content Security Policy
6. **CSRF**: SameSite cookies, CSRF tokens
7. **Rate Limiting**: Per-user and per-IP limits
8. **File Upload**: Type validation, size limits, virus scanning
9. **Secrets Management**: Environment variables, never in code
10. **HTTPS Only**: Enforce SSL in production

---

## Testing Strategy

### Backend Testing
- **Unit Tests**: Business logic, utilities (pytest)
- **Integration Tests**: API endpoints with test DB
- **Load Tests**: Performance testing (Locust)
- **Coverage Target**: 80%+ code coverage

### Frontend Testing
- **Unit Tests**: Component logic (Vitest)
- **Integration Tests**: User flows (React Testing Library)
- **E2E Tests**: Critical paths (Playwright)
- **Visual Tests**: Component snapshots

---

## Monitoring & Observability

1. **Error Tracking**: Sentry for backend and frontend
2. **Analytics**: Posthog for user behavior
3. **Logging**: Structured JSON logs (Loguru)
4. **Metrics**: Response times, error rates
5. **Uptime**: Health check endpoints
6. **Alerting**: Email/Slack for critical issues

---

## Documentation Structure

This documentation is organized as follows:

- `00-OVERVIEW.md` - This file (architecture overview)
- `01-DATABASE-SCHEMA.md` - Complete database design
- `02-API-SPECIFICATION.md` - All API endpoints
- `architecture/` - System design documents
- `implementation/` - Feature implementation guides
- `frontend/` - React component specifications
- `deployment/` - Infrastructure and deployment guides

---

## Getting Started

After reviewing this overview, proceed to:

1. **Database Schema** (`01-DATABASE-SCHEMA.md`) - Understand data models
2. **API Specification** (`02-API-SPECIFICATION.md`) - Learn the API contracts
3. **Implementation Guides** (`implementation/`) - Feature-by-feature build instructions
4. **Frontend Architecture** (`frontend/`) - UI component structure
5. **Deployment Guide** (`deployment/`) - Production setup

---

## Development Environment Setup

### Prerequisites
- Node.js 18+ and npm/yarn
- Python 3.11+
- PostgreSQL 15+
- Redis 7+
- Docker and Docker Compose (optional but recommended)

### Quick Start
```bash
# Clone repository
git clone <repo-url>
cd hiresight

# Backend setup
cd backend
python -m venv venv
source venv/bin/activate  # Windows: venv\Scripts\activate
pip install -r requirements.txt
alembic upgrade head
uvicorn app.main:app --reload

# Frontend setup (new terminal)
cd frontend
npm install
npm run dev
```

### With Docker
```bash
docker-compose up -d
# Access:
# Frontend: http://localhost:5173
# Backend: http://localhost:8000
# API Docs: http://localhost:8000/docs
```

---

## Next Steps

Ready to implement? Start with Phase 1 and follow the implementation guides in order. Each guide provides:
- Step-by-step instructions
- Code examples
- API contracts
- Testing strategies
- Common pitfalls to avoid

Good luck building HIRESIGHT!
