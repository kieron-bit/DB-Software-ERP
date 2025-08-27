# DB-Software-ERP-costruction
1. Introduzione

Questo schema database PostgreSQL supporta un'applicazione PWA (Progressive Web App) con backoffice amministrativo e componenti mobile per dispositivi iOS e Android. La soluzione è progettata per essere deploy-ready e include tutte le funzionalità necessarie per un ambiente produttivo multi-tenant.
Caratteristiche principali:

    Multi-tenancy con isolamento completo dei dati tra aziende

    Row Level Security (RLS) per la sicurezza a livello di riga

    Sistema di attivazione utenti con codici monouso

    Audit trail completo con log partizionati

    Workflow di correzione per le ore lavorate

    Notifiche real-time per le approvazioni

    Viste di supporto per reporting e dashboard

2. Requisiti di Sistema

    PostgreSQL ≥ 13 (versione consigliata: 14+)

    Estensione pgcrypto per la generazione di UUID crittograficamente sicuri

    Estensione pg_stat_statements opzionale per telemetria e monitoring

sql

-- Estensioni richieste
CREATE EXTENSION IF NOT EXISTS pgcrypto;

3. Struttura del Database
3.1 Tabelle Core
Tabella companies (Aziende)
sql

CREATE TABLE companies (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    nome VARCHAR(255) NOT NULL,
    activation_code VARCHAR(50) UNIQUE,  -- Codice master prima attivazione
    partita_iva VARCHAR(20),
    codice_fiscale VARCHAR(20),
    country_code CHAR(2),
    indirizzo VARCHAR(255),
    citta VARCHAR(100),
    cap VARCHAR(20),
    provincia VARCHAR(50),
    telefono VARCHAR(30),
    email_amministrativa VARCHAR(255),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

Tabella users (Utenti)
sql

CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    company_id UUID NOT NULL REFERENCES companies(id) ON DELETE CASCADE,
    activation_code_id UUID,  -- Link al codice di attivazione utilizzato
    nome VARCHAR(255),
    cognome VARCHAR(255),
    ruolo VARCHAR(20) NOT NULL CHECK (ruolo IN ('admin', 'supervisore', 'operaio')),
    email VARCHAR(255),
    password_hash VARCHAR(255),
    status VARCHAR(20) NOT NULL CHECK (status IN ('attivo', 'disattivato', 'eliminato')) DEFAULT 'attivo',
    onboarded BOOLEAN NOT NULL DEFAULT FALSE,  -- Completamento primo accesso
    last_login TIMESTAMP,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    UNIQUE(email)
);

Tabella activation_codes (Codici di Attivazione)
sql

CREATE TABLE activation_codes (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    company_id UUID NOT NULL REFERENCES companies(id) ON DELETE CASCADE,
    code VARCHAR(11) NOT NULL UNIQUE,  -- Formato: ABC-123-XYZ
    role VARCHAR(20) NOT NULL CHECK (role IN ('supervisore', 'operaio')),
    worker_name VARCHAR(255),  -- Nome precompilato per operai
    created_by UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    expires_at TIMESTAMP NOT NULL,
    status VARCHAR(20) NOT NULL CHECK (status IN ('attivo', 'utilizzato', 'scaduto')) DEFAULT 'attivo',
    CONSTRAINT activation_codes_code_format_chk CHECK (code ~ '^[A-Z0-9]{3}-[A-Z0-9]{3}-[A-Z0-9]{3}$')
);

3.2 Tabelle di Lavoro
Tabella projects (Progetti/Cantieri)
sql

CREATE TABLE projects (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    company_id UUID NOT NULL REFERENCES companies(id) ON DELETE CASCADE,
    nome VARCHAR(255) NOT NULL,
    status VARCHAR(20) CHECK (status IN ('attivo', 'completato', 'archiviato')) DEFAULT 'attivo',
    fine_prevista TIMESTAMP,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    UNIQUE (company_id, nome)  -- Nome univoco per azienda
);

Tabella work_hours (Ore Lavorate)
sql

CREATE TABLE work_hours (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    company_id UUID NOT NULL REFERENCES companies(id) ON DELETE CASCADE,
    project_id UUID NOT NULL REFERENCES projects(id),
    user_id UUID NOT NULL REFERENCES users(id),
    inserito_da UUID REFERENCES users(id),  -- Utente che ha inserito il record
    data DATE NOT NULL,
    ore_totali NUMERIC(4,2) NOT NULL CHECK (ore_totali BETWEEN 0 AND 24),
    is_festivo BOOLEAN NOT NULL,  -- Precalcolato automaticamente
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    UNIQUE (project_id, user_id, data)  -- Un registro per giorno per utente
);

3.3 Sistema di Audit
Tabella audit_log (Log Partizionato)
sql

CREATE TABLE audit_log (
    id UUID DEFAULT gen_random_uuid(),
    timestamp TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    user_id UUID,
    company_id UUID,
    entity_type VARCHAR(50) NOT NULL,
    entity_id UUID NOT NULL,
    operation VARCHAR(10) NOT NULL CHECK (operation IN ('INSERT','UPDATE','DELETE')),
    old_value JSONB,
    new_value JSONB,
    PRIMARY KEY (id, timestamp)
) PARTITION BY RANGE (timestamp);

Partizionamento Automatico
sql

-- Creazione partizione per l'anno corrente
CREATE TABLE audit_log_2023 PARTITION OF audit_log
    FOR VALUES FROM ('2023-01-01') TO ('2024-01-01');

3.4 Workflow Correzioni
Tabella time_correction_requests (Richieste di Correzione)
sql

CREATE TABLE time_correction_requests (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    company_id UUID NOT NULL REFERENCES companies(id) ON DELETE CASCADE,
    user_id UUID NOT NULL REFERENCES users(id),
    project_id UUID NOT NULL REFERENCES projects(id),
    correction_date DATE NOT NULL,
    original_hours NUMERIC(4,2) NOT NULL,
    requested_hours NUMERIC(4,2) NOT NULL CHECK (requested_hours BETWEEN 0 AND 24),
    reason TEXT NOT NULL,
    status VARCHAR(20) NOT NULL CHECK (status IN ('pending', 'approved', 'rejected')) DEFAULT 'pending',
    created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP,
    reviewed_by UUID REFERENCES users(id),
    reviewed_at TIMESTAMPTZ
);

4. Funzioni e Trigger Business
4.1 Gestione Automatica Stato Fatture
sql

CREATE OR REPLACE FUNCTION update_invoice_status() RETURNS TRIGGER AS $$
BEGIN
    NEW.pagata := (NEW.importo_pagato >= NEW.importo_dovuto);
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trigger_invoice_status 
    BEFORE INSERT OR UPDATE ON invoices 
    FOR EACH ROW EXECUTE FUNCTION update_invoice_status();

4.2 Precalcolo Festivi
sql

CREATE OR REPLACE FUNCTION precompute_holiday() RETURNS TRIGGER AS $$
BEGIN
    NEW.is_festivo := (
        EXTRACT(DOW FROM NEW.data) IN (0, 6) OR  -- Domenica=0, Sabato=6
        EXISTS (
            SELECT 1 FROM holidays h 
            WHERE h.date = NEW.data AND h.company_id = NEW.company_id
        )
    );
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trigger_precompute_holiday 
    BEFORE INSERT ON work_hours 
    FOR EACH ROW EXECUTE FUNCTION precompute_holiday();

4.3 Attivazione Utenti con Codice
sql

CREATE OR REPLACE FUNCTION consume_activation_code_and_create_user(
    p_code TEXT,
    p_nome TEXT,
    p_cognome TEXT,
    p_email TEXT,
    p_password_hash TEXT
) RETURNS UUID AS $$
DECLARE
    v_code activation_codes%ROWTYPE;
    v_user_id UUID;
BEGIN
    -- Verifica e lock del codice
    SELECT * INTO v_code FROM activation_codes 
    WHERE code = p_code AND status = 'attivo' AND expires_at > NOW() 
    FOR UPDATE;
    
    -- Inserimento utente e aggiornamento codice
    -- [Implementazione completa]
    RETURN v_user_id;
END;
$$ LANGUAGE plpgsql;

5. Row Level Security (RLS)
5.1 Configurazione Base
sql

-- Abilita RLS sulle tabelle
ALTER TABLE companies ENABLE ROW LEVEL SECURITY;
ALTER TABLE users ENABLE ROW LEVEL SECURITY;
-- [Altre tabelle...]

-- Policy per isolamento aziendale
CREATE POLICY companies_isolation_policy ON companies 
    USING (id = current_setting('app.current_company_id')::UUID);

5.2 Impostazione Variabili di Sessione
sql

-- All'inizio di ogni transazione
BEGIN;
SET LOCAL app.current_company_id = 'a1b2c3d4-e5f6-7890-abcd-ef1234567890';
SET LOCAL app.current_user_id = 'z1y2x3w4-v5u6-7890-ijkl-mn1234567890';
-- Query operative...
COMMIT;

6. Viste di Supporto
6.1 Vista Saldo Fatture
sql

CREATE OR REPLACE VIEW invoices_balance AS
SELECT 
    i.*,
    (i.importo_dovuto - i.importo_pagato) AS residuo
FROM invoices i;

6.2 Vista Utenti Attivi
sql

CREATE OR REPLACE VIEW active_users_report AS
SELECT 
    company_id, 
    COUNT(DISTINCT user_id) AS active_users
FROM work_hours 
WHERE data BETWEEN CURRENT_DATE - INTERVAL '30 days' AND CURRENT_DATE 
GROUP BY company_id;

7. Indici per Performance
7.1 Indici Principali
sql

-- Indici per ricerche temporali
CREATE INDEX idx_work_hours_project_date ON work_hours(project_id, data);
CREATE INDEX idx_photos_project_taken ON photos(project_id, taken_at);

-- Indici per ottimizzazione RLS
CREATE INDEX idx_users_company ON users(company_id);
CREATE INDEX idx_projects_company_id ON projects(company_id);

7.2 Indici per Ricerche Complesse
sql

-- Indice GIN per ricerche JSONB
CREATE INDEX idx_audit_old_gin ON audit_log USING GIN(old_value);
CREATE INDEX idx_audit_new_gin ON audit_log USING GIN(new_value);

-- Indice BRIN per tabelle temporali molto grandi
CREATE INDEX idx_audit_timestamp_brin ON audit_log USING BRIN(timestamp);

8. Gestione Operativa
8.1 Pulizia Codici Scaduti
sql

-- Funzione per pulizia periodica
CREATE OR REPLACE FUNCTION clean_expired_activation_codes() RETURNS VOID AS $$
BEGIN
    UPDATE activation_codes 
    SET status = 'scaduto' 
    WHERE status = 'attivo' AND expires_at < NOW();
END;
$$ LANGUAGE plpgsql;

-- Esecuzione manuale o via cron
SELECT clean_expired_activation_codes();

8.2 Notifiche Real-time
sql

-- Trigger per notifiche correzioni
CREATE OR REPLACE FUNCTION notify_supervisors_of_correction_request() 
RETURNS TRIGGER AS $$
BEGIN
    PERFORM pg_notify(
        'correction_request_channel',
        json_build_object(
            'request_id', NEW.id,
            'user_id', NEW.user_id,
            'project_id', NEW.project_id,
            'status', NEW.status
        )::text
    );
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

9. Considerazioni di Deployment
9.1 Backup e Recovery

    Utilizzare pg_dump con formato custom per backup consistenti

    Implementare WAL archiving per Point-in-Time Recovery

    Testare regolarmente procedure di restore

9.2 Monitoraggio

    Configurare alert su spazio disco e performance

    Monitorare la crescita delle partizioni di audit

    Tracciare gli accessi falliti nella tabella failed_logins

9.3 Manutenzione

    Pianificare vacuum e analyze regolari

    Monitorare la crescita degli indici

    Aggiornare le statistiche del query planner

10. Sicurezza
10.1 Hardening Database

    Limitare gli accessi di rete al database

    Utilizzare SSL per le connessioni

    Ruotare regolarmente le credenziali di amministrazione

10.2 Audit e Compliance

    Revisionare regolarmente i log di audit

    Implementare retention policy per i dati sensibili

    Eseguire penetration test periodici
