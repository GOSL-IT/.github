# PhysDev GOSLNet

## Projects

| Repo | Technologies | Role |
|---|---|---|
| `goslnet_backend` | Python, PostgreSQL (async psycopg), Pydantic, JWT, LDAP | Shared standard library: PostgreSQL models/tables, secrets, JWT, session auth, LDAP manager. Vendored into most other repos. |
| `goslnet_ldap` | Quart, Pydantic, JWT, PostgreSQL, Active Directory, nginx, Docker | SSO login microservice (`lrauthservice.goslnet.gov.lc`). One AD login → shared access/refresh JWT cookies reused by every downstream app. |
| `goslnet_ailab_server` | Quart, Ollama, Whisper, WebSockets, Pydantic, Docker | The "AI Lab": session-managed LLM server for Whisper transcription, OCR, image recognition, and chat (Ollama models). |
| `goslnet_ailab_api` | httpx, Pydantic, Ollama | Client SDK for the AI Lab; vendored as `lab_api`/`lap_api` into the apps that consume it. |
| `goslnet_document_retreival` | Quart, httpx, PostgreSQL, Ollama (via lab_api), nginx, Docker | Land Registry document tracker: intake, assignment, review, notary circulation, search, admin. Includes a background scan service (network folder → OCR/LLM → `LandRegisters` table). Three processes: app, scan, verification. |
| `goslnet_transcription_service` | Quart, Pydantic, JWT, SMTP, Whisper (via AI Lab), nginx, Docker | Audio drop-off web app: SSO login → queued Whisper transcription via the AI Lab → transcripts/subtitles emailed to the user's AD email. |
| `goslnet_email_service` | Python, smtplib (SMTP), Pydantic, LDAP (via backend) | Internal SMTP email helper built on the backend library's LDAP manager. |
| `goslnet_ailab_testsuite` | Pydantic, Ollama, mypy, black | Test suite measuring LLM-OCR accuracy on land-register scans (95% overall field accuracy). |
| `goslnet_style_guide` | Markdown | Org-wide coding standards. |

### Runtime topology

Solid arrows are runtime calls (HTTP / SSO / DB).

```mermaid
flowchart LR
    subgraph apps["User-facing apps"]
        DOC["goslnet_document_retreival<br>Land Registry tracker"]
        TRANS["goslnet_transcription_service<br>Audio transcription"]
        EMAIL["goslnet_email_service<br>Internal email"]
    end

    subgraph core["Core services"]
        LDAP["goslnet_ldap<br>SSO auth"]
        AILAB["goslnet_ailab_server<br>AI Lab"]
    end

    subgraph ext["External systems"]
        AD[("Active Directory")]
        PG[("PostgreSQL")]
        SMTP["SMTP relay"]
        LLM["Ollama LLMs"]
    end

    DOC -->|SSO login| LDAP
    DOC -->|OCR + LLM| AILAB
    DOC -->|land data| PG
    TRANS -->|verify SSO JWT| LDAP
    TRANS -->|transcribe| AILAB
    TRANS -->|results email| SMTP
    EMAIL -->|send mail| SMTP
    LDAP -->|LDAP bind| AD
    LDAP -->|sessions| PG
    AILAB -->|models| LLM
```

### Shared code (vendoring)

Dashed arrows are code included as git submodules.

```mermaid
flowchart LR
    subgraph shared["Shared code"]
        BACKEND["goslnet_backend<br>backend library"]
        AILABAPI["goslnet_ailab_api<br>AI Lab SDK"]
        AILAB["goslnet_ailab_server<br>AI Lab server"]
    end

    subgraph consumers["Consumers"]
        LDAP["goslnet_ldap"]
        DOC["goslnet_document_retreival"]
        TRANS["goslnet_transcription_service"]
        EMAIL["goslnet_email_service"]
        TEST["goslnet_ailab_testsuite"]
    end

    BACKEND -.->|vendored| DOC
    BACKEND -.->|vendored| TRANS
    BACKEND -.->|vendored| EMAIL
    BACKEND -.->|vendored| TEST
    BACKEND -.->|README only| LDAP

    AILABAPI -.->|lab_api| DOC
    AILABAPI -.->|lap_api| TRANS
    AILABAPI -.->|lab_api| TEST

    AILAB -.->|vendored| TEST
    TEST -.->|evaluates OCR pipeline of| DOC
```
