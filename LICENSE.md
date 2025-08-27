config:
  layout: elk
  theme: neo
title: PWA Multi-Tenant Database ER Diagram — Grouped by Area & 1–1 Relations
---
erDiagram
    direction TB
    companies {
        UUID id PK ""  
        string nome  ""  
        string activation_code  ""  
        string partita_iva  ""  
        string codice_fiscale  ""  
        string country_code  ""  
        string indirizzo  ""  
        string citta  ""  
        string cap  ""  
        string provincia  ""  
        string telefono  ""  
        string email_amministrativa  ""  
        timestamp created_at  ""  
    }
    users {
        UUID id PK ""  
        UUID company_id FK ""  
        UUID activation_code_id FK ""  
        string nome  ""  
        string cognome  ""  
        string ruolo  ""  
        string email  ""  
        string username UK ""  
        string password_hash  ""  
        string status  ""  
        boolean onboarded  ""  
        timestamp last_login  ""  
        timestamp created_at  ""  
    }
    projects {
        UUID id PK ""  
        UUID company_id FK ""  
        string nome  ""  
        string status  ""  
        timestamp fine_prevista  ""  
        timestamp created_at  ""  
    }
    work_hours {
        UUID id PK ""  
        UUID company_id FK ""  
        UUID project_id FK ""  
        UUID user_id FK ""  
        UUID inserito_da FK ""  
        date data  ""  
        numeric ore_totali  ""  
        boolean is_festivo  ""  
        timestamp created_at  ""  
    }
    activation_codes {
        UUID id PK ""  
        UUID company_id FK ""  
        string code  ""  
        string role  ""  
        UUID created_by FK ""  
        timestamp created_at  ""  
        timestamp expires_at  ""  
        string status  ""  
    }
    user_tokens {
        UUID id PK ""  
        UUID user_id FK ""  
        string jwt_token_id UK ""  
        boolean valid  ""  
        string device_id UK ""  
        boolean long_term_token  ""  
        timestamp created_at  ""  
        timestamp expires_at  ""  
    }
    password_resets {
        UUID id PK ""  
        UUID user_id FK ""  
        string token_hash UK ""  
        timestamp created_at  ""  
        timestamp expires_at  ""  
        boolean used  ""  
    }
    invoices {
        UUID id PK ""  
        UUID company_id FK ""  
        UUID project_id FK ""  
        string cliente_nome  ""  
        numeric importo_dovuto  ""  
        numeric importo_pagato  ""  
        boolean pagata  ""  
        date data_scadenza  ""  
        timestamp created_at  ""  
        string status  ""  
    }
    export_history {
        UUID id PK ""  
        UUID company_id FK ""  
        UUID user_id FK ""  
        string export_type  ""  
        string entity_type  ""  
        UUID entity_id  ""  
        jsonb filter_params  ""  
        text file_url  ""  
        timestamp created_at  ""  
    }
    audit_log {
        UUID id PK ""  
        timestamp timestamp  ""  
        UUID user_id FK ""  
        UUID company_id FK ""  
        string entity_type  ""  
        UUID entity_id  ""  
        string operation  ""  
        jsonb old_value  ""  
        jsonb new_value  ""  
    }
    activity_log {
        UUID id PK ""  
        UUID user_id FK ""  
        string action_type  ""  
        string table_name  ""  
        UUID record_id  ""  
        jsonb old_value  ""  
        jsonb new_value  ""  
        timestamp timestamp  ""  
    }
    failed_logins {
        UUID id PK ""  
        string username_attempted  ""  
        UUID user_id FK ""  
        inet ip_address  ""  
        text user_agent  ""  
        timestamp timestamp  ""  
    }
    time_correction_requests {
        UUID id PK ""  
        UUID company_id FK ""  
        UUID user_id FK ""  
        UUID project_id FK ""  
        date correction_date  ""  
        numeric original_hours  ""  
        numeric requested_hours  ""  
        text reason  ""  
        string status  ""  
        timestamp created_at  ""  
        UUID reviewed_by FK ""  
        timestamp reviewed_at  ""  
    }
    time_correction_history {
        UUID id PK ""  
        UUID company_id FK ""  
        UUID request_id FK ""  
        UUID work_hours_id FK ""  
        numeric old_hours  ""  
        numeric new_hours  ""  
        UUID corrected_by FK ""  
        timestamp corrected_at  ""  
    }
    holidays {
        UUID company_id FK ""  
        date date PK ""  
        string name  ""  
    }
    companies||--o{users:"has"
    companies||--o{activation_codes:"issues"
    companies||--o{projects:"owns"
    companies||--o{work_hours:"records"
    companies||--o{invoices:"bills"
    companies||--o{export_history:"exports"
    users||--o{user_tokens:"uses"
    users|o--||password_resets:"reset"
    users||--o{work_hours:"records"
    users||--o{time_correction_requests:"requests"
    projects||--o{work_hours:"work"
    projects||--o{invoices:"bills"
    projects||--o{time_correction_requests:"corrections"
    work_hours||--o{time_correction_requests:"may correct"
    time_correction_requests||--o{time_correction_history:"histories"
    work_hours||--o{time_correction_history:"histories"
    users||--o{audit_log:"has audits"
    users||--o{activity_log:"events"
    users||--o{failed_logins:"failed"
    time_correction_requests||--o{audit_log:"audited"
    export_history||--o{audit_log:"audited"
    holidays||--o{work_hours:"may influence"
    activation_codes|o--||users:"used by (if present)"
    companies:::core
    users:::core
    projects:::core
    work_hours:::core
    activation_codes:::sicurezza
    user_tokens:::sicurezza
    password_resets:::sicurezza
    invoices:::finanza
    export_history:::finanza
    audit_log:::audit
    activity_log:::audit
    failed_logins:::audit
    time_correction_requests:::correzioni
    time_correction_history:::correzioni
