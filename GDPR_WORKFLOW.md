# High-Level Design


##  GDPR Hard Delete Flow

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
