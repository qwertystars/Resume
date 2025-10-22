# Database Schema Design

## Overview
This document defines the complete PostgreSQL database schema for HIRESIGHT platform. All tables use UUID primary keys for security and distributed system compatibility.

---

## Core Tables

### users
Stores all user accounts (recruiters, admins, developers).

```sql
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email VARCHAR(255) UNIQUE NOT NULL,
    hashed_password VARCHAR(255) NOT NULL,
    full_name VARCHAR(255) NOT NULL,
    role VARCHAR(50) NOT NULL DEFAULT 'recruiter',
    -- Roles: 'admin', 'recruiter', 'viewer', 'developer'

    is_active BOOLEAN DEFAULT true,
    is_verified BOOLEAN DEFAULT false,
    email_verified_at TIMESTAMP,

    avatar_url TEXT,
    timezone VARCHAR(50) DEFAULT 'UTC',

    -- OAuth fields
    google_id VARCHAR(255) UNIQUE,
    github_id VARCHAR(255) UNIQUE,

    -- Security
    totp_secret VARCHAR(255),
    is_2fa_enabled BOOLEAN DEFAULT false,
    failed_login_attempts INTEGER DEFAULT 0,
    locked_until TIMESTAMP,

    -- Metadata
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    last_login_at TIMESTAMP,

    -- Soft delete
    deleted_at TIMESTAMP,

    CONSTRAINT check_role CHECK (role IN ('admin', 'recruiter', 'viewer', 'developer'))
);

CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_role ON users(role);
CREATE INDEX idx_users_google_id ON users(google_id);
CREATE INDEX idx_users_github_id ON users(github_id);
```

### teams
Multi-tenant workspace isolation.

```sql
CREATE TABLE teams (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL,
    slug VARCHAR(100) UNIQUE NOT NULL,

    owner_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,

    plan VARCHAR(50) DEFAULT 'free',
    -- Plans: 'free', 'starter', 'professional', 'enterprise'

    settings JSONB DEFAULT '{}',

    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    CONSTRAINT check_plan CHECK (plan IN ('free', 'starter', 'professional', 'enterprise'))
);

CREATE INDEX idx_teams_slug ON teams(slug);
CREATE INDEX idx_teams_owner_id ON teams(owner_id);
```

### team_members
Many-to-many relationship between users and teams.

```sql
CREATE TABLE team_members (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    team_id UUID NOT NULL REFERENCES teams(id) ON DELETE CASCADE,
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,

    role VARCHAR(50) NOT NULL DEFAULT 'member',
    -- Roles: 'owner', 'admin', 'member', 'viewer'

    invited_by UUID REFERENCES users(id),
    invited_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    joined_at TIMESTAMP,

    UNIQUE(team_id, user_id),
    CONSTRAINT check_team_role CHECK (role IN ('owner', 'admin', 'member', 'viewer'))
);

CREATE INDEX idx_team_members_team_id ON team_members(team_id);
CREATE INDEX idx_team_members_user_id ON team_members(user_id);
```

---

## Candidate Management

### candidates
Core candidate information.

```sql
CREATE TABLE candidates (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    team_id UUID NOT NULL REFERENCES teams(id) ON DELETE CASCADE,

    -- Personal info
    full_name VARCHAR(255) NOT NULL,
    email VARCHAR(255),
    phone VARCHAR(50),
    location VARCHAR(255),

    -- Profile links
    linkedin_url TEXT,
    github_url TEXT,
    portfolio_url TEXT,

    -- Status tracking
    status VARCHAR(50) DEFAULT 'new',
    -- Status: 'new', 'reviewing', 'interviewed', 'offered', 'hired', 'rejected', 'withdrawn'

    stage VARCHAR(50) DEFAULT 'applied',
    -- Stage: 'applied', 'screening', 'phone', 'technical', 'onsite', 'offer', 'hired'

    source VARCHAR(100),
    -- Source: 'job_board', 'referral', 'linkedin', 'direct_apply', 'recruiter'

    -- Calculated fields
    years_of_experience DECIMAL(4,2),
    education_level VARCHAR(50),
    -- Levels: 'high_school', 'associate', 'bachelor', 'master', 'phd'

    primary_skills TEXT[], -- Array of skill keywords

    -- Scoring
    overall_score DECIMAL(5,2) DEFAULT 0,
    resume_score DECIMAL(5,2) DEFAULT 0,
    github_score DECIMAL(5,2) DEFAULT 0,
    linkedin_score DECIMAL(5,2) DEFAULT 0,

    -- Metadata
    applied_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    last_contacted_at TIMESTAMP,

    created_by UUID REFERENCES users(id),
    assigned_to UUID REFERENCES users(id),

    notes TEXT,
    tags TEXT[],

    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    deleted_at TIMESTAMP,

    CONSTRAINT check_status CHECK (status IN ('new', 'reviewing', 'interviewed', 'offered', 'hired', 'rejected', 'withdrawn')),
    CONSTRAINT check_stage CHECK (stage IN ('applied', 'screening', 'phone', 'technical', 'onsite', 'offer', 'hired')),
    CONSTRAINT check_education CHECK (education_level IN ('high_school', 'associate', 'bachelor', 'master', 'phd', NULL))
);

CREATE INDEX idx_candidates_team_id ON candidates(team_id);
CREATE INDEX idx_candidates_email ON candidates(email);
CREATE INDEX idx_candidates_status ON candidates(status);
CREATE INDEX idx_candidates_stage ON candidates(stage);
CREATE INDEX idx_candidates_overall_score ON candidates(overall_score DESC);
CREATE INDEX idx_candidates_created_at ON candidates(created_at DESC);
CREATE INDEX idx_candidates_assigned_to ON candidates(assigned_to);

-- Full-text search index
CREATE INDEX idx_candidates_full_name_fts ON candidates USING GIN(to_tsvector('english', full_name));
CREATE INDEX idx_candidates_tags ON candidates USING GIN(tags);
CREATE INDEX idx_candidates_skills ON candidates USING GIN(primary_skills);
```

### resumes
Uploaded resume files and parsed data.

```sql
CREATE TABLE resumes (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    candidate_id UUID NOT NULL REFERENCES candidates(id) ON DELETE CASCADE,

    -- File info
    file_name VARCHAR(255) NOT NULL,
    file_size INTEGER NOT NULL, -- bytes
    file_type VARCHAR(50) NOT NULL,
    -- Types: 'pdf', 'docx', 'doc', 'txt'

    storage_path TEXT NOT NULL, -- S3 path or local path
    storage_url TEXT, -- Pre-signed URL for download

    -- Processing status
    status VARCHAR(50) DEFAULT 'pending',
    -- Status: 'pending', 'processing', 'completed', 'failed'

    error_message TEXT,

    -- Parsed data
    raw_text TEXT,
    parsed_data JSONB DEFAULT '{}',
    -- Structure:
    -- {
    --   "contact": {...},
    --   "experience": [...],
    --   "education": [...],
    --   "skills": [...],
    --   "certifications": [...]
    -- }

    -- Metadata
    uploaded_by UUID REFERENCES users(id),
    uploaded_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    processed_at TIMESTAMP,

    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    CONSTRAINT check_file_type CHECK (file_type IN ('pdf', 'docx', 'doc', 'txt')),
    CONSTRAINT check_status CHECK (status IN ('pending', 'processing', 'completed', 'failed'))
);

CREATE INDEX idx_resumes_candidate_id ON resumes(candidate_id);
CREATE INDEX idx_resumes_status ON resumes(status);
CREATE INDEX idx_resumes_uploaded_at ON resumes(uploaded_at DESC);

-- Full-text search on resume content
CREATE INDEX idx_resumes_raw_text_fts ON resumes USING GIN(to_tsvector('english', raw_text));
```

### candidate_github_profiles
GitHub profile analysis data.

```sql
CREATE TABLE candidate_github_profiles (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    candidate_id UUID NOT NULL UNIQUE REFERENCES candidates(id) ON DELETE CASCADE,

    -- GitHub info
    github_username VARCHAR(255) NOT NULL,
    github_user_id INTEGER,
    profile_url TEXT,

    -- Profile data
    public_repos INTEGER DEFAULT 0,
    followers INTEGER DEFAULT 0,
    following INTEGER DEFAULT 0,

    total_stars INTEGER DEFAULT 0,
    total_forks INTEGER DEFAULT 0,
    total_commits INTEGER DEFAULT 0,

    -- Language breakdown (JSON)
    languages JSONB DEFAULT '{}',
    -- Structure: {"Python": 45.2, "JavaScript": 32.1, "Go": 22.7}

    -- Activity metrics
    contributions_last_year INTEGER DEFAULT 0,
    longest_streak INTEGER DEFAULT 0,
    current_streak INTEGER DEFAULT 0,

    most_active_repo VARCHAR(255),
    recent_activity JSONB DEFAULT '[]',

    -- Dates
    github_created_at TIMESTAMP,
    last_commit_at TIMESTAMP,

    -- Fetching metadata
    last_fetched_at TIMESTAMP,
    fetch_error TEXT,

    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_github_profiles_candidate_id ON candidate_github_profiles(candidate_id);
CREATE INDEX idx_github_profiles_username ON candidate_github_profiles(github_username);
```

### candidate_linkedin_profiles
LinkedIn profile data (from Chrome extension).

```sql
CREATE TABLE candidate_linkedin_profiles (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    candidate_id UUID NOT NULL UNIQUE REFERENCES candidates(id) ON DELETE CASCADE,

    -- LinkedIn data
    linkedin_url TEXT,
    headline VARCHAR(500),

    current_position VARCHAR(255),
    current_company VARCHAR(255),

    connections INTEGER,
    recommendations INTEGER,

    -- Parsed data
    experience JSONB DEFAULT '[]',
    education JSONB DEFAULT '[]',
    skills JSONB DEFAULT '[]',
    certifications JSONB DEFAULT '[]',

    -- Screenshot/OCR
    screenshot_url TEXT,
    ocr_text TEXT,

    -- Metadata
    captured_at TIMESTAMP,
    captured_by UUID REFERENCES users(id),

    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_linkedin_profiles_candidate_id ON candidate_linkedin_profiles(candidate_id);
```

---

## Scoring System

### score_history
Track score changes over time.

```sql
CREATE TABLE score_history (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    candidate_id UUID NOT NULL REFERENCES candidates(id) ON DELETE CASCADE,

    score_type VARCHAR(50) NOT NULL,
    -- Types: 'overall', 'resume', 'github', 'linkedin', 'skills', 'experience'

    score_value DECIMAL(5,2) NOT NULL,
    previous_value DECIMAL(5,2),

    reason TEXT,
    metadata JSONB DEFAULT '{}',

    calculated_by VARCHAR(100), -- 'system' or user_id
    calculated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_score_history_candidate_id ON score_history(candidate_id);
CREATE INDEX idx_score_history_score_type ON score_history(score_type);
CREATE INDEX idx_score_history_calculated_at ON score_history(calculated_at DESC);
```

### scoring_rules
Configurable scoring rules per team.

```sql
CREATE TABLE scoring_rules (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    team_id UUID NOT NULL REFERENCES teams(id) ON DELETE CASCADE,

    rule_name VARCHAR(255) NOT NULL,
    rule_type VARCHAR(50) NOT NULL,
    -- Types: 'keyword_match', 'experience_years', 'education_level', 'github_activity', 'skills_match'

    conditions JSONB NOT NULL,
    -- Structure: {"keywords": ["python", "fastapi"], "min_years": 3}

    weight DECIMAL(5,2) DEFAULT 1.0,
    points INTEGER NOT NULL,

    is_active BOOLEAN DEFAULT true,

    created_by UUID REFERENCES users(id),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_scoring_rules_team_id ON scoring_rules(team_id);
CREATE INDEX idx_scoring_rules_is_active ON scoring_rules(is_active);
```

---

## Analytics & Tracking

### candidate_activities
Activity log for candidates.

```sql
CREATE TABLE candidate_activities (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    candidate_id UUID NOT NULL REFERENCES candidates(id) ON DELETE CASCADE,
    user_id UUID REFERENCES users(id),

    activity_type VARCHAR(100) NOT NULL,
    -- Types: 'status_change', 'note_added', 'email_sent', 'interview_scheduled', 'score_updated'

    description TEXT,
    metadata JSONB DEFAULT '{}',

    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_candidate_activities_candidate_id ON candidate_activities(candidate_id);
CREATE INDEX idx_candidate_activities_user_id ON candidate_activities(user_id);
CREATE INDEX idx_candidate_activities_type ON candidate_activities(activity_type);
CREATE INDEX idx_candidate_activities_created_at ON candidate_activities(created_at DESC);
```

### hiring_funnel_metrics
Aggregate metrics for hiring funnel analysis.

```sql
CREATE TABLE hiring_funnel_metrics (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    team_id UUID NOT NULL REFERENCES teams(id) ON DELETE CASCADE,

    date DATE NOT NULL,
    stage VARCHAR(50) NOT NULL,

    candidates_entered INTEGER DEFAULT 0,
    candidates_exited INTEGER DEFAULT 0,
    candidates_progressed INTEGER DEFAULT 0,
    candidates_dropped INTEGER DEFAULT 0,

    avg_time_in_stage INTERVAL,
    conversion_rate DECIMAL(5,2),

    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    UNIQUE(team_id, date, stage)
);

CREATE INDEX idx_funnel_metrics_team_date ON hiring_funnel_metrics(team_id, date DESC);
```

---

## Authentication & Security

### sessions
JWT refresh token storage.

```sql
CREATE TABLE sessions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,

    refresh_token VARCHAR(500) UNIQUE NOT NULL,
    access_token_jti VARCHAR(255), -- JWT ID for revocation

    ip_address INET,
    user_agent TEXT,

    expires_at TIMESTAMP NOT NULL,
    revoked_at TIMESTAMP,

    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_sessions_user_id ON sessions(user_id);
CREATE INDEX idx_sessions_refresh_token ON sessions(refresh_token);
CREATE INDEX idx_sessions_expires_at ON sessions(expires_at);
```

### api_keys
API keys for programmatic access.

```sql
CREATE TABLE api_keys (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    team_id UUID NOT NULL REFERENCES teams(id) ON DELETE CASCADE,
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,

    key_name VARCHAR(255) NOT NULL,
    key_prefix VARCHAR(20) NOT NULL, -- First 8 chars for identification
    hashed_key VARCHAR(255) NOT NULL,

    permissions JSONB DEFAULT '{}',
    -- Structure: {"candidates": ["read", "write"], "analytics": ["read"]}

    last_used_at TIMESTAMP,
    usage_count INTEGER DEFAULT 0,

    expires_at TIMESTAMP,
    revoked_at TIMESTAMP,

    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_api_keys_team_id ON api_keys(team_id);
CREATE INDEX idx_api_keys_key_prefix ON api_keys(key_prefix);
```

### audit_logs
Comprehensive audit trail.

```sql
CREATE TABLE audit_logs (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    team_id UUID REFERENCES teams(id) ON DELETE CASCADE,
    user_id UUID REFERENCES users(id) ON DELETE SET NULL,

    action VARCHAR(100) NOT NULL,
    -- Actions: 'user.login', 'candidate.create', 'resume.upload', 'settings.update'

    resource_type VARCHAR(50),
    resource_id UUID,

    ip_address INET,
    user_agent TEXT,

    request_method VARCHAR(10),
    request_path TEXT,

    changes JSONB,
    -- Structure: {"old": {...}, "new": {...}}

    status VARCHAR(20),
    -- Status: 'success', 'failure', 'unauthorized'

    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_audit_logs_team_id ON audit_logs(team_id);
CREATE INDEX idx_audit_logs_user_id ON audit_logs(user_id);
CREATE INDEX idx_audit_logs_action ON audit_logs(action);
CREATE INDEX idx_audit_logs_created_at ON audit_logs(created_at DESC);
CREATE INDEX idx_audit_logs_resource ON audit_logs(resource_type, resource_id);
```

---

## Integrations

### email_templates
Email templates for notifications.

```sql
CREATE TABLE email_templates (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    team_id UUID REFERENCES teams(id) ON DELETE CASCADE,

    template_name VARCHAR(255) NOT NULL,
    template_key VARCHAR(100) NOT NULL,
    -- Keys: 'candidate_welcome', 'interview_invitation', 'rejection'

    subject VARCHAR(500) NOT NULL,
    body_html TEXT NOT NULL,
    body_text TEXT,

    variables JSONB DEFAULT '[]',
    -- Structure: ["candidate_name", "interview_date", "position"]

    is_active BOOLEAN DEFAULT true,

    created_by UUID REFERENCES users(id),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    UNIQUE(team_id, template_key)
);

CREATE INDEX idx_email_templates_team_id ON email_templates(team_id);
```

### email_logs
Outgoing email tracking.

```sql
CREATE TABLE email_logs (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    team_id UUID REFERENCES teams(id) ON DELETE CASCADE,

    recipient_email VARCHAR(255) NOT NULL,
    recipient_name VARCHAR(255),

    subject VARCHAR(500),
    template_key VARCHAR(100),

    status VARCHAR(50) DEFAULT 'pending',
    -- Status: 'pending', 'sent', 'delivered', 'opened', 'clicked', 'bounced', 'failed'

    provider VARCHAR(50), -- 'sendgrid', 'resend'
    provider_message_id VARCHAR(255),

    error_message TEXT,

    sent_at TIMESTAMP,
    delivered_at TIMESTAMP,
    opened_at TIMESTAMP,
    clicked_at TIMESTAMP,

    metadata JSONB DEFAULT '{}',

    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_email_logs_team_id ON email_logs(team_id);
CREATE INDEX idx_email_logs_recipient_email ON email_logs(recipient_email);
CREATE INDEX idx_email_logs_status ON email_logs(status);
CREATE INDEX idx_email_logs_created_at ON email_logs(created_at DESC);
```

### webhooks
Webhook endpoint configurations.

```sql
CREATE TABLE webhooks (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    team_id UUID NOT NULL REFERENCES teams(id) ON DELETE CASCADE,

    url TEXT NOT NULL,
    secret VARCHAR(255) NOT NULL,

    events TEXT[] NOT NULL,
    -- Events: ['candidate.created', 'candidate.scored', 'resume.parsed']

    is_active BOOLEAN DEFAULT true,

    last_triggered_at TIMESTAMP,
    success_count INTEGER DEFAULT 0,
    failure_count INTEGER DEFAULT 0,

    created_by UUID REFERENCES users(id),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_webhooks_team_id ON webhooks(team_id);
CREATE INDEX idx_webhooks_is_active ON webhooks(is_active);
```

### webhook_logs
Webhook delivery logs.

```sql
CREATE TABLE webhook_logs (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    webhook_id UUID NOT NULL REFERENCES webhooks(id) ON DELETE CASCADE,

    event_type VARCHAR(100) NOT NULL,
    payload JSONB NOT NULL,

    status_code INTEGER,
    response_body TEXT,

    attempt_number INTEGER DEFAULT 1,

    status VARCHAR(50) DEFAULT 'pending',
    -- Status: 'pending', 'success', 'failed', 'retrying'

    error_message TEXT,

    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    delivered_at TIMESTAMP
);

CREATE INDEX idx_webhook_logs_webhook_id ON webhook_logs(webhook_id);
CREATE INDEX idx_webhook_logs_status ON webhook_logs(status);
CREATE INDEX idx_webhook_logs_created_at ON webhook_logs(created_at DESC);
```

---

## Background Jobs

### background_jobs
Celery task tracking.

```sql
CREATE TABLE background_jobs (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    team_id UUID REFERENCES teams(id) ON DELETE CASCADE,
    user_id UUID REFERENCES users(id) ON DELETE SET NULL,

    job_type VARCHAR(100) NOT NULL,
    -- Types: 'resume_parse', 'github_fetch', 'bulk_score', 'report_generate'

    status VARCHAR(50) DEFAULT 'pending',
    -- Status: 'pending', 'running', 'completed', 'failed', 'cancelled'

    progress INTEGER DEFAULT 0, -- 0-100

    celery_task_id VARCHAR(255),

    input_data JSONB DEFAULT '{}',
    result_data JSONB DEFAULT '{}',
    error_message TEXT,

    started_at TIMESTAMP,
    completed_at TIMESTAMP,

    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_background_jobs_team_id ON background_jobs(team_id);
CREATE INDEX idx_background_jobs_status ON background_jobs(status);
CREATE INDEX idx_background_jobs_celery_task_id ON background_jobs(celery_task_id);
CREATE INDEX idx_background_jobs_created_at ON background_jobs(created_at DESC);
```

---

## Billing

### subscriptions
Stripe subscription tracking.

```sql
CREATE TABLE subscriptions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    team_id UUID NOT NULL UNIQUE REFERENCES teams(id) ON DELETE CASCADE,

    stripe_customer_id VARCHAR(255) UNIQUE,
    stripe_subscription_id VARCHAR(255) UNIQUE,

    plan VARCHAR(50) NOT NULL,
    status VARCHAR(50) NOT NULL,
    -- Status: 'active', 'trialing', 'past_due', 'cancelled', 'unpaid'

    current_period_start TIMESTAMP,
    current_period_end TIMESTAMP,
    cancel_at_period_end BOOLEAN DEFAULT false,

    trial_ends_at TIMESTAMP,

    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_subscriptions_team_id ON subscriptions(team_id);
CREATE INDEX idx_subscriptions_stripe_customer_id ON subscriptions(stripe_customer_id);
CREATE INDEX idx_subscriptions_status ON subscriptions(status);
```

### invoices
Billing invoice history.

```sql
CREATE TABLE invoices (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    team_id UUID NOT NULL REFERENCES teams(id) ON DELETE CASCADE,
    subscription_id UUID REFERENCES subscriptions(id) ON DELETE SET NULL,

    stripe_invoice_id VARCHAR(255) UNIQUE,

    amount_due INTEGER NOT NULL, -- cents
    amount_paid INTEGER DEFAULT 0,
    currency VARCHAR(3) DEFAULT 'USD',

    status VARCHAR(50) NOT NULL,
    -- Status: 'draft', 'open', 'paid', 'void', 'uncollectible'

    invoice_pdf TEXT,

    period_start TIMESTAMP,
    period_end TIMESTAMP,
    due_date TIMESTAMP,
    paid_at TIMESTAMP,

    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_invoices_team_id ON invoices(team_id);
CREATE INDEX idx_invoices_stripe_invoice_id ON invoices(stripe_invoice_id);
CREATE INDEX idx_invoices_status ON invoices(status);
```

---

## Notifications

### notifications
In-app notifications.

```sql
CREATE TABLE notifications (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,

    title VARCHAR(255) NOT NULL,
    message TEXT NOT NULL,

    type VARCHAR(50) NOT NULL,
    -- Types: 'info', 'success', 'warning', 'error'

    category VARCHAR(100),
    -- Categories: 'candidate', 'system', 'billing', 'security'

    action_url TEXT,
    action_label VARCHAR(100),

    is_read BOOLEAN DEFAULT false,
    read_at TIMESTAMP,

    metadata JSONB DEFAULT '{}',

    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_notifications_user_id ON notifications(user_id);
CREATE INDEX idx_notifications_is_read ON notifications(is_read);
CREATE INDEX idx_notifications_created_at ON notifications(created_at DESC);
```

### notification_preferences
User notification settings.

```sql
CREATE TABLE notification_preferences (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL UNIQUE REFERENCES users(id) ON DELETE CASCADE,

    email_notifications BOOLEAN DEFAULT true,
    push_notifications BOOLEAN DEFAULT true,

    notify_candidate_added BOOLEAN DEFAULT true,
    notify_candidate_scored BOOLEAN DEFAULT true,
    notify_resume_processed BOOLEAN DEFAULT true,
    notify_team_activity BOOLEAN DEFAULT false,

    digest_frequency VARCHAR(50) DEFAULT 'daily',
    -- Frequency: 'realtime', 'daily', 'weekly', 'never'

    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

---

## Saved Searches & Views

### saved_searches
User-saved search queries.

```sql
CREATE TABLE saved_searches (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    team_id UUID NOT NULL REFERENCES teams(id) ON DELETE CASCADE,

    name VARCHAR(255) NOT NULL,
    query_params JSONB NOT NULL,
    -- Structure: {"status": ["new", "reviewing"], "min_score": 70, "skills": ["python"]}

    is_default BOOLEAN DEFAULT false,

    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_saved_searches_user_id ON saved_searches(user_id);
CREATE INDEX idx_saved_searches_team_id ON saved_searches(team_id);
```

---

## PostgreSQL Extensions

```sql
-- UUID generation
CREATE EXTENSION IF NOT EXISTS "pgcrypto";

-- Full-text search
CREATE EXTENSION IF NOT EXISTS "pg_trgm";

-- Array operations
CREATE EXTENSION IF NOT EXISTS "intarray";
```

---

## Triggers

### Update updated_at timestamp

```sql
CREATE OR REPLACE FUNCTION update_updated_at_column()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = CURRENT_TIMESTAMP;
    RETURN NEW;
END;
$$ language 'plpgsql';

-- Apply to all tables with updated_at
CREATE TRIGGER update_users_updated_at BEFORE UPDATE ON users
    FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();

CREATE TRIGGER update_candidates_updated_at BEFORE UPDATE ON candidates
    FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();

-- ... (apply to all relevant tables)
```

---

## Data Retention Policies

1. **Audit Logs**: Retain for 1 year, then archive to cold storage
2. **Email Logs**: Retain for 90 days
3. **Webhook Logs**: Retain for 30 days
4. **Sessions**: Auto-expire and delete after expiration
5. **Deleted Candidates**: Soft delete for 30 days, then hard delete
6. **Background Jobs**: Retain completed jobs for 7 days

---

## Backup Strategy

1. **Full Backup**: Daily at 2 AM UTC
2. **Incremental Backup**: Every 6 hours
3. **Point-in-Time Recovery**: Enabled with 7-day window
4. **Backup Retention**: 30 days
5. **Geographic Replication**: Multi-region for disaster recovery

---

## Performance Optimization

1. **Partitioning**: Consider partitioning `audit_logs`, `email_logs`, `webhook_logs` by date
2. **Archival**: Move old records to archive tables
3. **Materialized Views**: For complex analytics queries
4. **Connection Pooling**: PgBouncer for connection management
5. **Query Optimization**: Regular EXPLAIN ANALYZE on slow queries

---

## Security Considerations

1. **Row-Level Security**: Enable RLS for multi-tenant isolation
2. **Encrypted Columns**: Use pgcrypto for sensitive data
3. **No Sensitive Data**: Never store passwords in plain text
4. **Audit Everything**: All data modifications logged
5. **Soft Deletes**: Use deleted_at for data recovery

---

## Next Steps

1. Create migration files using Alembic
2. Seed initial data (default roles, email templates)
3. Set up database monitoring (pg_stat_statements)
4. Configure automated backups
5. Implement connection pooling

This schema provides a solid foundation for the HIRESIGHT platform with scalability, security, and maintainability in mind.
