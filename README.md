# DB-Software-ERP-costruction
# PWA Multi-Tenant Database

Schema database PostgreSQL progettato per una PWA multi-tenant con backoffice amministrativo e app mobile (iOS/Android).

## Caratteristiche principali
- Isolamento dati tra aziende (**multi-tenancy**)
- Sicurezza a livello di riga (**Row Level Security**)
- Sistema di attivazione utenti
- Audit trail partizionato e log attività
- Workflow correzioni ore lavorate e notifiche real-time
- Indici e viste ottimizzati per reporting

## Requisiti
- PostgreSQL ≥ 13 (consigliato 14+)
- Estensione `pgcrypto`
- Estensione opzionale `pg_stat_statements`

## Diagramma E/R
Visualizza l’ER diagram direttamente tramite [Mermaid Live Editor](https://mermaid.live/edit#pako:eNp1kE1OwzAMhu95CuSy_1SLQogR2pgQdVVQFdQTXGBV6OnBv9c3qRDU-6zCwL9aNfZfH7k7LpP7rLq6gA9w5nHk1Vg6mGzDWaE7BToqJwT4jWBBx2T3GL6w9gZ6p5E7M3A5A6GUOl1gqQxM5AgvBk1TVqBMCxqC5mpuE1kF3K7n8h5Ih7g3wYJ6Qn4F4bTzF1HPiW5qfJdcXGgqJ4ZJpPMz0XnL4zXsb7vl47cJkqJbHs6zhgFvKj5hy8XBesn9CVKQwRiR9ALWpvKp6N0uO6n2kVx6Xb8R1Fv0JZgAA__).

Oppure apri il file [docs/er-diagram.mmd](docs/er-diagram.mmd) nel tuo editor preferito compatibile Mermaid.

## Documentazione dettagliata

- [Schema SQL completo](docs/database-schema.sql)
- [Diagramma ER (Mermaid)](docs/er-diagram.mmd)
- [Note e best practice](docs/note-operationali.md)

## Avvio rapido

```sql
CREATE EXTENSION IF NOT EXISTS pgcrypto;
-- Vedi docs/database-schema.sql per lo schema completo
```

## Note operative

Per la sicurezza multi-tenant e la gestione sessione, imposta sempre:

```sql
SET LOCAL app.current_company_id = '<UUID-azienda>';
SET LOCAL app.current_user_id = '<UUID-utente>';
```

---

> Consulta la cartella `/docs` per tutti i dettagli tecnici, funzioni PL/pgSQL, trigger, viste e policy di sicurezza.
