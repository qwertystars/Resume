# Integrations Implementation Guide

## Overview
Implement integrations with external services: Email (SendGrid), Webhooks, Slack/Discord, Calendar APIs.

---

## Email Integration (SendGrid)

### Backend Service

**File: `app/services/email_service.py`**

```python
from sendgrid import SendGridAPIClient
from sendgrid.helpers.mail import Mail, Email, To, Content
from app.core.config import settings
from app.models.email_log import EmailLog
from app.models.email_template import EmailTemplate
from sqlalchemy.orm import Session
import logging

logger = logging.getLogger(__name__)

class EmailService:
    def __init__(self, db: Session):
        self.db = db
        self.client = SendGridAPIClient(settings.SENDGRID_API_KEY)

    def send_email(
        self,
        to_email: str,
        to_name: str,
        template_key: str,
        variables: dict,
        team_id: str
    ):
        """Send email using template"""

        try:
            # Get template
            template = self.db.query(EmailTemplate).filter(
                EmailTemplate.team_id == team_id,
                EmailTemplate.template_key == template_key,
                EmailTemplate.is_active == True
            ).first()

            if not template:
                raise ValueError(f"Template {template_key} not found")

            # Replace variables in subject and body
            subject = self._replace_variables(template.subject, variables)
            body_html = self._replace_variables(template.body_html, variables)

            # Create email
            message = Mail(
                from_email=Email(settings.FROM_EMAIL, "HireSight"),
                to_emails=To(to_email, to_name),
                subject=subject,
                html_content=Content("text/html", body_html)
            )

            # Send
            response = self.client.send(message)

            # Log email
            email_log = EmailLog(
                team_id=team_id,
                recipient_email=to_email,
                recipient_name=to_name,
                subject=subject,
                template_key=template_key,
                status="sent",
                provider="sendgrid",
                provider_message_id=response.headers.get("X-Message-Id"),
                sent_at=datetime.utcnow()
            )
            self.db.add(email_log)
            self.db.commit()

            logger.info(f"Email sent to {to_email}")

            return {"success": True, "message_id": response.headers.get("X-Message-Id")}

        except Exception as e:
            logger.error(f"Failed to send email to {to_email}: {str(e)}")

            # Log failure
            email_log = EmailLog(
                team_id=team_id,
                recipient_email=to_email,
                recipient_name=to_name,
                subject=subject if 'subject' in locals() else "",
                template_key=template_key,
                status="failed",
                error_message=str(e)
            )
            self.db.add(email_log)
            self.db.commit()

            raise

    def _replace_variables(self, text: str, variables: dict) -> str:
        """Replace {{variable}} placeholders"""
        for key, value in variables.items():
            text = text.replace(f"{{{{{key}}}}}", str(value))
        return text

    def send_bulk_email(self, recipients: list, template_key: str, team_id: str):
        """Send email to multiple recipients"""
        from app.workers.email_tasks import send_email_task

        for recipient in recipients:
            send_email_task.delay(
                recipient["email"],
                recipient["name"],
                template_key,
                recipient.get("variables", {}),
                team_id
            )
```

### Celery Task

**File: `app/workers/email_tasks.py`**

```python
from celery import shared_task
from app.services.email_service import EmailService
from app.core.database import SessionLocal

@shared_task(bind=True, max_retries=3)
def send_email_task(self, to_email, to_name, template_key, variables, team_id):
    """Send email in background"""
    db = SessionLocal()

    try:
        email_service = EmailService(db)
        email_service.send_email(to_email, to_name, template_key, variables, team_id)
    except Exception as e:
        logger.error(f"Email task failed: {str(e)}")
        raise self.retry(exc=e, countdown=60)
    finally:
        db.close()
```

### API Endpoints

**File: `app/api/v1/integrations.py`**

```python
from fastapi import APIRouter, Depends

router = APIRouter(prefix="/integrations", tags=["integrations"])

@router.post("/email/send")
async def send_email(
    data: dict,
    current_user = Depends(get_current_user),
    db = Depends(get_db)
):
    """Send email to candidate"""

    email_service = EmailService(db)

    result = email_service.send_email(
        to_email=data["email"],
        to_name=data["name"],
        template_key=data["template_key"],
        variables=data.get("variables", {}),
        team_id=current_user.team_id
    )

    return {
        "success": True,
        "data": result
    }

@router.get("/email/templates")
async def get_email_templates(
    current_user = Depends(get_current_user),
    db = Depends(get_db)
):
    """Get email templates"""

    templates = db.query(EmailTemplate).filter(
        EmailTemplate.team_id == current_user.team_id
    ).all()

    return {
        "success": True,
        "data": {"templates": [t.to_dict() for t in templates]}
    }

@router.post("/email/templates")
async def create_email_template(
    data: dict,
    current_user = Depends(get_current_user),
    db = Depends(get_db)
):
    """Create email template"""

    template = EmailTemplate(
        team_id=current_user.team_id,
        created_by=current_user.id,
        **data
    )

    db.add(template)
    db.commit()

    return {
        "success": True,
        "data": {"template": template.to_dict()}
    }
```

---

## Webhooks

### Webhook Service

**File: `app/services/webhook_service.py`**

```python
import httpx
import hmac
import hashlib
from app.models.webhook import Webhook
from app.models.webhook_log import WebhookLog

class WebhookService:
    def __init__(self, db: Session):
        self.db = db

    async def trigger_webhook(self, event: str, data: dict, team_id: str):
        """Trigger all webhooks for this event"""

        webhooks = self.db.query(Webhook).filter(
            Webhook.team_id == team_id,
            Webhook.is_active == True
        ).all()

        for webhook in webhooks:
            if event in webhook.events:
                await self._send_webhook(webhook, event, data)

    async def _send_webhook(self, webhook: Webhook, event: str, data: dict):
        """Send webhook request"""

        payload = {
            "event": event,
            "timestamp": datetime.utcnow().isoformat(),
            "data": data
        }

        # Generate signature
        signature = self._generate_signature(payload, webhook.secret)

        try:
            async with httpx.AsyncClient(timeout=10.0) as client:
                response = await client.post(
                    webhook.url,
                    json=payload,
                    headers={
                        "X-Webhook-Signature": signature,
                        "X-Webhook-Event": event
                    }
                )

            # Log success
            log = WebhookLog(
                webhook_id=webhook.id,
                event_type=event,
                payload=payload,
                status_code=response.status_code,
                response_body=response.text[:1000],
                status="success" if response.status_code < 400 else "failed"
            )

            webhook.last_triggered_at = datetime.utcnow()
            if response.status_code < 400:
                webhook.success_count += 1
            else:
                webhook.failure_count += 1

        except Exception as e:
            # Log failure
            log = WebhookLog(
                webhook_id=webhook.id,
                event_type=event,
                payload=payload,
                status="failed",
                error_message=str(e)
            )

            webhook.failure_count += 1

        self.db.add(log)
        self.db.commit()

    def _generate_signature(self, payload: dict, secret: str) -> str:
        """Generate HMAC signature"""
        import json

        message = json.dumps(payload, sort_keys=True).encode()
        signature = hmac.new(
            secret.encode(),
            message,
            hashlib.sha256
        ).hexdigest()

        return f"sha256={signature}"
```

### Webhook Endpoints

**File: `app/api/v1/integrations.py` (continued)**

```python
@router.get("/webhooks")
async def get_webhooks(
    current_user = Depends(get_current_user),
    db = Depends(get_db)
):
    """Get webhooks"""

    webhooks = db.query(Webhook).filter(
        Webhook.team_id == current_user.team_id
    ).all()

    return {
        "success": True,
        "data": {"webhooks": [w.to_dict() for w in webhooks]}
    }

@router.post("/webhooks")
async def create_webhook(
    data: dict,
    current_user = Depends(get_current_user),
    db = Depends(get_db)
):
    """Create webhook"""

    import secrets

    webhook = Webhook(
        team_id=current_user.team_id,
        url=data["url"],
        secret=secrets.token_urlsafe(32),
        events=data["events"],
        created_by=current_user.id
    )

    db.add(webhook)
    db.commit()

    return {
        "success": True,
        "data": {
            "webhook": webhook.to_dict(),
            "secret": webhook.secret
        }
    }

@router.get("/webhooks/{webhook_id}/logs")
async def get_webhook_logs(
    webhook_id: str,
    current_user = Depends(get_current_user),
    db = Depends(get_db)
):
    """Get webhook delivery logs"""

    logs = db.query(WebhookLog).filter(
        WebhookLog.webhook_id == webhook_id
    ).order_by(WebhookLog.created_at.desc()).limit(100).all()

    return {
        "success": True,
        "data": {"logs": [l.to_dict() for l in logs]}
    }
```

---

## Slack Integration

### Slack Service

**File: `app/services/slack_service.py`**

```python
import httpx
from app.core.config import settings

class SlackService:
    def __init__(self, webhook_url: str = None):
        self.webhook_url = webhook_url or settings.SLACK_WEBHOOK_URL

    async def send_message(self, channel: str, message: str, attachments: list = None):
        """Send message to Slack"""

        payload = {
            "channel": channel,
            "text": message,
            "attachments": attachments or []
        }

        async with httpx.AsyncClient() as client:
            response = await client.post(self.webhook_url, json=payload)
            response.raise_for_status()

        return {"success": True}

    async def notify_new_candidate(self, candidate: dict):
        """Notify about new candidate"""

        message = f"New candidate: {candidate['full_name']}"

        attachments = [{
            "color": "#3b82f6",
            "fields": [
                {
                    "title": "Email",
                    "value": candidate.get("email", "N/A"),
                    "short": True
                },
                {
                    "title": "Score",
                    "value": f"{candidate.get('overall_score', 0)}/100",
                    "short": True
                },
                {
                    "title": "Experience",
                    "value": f"{candidate.get('years_of_experience', 0)} years",
                    "short": True
                },
                {
                    "title": "Location",
                    "value": candidate.get("location", "N/A"),
                    "short": True
                }
            ],
            "actions": [
                {
                    "type": "button",
                    "text": "View Profile",
                    "url": f"{settings.FRONTEND_URL}/candidates/{candidate['id']}"
                }
            ]
        }]

        await self.send_message("#hiring", message, attachments)
```

---

## Calendar Integration (Google Calendar)

### Calendar Service

**File: `app/services/calendar_service.py`**

```python
from google.oauth2.credentials import Credentials
from googleapiclient.discovery import build
from datetime import datetime, timedelta

class CalendarService:
    def __init__(self, credentials_dict: dict):
        self.credentials = Credentials.from_authorized_user_info(credentials_dict)
        self.service = build('calendar', 'v3', credentials=self.credentials)

    def create_interview_event(
        self,
        candidate_name: str,
        candidate_email: str,
        interviewer_email: str,
        start_time: datetime,
        duration_minutes: int = 60
    ):
        """Create interview calendar event"""

        end_time = start_time + timedelta(minutes=duration_minutes)

        event = {
            'summary': f'Interview with {candidate_name}',
            'description': 'Technical interview',
            'start': {
                'dateTime': start_time.isoformat(),
                'timeZone': 'UTC',
            },
            'end': {
                'dateTime': end_time.isoformat(),
                'timeZone': 'UTC',
            },
            'attendees': [
                {'email': candidate_email},
                {'email': interviewer_email},
            ],
            'reminders': {
                'useDefault': False,
                'overrides': [
                    {'method': 'email', 'minutes': 24 * 60},
                    {'method': 'popup', 'minutes': 30},
                ],
            },
        }

        event = self.service.events().insert(
            calendarId='primary',
            body=event,
            sendUpdates='all'
        ).execute()

        return {
            "event_id": event['id'],
            "html_link": event.get('htmlLink')
        }

    def check_availability(
        self,
        email: str,
        start_time: datetime,
        end_time: datetime
    ):
        """Check if person is available"""

        body = {
            "timeMin": start_time.isoformat() + 'Z',
            "timeMax": end_time.isoformat() + 'Z',
            "items": [{"id": email}]
        }

        eventsResult = self.service.freebusy().query(body=body).execute()
        busy_times = eventsResult['calendars'][email]['busy']

        return len(busy_times) == 0
```

---

## Frontend Integration Components

### Email Template Builder

**File: `frontend/src/components/integrations/EmailTemplateBuilder.tsx`**

```typescript
import React, { useState } from 'react';
import { Button } from '../ui/Button';
import { Input } from '../ui/Input';

export const EmailTemplateBuilder: React.FC = () => {
  const [template, setTemplate] = useState({
    template_name: '',
    template_key: '',
    subject: '',
    body_html: '',
    variables: []
  });

  const handleSave = async () => {
    // Call API to save template
    await createEmailTemplate(template);
  };

  return (
    <div className="space-y-4">
      <Input
        label="Template Name"
        value={template.template_name}
        onChange={(e) => setTemplate({ ...template, template_name: e.target.value })}
      />

      <Input
        label="Template Key"
        value={template.template_key}
        onChange={(e) => setTemplate({ ...template, template_key: e.target.value })}
        helperText="e.g., interview_invitation"
      />

      <Input
        label="Subject"
        value={template.subject}
        onChange={(e) => setTemplate({ ...template, subject: e.target.value })}
        helperText="Use {{variable}} for dynamic content"
      />

      <div>
        <label className="block text-sm font-medium text-gray-700 mb-1">
          Email Body (HTML)
        </label>
        <textarea
          value={template.body_html}
          onChange={(e) => setTemplate({ ...template, body_html: e.target.value })}
          rows={10}
          className="w-full border rounded-lg px-3 py-2"
        />
      </div>

      <div className="flex justify-end">
        <Button onClick={handleSave}>Save Template</Button>
      </div>
    </div>
  );
};
```

This implementation provides comprehensive integration capabilities with external services for email, webhooks, notifications, and calendar management.
