# DB-Software-ERP-costruction
# PWA Multi-Tenant Database

PostgreSQL database schema designed for a multi-tenant Progressive Web App (PWA) with an administrative backoffice and mobile apps (iOS/Android).

## Main Features
- Data isolation between companies (**multi-tenancy**)
- Row Level Security (**RLS**)
- User activation system
- Partitioned audit trail and activity logs
- Correction workflow for work hours and real-time notifications
- Optimized indexes and views for reporting

## Requirements
- PostgreSQL ≥ 13 (recommended 14+)
- Extension `pgcrypto`
- Optional extension `pg_stat_statements`

## ER Diagram
You can view the ER diagram directly via [Mermaid Live Editor](https://mermaid.live/) or by opening [`Mermaid Architecture/er-diagram.mmd`](Mermaid%20Architecture/er-diagram.mmd) (copy/paste in the editor).

## Documentation

- [Full SQL Schema]
- [ER Diagram (Mermaid)]
- [Operational Notes & Best Practices]

## Quickstart

```sql
CREATE EXTENSION IF NOT EXISTS pgcrypto;
-- See docs/database-schema.sql for the complete schema
```

## Operational Notes

To ensure multi-tenant security and session context, always set:

```sql
SET LOCAL app.current_company_id = '<company-UUID>';
SET LOCAL app.current_user_id = '<user-UUID>';
```

---

> See the `/docs` and `/Mermaid Architecture` folders for all technical details, PL/pgSQL functions, triggers, views, and security policies.
