# High-Level Design

> 1. **System Architecture** — services and their interactions, assuming which is already done, if it required to develop from scratch, what could be possible service and their flow
> 2. **GDPR Hard Delete Flow** — swimlane diagram of the full deletion lifecycle

## Page 1: System Architecture

**5 consumers** → API Gateway (auth + RLS role mapping) → **7 backend services** → Unified PostgreSQL

### Backend Services

| Service | Responsibility | Interacts With |
|---|---|---|
| **Account Service** | User profiles, dependents, channels, platforms | → Application Service (creates services) |
| **Application Service** | Visa/permit/compliance lifecycle, workflow steps, family applications | → Document Service (requires docs), → Payment Service (charges) |
| **Document Service** | File upload, storage, staff review, status tracking | → AI Review Service (triggers on upload), ← Object Storage |
| **AI Review Service** | Async document review (approved/rejected/unsure), chat AI handoff | → Document Service (returns verdict), → Communication Service (handoff) |
| **Communication Service** | Conversations, lead qualification, account linking, multi-channel | ← Social Channels (webhooks), → Account Service (links lead to account) |
| **Payment Service** | Multi-provider (Stripe/Wise/Bank), 1 payment per service | ← Payment Providers (webhooks) |
| **Audit & Notification** | System-wide audit log, push/email/in-app notifications, GDPR orchestration | ← All services log mutations here |

### External Systems

- **Social Channels** (Instagram, TikTok, WhatsApp) → webhooks into Communication Service
- **Payment Providers** (Stripe, Wise, Bank) → webhooks into Payment Service
- **Object Storage** (GCS/S3) ↔ Document Service (file read/write)
- **Firebase Auth** → JWT tokens validated at API Gateway

## Page 2: GDPR Hard Delete Flow

Swimlane diagram across: Client → Account Service → PostgreSQL → Scheduled Job → Object Storage

1. **Client** requests account deletion
2. **Account Service** sets `deleted_at = now()` on the account row
3. **Response**: 202 Accepted — account is now soft-deleted and invisible to all consumers
4. **30-day grace period** — client can cancel; all data still exists but hidden from API
5. **GDPR Purge Job** (runs daily) picks up expired accounts:
   - Collects file storage paths before deletion
   - `DELETE FROM accounts WHERE deleted_at < now() - 30 days` — CASCADE removes all child rows (dependents, services, documents, files, reviews, payments, channels, links, notifications)
   - Anonymizes `audit_log` entries (replaces IDs with `[deleted]`, retains rows for compliance)
   - Bulk-deletes files from object storage using collected paths
6. **Conversations preserved** — only the link row is deleted; conversation records remain as unlinked leads
