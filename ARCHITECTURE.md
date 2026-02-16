# Architecture Deep Dive

## System Overview

Nurturing CRM is a monolithic Laravel application with a React SPA frontend. The system is designed to handle high-volume email campaigns while maintaining responsiveness for day-to-day CRM operations.

## Database Schema (Simplified)

```
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│     users       │     │    campaigns    │     │    contacts     │
├─────────────────┤     ├─────────────────┤     ├─────────────────┤
│ id              │     │ id              │     │ id              │
│ name            │     │ user_id (FK)    │     │ email           │
│ email           │     │ name            │     │ first_name      │
│ role            │     │ subject         │     │ last_name       │
│ created_at      │     │ template_id     │     │ status          │
└─────────────────┘     │ scheduled_at    │     │ tags (JSON)     │
                        │ status          │     │ created_at      │
                        │ sent_count      │     └─────────────────┘
                        │ created_at      │
                        └─────────────────┘
                                │
                                ▼
                        ┌─────────────────┐
                        │  email_logs     │
                        ├─────────────────┤
                        │ id              │
                        │ campaign_id     │
                        │ contact_id      │
                        │ status          │
                        │ sent_at         │
                        │ opened_at       │
                        │ clicked_at      │
                        │ bounced_at      │
                        └─────────────────┘
```

## Email Processing Pipeline

### 1. Campaign Creation
```
User creates campaign
        │
        ▼
Validate template & contacts
        │
        ▼
Save campaign (status: draft)
```

### 2. Campaign Scheduling
```
User schedules campaign
        │
        ▼
Create SendCampaign job (delayed)
        │
        ▼
Laravel Scheduler triggers at scheduled time
```

### 3. Email Sending (The Heavy Lifting)
```
SendCampaign job starts
        │
        ▼
Query contacts in chunks (1,000 per batch)
        │
        ▼
For each chunk:
    │
    ├── Create SendEmailBatch job
    │
    └── Dispatch to Redis queue
        │
        ▼
Multiple workers process batches in parallel
        │
        ▼
Each worker:
    ├── Render email template with contact data
    ├── Send via SMTP
    ├── Log result to email_logs
    └── Update campaign sent_count
```

### 4. Delivery Tracking
```
SMTP Provider sends webhook
        │
        ▼
POST /api/webhooks/email
        │
        ▼
ProcessWebhook job queued
        │
        ▼
Update email_logs with:
    - delivered_at
    - opened_at
    - clicked_at
    - bounced_at
    - unsubscribed_at
```

## Caching Strategy

### What we cache

| Data | TTL | Invalidation |
|------|-----|--------------|
| Campaign stats | 5 min | On webhook received |
| Contact counts | 10 min | On contact CRUD |
| User permissions | 1 hour | On role change |
| Dashboard metrics | 5 min | Automatic |

### Cache keys pattern
```
nurturing:campaigns:{id}:stats
nurturing:contacts:count:{filter_hash}
nurturing:users:{id}:permissions
nurturing:dashboard:{user_id}:metrics
```

## API Endpoints (Main ones)

### Campaigns
```
GET    /api/campaigns              # List with pagination
POST   /api/campaigns              # Create
GET    /api/campaigns/{id}         # Show with stats
PUT    /api/campaigns/{id}         # Update
DELETE /api/campaigns/{id}         # Soft delete
POST   /api/campaigns/{id}/send    # Trigger send
POST   /api/campaigns/{id}/pause   # Pause sending
```

### Contacts
```
GET    /api/contacts               # List with filters
POST   /api/contacts               # Create
POST   /api/contacts/import        # Bulk import CSV
GET    /api/contacts/{id}          # Show with history
PUT    /api/contacts/{id}          # Update
DELETE /api/contacts/{id}          # Soft delete
```

### Analytics
```
GET    /api/analytics/campaigns/{id}     # Campaign performance
GET    /api/analytics/dashboard          # Overview metrics
GET    /api/analytics/export/{id}        # Export to CSV
```

## Performance Optimizations

### Database
- Composite indexes on `(campaign_id, status)` for email_logs
- Partial indexes for active contacts only
- Connection pooling with PgBouncer

### Application
- Lazy loading disabled, explicit eager loading
- API responses use Resources (no raw models)
- Chunked processing for large datasets

### Frontend
- React Query for caching and deduplication
- Virtual scrolling for large contact lists
- Code splitting by route

## Deployment

```
GitHub Push
    │
    ▼
GitHub Actions CI
    ├── Run tests
    ├── Build React app
    └── Build Docker image
    │
    ▼
Push to Container Registry
    │
    ▼
Deploy to GCP Cloud Run
    ├── API containers (auto-scaling)
    └── Worker containers (queue processing)
```

## Monitoring

- **Errors:** Sentry for exception tracking
- **Performance:** Laravel Telescope (dev), custom metrics (prod)
- **Queues:** Laravel Horizon dashboard
- **Uptime:** GCP monitoring + alerts
