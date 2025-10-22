# API Specification

## Overview
Complete REST API documentation for HIRESIGHT platform. All endpoints return JSON and require authentication unless otherwise noted.

**Base URL**: `https://api.hiresight.com/api/v1`

---

## Authentication

All authenticated endpoints require a JWT Bearer token in the Authorization header:
```text
Authorization: Bearer <access_token>
```

### Authentication Endpoints

#### POST /auth/register
Register a new user account.

**Request:**
```json
{
  "email": "user@example.com",
  "password": "SecureP@ss123",
  "full_name": "John Doe",
  "team_name": "Acme Corp" // Optional, creates new team
}
```

**Response (201):**
```json
{
  "success": true,
  "data": {
    "user": {
      "id": "uuid",
      "email": "user@example.com",
      "full_name": "John Doe",
      "role": "recruiter"
    },
    "team": {
      "id": "uuid",
      "name": "Acme Corp",
      "slug": "acme-corp"
    },
    "access_token": "eyJ...",
    "refresh_token": "eyJ...",
    "token_type": "bearer",
    "expires_in": 3600
  }
}
```

#### POST /auth/login
Login with email and password.

**Request:**
```json
{
  "email": "user@example.com",
  "password": "SecureP@ss123"
}
```

**Response (200):**
```json
{
  "success": true,
  "data": {
    "access_token": "eyJ...",
    "refresh_token": "eyJ...",
    "token_type": "bearer",
    "expires_in": 3600,
    "user": {
      "id": "uuid",
      "email": "user@example.com",
      "full_name": "John Doe",
      "role": "recruiter"
    }
  }
}
```

#### POST /auth/refresh
Refresh access token using refresh token.

**Request:**
```json
{
  "refresh_token": "eyJ..."
}
```

**Response (200):**
```json
{
  "success": true,
  "data": {
    "access_token": "eyJ...",
    "token_type": "bearer",
    "expires_in": 3600
  }
}
```

#### POST /auth/logout
Revoke current refresh token.

**Response (200):**
```json
{
  "success": true,
  "message": "Logged out successfully"
}
```

#### POST /auth/forgot-password
Request password reset email.

**Request:**
```json
{
  "email": "user@example.com"
}
```

**Response (200):**
```json
{
  "success": true,
  "message": "Password reset email sent"
}
```

#### POST /auth/reset-password
Reset password with token from email.

**Request:**
```json
{
  "token": "reset_token_from_email",
  "new_password": "NewSecureP@ss123"
}
```

#### POST /auth/google
OAuth login with Google.

**Request:**
```json
{
  "id_token": "google_id_token"
}
```

#### POST /auth/github
OAuth login with GitHub.

**Request:**
```json
{
  "code": "github_oauth_code"
}
```

---

## Candidates

### GET /candidates
List all candidates with filtering, sorting, and pagination.

**Query Parameters:**
- `page` (default: 1)
- `per_page` (default: 20, max: 100)
- `status` (filter by status)
- `stage` (filter by stage)
- `min_score` (minimum overall score)
- `max_score` (maximum overall score)
- `skills` (comma-separated skill keywords)
- `location` (location filter)
- `source` (candidate source)
- `assigned_to` (user UUID)
- `tags` (comma-separated tags)
- `search` (full-text search on name, email)
- `sort_by` (field to sort by: created_at, overall_score, full_name)
- `sort_order` (asc or desc)

**Response (200):**
```json
{
  "success": true,
  "data": {
    "candidates": [
      {
        "id": "uuid",
        "full_name": "Jane Smith",
        "email": "jane@example.com",
        "phone": "+1-555-0100",
        "location": "San Francisco, CA",
        "status": "reviewing",
        "stage": "screening",
        "years_of_experience": 5.5,
        "education_level": "bachelor",
        "primary_skills": ["Python", "FastAPI", "React"],
        "overall_score": 85.5,
        "resume_score": 80.0,
        "github_score": 90.0,
        "linkedin_score": 86.0,
        "applied_at": "2025-10-15T10:30:00Z",
        "created_at": "2025-10-15T10:30:00Z",
        "updated_at": "2025-10-20T14:20:00Z"
      }
    ]
  },
  "metadata": {
    "page": 1,
    "per_page": 20,
    "total": 150,
    "total_pages": 8
  }
}
```

### POST /candidates
Create a new candidate.

**Request:**
```json
{
  "full_name": "Jane Smith",
  "email": "jane@example.com",
  "phone": "+1-555-0100",
  "location": "San Francisco, CA",
  "linkedin_url": "https://linkedin.com/in/janesmith",
  "github_url": "https://github.com/janesmith",
  "portfolio_url": "https://janesmith.dev",
  "source": "linkedin",
  "tags": ["senior", "backend"],
  "notes": "Excellent candidate with strong Python skills"
}
```

**Response (201):**
```json
{
  "success": true,
  "data": {
    "candidate": {
      "id": "uuid",
      "full_name": "Jane Smith",
      // ... full candidate object
    }
  }
}
```

### GET /candidates/{id}
Get detailed candidate information.

**Response (200):**
```json
{
  "success": true,
  "data": {
    "candidate": {
      "id": "uuid",
      "full_name": "Jane Smith",
      "email": "jane@example.com",
      // ... all fields
      "resume": {
        "id": "uuid",
        "file_name": "jane_smith_resume.pdf",
        "storage_url": "https://...",
        "uploaded_at": "2025-10-15T10:30:00Z"
      },
      "github_profile": {
        "github_username": "janesmith",
        "public_repos": 42,
        "total_stars": 1250,
        "languages": {
          "Python": 45.2,
          "JavaScript": 32.1
        }
      },
      "linkedin_profile": {
        "headline": "Senior Software Engineer",
        "current_position": "Lead Developer",
        "connections": 500
      },
      "activities": [
        {
          "id": "uuid",
          "activity_type": "status_change",
          "description": "Status changed from 'new' to 'reviewing'",
          "created_at": "2025-10-16T09:00:00Z"
        }
      ]
    }
  }
}
```

### PUT /candidates/{id}
Update candidate information.

**Request:**
```json
{
  "status": "interviewed",
  "stage": "technical",
  "notes": "Great technical interview",
  "tags": ["senior", "backend", "python-expert"]
}
```

### DELETE /candidates/{id}
Soft delete a candidate.

**Response (200):**
```json
{
  "success": true,
  "message": "Candidate deleted successfully"
}
```

### PATCH /candidates/{id}/assign
Assign candidate to a recruiter.

**Request:**
```json
{
  "user_id": "uuid"
}
```

### POST /candidates/{id}/activities
Add activity/note to candidate.

**Request:**
```json
{
  "activity_type": "note_added",
  "description": "Follow-up call scheduled for next week"
}
```

### POST /candidates/bulk-action
Perform bulk actions on multiple candidates.

**Request:**
```json
{
  "candidate_ids": ["uuid1", "uuid2", "uuid3"],
  "action": "update_status",
  "params": {
    "status": "rejected"
  }
}
```

**Actions:**
- `update_status`
- `update_stage`
- `add_tags`
- `remove_tags`
- `assign`
- `delete`
- `export`

### GET /candidates/export
Export candidates to CSV.

**Query Parameters:**
- Same filters as GET /candidates
- `format` (csv or excel)

**Response:** File download

---

## Resumes

### POST /resumes/upload
Upload resume file.

**Request:** multipart/form-data
- `file`: Resume file (PDF, DOCX, DOC, TXT)
- `candidate_id`: UUID (optional, creates new candidate if not provided)

**Response (201):**
```json
{
  "success": true,
  "data": {
    "resume": {
      "id": "uuid",
      "candidate_id": "uuid",
      "file_name": "resume.pdf",
      "file_size": 245678,
      "file_type": "pdf",
      "storage_url": "https://...",
      "status": "processing"
    }
  }
}
```

### POST /resumes/bulk-upload
Upload multiple resume files.

**Request:** multipart/form-data
- `files[]`: Array of resume files

**Response (201):**
```json
{
  "success": true,
  "data": {
    "job_id": "uuid",
    "total_files": 15,
    "status": "processing"
  }
}
```

### GET /resumes/{id}
Get resume details and parsed data.

**Response (200):**
```json
{
  "success": true,
  "data": {
    "resume": {
      "id": "uuid",
      "candidate_id": "uuid",
      "file_name": "resume.pdf",
      "status": "completed",
      "storage_url": "https://...",
      "parsed_data": {
        "contact": {
          "email": "jane@example.com",
          "phone": "+1-555-0100",
          "location": "San Francisco, CA"
        },
        "experience": [
          {
            "title": "Senior Software Engineer",
            "company": "Tech Corp",
            "location": "San Francisco, CA",
            "start_date": "2020-01",
            "end_date": "present",
            "description": "Led team of 5 engineers..."
          }
        ],
        "education": [
          {
            "degree": "Bachelor of Science",
            "field": "Computer Science",
            "school": "Stanford University",
            "graduation_year": 2018
          }
        ],
        "skills": [
          "Python", "FastAPI", "React", "PostgreSQL"
        ]
      }
    }
  }
}
```

### GET /resumes/{id}/download
Download original resume file.

**Response:** File download

### POST /resumes/{id}/reprocess
Trigger re-parsing of resume.

---

## GitHub Integration

### POST /github/connect
Connect GitHub account via OAuth.

**Request:**
```json
{
  "code": "github_oauth_code",
  "candidate_id": "uuid" // optional
}
```

### GET /github/profile/{username}
Fetch GitHub profile data for username.

**Response (200):**
```json
{
  "success": true,
  "data": {
    "profile": {
      "github_username": "janesmith",
      "github_user_id": 12345,
      "public_repos": 42,
      "followers": 150,
      "following": 75,
      "total_stars": 1250,
      "total_forks": 340,
      "total_commits": 5420,
      "languages": {
        "Python": 45.2,
        "JavaScript": 32.1,
        "Go": 22.7
      },
      "contributions_last_year": 1845,
      "most_active_repo": "awesome-project",
      "github_created_at": "2018-03-15T10:00:00Z",
      "last_commit_at": "2025-10-20T15:30:00Z"
    }
  }
}
```

### POST /github/analyze/{candidate_id}
Trigger GitHub analysis for candidate.

**Response (202):**
```json
{
  "success": true,
  "data": {
    "job_id": "uuid",
    "status": "processing"
  }
}
```

### GET /github/repositories/{username}
Get repository list for username.

**Response (200):**
```json
{
  "success": true,
  "data": {
    "repositories": [
      {
        "name": "awesome-project",
        "description": "An awesome project",
        "stars": 450,
        "forks": 120,
        "language": "Python",
        "last_updated": "2025-10-15T10:00:00Z"
      }
    ]
  }
}
```

---

## Scoring

### POST /scoring/calculate/{candidate_id}
Calculate/recalculate scores for candidate.

**Response (200):**
```json
{
  "success": true,
  "data": {
    "scores": {
      "overall_score": 85.5,
      "resume_score": 80.0,
      "github_score": 90.0,
      "linkedin_score": 86.0,
      "breakdown": {
        "experience": 25.0,
        "skills": 30.0,
        "education": 15.0,
        "github_activity": 20.0,
        "linkedin_presence": 10.0
      }
    }
  }
}
```

### GET /scoring/history/{candidate_id}
Get score history for candidate.

**Response (200):**
```json
{
  "success": true,
  "data": {
    "history": [
      {
        "score_type": "overall",
        "score_value": 85.5,
        "previous_value": 82.0,
        "reason": "GitHub analysis completed",
        "calculated_at": "2025-10-20T10:00:00Z"
      }
    ]
  }
}
```

### GET /scoring/rules
Get scoring rules for team.

**Response (200):**
```json
{
  "success": true,
  "data": {
    "rules": [
      {
        "id": "uuid",
        "rule_name": "Python Experience",
        "rule_type": "keyword_match",
        "conditions": {
          "keywords": ["python", "django", "fastapi"]
        },
        "weight": 1.5,
        "points": 20,
        "is_active": true
      }
    ]
  }
}
```

### POST /scoring/rules
Create new scoring rule.

**Request:**
```json
{
  "rule_name": "Senior Experience",
  "rule_type": "experience_years",
  "conditions": {
    "min_years": 5
  },
  "weight": 1.0,
  "points": 15
}
```

### PUT /scoring/rules/{id}
Update scoring rule.

### DELETE /scoring/rules/{id}
Delete scoring rule.

---

## Analytics

### GET /analytics/overview
Get dashboard overview metrics.

**Query Parameters:**
- `start_date` (ISO format)
- `end_date` (ISO format)

**Response (200):**
```json
{
  "success": true,
  "data": {
    "total_candidates": 245,
    "new_this_week": 18,
    "avg_score": 72.5,
    "by_status": {
      "new": 45,
      "reviewing": 80,
      "interviewed": 65,
      "offered": 25,
      "hired": 30
    },
    "by_stage": {
      "applied": 45,
      "screening": 60,
      "technical": 50,
      "onsite": 40,
      "offer": 25,
      "hired": 30
    },
    "top_skills": [
      {"skill": "Python", "count": 120},
      {"skill": "React", "count": 95},
      {"skill": "Node.js", "count": 80}
    ]
  }
}
```

### GET /analytics/funnel
Get hiring funnel metrics.

**Response (200):**
```json
{
  "success": true,
  "data": {
    "stages": [
      {
        "stage": "applied",
        "count": 245,
        "conversion_rate": 100.0,
        "avg_time_days": 0
      },
      {
        "stage": "screening",
        "count": 180,
        "conversion_rate": 73.5,
        "avg_time_days": 2.5
      },
      {
        "stage": "technical",
        "count": 120,
        "conversion_rate": 66.7,
        "avg_time_days": 5.2
      }
    ]
  }
}
```

### GET /analytics/trends
Get hiring trends over time.

**Query Parameters:**
- `metric` (applications, hires, avg_score)
- `period` (day, week, month)
- `start_date`
- `end_date`

**Response (200):**
```json
{
  "success": true,
  "data": {
    "series": [
      {
        "date": "2025-10-01",
        "value": 15
      },
      {
        "date": "2025-10-08",
        "value": 22
      }
    ]
  }
}
```

### GET /analytics/sources
Get candidate source effectiveness.

**Response (200):**
```json
{
  "success": true,
  "data": {
    "sources": [
      {
        "source": "linkedin",
        "total": 120,
        "hired": 25,
        "conversion_rate": 20.8,
        "avg_score": 78.5
      }
    ]
  }
}
```

### GET /analytics/team-performance
Get team metrics (GitHub-related).

**Response (200):**
```json
{
  "success": true,
  "data": {
    "total_commits": 15420,
    "total_prs": 842,
    "avg_pr_merge_time_hours": 18.5,
    "top_contributors": [
      {
        "name": "Jane Smith",
        "commits": 1234,
        "prs": 89
      }
    ],
    "language_breakdown": {
      "Python": 45.2,
      "JavaScript": 32.1
    }
  }
}
```

---

## Users & Teams

### GET /users/me
Get current user profile.

**Response (200):**
```json
{
  "success": true,
  "data": {
    "user": {
      "id": "uuid",
      "email": "user@example.com",
      "full_name": "John Doe",
      "role": "recruiter",
      "avatar_url": "https://...",
      "is_2fa_enabled": false,
      "created_at": "2025-09-01T10:00:00Z"
    }
  }
}
```

### PUT /users/me
Update current user profile.

**Request:**
```json
{
  "full_name": "John Smith",
  "avatar_url": "https://...",
  "timezone": "America/New_York"
}
```

### POST /users/change-password
Change user password.

**Request:**
```json
{
  "current_password": "OldPass123",
  "new_password": "NewPass456"
}
```

### POST /users/enable-2fa
Enable two-factor authentication.

**Response (200):**
```json
{
  "success": true,
  "data": {
    "secret": "BASE32SECRET",
    "qr_code_url": "data:image/png;base64,..."
  }
}
```

### POST /users/verify-2fa
Verify 2FA token.

**Request:**
```json
{
  "token": "123456"
}
```

### GET /teams/me
Get current team information.

### PUT /teams/me
Update team settings.

### GET /teams/members
List team members.

**Response (200):**
```json
{
  "success": true,
  "data": {
    "members": [
      {
        "user_id": "uuid",
        "full_name": "John Doe",
        "email": "john@example.com",
        "role": "admin",
        "joined_at": "2025-09-01T10:00:00Z"
      }
    ]
  }
}
```

### POST /teams/invite
Invite member to team.

**Request:**
```json
{
  "email": "newmember@example.com",
  "role": "member"
}
```

### DELETE /teams/members/{user_id}
Remove team member.

---

## Integrations

### Email

#### GET /integrations/email/templates
List email templates.

#### POST /integrations/email/templates
Create email template.

**Request:**
```json
{
  "template_name": "Interview Invitation",
  "template_key": "interview_invitation",
  "subject": "Interview Invitation for {{position}}",
  "body_html": "<html>...</html>",
  "variables": ["candidate_name", "position", "interview_date"]
}
```

#### POST /integrations/email/send
Send email to candidate.

**Request:**
```json
{
  "candidate_id": "uuid",
  "template_key": "interview_invitation",
  "variables": {
    "position": "Senior Engineer",
    "interview_date": "October 25, 2025"
  }
}
```

#### GET /integrations/email/logs
Get email delivery logs.

### Webhooks

#### GET /integrations/webhooks
List webhooks.

#### POST /integrations/webhooks
Create webhook.

**Request:**
```json
{
  "url": "https://your-app.com/webhook",
  "secret": "webhook_secret_key",
  "events": ["candidate.created", "candidate.scored"]
}
```

**Available Events:**
- `candidate.created`
- `candidate.updated`
- `candidate.deleted`
- `resume.uploaded`
- `resume.parsed`
- `score.calculated`
- `github.analyzed`

#### PUT /integrations/webhooks/{id}
Update webhook.

#### DELETE /integrations/webhooks/{id}
Delete webhook.

#### GET /integrations/webhooks/{id}/logs
Get webhook delivery logs.

### Slack/Discord

#### POST /integrations/slack/connect
Connect Slack workspace.

#### POST /integrations/slack/notify
Send notification to Slack channel.

**Request:**
```json
{
  "channel": "#hiring",
  "message": "New candidate Jane Smith scored 85/100"
}
```

---

## Background Jobs

### GET /jobs
List background jobs.

**Query Parameters:**
- `status` (pending, running, completed, failed)
- `job_type`

**Response (200):**
```json
{
  "success": true,
  "data": {
    "jobs": [
      {
        "id": "uuid",
        "job_type": "resume_parse",
        "status": "completed",
        "progress": 100,
        "started_at": "2025-10-20T10:00:00Z",
        "completed_at": "2025-10-20T10:05:00Z"
      }
    ]
  }
}
```

### GET /jobs/{id}
Get job details and results.

**Response (200):**
```json
{
  "success": true,
  "data": {
    "job": {
      "id": "uuid",
      "job_type": "bulk_score",
      "status": "running",
      "progress": 65,
      "result_data": {
        "processed": 65,
        "total": 100,
        "failed": 2
      }
    }
  }
}
```

### DELETE /jobs/{id}
Cancel running job.

---

## Billing

### GET /billing/subscription
Get current subscription.

**Response (200):**
```json
{
  "success": true,
  "data": {
    "subscription": {
      "plan": "professional",
      "status": "active",
      "current_period_end": "2025-11-20T00:00:00Z",
      "cancel_at_period_end": false
    }
  }
}
```

### POST /billing/checkout
Create Stripe checkout session.

**Request:**
```json
{
  "plan": "professional"
}
```

**Response (200):**
```json
{
  "success": true,
  "data": {
    "checkout_url": "https://checkout.stripe.com/..."
  }
}
```

### POST /billing/portal
Get Stripe customer portal URL.

**Response (200):**
```json
{
  "success": true,
  "data": {
    "portal_url": "https://billing.stripe.com/..."
  }
}
```

### GET /billing/invoices
List invoices.

### POST /billing/webhook
Stripe webhook endpoint (public).

---

## Notifications

### GET /notifications
Get user notifications.

**Query Parameters:**
- `is_read` (true/false)
- `category`

**Response (200):**
```json
{
  "success": true,
  "data": {
    "notifications": [
      {
        "id": "uuid",
        "title": "New Candidate",
        "message": "Jane Smith applied for Senior Engineer",
        "type": "info",
        "category": "candidate",
        "is_read": false,
        "action_url": "/candidates/uuid",
        "created_at": "2025-10-20T10:00:00Z"
      }
    ]
  },
  "metadata": {
    "unread_count": 5
  }
}
```

### PATCH /notifications/{id}/read
Mark notification as read.

### POST /notifications/read-all
Mark all notifications as read.

### GET /notifications/preferences
Get notification preferences.

### PUT /notifications/preferences
Update notification preferences.

---

## Developer Portal (for Candidates)

### POST /developer/signup
Developer self-registration.

### GET /developer/profile
Get developer profile and score.

**Response (200):**
```json
{
  "success": true,
  "data": {
    "profile": {
      "full_name": "Jane Smith",
      "email": "jane@example.com",
      "overall_score": 85.5,
      "profile_completeness": 80,
      "github_connected": true,
      "resume_uploaded": true,
      "linkedin_added": false
    },
    "suggestions": [
      "Connect your LinkedIn profile to increase your score",
      "Add more projects to your GitHub"
    ],
    "job_matches": [
      {
        "title": "Senior Python Engineer",
        "company": "Tech Corp",
        "match_score": 92
      }
    ]
  }
}
```

### PUT /developer/profile
Update developer profile.

### POST /developer/resume-upload
Upload resume (developer portal).

### POST /developer/github-connect
Connect GitHub (developer portal).

---

## API Keys

### GET /api-keys
List API keys for team.

**Response (200):**
```json
{
  "success": true,
  "data": {
    "keys": [
      {
        "id": "uuid",
        "key_name": "Production API",
        "key_prefix": "hs_live_abc",
        "last_used_at": "2025-10-20T10:00:00Z",
        "created_at": "2025-09-01T10:00:00Z"
      }
    ]
  }
}
```

### POST /api-keys
Create new API key.

**Request:**
```json
{
  "key_name": "Production API",
  "permissions": {
    "candidates": ["read", "write"],
    "analytics": ["read"]
  },
  "expires_at": "2026-10-20T00:00:00Z"
}
```

**Response (201):**
```json
{
  "success": true,
  "data": {
    // EXAMPLE: Replace with actual API key
    "api_key": "hs_live_abcdef123456789",
    "message": "Store this key securely. It won't be shown again."
  }
}
```

### DELETE /api-keys/{id}
Revoke API key.

---

## Search

### GET /search
Global search across candidates.

**Query Parameters:**
- `q` (search query)
- `type` (candidates, resumes, users)

**Response (200):**
```json
{
  "success": true,
  "data": {
    "results": [
      {
        "type": "candidate",
        "id": "uuid",
        "title": "Jane Smith",
        "description": "Senior Software Engineer with 5 years experience",
        "score": 0.95,
        "url": "/candidates/uuid"
      }
    ]
  }
}
```

---

## Health & Status

### GET /health
Health check endpoint (public).

**Response (200):**
```json
{
  "status": "healthy",
  "version": "1.0.0",
  "timestamp": "2025-10-22T10:00:00Z"
}
```

### GET /status
Detailed status check (authenticated).

**Response (200):**
```json
{
  "database": "connected",
  "redis": "connected",
  "celery": "running",
  "s3": "accessible"
}
```

---

## Rate Limiting

All endpoints are rate limited:
- **Authenticated users**: 1000 requests per hour
- **API keys**: Based on plan (10k-100k per hour)
- **Public endpoints**: 100 requests per hour per IP

Rate limit headers:
```
X-RateLimit-Limit: 1000
X-RateLimit-Remaining: 950
X-RateLimit-Reset: 1635340800
```

---

## Error Codes

| Code | Message | Description |
|------|---------|-------------|
| 400 | Bad Request | Invalid request data |
| 401 | Unauthorized | Missing or invalid auth token |
| 403 | Forbidden | Insufficient permissions |
| 404 | Not Found | Resource doesn't exist |
| 409 | Conflict | Duplicate resource |
| 422 | Validation Error | Invalid input data |
| 429 | Too Many Requests | Rate limit exceeded |
| 500 | Internal Server Error | Server error |
| 503 | Service Unavailable | Temporary outage |

---

## Pagination

All list endpoints support cursor-based pagination:

**Request:**
```
GET /candidates?page=2&per_page=20
```

**Response:**
```json
{
  "data": [...],
  "metadata": {
    "page": 2,
    "per_page": 20,
    "total": 150,
    "total_pages": 8,
    "has_next": true,
    "has_prev": true
  }
}
```

---

## Filtering & Sorting

Use query parameters for filtering:
```
GET /candidates?status=reviewing&min_score=70&sort_by=overall_score&sort_order=desc
```

---

## Webhooks Payload Format

All webhook events follow this structure:

```json
{
  "event": "candidate.created",
  "timestamp": "2025-10-22T10:00:00Z",
  "data": {
    "candidate": {
      "id": "uuid",
      // ... candidate object
    }
  }
}
```

Webhook signature is in the header:
```
X-Webhook-Signature: sha256=...
```

---

## SDK Examples

### Python
```python
from hiresight import HireSight

client = HireSight(api_key="hs_live_...")

# List candidates
candidates = client.candidates.list(min_score=70)

# Create candidate
candidate = client.candidates.create(
    full_name="Jane Smith",
    email="jane@example.com"
)

# Upload resume
resume = client.resumes.upload(
    file=open("resume.pdf", "rb"),
    candidate_id=candidate.id
)
```

### JavaScript
```javascript
const HireSight = require('hiresight-sdk');

const client = new HireSight({ apiKey: 'hs_live_...' });

// List candidates
const candidates = await client.candidates.list({ minScore: 70 });

// Create candidate
const candidate = await client.candidates.create({
  fullName: 'Jane Smith',
  email: 'jane@example.com'
});
```

---

This API specification provides the foundation for building the HIRESIGHT platform. All endpoints follow REST conventions and return consistent JSON responses.
