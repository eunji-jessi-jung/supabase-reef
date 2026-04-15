---
generated: true
---
# Reef Index — supabase-reef

> Auto-generated catalog. Do not edit manually.

## Systems
- [[SYS-AUTH]] — Supabase Auth Service (draft)
- [[SYS-CLIENT-SDK]] — Supabase JS SDK (draft)
- [[SYS-REALTIME]] — Supabase Realtime Service (draft)
- [[SYS-STORAGE]] — Supabase Storage Service (draft)
- [[SYS-STUDIO]] — Supabase Studio (draft)

## Schemas
- [[SCH-AUTH]] — Auth Data Model (active)
- [[SCH-REALTIME]] — Realtime Data Model (draft)
- [[SCH-REALTIME-FIELD-LINEAGE]] — Realtime CDC Field Lineage (draft)
- [[SCH-STORAGE]] — Storage Data Model (active)
- [[SCH-STUDIO-PG-META]] — Pg-Meta Postgres Introspection Model (draft)

## APIs
- [[API-AUTH]] — Auth REST API (active)
- [[API-CLIENT-SDK]] — Supabase JS SDK Public API (draft)
- [[API-REALTIME]] — Realtime API and WebSocket Interface (draft)
- [[API-STORAGE]] — Storage Multi-Protocol API (active)
- [[API-STUDIO]] — Studio API Surface (draft)
- [[API-STUDIO-API-TYPES]] — Studio API Types Package (draft)

## Processes
- [[PROC-AUTH-AUTH]] — Auth Authentication Mechanism (draft)
- [[PROC-AUTH-CUSTOM-OAUTH-PROVIDERS-LIFECYCLE]] — Auth Custom OAuth/OIDC Providers Entity Lifecycle (draft)
- [[PROC-AUTH-ERROR-HANDLING]] — Auth Error Handling Patterns (draft)
- [[PROC-AUTH-IDENTITIES-LIFECYCLE]] — Auth Identities Entity Lifecycle (draft)
- [[PROC-AUTH-MFA-FACTORS-LIFECYCLE]] — Auth MFA Factors Entity Lifecycle (draft)
- [[PROC-AUTH-REFRESH-TOKENS-LIFECYCLE]] — Auth Refresh Tokens Entity Lifecycle (draft)
- [[PROC-AUTH-SAML-PROVIDERS-LIFECYCLE]] — Auth SAML Providers Entity Lifecycle (draft)
- [[PROC-AUTH-SESSIONS-LIFECYCLE]] — Auth Sessions Entity Lifecycle (draft)
- [[PROC-AUTH-USERS-LIFECYCLE]] — Auth Users Entity Lifecycle (draft)
- [[PROC-AUTH-WEBAUTHN-CREDENTIALS-LIFECYCLE]] — Auth WebAuthn Credentials (Passkeys) Entity Lifecycle (draft)
- [[PROC-CLIENT-SDK-AUTH]] — Client SDK Authentication and Token Lifecycle (draft)
- [[PROC-CLIENT-SDK-ERROR-HANDLING]] — Client SDK Error Handling Across Sub-Clients (draft)
- [[PROC-REALTIME-AUTH]] — Realtime Authentication and Authorization Flow (draft)
- [[PROC-REALTIME-ERROR-HANDLING]] — Realtime Error Handling and Recovery Processes (draft)
- [[PROC-STORAGE-AUTH]] — Storage Authentication and Authorization Flow (draft)
- [[PROC-STORAGE-ERROR-HANDLING]] — Storage Error Handling Across Protocols (draft)
- [[PROC-STORAGE-ICEBERG-TABLES-LIFECYCLE]] — Storage Iceberg Tables Entity Lifecycle (draft)
- [[PROC-STORAGE-OBJECTS-LIFECYCLE]] — Storage Objects Entity Lifecycle (draft)
- [[PROC-STORAGE-S3-MULTIPART-UPLOADS-LIFECYCLE]] — Storage S3 Multipart Uploads Entity Lifecycle (draft)
- [[PROC-STORAGE-S3-MULTIPART-UPLOADS-PARTS-LIFECYCLE]] — Storage S3 Multipart Upload Parts Entity Lifecycle (draft)
- [[PROC-STORAGE-VECTOR-INDEXES-LIFECYCLE]] — Storage Vector Indexes Entity Lifecycle (draft)
- [[PROC-STUDIO-AUTH]] — Studio Authentication and Authorization Mechanism (draft)
- [[PROC-STUDIO-ERROR-HANDLING]] — Studio Error Handling and Display Pipeline (draft)
- [[PROC-STUDIO-MULTI-APP-COMPARISON]] — Studio Multi-App Comparison: Studio vs Api-Types vs Pg-Meta (draft)

## Decisions
- [[DEC-AUTH-DUAL-TRANSPORT-HOOKS]] — Auth Hooks: Dual-Transport Extensibility via Postgres Functions and HTTP Webhooks (active)
- [[DEC-CLIENT-SDK-AUTH-TOKEN-INJECTION]] — Cross-Cutting Auth Token Injection via fetchWithAuth (draft)
- [[DEC-MULTI-PROTOCOL-STORAGE]] — Storage Multi-Protocol Design Decision (draft)
- [[DEC-REALTIME-GENRPC-PUBSUB]] — Realtime Uses Gen_rpc Instead of Distributed Erlang for Cross-Node Messaging (draft)
- [[DEC-STORAGE-RLS-AUTH-DELEGATION]] — Storage Delegates Authorization to Postgres Row Level Security (draft)
- [[DEC-STUDIO-BFF-PROXY]] — Studio BFF Proxy Pattern via Next.js API Routes (draft)

## Glossary
- [[GLOSSARY-AUTH]] — Auth Domain Glossary (active)
- [[GLOSSARY-CLIENT-SDK]] — Client SDK Domain Glossary (draft)
- [[GLOSSARY-REALTIME]] — Realtime Domain Glossary (draft)
- [[GLOSSARY-SOURCE-INDEX]] — Cross-Service Source Index (draft)
- [[GLOSSARY-STORAGE]] — Storage Domain Glossary (draft)
- [[GLOSSARY-STUDIO]] — Studio Domain Glossary (draft)
- [[GLOSSARY-SUPABASE-REEF]] — Supabase Platform Unified Glossary (draft)

## Contracts
- [[CON-AUTH-CLIENT-SDK]] — Auth to Client SDK Contract (draft)
- [[CON-AUTH-REALTIME]] — Auth to Realtime JWT Contract (draft)
- [[CON-AUTH-STORAGE]] — Auth to Storage JWT Contract (draft)
- [[CON-STUDIO-SERVICES]] — Studio to Backend Services Integration (draft)

## Risks
- [[RISK-AUTH-KNOWN-GAPS]] — Auth Known Risks, Gaps, and Backlog Candidates (draft)
- [[RISK-CLIENT-SDK-KNOWN-GAPS]] — Client SDK Known Risks, Gaps, and Backlog Candidates (draft)
- [[RISK-REALTIME-KNOWN-GAPS]] — Realtime Known Risks, Gaps, and Backlog Candidates (draft)
- [[RISK-STORAGE-KNOWN-GAPS]] — Storage Known Risks, Gaps, and Backlog Candidates (draft)
- [[RISK-STUDIO-KNOWN-GAPS]] — Studio Known Risks, Gaps, and Backlog Candidates (draft)

## Patterns
- [[PAT-CROSS-SERVICE-AUTH]] — Cross-Service Authentication Pattern (draft)
- [[PAT-CROSS-SERVICE-CACHE]] — Cross-Service Caching Pattern (draft)
- [[PAT-CROSS-SERVICE-DATA-FLOW]] — Cross-Service Data Flow Pattern (draft)
- [[PAT-CROSS-SERVICE-MIDDLEWARE]] — Cross-Service Middleware Pattern (draft)
- [[PAT-CROSS-SERVICE-QUEUE]] — Cross-Service Queue and Async Job Pattern (draft)
- [[PAT-CROSS-SERVICE-RETRY]] — Cross-Service Retry Pattern (draft)
- [[PAT-CROSS-SERVICE-STORAGE]] — Cross-Service Storage and File Handling Pattern (draft)
