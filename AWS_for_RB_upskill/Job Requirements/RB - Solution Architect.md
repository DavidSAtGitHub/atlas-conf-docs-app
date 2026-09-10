# Analýza role: Solution Architect pro digitální bankovní kanály

## 1. Dekompozice požadavků z popisu role

Z inzerátu lze identifikovat tyto klíčové oblasti odpovědnosti:

|Oblast|Konkrétní očekávání|
|---|---|
|**Architektura řešení**|Návrh pro mobilní a internetové bankovnictví, dopad na miliony uživatelů|
|**Technická dokumentace**|Příprava zadání a specifikací pro vývojáře|
|**Komunikace**|Spolupráce napříč týmy (dev, PO, experti), vedení technických diskusí|
|**Modelování**|Systémy a procesy se zaměřením na bezpečnost, škálovatelnost, spolehlivost|
|**Technologie**|AWS cloud, integrační vzory, generativní AI|
|**NFR**|Výkon, dostupnost, ochrana dat|

---

## 2. Doménové znalosti: Digitální bankovnictví

### 2.1 Architektura digitálních kanálů

```
┌─────────────────────────────────────────────────────────────────┐
│                     DIGITÁLNÍ KANÁLY                            │
├─────────────────┬─────────────────┬─────────────────────────────┤
│  Mobile App     │  Internet       │  Další kanály               │
│  (iOS/Android)  │  Banking (Web)  │  (chatbot, voice, wearables)│
└────────┬────────┴────────┬────────┴──────────────┬──────────────┘
         │                 │                       │
         ▼                 ▼                       ▼
┌─────────────────────────────────────────────────────────────────┐
│                 API Gateway / BFF Layer                         │
│   (Backend for Frontend - dedikované API pro každý kanál)       │
└─────────────────────────────┬───────────────────────────────────┘
                              │
┌─────────────────────────────▼───────────────────────────────────┐
│              Orchestrační / Kompozitní vrstva                   │
│         (agregace služeb, workflow, transformace)               │
└─────────────────────────────┬───────────────────────────────────┘
                              │
┌─────────────────────────────▼───────────────────────────────────┐
│                    Domain Services                              │
│  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐   │
│  │Accounts │ │Payments │ │ Cards   │ │ Loans   │ │Products │   │
│  └─────────┘ └─────────┘ └─────────┘ └─────────┘ └─────────┘   │
└─────────────────────────────┬───────────────────────────────────┘
                              │
┌─────────────────────────────▼───────────────────────────────────┐
│                    Core Banking System                          │
│              (legacy mainframe / moderní CBS)                   │
└─────────────────────────────────────────────────────────────────┘
```

### 2.2 Klíčové funkcionality mobilního/internetového bankovnictví

**Základní služby (must-have):**

- Přehled účtů a zůstatků (real-time vs. batch)
- Historie transakcí s vyhledáváním a filtrováním
- Tuzemské a zahraniční platby (SEPA, SWIFT)
- Trvalé příkazy a inkasa
- Správa karet (limity, blokace, PIN, aktivace)
- Správa uživatelského profilu a kontaktů

**Rozšířené služby:**

- Okamžité platby (Instant Payments / TIPS)
- P2P platby a QR platby
- Správa úvěrů a hypoték
- Investiční produkty
- Pojištění
- PFM (Personal Finance Management) - kategorizace, rozpočty, analýzy
- Notifikace a alerty (push, SMS, email)
- Biometrické přihlášení (Face ID, Touch ID, behavioral biometrics)

**Inovativní funkce:**

- Chatbot / virtuální asistent (zde vstupuje Gen AI)
- Hlasové ovládání
- Prediktivní analýzy a doporučení
- Open Banking / PSD2 agregace účtů
- Marketplace třetích stran

---

## 3. Technické znalosti pro Solution Architekta

### 3.1 AWS Cloud služby relevantní pro bankovnictví

|Kategorie|Služby|Použití v digital banking|
|---|---|---|
|**Compute**|ECS, EKS, Lambda, Fargate|Kontejnerizované microservices, serverless funkce|
|**API Management**|API Gateway, AppSync|REST/GraphQL API, rate limiting, throttling|
|**Messaging**|SQS, SNS, EventBridge, MSK (Kafka)|Asynchronní zpracování, event-driven architektura|
|**Data**|RDS, Aurora, DynamoDB, ElastiCache|Transakční data, session cache, real-time data|
|**Security**|KMS, Secrets Manager, WAF, Shield, Cognito|Šifrování, správa secrets, ochrana před DDoS|
|**Networking**|VPC, PrivateLink, Transit Gateway, Direct Connect|Izolace, hybridní konektivita s on-prem|
|**Observability**|CloudWatch, X-Ray, CloudTrail|Monitoring, tracing, audit logging|
|**AI/ML**|Bedrock, SageMaker, Comprehend, Lex|Gen AI asistenti, fraud detection, NLP|

### 3.2 Architektonické principy a patterny

**Fundamentální principy:**

- **Defense in Depth** - vícevrstevná bezpečnost
- **Zero Trust Architecture** - nikdy nedůvěřuj, vždy ověřuj
- **Separation of Concerns** - oddělení odpovědností
- **Loose Coupling** - volné vazby mezi komponentami
- **High Cohesion** - související funkce pohromadě
- **Idempotency** - kritické pro platební operace
- **Eventually Consistent vs. Strong Consistency** - trade-off

**Klíčové architektonické patterny:**

```
┌─────────────────────────────────────────────────────────────┐
│                    INTEGRAČNÍ PATTERNY                      │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  1. API Gateway Pattern                                     │
│     ┌──────┐     ┌─────────┐     ┌──────────┐              │
│     │Client│────▶│ Gateway │────▶│ Services │              │
│     └──────┘     └─────────┘     └──────────┘              │
│     - Rate limiting, throttling                             │
│     - Authentication/Authorization                          │
│     - Request/Response transformation                       │
│     - Caching                                               │
│                                                             │
│  2. Backend for Frontend (BFF)                              │
│     ┌────────┐     ┌─────────┐                             │
│     │Mobile  │────▶│Mobile BFF│──┐                         │
│     └────────┘     └─────────┘  │   ┌──────────┐           │
│     ┌────────┐     ┌─────────┐  ├──▶│ Backend  │           │
│     │  Web   │────▶│ Web BFF │──┘   │ Services │           │
│     └────────┘     └─────────┘      └──────────┘           │
│     - Optimalizace pro konkrétní kanál                      │
│     - Agregace dat                                          │
│                                                             │
│  3. Saga Pattern (distribuované transakce)                  │
│     ┌─────┐    ┌─────┐    ┌─────┐    ┌─────┐              │
│     │Step1│───▶│Step2│───▶│Step3│───▶│Step4│              │
│     └──┬──┘    └──┬──┘    └──┬──┘    └─────┘              │
│        │         │         │                               │
│        ▼         ▼         ▼                               │
│     Compensate Compensate Compensate (při selhání)         │
│     - Choreography vs. Orchestration                        │
│                                                             │
│  4. Event Sourcing + CQRS                                   │
│     ┌────────┐        ┌─────────────┐                      │
│     │Commands│───────▶│Event Store  │                      │
│     └────────┘        └──────┬──────┘                      │
│                              │                              │
│     ┌────────┐        ┌──────▼──────┐                      │
│     │Queries │◀───────│Read Models  │                      │
│     └────────┘        └─────────────┘                      │
│     - Audit trail, replay schopnost                         │
│                                                             │
│  5. Circuit Breaker                                         │
│     - Ochrana před kaskádovým selháním                      │
│     - States: Closed → Open → Half-Open                     │
│                                                             │
│  6. Bulkhead Pattern                                        │
│     - Izolace selhání, thread pool isolation                │
│                                                             │
│  7. Strangler Fig Pattern                                   │
│     - Postupná migrace z legacy systémů                     │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 3.3 Integrační vzory pro bankovnictví

**Synchronní integrace:**

- REST API (JSON over HTTPS)
- gRPC (vysoký výkon, strongly typed)
- GraphQL (flexibilní dotazování, vhodné pro BFF)

**Asynchronní integrace:**

- Message Queues (SQS, RabbitMQ) - point-to-point
- Publish/Subscribe (SNS, Kafka) - event distribution
- Event-driven architecture

**Specifické bankovní integrace:**

- **SWIFT** - mezinárodní platby (MT/MX formáty, ISO 20022 migrace)
- **SEPA** - evropské platby (SCT, SDD, SCT Inst)
- **Okamžité platby** - TIPS, národní schémata
- **Karetní sítě** - Visa, Mastercard (ISO 8583)
- **Core Banking System** - integrace s hlavním bankovním systémem
- **AML/KYC systémy** - compliance integrace
- **Credit bureaus** - úvěrové registry

**Anti-Corruption Layer (ACL):**

```
┌────────────┐     ┌─────────────────┐     ┌──────────────┐
│ Modern     │     │ Anti-Corruption │     │ Legacy       │
│ Services   │────▶│ Layer           │────▶│ Core Banking │
└────────────┘     │ - Translation   │     │ (Mainframe)  │
                   │ - Mapping       │     └──────────────┘
                   │ - Protocol conv.│
                   └─────────────────┘
```

---

## 4. Bezpečnost v digitálním bankovnictví

### 4.1 Autentizace a autorizace

```
┌─────────────────────────────────────────────────────────────┐
│              AUTENTIZAČNÍ MECHANISMY                        │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Multi-Factor Authentication (MFA):                         │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐         │
│  │ Something   │  │ Something   │  │ Something   │         │
│  │ you KNOW    │  │ you HAVE    │  │ you ARE     │         │
│  │ (password)  │  │ (phone/token)│  │ (biometric) │         │
│  └─────────────┘  └─────────────┘  └─────────────┘         │
│                                                             │
│  Typy autentizace:                                          │
│  - Username/Password + OTP (SMS/TOTP/Push)                  │
│  - Biometrie (Face ID, Touch ID, voice)                     │
│  - Behavioral biometrics (typing pattern, device usage)     │
│  - Device binding (device fingerprinting)                   │
│  - PKI / certifikáty (pro korporátní klienty)              │
│                                                             │
│  Session Management:                                         │
│  - JWT tokens (access + refresh tokens)                     │
│  - Token rotation                                           │
│  - Session timeout policies                                 │
│  - Concurrent session limits                                │
│                                                             │
│  Autorizace:                                                │
│  - RBAC (Role-Based Access Control)                         │
│  - ABAC (Attribute-Based Access Control)                    │
│  - OAuth 2.0 / OpenID Connect                              │
│  - Transaction signing (pro high-value operace)             │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 4.2 Strong Customer Authentication (SCA) - PSD2

Regulatorní požadavek pro EU - nutné pro:

- Přihlášení k účtu
- Elektronické platby
- Akce s rizikem podvodu

Výjimky z SCA:

- Low-value transactions (< 30 EUR, max 5x nebo 100 EUR kumulativně)
- Recurring payments (po prvním SCA)
- Trusted beneficiaries
- Transaction Risk Analysis (TRA) exemption

### 4.3 Bezpečnostní vrstvy

|Vrstva|Mechanismy|
|---|---|
|**Network**|WAF, DDoS protection (Shield), VPC isolation, PrivateLink|
|**Transport**|TLS 1.3, certificate pinning, mutual TLS|
|**Application**|Input validation, output encoding, OWASP Top 10 mitigace|
|**Data**|Encryption at rest (KMS), encryption in transit, tokenization, masking|
|**Identity**|IAM, MFA, session management, fraud detection|
|**Audit**|Logging (CloudTrail), SIEM integration, tamper-proof audit trails|

### 4.4 Fraud Detection a Prevention

```
┌─────────────────────────────────────────────────────────────┐
│                FRAUD DETECTION PIPELINE                     │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌──────────┐     ┌──────────┐     ┌──────────┐            │
│  │Transaction│────▶│ Rules    │────▶│ ML Model │           │
│  │  Event   │     │ Engine   │     │ Scoring  │            │
│  └──────────┘     └──────────┘     └────┬─────┘            │
│                                         │                   │
│                   ┌─────────────────────▼─────────────────┐ │
│                   │         Decision Engine               │ │
│                   │  ┌─────────┐ ┌─────────┐ ┌─────────┐ │ │
│                   │  │ APPROVE │ │ REVIEW  │ │ DECLINE │ │ │
│                   │  └─────────┘ └─────────┘ └─────────┘ │ │
│                   └───────────────────────────────────────┘ │
│                                                             │
│  Signály pro detekci:                                       │
│  - Device anomalies (nové zařízení, root/jailbreak)        │
│  - Behavioral anomalies (unusual time, location, amount)    │
│  - Velocity checks (frequency of transactions)              │
│  - Network analysis (IP reputation, proxy/VPN detection)    │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## 5. Nefunkční požadavky (NFR)

### 5.1 Performance

|Metrika|Typický target|Poznámka|
|---|---|---|
|**API Latency (P99)**|< 500ms|Pro kritické endpointy < 200ms|
|**Throughput**|10,000+ TPS|Peak load během salary days|
|**Page Load Time**|< 3s|First Contentful Paint < 1.5s|
|**Time to Interactive**|< 5s|Mobile app startup|

**Optimalizační techniky:**

- Caching (Redis/ElastiCache, CDN)
- Connection pooling
- Async processing pro non-critical paths
- Database query optimization
- Lazy loading, pagination

### 5.2 Availability a Reliability

```
┌─────────────────────────────────────────────────────────────┐
│                AVAILABILITY TARGETS                         │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  99.9% (three nines)  = 8.76 hours downtime/year           │
│  99.95%               = 4.38 hours downtime/year           │
│  99.99% (four nines)  = 52.6 minutes downtime/year         │
│                                                             │
│  Bankovnictví typicky: 99.9% - 99.95%                      │
│  Kritické služby (platby): 99.99%                          │
│                                                             │
│  Strategie:                                                 │
│  - Multi-AZ deployment                                      │
│  - Active-Active nebo Active-Passive DR                     │
│  - Database replication (sync/async)                        │
│  - Auto-scaling                                             │
│  - Health checks a self-healing                             │
│  - Graceful degradation                                     │
│  - Feature flags pro rychlý rollback                        │
│                                                             │
│  RPO/RTO:                                                   │
│  - RPO (Recovery Point Objective): 0-15 min pro kritické   │
│  - RTO (Recovery Time Objective): < 1 hour                 │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 5.3 Scalability

**Horizontální škálování:**

- Stateless services (session externalizace)
- Container orchestration (EKS/ECS)
- Auto Scaling Groups
- Database read replicas

**Vertikální škálování:**

- Instance sizing pro peak loads
- Reserved capacity pro kritické služby

### 5.4 Observability

```
Three Pillars:
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│   METRICS   │     │   LOGS      │     │   TRACES    │
│ (CloudWatch)│     │(CloudWatch  │     │  (X-Ray)    │
│             │     │ Logs)       │     │             │
└─────────────┘     └─────────────┘     └─────────────┘

Klíčové metriky:
- Business: Transaction volume, success rate, conversion
- Technical: Latency, error rate, throughput
- Infrastructure: CPU, memory, network, disk
- Security: Failed auth attempts, anomaly scores
```

---

## 6. Regulatorní a compliance aspekty

### 6.1 Klíčové regulace

|Regulace|Oblast|Dopad na architekturu|
|---|---|---|
|**PSD2/PSD3**|Open Banking, SCA|API pro TPP, autentizace, consent management|
|**GDPR**|Ochrana osobních údajů|Data minimization, right to erasure, encryption|
|**DORA**|Digitální operační odolnost|ICT risk management, incident reporting, testing|
|**NIS2**|Kybernetická bezpečnost|Security measures, reporting|
|**AML/KYC**|Anti-money laundering|Transaction monitoring, customer due diligence|
|**Basel III/IV**|Kapitálová přiměřenost|Risk calculation, reporting|

### 6.2 Open Banking (PSD2)

```
┌─────────────────────────────────────────────────────────────┐
│                    OPEN BANKING ARCHITECTURE                │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌─────────┐                              ┌─────────┐      │
│  │  AISP   │◀────── Account Info ────────▶│         │      │
│  │(Agregátor)│                            │         │      │
│  └─────────┘                              │  ASPSP  │      │
│                                           │ (Banka) │      │
│  ┌─────────┐                              │         │      │
│  │  PISP   │◀─── Payment Initiation ─────▶│         │      │
│  │(Platební)│                             └─────────┘      │
│  └─────────┘                                               │
│                                                             │
│  Komponenty:                                                │
│  - Consent Management (správa souhlasů)                    │
│  - TPP Registry (registr třetích stran)                    │
│  - API Gateway s OAuth 2.0 / OIDC                         │
│  - Dedicated Interface (API) vs. Fallback (screen scraping)│
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## 7. Generativní AI v bankovnictví

### 7.1 Use cases

|Use Case|Popis|Architektonické úvahy|
|---|---|---|
|**Virtual Assistant**|Chatbot pro customer support|RAG, conversation history, guardrails|
|**Document Processing**|Extrakce dat z dokumentů|OCR + LLM, validation|
|**Fraud Narrative**|Generování popisů pro vyšetřování|Audit trail, explainability|
|**Code Generation**|Asistence vývojářům|Security review, sandboxing|
|**Personalization**|Produktová doporučení|Privacy, bias mitigation|

### 7.2 Architektura pro Gen AI

```
┌─────────────────────────────────────────────────────────────┐
│              GEN AI ARCHITECTURE (AWS Bedrock)              │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌────────┐     ┌──────────────┐     ┌──────────────┐      │
│  │ User   │────▶│ Orchestrator │────▶│   Bedrock    │      │
│  │ Query  │     │ (Lambda)     │     │   (Claude)   │      │
│  └────────┘     └──────┬───────┘     └──────────────┘      │
│                        │                                    │
│                 ┌──────▼───────┐                           │
│                 │ RAG Pipeline │                           │
│                 │ - Embedding  │                           │
│                 │ - Vector DB  │◀── Knowledge Base         │
│                 │ - Retrieval  │    (produkty, FAQ, docs)  │
│                 └──────────────┘                           │
│                                                             │
│  Guardrails:                                                │
│  - PII detection/redaction                                  │
│  - Topic filtering (no financial advice)                    │
│  - Hallucination detection                                  │
│  - Content moderation                                       │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## 8. Metodiky a procesy

### 8.1 Architektonická dokumentace

| Artefakt                                | Účel                       | Nástroje             |
| --------------------------------------- | -------------------------- | -------------------- |
| **Context Diagram (C4 L1)**             | Systém v kontextu          | Structurizr, draw.io |
| **Container Diagram (C4 L2)**           | Hlavní komponenty          | Structurizr          |
| **Component Diagram (C4 L3)**           | Detailní struktura         | Structurizr          |
| **Sequence Diagrams**                   | Interakce komponent        | PlantUML, Mermaid    |
| **Architecture Decision Records (ADR)** | Rozhodnutí a jejich důvody | Markdown             |
| **Technical Specifications**            | Detailní zadání pro dev    | Confluence           |

### 8.2 Agilní vývoj v enterprise kontextu

```
┌─────────────────────────────────────────────────────────────┐
│              SCALED AGILE (SAFe / LeSS)                     │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Portfolio Level:                                           │
│  - Strategic themes                                         │
│  - Epic management                                          │
│  - Architecture runway                                      │
│                                                             │
│  Program Level (ART):                                       │
│  - PI Planning                                              │
│  - System demos                                             │
│  - Solution architect role                                  │
│                                                             │
│  Team Level:                                                │
│  - Sprint planning                                          │
│  - Daily standups                                           │
│  - Retrospectives                                           │
│                                                             │
│  Solution Architect odpovědnosti:                           │
│  - Architectural runway (předstih před delivery)            │
│  - Technical enablers                                       │
│  - Non-functional requirements                              │
│  - Cross-team coordination                                  │
│  - Technical debt management                                │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## 9. Typické výzvy a jejich řešení

|Výzva|Řešení|
|---|---|
|**Legacy integrace**|Anti-Corruption Layer, Strangler Fig pattern, API wrapping|
|**Real-time vs. batch**|Event-driven architecture, CDC (Change Data Capture), Kafka|
|**Vysoká dostupnost**|Multi-AZ, circuit breakers, graceful degradation, chaos engineering|
|**Konzistence dat**|Saga pattern, eventual consistency s kompenzačními transakcemi|
|**Vendor lock-in**|Abstrakční vrstvy, multi-cloud strategie, open standards|
|**Regulatory changes**|Feature flags, configurable business rules, modular architecture|
|**Peak load handling**|Auto-scaling, queue-based load leveling, caching|
|**Security threats**|Defense in depth, zero trust, continuous security testing|

---

## 10. Checklist pro přípravu na pohovor

### Technické znalosti

- [ ] AWS služby (zejména ECS/EKS, Lambda, API Gateway, RDS/Aurora, SQS/SNS/Kafka)
- [ ] REST API design, OpenAPI/Swagger
- [ ] Microservices patterns (Saga, CQRS, Event Sourcing)
- [ ] Containerization (Docker, Kubernetes basics)
- [ ] CI/CD pipelines
- [ ] Infrastructure as Code (Terraform/CloudFormation)

### Architektonické dovednosti

- [ ] C4 model a architektonická dokumentace
- [ ] ADR (Architecture Decision Records)
- [ ] Trade-off analysis
- [ ] NFR definice a měření

### Doménové znalosti

- [ ] Základy bankovních produktů (účty, platby, karty)
- [ ] PSD2/Open Banking principy
- [ ] GDPR a data protection
- [ ] AML/KYC basics

### Soft skills

- [ ] Komunikace technických konceptů non-technickým stakeholderům
- [ ] Vedení technických diskusí
- [ ] Spolupráce s product ownery
- [ ] Mentoring vývojářů

---

Chceš, abych některou z těchto oblastí rozvedl detailněji, nebo připravil konkrétní příklady otázek a odpovědí na pohovor?

# API Gateway v kontextu digitálního bankovnictví

## 1. Role API Gateway v architektuře

API Gateway je centrální vstupní bod pro všechny klientské požadavky. V bankovním kontextu je to kritická komponenta, protože:

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         KLIENTI                                         │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐               │
│  │Mobile App│  │   Web    │  │   TPP    │  │ Partner  │               │
│  │(iOS/And) │  │ Browser  │  │(PSD2 API)│  │  Systems │               │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘  └────┬─────┘               │
│       │             │             │             │                      │
└───────┼─────────────┼─────────────┼─────────────┼──────────────────────┘
        │             │             │             │
        ▼             ▼             ▼             ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                        API GATEWAY                                      │
│  ┌────────────────────────────────────────────────────────────────┐    │
│  │  Cross-cutting concerns:                                        │    │
│  │  • Authentication & Authorization                               │    │
│  │  • Rate Limiting & Throttling                                   │    │
│  │  • Request/Response Transformation                              │    │
│  │  • Caching                                                      │    │
│  │  • Logging & Monitoring                                         │    │
│  │  • SSL Termination                                              │    │
│  │  • API Versioning                                               │    │
│  │  • Request Validation                                           │    │
│  └────────────────────────────────────────────────────────────────┘    │
└───────────────────────────────┬─────────────────────────────────────────┘
                                │
        ┌───────────────────────┼───────────────────────┐
        ▼                       ▼                       ▼
┌──────────────┐       ┌──────────────┐       ┌──────────────┐
│   Accounts   │       │   Payments   │       │    Cards     │
│   Service    │       │   Service    │       │   Service    │
└──────────────┘       └──────────────┘       └──────────────┘
```

---

## 2. Klíčové funkce API Gateway pro bankovnictví

### 2.1 Security Functions (kritické pro banky)

| Funkce              | Popis                       | Bankovní kontext                     |
| ------------------- | --------------------------- | ------------------------------------ |
| **Authentication**  | Ověření identity volajícího | OAuth 2.0, JWT validace, API keys    |
| **Authorization**   | Kontrola oprávnění          | Scopes, RBAC, policy enforcement     |
| **Rate Limiting**   | Omezení počtu požadavků     | Ochrana před DDoS, fair usage        |
| **Throttling**      | Zpomalení při přetížení     | Graceful degradation                 |
| **IP Whitelisting** | Povolené IP adresy          | Pro TPP (PSD2), partnery             |
| **mTLS**            | Vzájemná TLS autentizace    | B2B integrace, PSD2 QWAC certifikáty |
| **WAF Integration** | Web Application Firewall    | OWASP Top 10, SQL injection, XSS     |

### 2.2 Traffic Management

```
┌─────────────────────────────────────────────────────────────┐
│                   RATE LIMITING STRATEGIE                   │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Per-API Key limits:                                        │
│  ┌─────────────────────────────────────────────────────┐   │
│  │ TPP Partner A:     1000 req/min (PSD2 API)          │   │
│  │ Mobile App:        5000 req/min (interní)           │   │
│  │ Premium Partner:   10000 req/min                    │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  Per-User limits (burst protection):                        │
│  ┌─────────────────────────────────────────────────────┐   │
│  │ Login attempts:    5/min per user                   │   │
│  │ Payment init:      20/min per user                  │   │
│  │ Balance check:     60/min per user                  │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  Token bucket vs. Sliding window algoritmy                  │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 2.3 Request/Response Processing

```
Request Flow:
┌────────┐     ┌─────────────────────────────────────────┐     ┌─────────┐
│ Client │────▶│              API Gateway                │────▶│ Backend │
└────────┘     │                                         │     └─────────┘
               │  1. SSL Termination                     │
               │  2. Request Validation (schema)         │
               │  3. Authentication (JWT verify)         │
               │  4. Authorization (scope check)         │
               │  5. Rate Limit Check                    │
               │  6. Request Transformation              │
               │  7. Caching Check                       │
               │  8. Routing                             │
               └─────────────────────────────────────────┘

Response Flow:
               ┌─────────────────────────────────────────┐
               │  1. Response Transformation             │
               │  2. Error Mapping (standardizace)       │
               │  3. Response Caching                    │
               │  4. Logging & Metrics                   │
               │  5. Compression                         │
               └─────────────────────────────────────────┘
```

---

## 3. AWS API Gateway: REST API vs HTTP API

AWS nabízí dva hlavní typy API Gateway. Toto je klíčová znalost pro Solution Architekta:

### 3.1 Přehled rozdílů

|Aspekt|REST API|HTTP API|
|---|---|---|
|**Cena**|~$3.50 / milion requests|~$1.00 / milion requests (70% levnější)|
|**Latence**|Vyšší (~30-50ms overhead)|Nižší (~10ms overhead)|
|**Protokol**|REST (HTTP/1.1)|HTTP/1.1, HTTP/2, WebSocket|
|**Komplexita**|Plně vybavený, více funkcí|Jednodušší, rychlejší|
|**Ideální pro**|Enterprise, komplexní požadavky|High-volume, low-latency|

### 3.2 Detailní srovnání funkcí

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    FEATURE COMPARISON                                   │
├─────────────────────────────┬───────────────────┬───────────────────────┤
│         Feature             │     REST API      │      HTTP API         │
├─────────────────────────────┼───────────────────┼───────────────────────┤
│ Request Validation          │        ✅         │         ❌            │
│ (JSON Schema)               │                   │                       │
├─────────────────────────────┼───────────────────┼───────────────────────┤
│ Request/Response Transform  │        ✅         │         ❌            │
│ (VTL Templates)             │   (plná podpora)  │  (jen parameter map)  │
├─────────────────────────────┼───────────────────┼───────────────────────┤
│ Caching                     │        ✅         │         ❌            │
│                             │  (built-in cache) │                       │
├─────────────────────────────┼───────────────────┼───────────────────────┤
│ API Keys & Usage Plans      │        ✅         │         ❌            │
│                             │   (rate limiting  │  (jen throttling)     │
│                             │    per API key)   │                       │
├─────────────────────────────┼───────────────────┼───────────────────────┤
│ Resource Policies           │        ✅         │         ❌            │
│ (IAM-based access)          │                   │                       │
├─────────────────────────────┼───────────────────┼───────────────────────┤
│ WAF Integration             │        ✅         │         ❌            │
├─────────────────────────────┼───────────────────┼───────────────────────┤
│ Private API (VPC only)      │        ✅         │         ✅            │
├─────────────────────────────┼───────────────────┼───────────────────────┤
│ Custom Domain               │        ✅         │         ✅            │
├─────────────────────────────┼───────────────────┼───────────────────────┤
│ Lambda Authorizer           │        ✅         │         ✅            │
├─────────────────────────────┼───────────────────┼───────────────────────┤
│ JWT Authorizer (native)     │        ❌         │         ✅            │
│                             │ (needs Lambda)    │    (built-in)         │
├─────────────────────────────┼───────────────────┼───────────────────────┤
│ Cognito Integration         │        ✅         │         ✅            │
├─────────────────────────────┼───────────────────┼───────────────────────┤
│ OpenAPI Import              │        ✅         │         ✅            │
├─────────────────────────────┼───────────────────┼───────────────────────┤
│ X-Ray Tracing               │        ✅         │         ✅            │
├─────────────────────────────┼───────────────────┼───────────────────────┤
│ mTLS (Client Certificates)  │        ✅         │         ✅            │
├─────────────────────────────┼───────────────────┼───────────────────────┤
│ Canary Deployments          │        ✅         │         ❌            │
├─────────────────────────────┼───────────────────┼───────────────────────┤
│ Stage Variables             │        ✅         │         ✅            │
├─────────────────────────────┼───────────────────┼───────────────────────┤
│ Access Logging              │        ✅         │         ✅            │
├─────────────────────────────┼───────────────────┼───────────────────────┤
│ AWS Service Integration     │        ✅         │         ✅            │
│ (direct to SQS, DynamoDB)   │   (více služeb)   │   (omezený výběr)     │
└─────────────────────────────┴───────────────────┴───────────────────────┘
```

### 3.3 Rozhodovací strom: Kdy použít který typ

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    ROZHODOVACÍ STROM                                    │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  Potřebuješ Request Validation (JSON Schema)?                          │
│  │                                                                      │
│  ├── ANO ──────────────────────────────────▶ REST API                  │
│  │                                                                      │
│  └── NE                                                                 │
│      │                                                                  │
│      Potřebuješ API Caching na úrovni Gateway?                         │
│      │                                                                  │
│      ├── ANO ──────────────────────────────▶ REST API                  │
│      │                                                                  │
│      └── NE                                                             │
│          │                                                              │
│          Potřebuješ WAF (Web Application Firewall)?                    │
│          │                                                              │
│          ├── ANO ──────────────────────────▶ REST API                  │
│          │                                                              │
│          └── NE                                                         │
│              │                                                          │
│              Potřebuješ Usage Plans & API Keys pro partnery?           │
│              │                                                          │
│              ├── ANO ──────────────────────▶ REST API                  │
│              │                                                          │
│              └── NE                                                     │
│                  │                                                      │
│                  Potřebuješ VTL transformace request/response?         │
│                  │                                                      │
│                  ├── ANO ──────────────────▶ REST API                  │
│                  │                                                      │
│                  └── NE                                                 │
│                      │                                                  │
│                      Je prioritou nízká latence a cena?                │
│                      │                                                  │
│                      ├── ANO ──────────────▶ HTTP API                  │
│                      │                                                  │
│                      └── NE ───────────────▶ REST API (default choice) │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 4. Praktická doporučení pro bankovnictví

### 4.1 Kdy použít REST API (typické pro banky)

**PSD2 / Open Banking API:**

```
Důvody pro REST API:
├── WAF integrace (povinná ochrana pro externí API)
├── Request validation (validace dle Berlin Group / UK Open Banking spec)
├── Usage Plans (rate limiting per TPP)
├── API Keys (identifikace TPP partnerů)
├── Resource Policies (IP whitelisting pro certifikované TPP)
└── Canary deployments (postupné rollout změn)
```

**Partner/B2B API:**

```
Důvody pro REST API:
├── mTLS (vzájemná autentizace certifikáty)
├── Usage Plans (SLA per partner)
├── Request transformation (adaptace na různé formáty partnerů)
└── Caching (snížení zátěže backend systémů)
```

### 4.2 Kdy použít HTTP API

**Interní mobilní/web BFF:**

```
Důvody pro HTTP API:
├── Nízká latence (lepší UX)
├── Nižší cena (vysoký objem requestů)
├── JWT Authorizer (native podpora, bez Lambda)
├── Jednodušší konfigurace
└── Validace může být na úrovni backendu (GraphQL, Lambda)
```

**Microservices komunikace:**

```
Důvody pro HTTP API:
├── Service-to-service volání
├── Nízká latence kritická
├── Private API ve VPC
└── Jednoduchý routing bez transformací
```

### 4.3 Hybridní architektura (běžná v bankách)

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    HYBRIDNÍ ARCHITEKTURA                                │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  ┌─────────────┐     ┌─────────────────┐                               │
│  │ TPP (PSD2)  │────▶│   REST API      │──┐                            │
│  │ Partners    │     │ (Open Banking)  │  │                            │
│  └─────────────┘     │ + WAF + Caching │  │                            │
│                      └─────────────────┘  │                            │
│                                           │    ┌──────────────────┐    │
│                                           ├───▶│                  │    │
│  ┌─────────────┐     ┌─────────────────┐  │    │  Backend         │    │
│  │ Mobile App  │────▶│   HTTP API      │──┤    │  Services        │    │
│  │ Web App     │     │ (Low latency)   │  │    │  (ECS/Lambda)    │    │
│  └─────────────┘     │ + JWT Auth      │  │    │                  │    │
│                      └─────────────────┘  │    └──────────────────┘    │
│                                           │                            │
│  ┌─────────────┐     ┌─────────────────┐  │                            │
│  │ Admin       │────▶│   REST API      │──┘                            │
│  │ Portal      │     │ (Internal)      │                               │
│  └─────────────┘     │ + IAM Auth      │                               │
│                      └─────────────────┘                               │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 5. Důležité koncepty pro pohovor

### 5.1 API Versioning strategie

```
URL Path Versioning:
  /v1/accounts/{id}
  /v2/accounts/{id}
  
  ✅ Explicitní, cache-friendly
  ❌ Může vést k duplicitě kódu

Header Versioning:
  GET /accounts/{id}
  Accept: application/vnd.bank.v2+json
  
  ✅ Čistší URL
  ❌ Složitější debugging, caching

Query Parameter:
  /accounts/{id}?version=2
  
  ✅ Jednoduchá implementace
  ❌ Méně RESTful

Doporučení pro banky: URL Path Versioning
- Jednoznačné pro TPP partnery
- Umožňuje paralelní provoz verzí (důležité pro PSD2 compliance)
- Snadnější traffic routing a monitoring
```

### 5.2 Error Handling standardizace

```json
// RFC 7807 Problem Details (doporučeno pro bankovní API)
{
  "type": "https://api.bank.cz/errors/insufficient-funds",
  "title": "Insufficient Funds",
  "status": 400,
  "detail": "Account CZ1234 has insufficient funds for this transaction",
  "instance": "/payments/12345",
  "errorCode": "PAY_001",
  "traceId": "abc-123-def-456"
}
```

### 5.3 Idempotency (kritické pro platby)

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    IDEMPOTENCY PATTERN                                  │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  Request:                                                               │
│  POST /payments                                                         │
│  Idempotency-Key: 550e8400-e29b-41d4-a716-446655440000                 │
│                                                                         │
│  Flow:                                                                  │
│  ┌────────┐     ┌─────────────┐     ┌─────────────┐     ┌─────────┐   │
│  │ Client │────▶│ API Gateway │────▶│ Idempotency │────▶│ Payment │   │
│  └────────┘     └─────────────┘     │   Store     │     │ Service │   │
│                                     │ (DynamoDB)  │     └─────────┘   │
│                                     └──────┬──────┘                    │
│                                            │                           │
│  Scénáře:                                  │                           │
│  1. Nový klíč ──▶ Zpracuj platbu, ulož výsledek                       │
│  2. Existující klíč ──▶ Vrať uložený výsledek (bez opakování)         │
│  3. Probíhající zpracování ──▶ HTTP 409 Conflict                      │
│                                                                         │
│  TTL: Typicky 24-48 hodin                                              │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 5.4 Circuit Breaker na API Gateway

```
// AWS: Není nativní, ale lze implementovat přes:

1. Lambda Authorizer s circuit breaker logikou
2. Backend integration s timeout nastavením
3. Step Functions pro orchestraci s error handling

// Alternativa: Service Mesh (App Mesh) nebo 
// application-level (Resilience4j, Polly)
```

---

## 6. Možné otázky na pohovoru

**Q: Jak byste navrhli API Gateway architekturu pro nové internetové bankovnictví?**

Klíčové body odpovědi:

1. HTTP API pro interní BFF (mobilní/web klienti) - nízká latence, JWT auth
2. REST API pro Open Banking/PSD2 - WAF, usage plans, request validation
3. Oddělené API Gateway per domain (accounts, payments, cards) nebo centrální s routing
4. Caching strategie pro read-heavy operace (balance, transaction history)
5. Observability (X-Ray, CloudWatch, structured logging)

**Q: Jaký je rozdíl mezi REST API a HTTP API v AWS?**

Viz tabulka výše - klíčové body: cena, latence, features (validation, caching, WAF, usage plans)

**Q: Jak řešíte rate limiting pro různé typy klientů?**

REST API s Usage Plans - různé limity pro mobile app, web, TPP partnery, interní systémy

---

Chceš, abych rozvedl nějakou konkrétní část - například detaily OAuth 2.0 flows, request validation schémata, nebo konkrétní Terraform/CloudFormation příklady?

# Cross-Cutting Concerns v API Gateway: Proč a Jak v AWS

## 1. Authentication & Authorization

### Proč je potřeba řešit

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         RIZIKA BEZ AUTENTIZACE                          │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  ❌ Neoprávněný přístup k účtům klientů                                │
│  ❌ Krádež finančních prostředků                                       │
│  ❌ Únik citlivých osobních dat (GDPR porušení)                        │
│  ❌ Podvodné transakce                                                 │
│  ❌ Regulatorní sankce (PSD2 vyžaduje SCA)                             │
│  ❌ Reputační škody                                                    │
│                                                                         │
│  V bankovnictví:                                                        │
│  • Každý request musí být přiřazen konkrétní identitě                  │
│  • Různé úrovně oprávnění (read-only vs. transakční)                   │
│  • Audit trail pro compliance                                           │
│  • PSD2 vyžaduje Strong Customer Authentication (SCA)                  │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### Jak v AWS

**Možnost 1: Amazon Cognito (REST API i HTTP API)**

```
┌──────────┐     ┌─────────────┐     ┌─────────────┐     ┌─────────┐
│  Client  │────▶│   Cognito   │────▶│ API Gateway │────▶│ Backend │
│          │◀────│ (JWT token) │     │(token valid)│     │         │
└──────────┘     └─────────────┘     └─────────────┘     └─────────┘

Konfigurace (REST API - Cognito Authorizer):
- API Gateway → Authorizers → Create New Authorizer
- Type: Cognito
- Cognito User Pool: vybrat pool
- Token Source: Authorization header

Výhody:
✅ Managed služba (bez kódu)
✅ Built-in user management, MFA
✅ OAuth 2.0 / OIDC compliant
✅ Hosted UI pro login

Nevýhody:
❌ Méně flexibilní pro komplexní business logic
❌ Vendor lock-in
```

**Možnost 2: JWT Authorizer (pouze HTTP API)**

```
HTTP API → Authorization → Create → JWT

Konfigurace:
{
  "identitySource": "$request.header.Authorization",
  "issuerUrl": "https://cognito-idp.eu-central-1.amazonaws.com/eu-central-1_xxx",
  "audience": ["your-app-client-id"]
}

Výhody:
✅ Nativní podpora v HTTP API
✅ Nízká latence (žádná Lambda)
✅ Funguje s jakýmkoliv OIDC providerem (Cognito, Auth0, Okta, Azure AD)
```

**Možnost 3: Lambda Authorizer (REST API i HTTP API)**

```
┌──────────┐     ┌─────────────┐     ┌─────────────┐     ┌─────────┐
│  Client  │────▶│ API Gateway │────▶│   Lambda    │     │ Backend │
│ + Token  │     │             │     │ Authorizer  │     │         │
└──────────┘     └──────┬──────┘     └──────┬──────┘     └─────────┘
                        │                   │
                        │    IAM Policy     │
                        │◀──────────────────┘
                        │
                        ▼
                   Route to Backend
```

**Možnost 4: IAM Authorization (pro service-to-service)**

```
Použití: Interní microservices, AWS služby volající API

Client (s IAM credentials) → SigV4 podpis → API Gateway → Backend

Konfigurace:
- Method → Authorization: AWS_IAM
- Client musí použít AWS SDK pro signing
```

### Autorizace - Scopes a Permissions

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    SCOPE-BASED AUTHORIZATION                            │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  OAuth 2.0 Scopes pro bankovnictví:                                    │
│                                                                         │
│  accounts:read      - Čtení zůstatků a informací o účtech              │
│  accounts:write     - Změny nastavení účtu                             │
│  transactions:read  - Historie transakcí                                │
│  payments:create    - Vytvoření platby                                  │
│  payments:approve   - Schválení platby (pro firemní účty)              │
│  cards:read         - Informace o kartách                               │
│  cards:manage       - Blokace, limity, PIN                             │
│                                                                         │
│  PSD2 specifické:                                                       │
│  aisp              - Account Information Service Provider               │
│  pisp              - Payment Initiation Service Provider                │
│  cbpii             - Card-Based Payment Instrument Issuer               │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 2. Rate Limiting & Throttling

### Proč je potřeba řešit

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    DŮVODY PRO RATE LIMITING                             │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  🛡️ OCHRANA SYSTÉMU                                                    │
│  • DDoS útoky - zahlcení systému                                       │
│  • Brute force útoky na login                                          │
│  • Credential stuffing                                                  │
│  • Scraping dat                                                         │
│                                                                         │
│  💰 COST CONTROL                                                        │
│  • Nekontrolované volání = vysoké náklady na compute                   │
│  • Partner s bugem může generovat miliony requestů                     │
│                                                                         │
│  ⚖️ FAIR USAGE                                                          │
│  • Jeden klient nesmí degradovat službu pro ostatní                    │
│  • SLA garantování pro premium partnery                                │
│                                                                         │
│  📋 REGULATORNÍ                                                         │
│  • PSD2 vyžaduje "reasonable rate limits" pro TPP                      │
│  • Nesmí být diskriminační mezi vlastní app a TPP                      │
│                                                                         │
│  Typické útoky:                                                         │
│  • Balance check flooding (zjištění stavu před útokem)                 │
│  • Payment enumeration                                                  │
│  • Account enumeration přes login                                      │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### Jak v AWS

**REST API - Usage Plans & API Keys**

```
┌─────────────────────────────────────────────────────────────────────────┐
│                      USAGE PLANS ARCHITEKTURA                           │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  ┌─────────────────┐                                                   │
│  │   API Key:      │                                                   │
│  │   "TPP-PartnerA"│───────┐                                           │
│  └─────────────────┘       │      ┌─────────────────────────┐          │
│                            ├─────▶│  Usage Plan: "TPP-Basic"│          │
│  ┌─────────────────┐       │      │  • Rate: 100 req/sec    │          │
│  │   API Key:      │───────┘      │  • Burst: 200           │          │
│  │   "TPP-PartnerB"│              │  • Quota: 100,000/day   │          │
│  └─────────────────┘              └─────────────────────────┘          │
│                                                                         │
│  ┌─────────────────┐              ┌─────────────────────────┐          │
│  │   API Key:      │─────────────▶│  Usage Plan: "Premium"  │          │
│  │   "BigPartner"  │              │  • Rate: 1000 req/sec   │          │
│  └─────────────────┘              │  • Burst: 2000          │          │
│                                   │  • Quota: unlimited     │          │
│                                   └─────────────────────────┘          │
│                                                                         │
│  ┌─────────────────┐              ┌─────────────────────────┐          │
│  │   API Key:      │─────────────▶│  Usage Plan: "Internal" │          │
│  │   "MobileApp"   │              │  • Rate: 5000 req/sec   │          │
│  └─────────────────┘              │  • No quota             │          │
│                                   └─────────────────────────┘          │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```


**Rozdíl Rate vs Burst vs Quota:**

```
┌─────────────────────────────────────────────────────────────────────────┐
│                                                                         │
│  RATE LIMIT (steady state):                                            │
│  ════════════════════════                                              │
│  Počet požadavků za sekundu, které systém trvale zvládne               │
│                                                                         │
│  ───────────────────────────────────────────────────────▶ čas          │
│  ▓▓▓ ▓▓▓ ▓▓▓ ▓▓▓ ▓▓▓ ▓▓▓ ▓▓▓ (100 req/sec kontinuálně)                │
│                                                                         │
│  BURST LIMIT (spike tolerance):                                        │
│  ══════════════════════════════                                        │
│  Maximální počet současných požadavků (peak)                           │
│                                                                         │
│                    ▓▓▓▓▓▓▓▓▓▓                                          │
│  ───────────────────▓▓▓▓▓▓▓▓▓▓──────────────────────────▶ čas          │
│  ▓▓▓ ▓▓▓ ▓▓▓ ▓▓▓ ▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓ ▓▓▓ ▓▓▓                              │
│                    (burst 200)                                          │
│                                                                         │
│  QUOTA (total volume):                                                  │
│  ═════════════════════                                                  │
│  Celkový počet požadavků za období (den/měsíc)                         │
│                                                                         │
│  ████████████████████████████░░░░░░░░░░░░░░                            │
│  [====== 75,000 / 100,000 denní limit ======]                          │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

**Response při překročení limitu:**

```http
HTTP/1.1 429 Too Many Requests
Retry-After: 1
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 0
X-RateLimit-Reset: 1640000000

{
  "message": "Rate limit exceeded",
  "type": "https://api.bank.cz/errors/rate-limit-exceeded"
}
```

---

## 3. Request/Response Transformation

### Proč je potřeba řešit

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    DŮVODY PRO TRANSFORMACI                              │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  🔄 LEGACY INTEGRACE                                                    │
│  • Core banking vrací XML, klient potřebuje JSON                       │
│  • Mainframe vrací EBCDIC/fixed-width, potřebujeme REST                │
│  • Různé datové formáty (ISO 8583, ISO 20022, proprietární)            │
│                                                                         │
│  🎯 API DESIGN DECOUPLING                                              │
│  • Interní struktura ≠ externí API kontrakt                            │
│  • Změny v backendu neovlivní klienty                                  │
│  • Agregace dat z více backendů do jednoho response                    │
│                                                                         │
│  🔒 SECURITY                                                            │
│  • Maskování citlivých dat (číslo účtu, rodné číslo)                  │
│  • Odstranění interních fieldů (debug info, internal IDs)              │
│  • Přidání security headers                                            │
│                                                                         │
│  📊 ENRICHMENT                                                          │
│  • Přidání computed fields                                             │
│  • Lokalizace (currency formatting, date formats)                      │
│  • Přidání HATEOAS linků                                               │
│                                                                         │
│  Příklad v bankovnictví:                                               │
│  Core Banking: { "acct_no": "123", "bal": 1000, "ccy": "CZK" }        │
│  API Response: { "accountNumber": "***123",                            │
│                  "balance": { "amount": 1000.00,                       │
│                               "currency": "CZK" },                     │
│                  "formattedBalance": "1 000,00 Kč" }                   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### Jak v AWS

**REST API - Velocity Template Language (VTL)**

```
┌──────────┐     ┌─────────────────────────────┐     ┌──────────┐
│  Client  │────▶│       API Gateway           │────▶│ Backend  │
│          │     │                             │     │          │
│  JSON    │     │  Request:                   │     │  JSON/   │
│          │     │  ┌───────────────────────┐  │     │  XML     │
│          │     │  │ Integration Request   │  │     │          │
│          │     │  │ Template (VTL)        │  │     │          │
│          │     │  └───────────────────────┘  │     │          │
│          │◀────│                             │◀────│          │
│  JSON    │     │  Response:                  │     │  JSON/   │
│          │     │  ┌───────────────────────┐  │     │  XML     │
│          │     │  │ Integration Response  │  │     │          │
│          │     │  │ Template (VTL)        │  │     │          │
│          │     │  └───────────────────────┘  │     │          │
└──────────┘     └─────────────────────────────┘     └──────────┘
```

**Komplexní transformace - Lambda Integration**

---

## 4. Caching

### Proč je potřeba řešit

```
┌─────────────────────────────────────────────────────────────────────────┐
│                       DŮVODY PRO CACHING                                │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  ⚡ PERFORMANCE                                                         │
│  • Snížení latence (cache hit < 10ms vs backend 100-500ms)             │
│  • Lepší uživatelská zkušenost                                         │
│  • Rychlejší načítání mobilní aplikace                                 │
│                                                                         │
│  💰 COST REDUCTION                                                      │
│  • Méně volání do backendu = nižší compute náklady                     │
│  • Snížení zátěže na core banking (často drahé legacy systémy)         │
│  • Licence core banking často per-transaction                          │
│                                                                         │
│  🛡️ RESILIENCE                                                         │
│  • Cache může sloužit při výpadku backendu (stale-while-revalidate)    │
│  • Ochrana před spike traffic                                          │
│                                                                         │
│  📊 TYPICKÉ CACHE CANDIDATES V BANKOVNICTVÍ                            │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ Endpoint              │ TTL      │ Důvod                        │   │
│  ├───────────────────────┼──────────┼──────────────────────────────│   │
│  │ Exchange rates        │ 5 min    │ Mění se periodicky           │   │
│  │ Product catalog       │ 1 hour   │ Zřídka se mění               │   │
│  │ Branch locations      │ 24 hours │ Statická data                │   │
│  │ Account balance       │ ❌        │ Real-time requirement        │   │
│  │ Transactions          │ ❌        │ Konzistence kritická         │   │
│  │ User profile          │ 5 min    │ Čtení >> zápis               │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  ⚠️ CO NECACHOVAT                                                      │
│  • Finanční transakce a zůstatky (konzistence)                        │
│  • POST/PUT/DELETE operace                                             │
│  • Personalizovaná data s vysokou volatilitou                         │
│  • Security-sensitive operace                                          │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### Jak v AWS

**REST API - Built-in Caching**

```
┌──────────┐     ┌─────────────────────────────────────┐     ┌──────────┐
│  Client  │────▶│          API Gateway                │────▶│ Backend  │
│          │     │     ┌───────────────────┐           │     │          │
│          │     │     │   Cache Layer     │           │     │          │
│          │◀────│     │  (ElastiCache)    │           │     │          │
│          │     │     │                   │           │     │          │
│          │     │     │  Cache Hit: <10ms │           │     │          │
│          │     │     │  Cache Miss: pass │           │     │          │
│          │     │     └───────────────────┘           │     │          │
└──────────┘     └─────────────────────────────────────┘     └──────────┘
```

## Různé cache klíče pro různé uživatele/requesty

GET /exchange-rates?from=CZK&to=EUR
Cache Key: from=CZK&to=EUR
→ Sdíleno mezi všemi uživateli (veřejná data)

GET /accounts/{accountId}/profile
Cache Key: accountId + Authorization header
→ Per-user cache (personalizovaná data)

GET /products?segment=premium
Cache Key: segment + Accept-Language header
→ Per-segment + per-locale cache

**HTTP API - Žádný built-in cache**

```
HTTP API nemá built-in caching, alternativy:

1. CloudFront před API Gateway
   ┌────────┐     ┌────────────┐     ┌──────────┐     ┌─────────┐
   │ Client │────▶│ CloudFront │────▶│ HTTP API │────▶│ Backend │
   └────────┘     │   (cache)  │     └──────────┘     └─────────┘
                  └────────────┘

2. ElastiCache (Redis) v aplikační vrstvě
   ┌──────────┐     ┌─────────────┐     ┌─────────┐
   │ HTTP API │────▶│   Lambda    │────▶│ Backend │
   └──────────┘     │     ↓↑      │     └─────────┘
                    │ ElastiCache │
                    └─────────────┘
```

---

## 5. Logging & Monitoring

### Proč je potřeba řešit

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    DŮVODY PRO LOGGING & MONITORING                      │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  📋 REGULATORY COMPLIANCE                                               │
│  • Audit trail pro všechny transakce (AML, PSD2)                       │
│  • Uchovávání logů 5-10 let (zákonné požadavky)                        │
│  • Schopnost rekonstruovat události pro vyšetřování                    │
│  • GDPR - kdo kdy přistoupil k osobním datům                          │
│                                                                         │
│  🔍 DEBUGGING & TROUBLESHOOTING                                         │
│  • Identifikace problémů v produkci                                    │
│  • Root cause analysis                                                  │
│  • Performance bottleneck detection                                     │
│                                                                         │
│  🔒 SECURITY                                                            │
│  • Detekce anomálií a útoků                                            │
│  • Failed login attempts tracking                                       │
│  • Unusual access patterns                                              │
│  • SIEM integrace                                                       │
│                                                                         │
│  📊 BUSINESS INTELLIGENCE                                               │
│  • API usage analytics                                                  │
│  • Popular endpoints                                                    │
│  • Partner usage tracking                                               │
│  • Capacity planning                                                    │
│                                                                         │
│  🚨 ALERTING                                                            │
│  • Error rate spikes                                                    │
│  • Latency degradation                                                  │
│  • Availability issues                                                  │
│  • Rate limit breaches                                                  │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### Jak v AWS

**CloudWatch Logs - Access Logging**

**CloudWatch Metrics & Alarms**

```yaml
# Automaticky dostupné metriky:
# - Count (počet requestů)
# - Latency (response time)
# - IntegrationLatency (backend time)
# - 4XXError, 5XXError
# - CacheHitCount, CacheMissCount

**X-Ray Tracing (distributed tracing)**

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         X-RAY TRACE                                     │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  Request ID: abc-123-def                                               │
│  Total Duration: 245ms                                                  │
│                                                                         │
│  ├── API Gateway (15ms)                                                │
│  │   ├── Authorization (5ms)                                           │
│  │   └── Request Processing (10ms)                                     │
│  │                                                                      │
│  ├── Lambda - Account Service (180ms)                                  │
│  │   ├── Cold Start (50ms) ⚠️                                         │
│  │   ├── DynamoDB GetItem (20ms)                                       │
│  │   ├── Core Banking API Call (100ms)                                 │
│  │   └── Response Processing (10ms)                                    │
│  │                                                                      │
│  └── API Gateway Response (50ms)                                       │
│      └── Response Transformation (50ms)                                │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘

```

**Log Insights Queries (analýza)**

```sql
-- Top 10 nejpomalejších endpointů
fields @timestamp, resourcePath, responseLatency
| filter responseLatency > 1000
| sort responseLatency desc
| limit 10

-- Error rate per endpoint
fields resourcePath, status
| filter status >= 400
| stats count() as errors by resourcePath
| sort errors desc

-- Requests per user (pro fraud detection)
fields userId, @timestamp
| filter userId != ""
| stats count() as requestCount by userId
| filter requestCount > 1000
| sort requestCount desc

-- Failed login attempts
fields ip, status, resourcePath
| filter resourcePath like /login/ and status = 401
| stats count() as failedAttempts by ip
| filter failedAttempts > 5
```

---

## 6. SSL Termination

### Proč je potřeba řešit

```
┌─────────────────────────────────────────────────────────────────────────┐
│                      DŮVODY PRO SSL TERMINATION                         │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  🔐 ENCRYPTION IN TRANSIT                                               │
│  • Ochrana dat při přenosu (GDPR, PCI-DSS requirement)                 │
│  • Prevence man-in-the-middle útoků                                    │
│  • Integrita dat                                                        │
│                                                                         │
│  📜 CERTIFICATE MANAGEMENT                                              │
│  • Centralizovaná správa certifikátů                                   │
│  • Automatická rotace                                                   │
│  • Snížení komplexity na backend serverech                             │
│                                                                         │
│  ⚡ PERFORMANCE                                                         │
│  • SSL handshake na edge (blíže klientovi)                             │
│  • Backend komunikuje přes interní síť (rychlejší)                     │
│  • Možnost HTTP/2 multiplexing                                         │
│                                                                         │
│  🏢 COMPLIANCE                                                          │
│  • PCI-DSS vyžaduje TLS 1.2+                                           │
│  • Bankovní regulace vyžadují strong encryption                        │
│  • Audit schopnost (cipher suites, protocol versions)                  │
│                                                                         │
│  Flow:                                                                  │
│  ┌────────┐  HTTPS   ┌─────────────┐  HTTP/HTTPS  ┌─────────┐          │
│  │ Client │─────────▶│ API Gateway │─────────────▶│ Backend │          │
│  └────────┘  TLS 1.3 │(terminates) │   (VPC)      └─────────┘          │
│                      └─────────────┘                                    │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### Jak v AWS

**Custom Domain s ACM certifikátem**

```yaml
# Terraform - Custom Domain pro API Gateway

# 1. Vytvoření certifikátu v ACM

# 2. DNS validace

# 3. Custom domain pro REST API

# 4. Base path mapping


# 5. Route53 record
```

**Security Policy (TLS versions & cipher suites)**

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    TLS SECURITY POLICIES                                │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  TLS_1_2 (doporučeno pro bankovnictví):                                │
│  ├── Protokoly: TLS 1.2, TLS 1.3                                       │
│  ├── Cipher Suites:                                                     │
│  │   ├── TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384                        │
│  │   ├── TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256                        │
│  │   └── TLS_ECDHE_ECDSA_WITH_AES_256_GCM_SHA384                      │
│  └── ❌ Zakázáno: TLS 1.0, TLS 1.1, SSLv3                              │
│                                                                         │
│  TLS_1_0 (legacy, nedoporučeno):                                       │
│  ├── Protokoly: TLS 1.0, 1.1, 1.2, 1.3                                 │
│  └── ⚠️ Pouze pro zpětnou kompatibilitu se starými klienty            │
│                                                                         │
│  Pro PCI-DSS compliance: MUSÍ být TLS_1_2                              │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

**Backend Integration - HTTPS vs HTTP**

```yaml
# HTTPS k backendu (end-to-end encryption)

# HTTP k backendu (pouze v rámci VPC - přijatelné)

  # Důležité: Pouze přes VPC Link, nikdy přes internet
  connection_type = "VPC_LINK"
}
```

---

## 7. API Versioning

### Proč je potřeba řešit

```
┌─────────────────────────────────────────────────────────────────────────┐
│                      DŮVODY PRO API VERSIONING                          │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  🔄 BACKWARD COMPATIBILITY                                              │
│  • Existující klienti nesmí přestat fungovat                           │
│  • Mobilní aplikace nelze vynutit k okamžitému update                  │
│  • TPP partneři potřebují čas na migraci                               │
│                                                                         │
│  📋 REGULATORY REQUIREMENTS                                             │
│  • PSD2 vyžaduje oznámení změn 3 měsíce předem                        │
│  • Dokumentované změny pro audit                                        │
│  • Parallel running období                                              │
│                                                                         │
│  🔧 EVOLUTION                                                           │
│  • Breaking changes (změna struktury response)                         │
│  • Nové povinné fieldy                                                 │
│  • Změna business logic                                                 │
│  • Deprecation starých funkcí                                          │
│                                                                         │
│  📊 ANALYTICS                                                           │
│  • Sledování adopce nových verzí                                       │
│  • Identifikace klientů na starých verzích                             │
│  • Plánování sunset starých verzí                                      │
│                                                                         │
│  Příklad breaking change:                                               │
│  v1: { "balance": 1000.50 }                                            │
│  v2: { "balance": { "amount": 1000.50, "currency": "CZK" } }          │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### Jak v AWS

**Strategie 1: URL Path Versioning (doporučeno)**

```
┌─────────────────────────────────────────────────────────────────────────┐
│                     URL PATH VERSIONING                                 │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  https://api.banka.cz/v1/accounts                                      │
│  https://api.banka.cz/v2/accounts                                      │
│                                                                         │
│  ┌───────────────┐                                                     │
│  │  API Gateway  │                                                     │
│  │               │                                                     │
│  │  /v1/* ───────┼──────▶ Backend V1 (nebo Lambda V1)                 │
│  │               │                                                     │
│  │  /v2/* ───────┼──────▶ Backend V2 (nebo Lambda V2)                 │
│  │               │                                                     │
│  └───────────────┘                                                     │
│                                                                         │
│  Výhody:                                                                │
│  ✅ Explicitní a jasné                                                 │
│  ✅ Cache-friendly (různé URL = různé cache entries)                   │
│  ✅ Snadné logování a monitoring per verze                             │
│  ✅ Snadný routing                                                      │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

```yaml
# Terraform - Path-based versioning

# Možnost A: Jeden API Gateway, různé resources

# Možnost B: Base path mapping (čistší)

```

**Strategie 2: Header Versioning**

```
┌─────────────────────────────────────────────────────────────────────────┐
│                      HEADER VERSIONING                                  │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  GET /accounts                                                          │
│  Accept: application/vnd.banka.v2+json                                 │
│                                                                         │
│  nebo:                                                                  │
│  GET /accounts                                                          │
│  X-API-Version: 2                                                       │
│                                                                         │
│  Implementace v API Gateway:                                            │
│  → Lambda Authorizer čte header                                        │
│  → Nastaví context variable                                            │
│  → Integration routing podle context                                    │
│                                                                         │
│  Výhody:                                                                │
│  ✅ Čistší URL                                                          │
│  ✅ RESTful puristi preferují                                           │
│                                                                         │
│  Nevýhody:                                                              │
│  ❌ Složitější implementace                                             │
│  ❌ Horší pro debugging (URL nevypovídá o verzi)                       │
│  ❌ Cache komplikace (Vary header)                                      │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

**Stage Variables pro routing**

```yaml
# Stage variables pro dynamický routing

# V integraci použít: ${stageVariables.accountsServiceUrl}
```

**Deprecation Headers**

```python
# Lambda - přidání deprecation headers pro staré verze

```

---

## 8. Request Validation

### Proč je potřeba řešit

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    DŮVODY PRO REQUEST VALIDATION                        │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  🔒 SECURITY (Input Validation = First Line of Defense)                │
│  • SQL Injection prevence                                               │
│  • XSS prevence                                                         │
│  • Command injection                                                    │
│  • Path traversal                                                       │
│  • Oversized payloads (DoS)                                            │
│                                                                         │
│  💰 COST OPTIMIZATION                                                   │
│  • Neplatné requesty zastavit PŘED voláním backendu                    │
│  • Úspora Lambda invokací                                              │
│  • Úspora compute na backendu                                          │
│                                                                         │
│  📊 DATA QUALITY                                                        │
│  • Zajištění správných typů (string vs number)                         │
│  • Povinné fieldy                                                       │
│  • Formáty (email, IBAN, telefon)                                      │
│  • Business rules (amount > 0)                                         │
│                                                                         │
│  ⚡ BETTER UX                                                           │
│  • Rychlá zpětná vazba (400 Bad Request ihned)                        │
│  • Jasné error messages                                                 │
│  • Konzistentní chování API                                            │
│                                                                         │
│  Příklad - platební příkaz:                                            │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ {                                                                │   │
│  │   "amount": 1000.50,        ← MUSÍ být kladné číslo            │   │
│  │   "currency": "CZK",        ← MUSÍ být ISO 4217                │   │
│  │   "creditorIban": "CZ...",  ← MUSÍ být validní IBAN            │   │
│  │   "reference": "INV-001"    ← MAX 140 znaků                    │   │
│  │ }                                                                │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### Jak v AWS

**REST API - JSON Schema Validation (built-in)**

```yaml
# Terraform - Request Validator
resource "aws_api_gateway_request_validator" "full" {
}

resource "aws_api_gateway_request_validator" "params_only" {
}
```

```json
// JSON Schema Model pro platební příkaz
{
  "$schema": "http://json-schema.org/draft-04/schema#",
  "title": "PaymentInitiationRequest",
  "type": "object",
  "required": ["debtorAccount", "creditorAccount", "amount", "currency"],
  "properties": {
    "debtorAccount": {
      "type": "object",
      "required": ["iban"],
      "properties": {
        "iban": {
          "type": "string",
          "pattern": "^[A-Z]{2}[0-9]{2}[A-Z0-9]{1,30}$",
          "description": "IBAN account number"
        }
      }
    },
    "creditorAccount": {
      "type": "object",
      "required": ["iban"],
      "properties": {
        "iban": {
          "type": "string",
          "pattern": "^[A-Z]{2}[0-9]{2}[A-Z0-9]{1,30}$"
        },
        "creditorName": {
          "type": "string",
          "maxLength": 70
        }
      }
    },
    "amount": {
      "type": "number",
      "minimum": 0.01,
      "maximum": 999999999.99,
      "description": "Transaction amount"
    },
    "currency": {
      "type": "string",
      "enum": ["CZK", "EUR", "USD", "GBP"],
      "description": "ISO 4217 currency code"
    },
    "reference": {
      "type": "string",
      "maxLength": 140,
      "description": "Payment reference"
    },
    "requestedExecutionDate": {
      "type": "string",
      "format": "date",
      "description": "ISO 8601 date"
    }
  },
  "additionalProperties": false
}
```

```yaml
# Terraform - Přiřazení modelu k metodě
resource "aws_api_gateway_model" "payment_request" {
  rest_api_id  = aws_api_gateway_rest_api.banking.id
  name         = "PaymentRequest"
  description  = "Payment initiation request schema"
  content_type = "application/json"
  
  schema = file("schemas/payment-request.json")
}

resource "aws_api_gateway_method" "create_payment" {
  rest_api_id   = aws_api_gateway_rest_api.banking.id
  resource_id   = aws_api_gateway_resource.payments.id
  http_method   = "POST"
  authorization = "CUSTOM"
  authorizer_id = aws_api_gateway_authorizer.jwt.id
  
  request_validator_id = aws_api_gateway_request_validator.full.id
  
  request_models = {
    "application/json" = aws_api_gateway_model.payment_request.name
  }
  
  # Query parameter validation
  request_parameters = {
    "method.request.querystring.idempotencyKey" = true  # required
  }
}
```

**HTTP API - Validace v Lambda (není built-in)**

```python
# Validace pomocí knihoven (jsonschema, pydantic, marshmallow)
```

**Gateway Response Customization**

```yaml
# Vlastní error response pro validační chyby
resource "aws_api_gateway_gateway_response" "bad_request" {
  rest_api_id   = aws_api_gateway_rest_api.banking.id
  response_type = "BAD_REQUEST_BODY"
  status_code   = "400"

  response_templates = {
    "application/json" = jsonencode({
      type    = "https://api.bank.cz/errors/validation-error"
      title   = "Request validation failed"
      status  = 400
      detail  = "$context.error.validationErrorString"
      traceId = "$context.requestId"
    })
  }
  
  response_parameters = {
    "gatewayresponse.header.Content-Type" = "'application/problem+json'"
  }
}
```

---

## 9. mTLS (Mutual TLS)

### Proč je potřeba řešit

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         DŮVODY PRO mTLS                                 │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  🔐 VZÁJEMNÁ AUTENTIZACE                                               │
│                                                                         │
│  Běžné TLS (jednosměrné):                                              │
│  ┌────────┐                    ┌────────┐                              │
│  │ Client │──── Verify ───────▶│ Server │                              │
│  │        │    server cert     │        │                              │
│  └────────┘                    └────────┘                              │
│  Klient ověřuje server, ale server neví KDO je klient                  │
│                                                                         │
│  mTLS (obousměrné):                                                    │
│  ┌────────┐                    ┌────────┐                              │
│  │ Client │◀─── Verify ───────▶│ Server │                              │
│  │  cert  │    both certs      │  cert  │                              │
│  └────────┘                    └────────┘                              │
│  Obě strany se navzájem ověří certifikáty                              │
│                                                                         │
│  📋 PSD2 REQUIREMENT                                                    │
│  • TPP (Third Party Providers) MUSÍ použít QWAC certifikáty           │
│  • eIDAS qualified certificates                                        │
│  • Identifikace TPP podle certifikátu                                  │
│  • Nelze se vydávat za jiného TPP                                      │
│                                                                         │
│  🏢 B2B INTEGRACE                                                       │
│  • Partnerské systémy (payment processors, card networks)              │
│  • Vyšší bezpečnost než API keys                                       │
│  • Certificate-based identity                                          │
│                                                                         │
│  🔒 ZERO TRUST ARCHITECTURE                                            │
│  • "Never trust, always verify"                                        │
│  • Network location není důvěryhodná                                   │
│  • Každý request musí prokázat identitu                                │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### Jak v AWS

**REST API & HTTP API - mTLS konfigurace**

```yaml
# Terraform - mTLS pro API Gateway

# 1. Truststore v S3 (obsahuje CA certifikáty pro ověření klientů)

# 2. Custom domain s mTLS

```

**Přístup k client certificate v Lambda**

```python
# Lambda - čtení informací z klientského certifikátu

```

**QWAC Certificate validation pro PSD2**

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    PSD2 QWAC CERTIFICATE FLOW                           │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  ┌─────────┐     ┌─────────────┐     ┌───────────────┐                 │
│  │   TPP   │────▶│ API Gateway │────▶│    Lambda     │                 │
│  │ + QWAC  │     │   (mTLS)    │     │  Authorizer   │                 │
│  └─────────┘     └──────┬──────┘     └───────┬───────┘                 │
│                         │                    │                          │
│                         │                    ▼                          │
│                         │            ┌───────────────┐                 │
│                         │            │ TPP Registry  │                 │
│                         │            │ (check status)│                 │
│                         │            └───────────────┘                 │
│                         │                                               │
│  QWAC obsahuje:                                                         │
│  • Organization ID (TPP identifikace)                                  │
│  • Authorization Number (PSDXX-NCA-XXXXXX)                            │
│  • PSD2 Roles (AISP, PISP, CBPII)                                     │
│  • NCA (National Competent Authority)                                  │
│                                                                         │
│  Validace:                                                              │
│  1. ✅ Certifikát podepsán důvěryhodnou CA (QTSP)                      │
│  2. ✅ Certifikát není expirovaný/revokovaný                           │
│  3. ✅ TPP je registrován v národním registru                          │
│  4. ✅ TPP má oprávnění pro požadovanou operaci                        │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 10. WAF Integration

### Proč je potřeba řešit

```
┌─────────────────────────────────────────────────────────────────────────┐
│                       DŮVODY PRO WAF                                    │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  🛡️ OWASP TOP 10 PROTECTION                                            │
│  • SQL Injection                                                        │
│  • Cross-Site Scripting (XSS)                                          │
│  • Server-Side Request Forgery (SSRF)                                  │
│  • Local File Inclusion (LFI)                                          │
│                                                                         │
│  🤖 BOT PROTECTION                                                      │
│  • Credential stuffing                                                  │
│  • Account takeover attempts                                            │
│  • Scraping                                                             │
│  • Automated vulnerability scanning                                     │
│                                                                         │
│  🌊 DDOS MITIGATION                                                     │
│  • Rate-based rules                                                     │
│  • Geographic blocking                                                  │
│  • IP reputation                                                        │
│                                                                         │
│  📋 COMPLIANCE                                                          │
│  • PCI-DSS requirement 6.6 (WAF nebo code review)                      │
│  • Prokazatelná ochrana pro auditory                                   │
│  • Logging pro forenzní analýzu                                        │
│                                                                         │
│  🎯 CUSTOM PROTECTION                                                   │
│  • Blocking specific attack patterns                                    │
│  • Geo-restrictions (OFAC countries)                                   │
│  • Business logic protection                                           │
│                                                                         │
│  ⚠️ POUZE PRO REST API                                                 │
│  HTTP API nepodporuje WAF integraci!                                   │
│  Alternativa: CloudFront + WAF před HTTP API                           │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### Jak v AWS

**AWS WAF Konfigurace**

```yaml
# Terraform - WAF pro API Gateway

# 1. Web ACL

  # AWS Managed Rules - OWASP Core Rule Set

  # SQL Injection Protection

  # Known Bad Inputs

  # Rate Limiting - Brute Force Protection

  # Geo Blocking (OFAC sanctioned countries)

  # Custom Rule - Block specific patterns

# 2. Asociace WAF s API Gateway Stage

# 3. Logging

```

**WAF Dashboard a Alarmy**

```yaml
# CloudWatch Alarm pro blocked requests
resource "aws_cloudwatch_metric_alarm" "waf_blocked" {
  alarm_name          = "waf-high-blocked-requests"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 2
  metric_name         = "BlockedRequests"
  namespace           = "AWS/WAFV2"
  period              = 300
  statistic           = "Sum"
  threshold           = 1000
  alarm_description   = "High number of WAF blocked requests"
  
  dimensions = {
    WebACL = aws_wafv2_web_acl.banking_api.name
    Rule   = "ALL"
    Region = "eu-central-1"
  }
  
  alarm_actions = [aws_sns_topic.security_alerts.arn]
}
```

---

## Shrnutí: Feature Matrix

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    FEATURE AVAILABILITY MATRIX                          │
├────────────────────────┬───────────────────┬────────────────────────────┤
│        Feature         │     REST API      │         HTTP API           │
├────────────────────────┼───────────────────┼────────────────────────────┤
│ Authentication         │ Cognito, Lambda,  │ JWT (native), Lambda,      │
│                        │ IAM               │ IAM                        │
├────────────────────────┼───────────────────┼────────────────────────────┤
│ Rate Limiting          │ ✅ Usage Plans    │ ⚠️ Basic throttling only   │
├────────────────────────┼───────────────────┼────────────────────────────┤
│ Request Transformation │ ✅ VTL Templates  │ ⚠️ Parameter mapping only  │
├────────────────────────┼───────────────────┼────────────────────────────┤
│ Caching                │ ✅ Built-in       │ ❌ (use CloudFront)        │
├────────────────────────┼───────────────────┼────────────────────────────┤
│ Logging & Monitoring   │ ✅ Full           │ ✅ Full                    │
├────────────────────────┼───────────────────┼────────────────────────────┤
│ SSL Termination        │ ✅ ACM            │ ✅ ACM                     │
├────────────────────────┼───────────────────┼────────────────────────────┤
│ API Versioning         │ ✅ Base path      │ ✅ Base path               │
├────────────────────────┼───────────────────┼────────────────────────────┤
│ Request Validation     │ ✅ JSON Schema    │ ❌ (in Lambda)             │
├────────────────────────┼───────────────────┼────────────────────────────┤
│ mTLS                   │ ✅                │ ✅                         │
├────────────────────────┼───────────────────┼────────────────────────────┤
│ WAF Integration        │ ✅                │ ❌ (use CloudFront+WAF)    │
├────────────────────────┼───────────────────┼────────────────────────────┤
│ Cena                   │ $3.50/million     │ $1.00/million              │
├────────────────────────┼───────────────────┼────────────────────────────┤
│ Latence                │ ~30-50ms overhead │ ~10ms overhead             │
└────────────────────────┴───────────────────┴────────────────────────────┘

Doporučení pro bankovnictví:
• PSD2/Open Banking API → REST API (WAF, validation, usage plans)
• Interní Mobile/Web BFF → HTTP API (nízká latence, JWT auth)
• B2B Partner API → REST API (mTLS, caching, transformation)
```

---

Chceš, abych rozvedl nějakou konkrétní část - například praktické příklady Terraform konfigurace pro konkrétní use case, nebo detaily OAuth 2.0 flows pro bankovnictví?

# Architektonické a bezpečnostní principy pro digitální bankovnictví

---

## 1. Defense in Depth (Vícevrstevná bezpečnost)

### Co to je

Princip vycházející z vojenské strategie - vytvoření více nezávislých bezpečnostních vrstev, kde selhání jedné vrstvy neznamená kompromitaci celého systému. Útočník musí překonat všechny vrstvy, ne jen jednu.

```
┌─────────────────────────────────────────────────────────────────────────┐
│                      DEFENSE IN DEPTH - VRSTVY                          │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                    PERIMETER SECURITY                            │   │
│  │              (WAF, DDoS Protection, Edge Firewall)               │   │
│  │  ┌─────────────────────────────────────────────────────────┐    │   │
│  │  │                 NETWORK SECURITY                         │    │   │
│  │  │           (VPC, Security Groups, NACLs, PrivateLink)     │    │   │
│  │  │  ┌─────────────────────────────────────────────────┐    │    │   │
│  │  │  │              IDENTITY SECURITY                   │    │    │   │
│  │  │  │         (Authentication, Authorization, MFA)     │    │    │   │
│  │  │  │  ┌─────────────────────────────────────────┐    │    │    │   │
│  │  │  │  │          APPLICATION SECURITY            │    │    │    │   │
│  │  │  │  │    (Input Validation, OWASP, Secure Code)│    │    │    │   │
│  │  │  │  │  ┌─────────────────────────────────┐    │    │    │    │   │
│  │  │  │  │  │         DATA SECURITY            │    │    │    │    │   │
│  │  │  │  │  │  (Encryption, Masking, Tokenization) │    │    │    │   │
│  │  │  │  │  │                                  │    │    │    │    │   │
│  │  │  │  │  │         💎 DATA 💎              │    │    │    │    │   │
│  │  │  │  │  │                                  │    │    │    │    │   │
│  │  │  │  │  └─────────────────────────────────┘    │    │    │    │   │
│  │  │  │  └─────────────────────────────────────────┘    │    │    │   │
│  │  │  └─────────────────────────────────────────────────┘    │    │   │
│  │  └─────────────────────────────────────────────────────────┘    │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  + MONITORING & DETECTION (průřezově všemi vrstvami)                   │
│  + INCIDENT RESPONSE (reakce na průnik)                                │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### Proč je kritický v bankovnictví

|Důvod|Vysvětlení|
|---|---|
|**Vysoká hodnota cíle**|Banky jsou primární cíl útočníků - přímý přístup k penězům|
|**Regulatorní požadavky**|PCI-DSS, DORA, NIS2 explicitně vyžadují vícevrstevnou ochranu|
|**Sofistikované útoky**|APT (Advanced Persistent Threats) překonávají jednotlivé vrstvy|
|**Insider threats**|Interní útočníci už jsou "uvnitř" - potřeba vnitřních vrstev|
|**Supply chain attacks**|Kompromitace dodavatele obchází perimetr|

### Implementace v AWS bankovní architektuře

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    PRAKTICKÁ IMPLEMENTACE AWS                           │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  VRSTVA 1: Edge/Perimeter                                              │
│  ├── AWS Shield (DDoS protection)                                      │
│  ├── AWS WAF (OWASP rules, rate limiting, geo-blocking)               │
│  ├── CloudFront (edge caching, TLS termination)                        │
│  └── Route 53 (DNS filtering, health checks)                           │
│                                                                         │
│  VRSTVA 2: Network                                                      │
│  ├── VPC isolation (separate VPCs per environment)                     │
│  ├── Private subnets (no direct internet access)                       │
│  ├── Security Groups (stateful, least privilege)                       │
│  ├── NACLs (stateless, subnet-level)                                   │
│  ├── VPC Endpoints/PrivateLink (no internet for AWS services)         │
│  └── Transit Gateway (controlled inter-VPC traffic)                    │
│                                                                         │
│  VRSTVA 3: Identity                                                     │
│  ├── API Gateway authentication (Cognito, JWT, Lambda Authorizer)     │
│  ├── IAM roles (least privilege, no long-term credentials)            │
│  ├── MFA enforcement                                                    │
│  ├── Session management (token expiry, rotation)                       │
│  └── Service-to-service authentication (IAM roles, mTLS)              │
│                                                                         │
│  VRSTVA 4: Application                                                  │
│  ├── Input validation (API Gateway + application level)               │
│  ├── Output encoding                                                    │
│  ├── Secure dependencies (vulnerability scanning)                      │
│  ├── Runtime protection (Lambda layers, container security)           │
│  └── Secrets management (Secrets Manager, no hardcoded secrets)       │
│                                                                         │
│  VRSTVA 5: Data                                                         │
│  ├── Encryption at rest (KMS, customer-managed keys)                  │
│  ├── Encryption in transit (TLS 1.2+)                                  │
│  ├── Field-level encryption (citlivá pole)                            │
│  ├── Tokenization (PAN, personal data)                                 │
│  ├── Data masking (logs, non-prod environments)                       │
│  └── Backup encryption                                                  │
│                                                                         │
│  VRSTVA 6: Monitoring & Response                                        │
│  ├── CloudTrail (API audit logging)                                    │
│  ├── GuardDuty (threat detection)                                      │
│  ├── Security Hub (centralized findings)                               │
│  ├── SIEM integration                                                   │
│  └── Automated incident response (EventBridge + Lambda)               │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### Příklad průniku a jak vrstvy pomáhají

```
Scénář: Útočník získal přístup k API credentials

┌────────────────────────────────────────────────────────────────────────┐
│ BEZ Defense in Depth:                                                  │
│ Credentials → API → Data → BREACH                                      │
│                                                                        │
│ S Defense in Depth:                                                    │
│ Credentials                                                            │
│     ↓                                                                  │
│ WAF → ✅ Prošel (validní request format)                              │
│     ↓                                                                  │
│ API Gateway Auth → ✅ Prošel (platné credentials)                     │
│     ↓                                                                  │
│ Rate Limiting → ⚠️ Omezen na 100 req/min                              │
│     ↓                                                                  │
│ Network (Security Group) → ✅ Prošel                                  │
│     ↓                                                                  │
│ Application → ⚠️ Přístup pouze k vlastním datům (authz check)        │
│     ↓                                                                  │
│ Data → 🔒 Šifrováno, pouze omezeně čitelné                           │
│     ↓                                                                  │
│ Monitoring → 🚨 GuardDuty detekuje anomální chování                   │
│     ↓                                                                  │
│ Response → 🔒 Automatická revokace credentials                        │
│                                                                        │
│ Výsledek: Útočník získal omezená data, byl detekován a zastaven      │
└────────────────────────────────────────────────────────────────────────┘
```

---

## 2. Zero Trust Architecture

### Co to je

Bezpečnostní model založený na principu "nikdy nedůvěřuj, vždy ověřuj" - na rozdíl od tradičního perimeter-based modelu, kde se důvěřuje všemu uvnitř sítě.

```
┌─────────────────────────────────────────────────────────────────────────┐
│              TRADIČNÍ MODEL vs. ZERO TRUST                              │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  TRADIČNÍ (Castle & Moat):                                             │
│  ┌─────────────────────────────────────────────┐                       │
│  │ 🏰 CORPORATE NETWORK                         │                       │
│  │                                              │                       │
│  │   😊 Trusted    😊 Trusted    😊 Trusted    │   🌐 Internet         │
│  │                                              │       │               │
│  │   Všichni uvnitř jsou důvěryhodní           │       │               │
│  │                                              │◀──────┤ Firewall     │
│  │   😈 Insider threat = plný přístup          │       │               │
│  │                                              │   🔒 Perimeter       │
│  └─────────────────────────────────────────────┘                       │
│                                                                         │
│  ZERO TRUST:                                                           │
│  ┌─────────────────────────────────────────────┐                       │
│  │                                              │                       │
│  │   🔐──🔐──🔐    Každý přístup ověřen        │   🌐 Internet         │
│  │    │    │    │                               │       │               │
│  │   🔐──🔐──🔐    Mikrosegmentace              │       │               │
│  │    │    │    │                               │◀──────┤               │
│  │   🔐──🔐──🔐    Least privilege             │       │               │
│  │                                              │                       │
│  │   😈 Insider = stále musí se autentizovat   │                       │
│  └─────────────────────────────────────────────┘                       │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### Základní pilíře Zero Trust

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    PILÍŘE ZERO TRUST                                    │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  1. VERIFY EXPLICITLY (Explicitně ověřuj)                              │
│     ├── Autentizuj každý request (ne jen session)                      │
│     ├── Validuj kontext (device, location, time, behavior)             │
│     ├── Continuous authentication (ne jednorázové)                     │
│     └── Strong authentication (MFA vždy)                               │
│                                                                         │
│  2. LEAST PRIVILEGE ACCESS (Minimální oprávnění)                       │
│     ├── Just-in-time access (oprávnění jen když potřeba)              │
│     ├── Just-enough-access (jen nezbytná oprávnění)                   │
│     ├── Risk-based adaptive policies                                   │
│     └── Segmentace na základě citlivosti                               │
│                                                                         │
│  3. ASSUME BREACH (Předpokládej průnik)                                │
│     ├── Minimalizuj blast radius (segmentace)                          │
│     ├── End-to-end encryption                                          │
│     ├── Continuous monitoring                                          │
│     └── Automated threat response                                      │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### Implementace v bankovní architektuře

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    ZERO TRUST V PRAXI                                   │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  USER/DEVICE                 POLICY ENGINE              RESOURCE        │
│  ┌─────────┐                ┌─────────────┐            ┌─────────┐     │
│  │ Mobile  │                │             │            │ Account │     │
│  │ App     │───Request────▶│  Evaluate:  │──Allow───▶│ Service │     │
│  │         │                │  • Identity │            │         │     │
│  │ Context:│                │  • Device   │            └─────────┘     │
│  │ • DeviceID│              │  • Location │                            │
│  │ • Location│              │  • Behavior │            ┌─────────┐     │
│  │ • Time   │               │  • Risk score│           │ Payment │     │
│  │ • Behavior│              │             │──Deny────▶│ Service │     │
│  └─────────┘                └─────────────┘            │ (blocked)│    │
│                                    │                   └─────────┘     │
│                                    ▼                                    │
│                             ┌─────────────┐                            │
│                             │   Logging   │                            │
│                             │  & Analytics│                            │
│                             └─────────────┘                            │
│                                                                         │
│  SIGNÁLY PRO ROZHODOVÁNÍ:                                              │
│  ┌────────────────────────────────────────────────────────────────┐   │
│  │ Signal              │ Low Risk        │ High Risk              │   │
│  ├─────────────────────┼─────────────────┼────────────────────────│   │
│  │ Device              │ Known, managed  │ Unknown, jailbroken    │   │
│  │ Location            │ Usual country   │ New country, VPN       │   │
│  │ Time                │ Business hours  │ 3 AM, unusual          │   │
│  │ Behavior            │ Normal patterns │ Rapid transactions     │   │
│  │ Network             │ Corporate/home  │ Tor, suspicious IP     │   │
│  │ Transaction         │ Usual amount    │ Max limit, new payee   │   │
│  └────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  AKCE NA ZÁKLADĚ RIZIKA:                                               │
│  • Low risk → Allow                                                     │
│  • Medium risk → Step-up authentication (additional MFA)               │
│  • High risk → Block + alert                                           │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### Service-to-Service Zero Trust

```
┌─────────────────────────────────────────────────────────────────────────┐
│              ZERO TRUST MEZI MICROSERVICES                              │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  Tradiční přístup:                                                     │
│  Account Service ────── VPC Network ────── Payment Service             │
│        │                                         │                      │
│        └── "Jsme ve stejné VPC, důvěřuji ti" ───┘                      │
│            ❌ Problém: Kompromitovaná služba má přístup ke všemu       │
│                                                                         │
│  Zero Trust přístup:                                                   │
│  ┌─────────────┐         ┌─────────────┐         ┌─────────────┐       │
│  │  Account    │──mTLS──▶│   Service   │──mTLS──▶│  Payment    │       │
│  │  Service    │         │   Mesh      │         │  Service    │       │
│  │             │         │  (Envoy)    │         │             │       │
│  │ IAM Role A  │         │             │         │ IAM Role P  │       │
│  └─────────────┘         └──────┬──────┘         └─────────────┘       │
│                                 │                                       │
│                          ┌──────▼──────┐                               │
│                          │   Policy:    │                               │
│                          │ A → P: Allow │                               │
│                          │ A → C: Deny  │                               │
│                          └─────────────┘                               │
│                                                                         │
│  Implementační kroky:                                                   │
│  1. Každá služba má vlastní identitu (IAM Role, Service Account)       │
│  2. mTLS mezi všemi službami (vzájemná autentizace)                   │
│  3. Explicitní autorizační politiky (kdo může volat koho)             │
│  4. Network policies (i když mTLS, omezit na síťové úrovni)           │
│  5. Audit logging všech service-to-service volání                      │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 3. Separation of Concerns (Oddělení odpovědností)

### Co to je

Princip, který říká, že každá komponenta systému by měla být zodpovědná za jednu jasně definovanou oblast funkcionality. Změna v jedné oblasti by neměla vyžadovat změny v jiných oblastech.

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    SEPARATION OF CONCERNS                               │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  ❌ ŠPATNĚ - Monolitická "God Service":                                │
│  ┌────────────────────────────────────────────────────────────────┐   │
│  │                     BANKING SERVICE                             │   │
│  │  ┌──────────────────────────────────────────────────────────┐  │   │
│  │  │ • Autentizace uživatelů                                   │  │   │
│  │  │ • Správa účtů                                             │  │   │
│  │  │ • Platby                                                  │  │   │
│  │  │ • Karty                                                   │  │   │
│  │  │ • Reporting                                               │  │   │
│  │  │ • Notifikace                                              │  │   │
│  │  │ • Fraud detection                                         │  │   │
│  │  │ • Audit logging                                           │  │   │
│  │  └──────────────────────────────────────────────────────────┘  │   │
│  └────────────────────────────────────────────────────────────────┘   │
│  Problémy: Změna v platbách může rozbít karty, nelze škálovat         │
│            nezávisle, jeden tým nemůže deployovat bez druhého          │
│                                                                         │
│  ✅ SPRÁVNĚ - Oddělené domény:                                         │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐    │
│  │ Identity │ │ Accounts │ │ Payments │ │  Cards   │ │Notificat.│    │
│  │ Service  │ │ Service  │ │ Service  │ │ Service  │ │ Service  │    │
│  ├──────────┤ ├──────────┤ ├──────────┤ ├──────────┤ ├──────────┤    │
│  │• AuthN   │ │• Balances│ │• Domestic│ │• Limits  │ │• Email   │    │
│  │• AuthZ   │ │• Statemnt│ │• SEPA    │ │• Block   │ │• SMS     │    │
│  │• MFA     │ │• Limits  │ │• Instant │ │• PIN     │ │• Push    │    │
│  │• Session │ │          │ │• FX      │ │• Virtual │ │          │    │
│  └──────────┘ └──────────┘ └──────────┘ └──────────┘ └──────────┘    │
│       │            │            │            │            │           │
│       └────────────┴────────────┴────────────┴────────────┘           │
│                              Event Bus                                 │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### Vrstvy separace v bankovní aplikaci

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    HORIZONTÁLNÍ SEPARACE (VRSTVY)                       │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                    PRESENTATION LAYER                            │   │
│  │  • API Gateway (REST/GraphQL exposure)                          │   │
│  │  • Request/Response transformation                               │   │
│  │  • Input validation                                              │   │
│  │  • Rate limiting                                                 │   │
│  │  Odpovědnost: JAK se data prezentují externě                    │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                              │                                          │
│                              ▼                                          │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                    APPLICATION LAYER                             │   │
│  │  • Orchestrace workflow                                          │   │
│  │  • Business process coordination                                 │   │
│  │  • Transaction management                                        │   │
│  │  Odpovědnost: JAK se procesy koordinují                         │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                              │                                          │
│                              ▼                                          │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                    DOMAIN LAYER                                  │   │
│  │  • Business rules                                                │   │
│  │  • Domain entities                                               │   │
│  │  • Business validations                                          │   │
│  │  Odpovědnost: CO je business logika                             │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                              │                                          │
│                              ▼                                          │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                    INFRASTRUCTURE LAYER                          │   │
│  │  • Database access                                               │   │
│  │  • External service integration                                  │   │
│  │  • Messaging                                                     │   │
│  │  Odpovědnost: JAK se data ukládají a přenášejí                  │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### Vertikální separace (Domain-Driven Design)

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    BOUNDED CONTEXTS V BANCE                             │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  Každý bounded context:                                                 │
│  • Vlastní datový model                                                 │
│  • Vlastní terminologie (ubiquitous language)                          │
│  • Vlastní tým                                                          │
│  • Vlastní deployment lifecycle                                         │
│                                                                         │
│  ┌───────────────┐  ┌───────────────┐  ┌───────────────┐               │
│  │   CUSTOMER    │  │   ACCOUNTS    │  │   PAYMENTS    │               │
│  │   CONTEXT     │  │   CONTEXT     │  │   CONTEXT     │               │
│  ├───────────────┤  ├───────────────┤  ├───────────────┤               │
│  │               │  │               │  │               │               │
│  │ "Customer"    │  │ "Account      │  │ "Payment      │               │
│  │ "Segment"     │  │  Holder"      │  │  Initiator"   │               │
│  │ "KYC Status"  │  │ "Balance"     │  │ "Beneficiary" │               │
│  │               │  │ "Statement"   │  │ "Amount"      │               │
│  │               │  │               │  │               │               │
│  │ Team: CRM     │  │ Team: Core    │  │ Team: Payments│               │
│  └───────┬───────┘  └───────┬───────┘  └───────┬───────┘               │
│          │                  │                  │                        │
│          │    CONTEXT MAP (jak spolu komunikují)                       │
│          │                  │                  │                        │
│          └──────────────────┼──────────────────┘                        │
│                             │                                           │
│                    Anti-Corruption Layer                                │
│                    (překlad mezi kontexty)                             │
│                                                                         │
│  Příklad:                                                               │
│  • V CUSTOMER context: "Customer" má demografické údaje               │
│  • V ACCOUNTS context: "Account Holder" má jen ID a jméno             │
│  • V PAYMENTS context: "Payment Initiator" má jen autorizační info    │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### Výhody v bankovním kontextu

|Výhoda|Příklad v bankovnictví|
|---|---|
|**Nezávislý vývoj**|Tým Payments může deployovat bez čekání na tým Cards|
|**Nezávislé škálování**|Payment service škáluje 10x během salary day, ostatní ne|
|**Izolace selhání**|Bug v notifikacích neovlivní platby|
|**Specializace týmů**|Experti na fraud detection vs. experti na UX|
|**Compliance**|PCI-DSS scope omezen jen na relevantní komponenty|
|**Testovatelnost**|Jednotlivé komponenty lze testovat izolovaně|

---

## 4. Loose Coupling (Volné vazby)

### Co to je

Komponenty systému by měly mít minimální závislosti na interních detailech jiných komponent. Změna implementace jedné komponenty by neměla vyžadovat změnu jiných komponent.

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    TIGHT vs. LOOSE COUPLING                             │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  ❌ TIGHT COUPLING (Těsné vazby):                                      │
│                                                                         │
│  ┌──────────┐    Synchronní volání     ┌──────────┐                   │
│  │ Payment  │─────────────────────────▶│ Account  │                   │
│  │ Service  │    Zná interní strukturu │ Service  │                   │
│  │          │    Čeká na odpověď       │          │                   │
│  │          │◀─────────────────────────│          │                   │
│  └──────────┘    Sdílená databáze?     └──────────┘                   │
│        │                                     │                         │
│        └─────────────┬───────────────────────┘                         │
│                      ▼                                                  │
│             Změna v Account Service                                     │
│             = změna v Payment Service                                   │
│             = deployment obou současně                                  │
│             = single point of failure                                   │
│                                                                         │
│  ✅ LOOSE COUPLING (Volné vazby):                                      │
│                                                                         │
│  ┌──────────┐                          ┌──────────┐                   │
│  │ Payment  │──────▶ Event ──────────▶│ Account  │                   │
│  │ Service  │      "PaymentCreated"    │ Service  │                   │
│  │          │                          │          │                   │
│  │          │       Nezná detaily      │          │                   │
│  │          │       implementace       │          │                   │
│  └──────────┘                          └──────────┘                   │
│        │                                     │                         │
│        │    Komunikace přes:                 │                         │
│        │    • Definované kontrakty (API)     │                         │
│        │    • Asynchronní zprávy (events)    │                         │
│        │    • Abstraktní interface           │                         │
│        │                                     │                         │
│        └───── Nezávislý deployment ──────────┘                         │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### Techniky pro dosažení loose coupling

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    TECHNIKY LOOSE COUPLING                              │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  1. ASYNCHRONNÍ KOMUNIKACE                                             │
│  ┌──────────┐     ┌─────────────┐     ┌──────────┐                    │
│  │ Producer │────▶│ Message     │────▶│ Consumer │                    │
│  │          │     │ Queue/Topic │     │          │                    │
│  └──────────┘     └─────────────┘     └──────────┘                    │
│  • Producer nezná consumera                                            │
│  • Consumer zpracuje když může                                         │
│  • Fronta absorbuje špičky                                            │
│  • AWS: SQS, SNS, EventBridge, MSK (Kafka)                            │
│                                                                         │
│  2. API CONTRACTS (OpenAPI/AsyncAPI)                                   │
│  ┌──────────┐     ┌─────────────┐     ┌──────────┐                    │
│  │ Service  │────▶│  Contract   │◀────│ Service  │                    │
│  │    A     │     │ (OpenAPI)   │     │    B     │                    │
│  └──────────┘     └─────────────┘     └──────────┘                    │
│  • Služby závisí na kontraktu, ne na sobě                             │
│  • Contract-first development                                          │
│  • Backward compatibility pravidla                                     │
│                                                                         │
│  3. EVENT-DRIVEN ARCHITECTURE                                          │
│  ┌──────────┐                         ┌──────────┐                    │
│  │ Account  │──"AccountCreated"─────▶│ Notification│                  │
│  │ Service  │                         │ Service   │                   │
│  └──────────┘──"AccountCreated"─────▶┌──────────┐                    │
│                                       │ Analytics │                    │
│  • Publisher neví kdo poslouchá       │ Service   │                   │
│  • Noví subscribers bez změny         └──────────┘                    │
│                                                                         │
│  4. ANTI-CORRUPTION LAYER                                              │
│  ┌──────────┐     ┌─────────────┐     ┌──────────┐                    │
│  │ Modern   │────▶│    ACL      │────▶│ Legacy   │                    │
│  │ Service  │     │ (Translator)│     │ System   │                    │
│  └──────────┘     └─────────────┘     └──────────┘                    │
│  • Izolace od legacy systému                                          │
│  • Překlad mezi modely                                                │
│  • Změna legacy nevyžaduje změnu moderního                            │
│                                                                         │
│  5. INTERFACE SEGREGATION                                              │
│  • Malé, specifické interface                                          │
│  • Klient závisí jen na tom, co potřebuje                             │
│  • Změny neovlivní nezávislé klienty                                  │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### Příklad: Platební workflow s loose coupling

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    PAYMENT WORKFLOW                                     │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  ❌ Tight coupling (synchronní řetěz):                                 │
│                                                                         │
│  Client → Payment → Account → Fraud → AML → Core → Notification       │
│     ↑                                                    │              │
│     └────────────── Čeká na celý řetěz ──────────────────┘              │
│                                                                         │
│  Problémy:                                                              │
│  • Latence = suma všech služeb                                         │
│  • Selhání jedné = selhání celého flow                                 │
│  • Nelze škálovat nezávisle                                            │
│                                                                         │
│  ✅ Loose coupling (event-driven):                                     │
│                                                                         │
│  Client → Payment API                                                   │
│              │                                                          │
│              ▼                                                          │
│         ┌─────────┐                                                    │
│         │ Payment │──"PaymentInitiated"──┬──▶ Fraud Service           │
│         │ Service │                      │                             │
│         └────┬────┘                      ├──▶ AML Service              │
│              │                           │                             │
│         Responds                         ├──▶ Account Service          │
│         immediately                      │                             │
│         with PaymentId                   └──▶ Notification Service    │
│              │                                                          │
│              ▼                                                          │
│         Client                                                          │
│         (polls status or webhook)                                       │
│                                                                         │
│  Výhody:                                                                │
│  • Okamžitá odpověď klientovi                                          │
│  • Služby zpracovávají nezávisle                                       │
│  • Selhání Fraud nezablokuje AML                                       │
│  • Každá služba škáluje podle potřeby                                  │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 5. High Cohesion (Vysoká soudržnost)

### Co to je

Komponenta by měla obsahovat funkcionalitu, která logicky patří k sobě a pracuje se stejnými daty. Opak "spaghetti code" kde nesouvisející funkce jsou smíchány dohromady.

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    LOW vs. HIGH COHESION                                │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  ❌ LOW COHESION (Nízká soudržnost):                                   │
│                                                                         │
│  ┌────────────────────────────────────────┐                            │
│  │         UTILITY SERVICE                 │                            │
│  │  ┌────────────────────────────────┐    │                            │
│  │  │ • Send email                   │    │ ← Notifikace               │
│  │  │ • Calculate interest           │    │ ← Finanční výpočty         │
│  │  │ • Validate IBAN                │    │ ← Validace                 │
│  │  │ • Generate PDF statement       │    │ ← Dokumenty                │
│  │  │ • Convert currency             │    │ ← FX                       │
│  │  │ • Check fraud rules            │    │ ← Security                 │
│  │  └────────────────────────────────┘    │                            │
│  └────────────────────────────────────────┘                            │
│  Problém: Změna v email logice vyžaduje retest fraud rules            │
│                                                                         │
│  ✅ HIGH COHESION (Vysoká soudržnost):                                 │
│                                                                         │
│  ┌──────────────┐ ┌──────────────┐ ┌──────────────┐ ┌──────────────┐  │
│  │ Notification │ │   Interest   │ │  Validation  │ │   Document   │  │
│  │   Service    │ │   Service    │ │   Service    │ │   Service    │  │
│  ├──────────────┤ ├──────────────┤ ├──────────────┤ ├──────────────┤  │
│  │ • Email      │ │ • Calculate  │ │ • IBAN       │ │ • PDF        │  │
│  │ • SMS        │ │ • Accrue     │ │ • BIC        │ │ • Statement  │  │
│  │ • Push       │ │ • Compound   │ │ • Account#   │ │ • Contract   │  │
│  │ • Templates  │ │ • Day count  │ │ • Amount     │ │ • Receipt    │  │
│  └──────────────┘ └──────────────┘ └──────────────┘ └──────────────┘  │
│                                                                         │
│  Každá služba má JEDEN důvod ke změně (Single Responsibility)         │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### Měření koheze

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    INDIKÁTORY KOHEZE                                    │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  VYSOKÁ KOHEZE (dobře):                                                │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ Account Service                                                  │   │
│  │                                                                  │   │
│  │  Všechny metody pracují s:                                      │   │
│  │  • Account entity                                                │   │
│  │  • Account database table                                        │   │
│  │  • Account-related business rules                               │   │
│  │                                                                  │   │
│  │  getBalance() ─────┐                                            │   │
│  │  getTransactions() ├──── Všechny sdílejí Account context        │   │
│  │  updateLimits() ───┤                                            │   │
│  │  closeAccount() ───┘                                            │   │
│  │                                                                  │   │
│  │  Test: Lze popsat účel služby JEDNOU větou?                     │   │
│  │  "Spravuje životní cyklus a stav bankovních účtů"  ✅           │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  NÍZKÁ KOHEZE (špatně):                                                │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ Banking Utils Service                                            │   │
│  │                                                                  │   │
│  │  calculateInterest() ── Finance                                 │   │
│  │  sendNotification() ─── Messaging                               │   │
│  │  validateIBAN() ─────── Validation                              │   │
│  │  generateReport() ───── Reporting                               │   │
│  │                                                                  │   │
│  │  Test: Lze popsat účel služby JEDNOU větou?                     │   │
│  │  "Dělá různé věci pro bankovnictví"  ❌                         │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  PRAVIDLA PRO VYSOKOU KOHEZI:                                          │
│  • Jedna služba = jedna business capability                            │
│  • Všechny metody sdílejí stejná data/entity                          │
│  • Změny přicházejí ze stejného důvodu                                │
│  • Tým vlastní celou doménu end-to-end                                │
│  • Službu lze pojmenovat podstatným jménem, ne slovesem               │
│    ✅ "PaymentService" ne ❌ "DoThingsService"                        │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### Vztah mezi Cohesion a Coupling

```
┌─────────────────────────────────────────────────────────────────────────┐
│              COHESION vs. COUPLING MATRIX                               │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│                        COUPLING                                         │
│                   Low            High                                   │
│              ┌─────────────┬─────────────┐                             │
│         High │     ✅      │     ⚠️      │                             │
│   COHESION   │   IDEAL     │  Problematic │                             │
│              │             │  (fixable)   │                             │
│              ├─────────────┼─────────────┤                             │
│         Low  │     ⚠️      │     ❌      │                             │
│              │  Scattered  │   WORST     │                             │
│              │  (review)   │  (refactor) │                             │
│              └─────────────┴─────────────┘                             │
│                                                                         │
│  Cíl: High Cohesion + Low Coupling                                     │
│                                                                         │
│  Příklad v bance:                                                       │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │ ✅ High Cohesion, Low Coupling:                                  │  │
│  │    Payment Service                                                │  │
│  │    • Vše o platbách na jednom místě                              │  │
│  │    • Komunikuje s ostatními přes eventy                          │  │
│  │    • Vlastní databáze                                            │  │
│  ├──────────────────────────────────────────────────────────────────┤  │
│  │ ❌ Low Cohesion, High Coupling:                                  │  │
│  │    TransactionProcessor                                           │  │
│  │    • Zpracovává platby, karty, FX, fees                          │  │
│  │    • Synchronně volá 10 dalších služeb                           │  │
│  │    • Sdílí databázi s Account Service                            │  │
│  └──────────────────────────────────────────────────────────────────┘  │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 6. Idempotency (Idempotence)

### Co to je

Operace je idempotentní, pokud její opakované provedení má stejný efekt jako jednorázové provedení. Kritické pro distribuované systémy, kde může dojít k opakovanému volání (retry, network issues, duplicate messages).

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    IDEMPOTENCY - ZÁKLADNÍ KONCEPT                       │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  Matematicky: f(f(x)) = f(x)                                           │
│                                                                         │
│  PŘIROZENĚ IDEMPOTENTNÍ operace:                                       │
│  • GET /accounts/123        → Vždy vrátí stejný stav                   │
│  • PUT /accounts/123        → Nastaví stav (ne přičte)                 │
│  • DELETE /accounts/123     → Po prvním smazání už nic nedělá         │
│                                                                         │
│  NON-IDEMPOTENTNÍ operace (nebezpečné):                               │
│  • POST /payments           → Každé volání vytvoří novou platbu!       │
│  • POST /transfers          → Každé volání převede peníze!            │
│  • PATCH /accounts/123/balance (increment) → Každé volání přičte!     │
│                                                                         │
│  PROBLÉM V DISTRIBUOVANÉM SYSTÉMU:                                     │
│                                                                         │
│  Client ──POST /payments──▶ API ──────▶ Backend                        │
│     │                          │           │                            │
│     │                          │    ✅ Platba provedena                │
│     │                          │           │                            │
│     │     ⚡ Network timeout   │◀──────────┘                            │
│     │                          X                                        │
│     │                                                                   │
│     │  Client neví jestli platba prošla!                               │
│     │  Co udělá? → RETRY                                               │
│     │                                                                   │
│     └──POST /payments──▶ API ──────▶ Backend                           │
│                                         │                               │
│                               💥 DUPLICITNÍ PLATBA!                    │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### Proč je kritická v bankovnictví

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    DŮSLEDKY BEZ IDEMPOTENCE                             │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  Scénář 1: Duplicitní platba                                           │
│  • Klient platí 10,000 CZK za nájem                                    │
│  • Timeout při potvrzení                                               │
│  • Klient zkusí znovu                                                  │
│  • Výsledek: 2x 10,000 CZK strženo                                     │
│  • Dopad: Reklamace, náklady, ztráta důvěry                           │
│                                                                         │
│  Scénář 2: Duplicitní příkaz k nákupu                                  │
│  • Investor kupuje akcie za 100,000 CZK                                │
│  • Pomalá odpověď, refresh stránky                                     │
│  • Výsledek: Koupeno 2x více akcií                                     │
│  • Dopad: Právní spory, regulatorní problémy                          │
│                                                                         │
│  Scénář 3: At-least-once delivery (messaging)                          │
│  • Kafka/SQS může doručit zprávu vícekrát                             │
│  • Bez idempotence = vícenásobné zpracování                           │
│  • Dopad: Nekonzistentní data, chybné zůstatky                        │
│                                                                         │
│  REGULATORNÍ DŮSLEDKY:                                                  │
│  • PSD2 vyžaduje přesné zpracování plateb                             │
│  • Finanční ztráty = odpovědnost banky                                │
│  • Možné sankce od regulátora                                          │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### Implementační strategie

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    STRATEGIE IDEMPOTENCE                                │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  STRATEGIE 1: Idempotency Key (nejběžnější pro API)                   │
│                                                                         │
│  ┌────────┐                    ┌─────────────┐                         │
│  │ Client │───────────────────▶│ API Gateway │                         │
│  │        │ POST /payments     │             │                         │
│  │        │ Idempotency-Key:   │             │                         │
│  │        │ "uuid-123-abc"     │             │                         │
│  └────────┘                    └──────┬──────┘                         │
│                                       │                                 │
│                                       ▼                                 │
│                             ┌─────────────────┐                        │
│                             │ Idempotency     │                        │
│                             │ Store (DynamoDB)│                        │
│                             │                 │                        │
│                             │ Key: uuid-123   │                        │
│                             │ Status: pending │                        │
│                             │ Created: now    │                        │
│                             └────────┬────────┘                        │
│                                      │                                  │
│                   ┌──────────────────┼──────────────────┐              │
│                   │                  │                  │              │
│                   ▼                  ▼                  ▼              │
│            Key neexistuje     Key existuje,      Key existuje,         │
│                               status=pending     status=complete       │
│                   │                  │                  │              │
│                   ▼                  ▼                  ▼              │
│            Zpracuj platbu      Vrať 409         Vrať uložený           │
│            Ulož výsledek       Conflict         výsledek               │
│                                (processing)     (deduplikace)          │
│                                                                         │
│  TTL pro idempotency keys: typicky 24-48 hodin                        │
│                                                                         │
│  STRATEGIE 2: Natural Idempotency Key                                  │
│                                                                         │
│  Použití business identifikátorů místo UUID:                          │
│  • Číslo faktury jako payment reference                                │
│  • Kombinace: accountId + date + sequence                              │
│  • Hash request body                                                    │
│                                                                         │
│  Výhoda: Client nemusí generovat UUID                                  │
│  Nevýhoda: Složitější logika, možné kolize                            │
│                                                                         │
│  STRATEGIE 3: Conditional Writes (Database level)                      │
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ INSERT INTO payments (id, amount, status)                        │   │
│  │ VALUES ('pay-123', 1000, 'pending')                             │   │
│  │ ON CONFLICT (id) DO NOTHING;                                    │   │
│  │                                                                  │   │
│  │ -- nebo --                                                       │   │
│  │                                                                  │   │
│  │ DynamoDB: ConditionExpression: "attribute_not_exists(id)"       │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  STRATEGIE 4: Event Deduplication (pro messaging)                      │
│                                                                         │
│  ┌───────────┐     ┌───────────┐     ┌───────────┐                    │
│  │  Message  │────▶│   Dedup   │────▶│  Consumer │                    │
│  │   Queue   │     │   Store   │     │           │                    │
│  │           │     │           │     │           │                    │
│  │ msg-id:X  │     │ Seen: X?  │     │ Process   │                    │
│  │ msg-id:X  │     │ Yes→Skip  │     │ only once │                    │
│  └───────────┘     └───────────┘     └───────────┘                    │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### Idempotency v různých vrstvách

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    IDEMPOTENCY NA KAŽDÉ VRSTVĚ                          │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  VRSTVA              MECHANISMUS                PŘÍKLAD                 │
│  ─────────────────────────────────────────────────────────────────────  │
│                                                                         │
│  API Gateway         Idempotency-Key header     POST /payments s UUID   │
│                      Request deduplication      Cachování response      │
│                                                                         │
│  Application         Idempotency store          DynamoDB tabulka        │
│                      Status tracking            pending → complete      │
│                                                                         │
│  Database            Unique constraints         payment_reference UNIQUE│
│                      Conditional writes         INSERT ... ON CONFLICT  │
│                      Optimistic locking         Version field check     │
│                                                                         │
│  Messaging           Message ID tracking        Kafka offset, SQS dedup │
│                      Consumer deduplication     Processed message store │
│                                                                         │
│  External APIs       Transaction IDs            Core banking TX ID      │
│                      Inquiry before action      Check if already done   │
│                                                                         │
│  BEST PRACTICE: Idempotence na KAŽDÉ vrstvě (Defense in Depth)        │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 7. Eventually Consistent vs. Strong Consistency

### Co to je

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    CONSISTENCY MODELS                                   │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  STRONG CONSISTENCY (Silná konzistence):                               │
│  ════════════════════════════════════════                              │
│  Po dokončení zápisu všechny následné čtení vrátí novou hodnotu.      │
│                                                                         │
│  ┌────────┐     Write X=5     ┌────────┐                              │
│  │ Client │──────────────────▶│  DB    │                              │
│  │   A    │                   │        │                              │
│  └────────┘                   │  X=5   │◀─────┐                       │
│                               └────────┘      │                        │
│  ┌────────┐     Read X        ┌────────┐      │                       │
│  │ Client │──────────────────▶│  DB    │──────┘                       │
│  │   B    │◀──── X=5 ─────────│        │   Vždy vidí 5               │
│  └────────┘                   └────────┘                              │
│                                                                         │
│  Čas: ──Write──┬──────────────────────────────────▶                   │
│                │                                                        │
│                └── Od tohoto okamžiku všichni vidí novou hodnotu      │
│                                                                         │
│  EVENTUAL CONSISTENCY (Konečná konzistence):                           │
│  ════════════════════════════════════════════                          │
│  Po zápisu mohou čtení dočasně vracet starou hodnotu.                 │
│  Systém se NAKONEC dostane do konzistentního stavu.                   │
│                                                                         │
│  ┌────────┐     Write X=5     ┌────────┐                              │
│  │ Client │──────────────────▶│ Node 1 │                              │
│  │   A    │                   │  X=5   │                              │
│  └────────┘                   └───┬────┘                              │
│                                   │ Replication                        │
│  ┌────────┐     Read X        ┌───▼────┐                              │
│  │ Client │──────────────────▶│ Node 2 │                              │
│  │   B    │◀──── X=1 ─────────│  X=1   │   Ještě stará hodnota!      │
│  └────────┘                   └────────┘                              │
│                                                                         │
│  Čas: ──Write──┬────────────┬───────────────────▶                     │
│                │            │                                          │
│                │ Replication│                                          │
│                │   delay    │                                          │
│                │            └── Teprve teď všichni vidí 5             │
│                │                                                        │
│                └── V tomto okně mohou být nekonzistentní data         │
│                    (stale reads)                                       │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### Trade-offs (CAP Theorem)

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    CAP THEOREM                                          │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  V distribuovaném systému můžeš mít pouze 2 ze 3:                      │
│                                                                         │
│                         Consistency                                     │
│                             △                                           │
│                            ╱ ╲                                          │
│                           ╱   ╲                                         │
│                          ╱     ╲                                        │
│                         ╱  CP   ╲                                       │
│                        ╱ systems ╲                                      │
│                       ╱           ╲                                     │
│                      ╱             ╲                                    │
│                     ╱───────────────╲                                   │
│                    ╱    CA systems   ╲                                  │
│                   ╱   (single node)   ╲                                 │
│                  ╱                     ╲                                │
│                 △───────────────────────△                               │
│         Availability                 Partition                          │
│                        AP systems     Tolerance                         │
│                                                                         │
│  V praxi: Network partitions SE STÁVAJÍ                                │
│  → Musíš si vybrat mezi C a A                                          │
│                                                                         │
│  ┌────────────────────────────────────────────────────────────────┐   │
│  │ Volba            │ Příklad              │ Trade-off             │   │
│  ├───────────────────┼──────────────────────┼───────────────────────│   │
│  │ CP (Consistency) │ Tradiční RDBMS       │ Při partition:        │   │
│  │                  │ Bank core system     │ systém nedostupný     │   │
│  ├───────────────────┼──────────────────────┼───────────────────────│   │
│  │ AP (Availability)│ DynamoDB, Cassandra  │ Při partition:        │   │
│  │                  │ Cache, Session store │ možné stale reads     │   │
│  └────────────────────────────────────────────────────────────────┘   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### Kdy použít co v bankovnictví

```
┌─────────────────────────────────────────────────────────────────────────┐
│              ROZHODOVACÍ MATICE PRO BANKOVNICTVÍ                        │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  STRONG CONSISTENCY - VYŽADOVÁNO PRO:                                  │
│  ══════════════════════════════════════                                │
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ Use Case                  │ Důvod                               │   │
│  ├────────────────────────────┼─────────────────────────────────────│   │
│  │ Account balance           │ Nesmí být záporný ani nesprávný    │   │
│  │ Payment execution         │ Peníze nesmí zmizet/zdvojit        │   │
│  │ Available funds check     │ Předschválení musí být přesné      │   │
│  │ Card authorization        │ Limit musí být aktuální            │   │
│  │ Loan disbursement         │ Přesná částka, přesný čas          │   │
│  │ Standing order execution  │ Musí proběhnout právě jednou       │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  Implementace AWS:                                                      │
│  • RDS/Aurora s synchronní replikací                                   │
│  • DynamoDB s ConsistentRead=true                                      │
│  • Transakce přes DynamoDB Transactions                                │
│                                                                         │
│  EVENTUAL CONSISTENCY - AKCEPTOVATELNÁ PRO:                            │
│  ════════════════════════════════════════════                          │
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ Use Case                  │ Důvod                               │   │
│  ├────────────────────────────┼─────────────────────────────────────│   │
│  │ Transaction history       │ Krátké zpoždění akceptovatelné     │   │
│  │ Analytics dashboards      │ Data mohou být minuty/hodiny stará │   │
│  │ Notification delivery     │ Zpoždění v řádu sekund OK          │   │
│  │ User preferences          │ Není kritické                       │   │
│  │ Product catalog           │ Změny jsou vzácné                   │   │
│  │ Branch/ATM locator        │ Statická data                       │   │
│  │ Fraud scoring (read)      │ Model se aktualizuje periodicky    │   │
│  │ Audit logs                │ Write-heavy, read rarely           │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  Implementace AWS:                                                      │
│  • DynamoDB (default = eventually consistent)                          │
│  • ElastiCache (read replicas)                                         │
│  • Aurora read replicas                                                 │
│  • Event-driven updates (SNS → Lambda → DynamoDB)                      │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### Praktické patterny pro konzistenci

```
┌─────────────────────────────────────────────────────────────────────────┐
│              PATTERNY PRO ŘÍZENÍ KONZISTENCE                            │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  PATTERN 1: Read Your Own Writes                                       │
│  ═══════════════════════════════                                       │
│  Uživatel vždy vidí své vlastní změny.                                 │
│                                                                         │
│  ┌────────┐  Write   ┌────────┐                                       │
│  │ Client │─────────▶│ Primary│                                       │
│  │        │          └────┬───┘                                       │
│  │        │               │                                            │
│  │        │  Read    ┌────▼───┐                                       │
│  │        │─────────▶│ Primary│  ← Čti z primary po vlastním zápisu   │
│  └────────┘          └────────┘                                       │
│                                                                         │
│  Implementace:                                                          │
│  • Session affinity k primary                                          │
│  • Timestamp/version tracking                                          │
│  • DynamoDB: ConsistentRead pro vlastní data                          │
│                                                                         │
│  PATTERN 2: Monotonic Reads                                            │
│  ═══════════════════════════                                           │
│  Uživatel nikdy nevidí starší data než dříve.                         │
│                                                                         │
│  ┌────────┐  Read 1   ┌────────┐                                      │
│  │ Client │─────────▶│ Replica│──▶ X=5                                │
│  │        │          └────────┘                                       │
│  │        │  Read 2   ┌────────┐                                      │
│  │        │─────────▶│ Replica│──▶ X=5 nebo novější (ne X=3!)        │
│  └────────┘          └────────┘                                       │
│                                                                         │
│  Implementace:                                                          │
│  • Sticky sessions k jedné replice                                     │
│  • Version vectors                                                      │
│                                                                         │
│  PATTERN 3: Saga s Compensating Transactions                           │
│  ═══════════════════════════════════════════                           │
│  Pro distribuované transakce bez 2PC.                                  │
│                                                                         │
│  ┌─────────┐    ┌─────────┐    ┌─────────┐    ┌─────────┐            │
│  │ Reserve │───▶│ Debit   │───▶│ Credit  │───▶│ Confirm │            │
│  │ Funds   │    │ Account │    │ Account │    │ Transfer│            │
│  └────┬────┘    └────┬────┘    └────┬────┘    └─────────┘            │
│       │              │              │                                  │
│       ▼              ▼              ▼                                  │
│    Compensate    Compensate    Compensate                              │
│    (release)     (refund)      (reverse)                               │
│                                                                         │
│  Každý krok má kompenzační akci pro rollback.                         │
│  Event sourcing umožňuje audit a replay.                              │
│                                                                         │
│  PATTERN 4: Outbox Pattern                                             │
│  ═════════════════════════                                             │
│  Atomický zápis do DB + publikace eventu.                             │
│                                                                         │
│  ┌──────────────────────────────────────────┐                         │
│  │            SAME TRANSACTION              │                         │
│  │                                          │                         │
│  │  1. UPDATE accounts SET balance = ...   │                         │
│  │  2. INSERT INTO outbox (event_data)     │                         │
│  │                                          │                         │
│  └──────────────────────────────────────────┘                         │
│                      │                                                  │
│                      ▼                                                  │
│              Outbox processor                                           │
│              (async, idempotent)                                        │
│                      │                                                  │
│                      ▼                                                  │
│              Publish to EventBridge/Kafka                              │
│                                                                         │
│  Výhoda: Žádná ztráta eventů při selhání.                             │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### Shrnutí trade-offs

```
┌─────────────────────────────────────────────────────────────────────────┐
│              SUMMARY: CONSISTENCY TRADE-OFFS                            │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│                    STRONG              EVENTUAL                         │
│                    ══════              ════════                         │
│                                                                         │
│  Latence           Vyšší               Nižší                           │
│                    (čeká na sync)      (okamžitá odpověď)              │
│                                                                         │
│  Throughput        Nižší               Vyšší                           │
│                    (locking)           (no coordination)               │
│                                                                         │
│  Availability      Nižší               Vyšší                           │
│                    (při partition)     (při partition)                 │
│                                                                         │
│  Complexity        Nižší               Vyšší                           │
│                    (jednodušší logic)  (conflict resolution)           │
│                                                                         │
│  Cost              Vyšší               Nižší                           │
│                    (sync replication)  (async replication)             │
│                                                                         │
│  Data integrity    Garantovaná         Best effort                     │
│                                        (+ compensation)                │
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ GOLDEN RULE PRO BANKOVNICTVÍ:                                    │   │
│  │                                                                  │   │
│  │ • Peníze a zůstatky → STRONG CONSISTENCY                        │   │
│  │ • Vše ostatní → EVENTUAL je často OK                            │   │
│  │ • Pokud eventual, zajisti IDEMPOTENCY + COMPENSATION            │   │
│  │ • Vždy informuj uživatele o stavu ("zpracovává se...")         │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## Shrnutí: Jak principy spolu souvisí

```
┌─────────────────────────────────────────────────────────────────────────┐
│              VZTAHY MEZI PRINCIPY                                       │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│                    ┌─────────────────────┐                             │
│                    │  Defense in Depth   │                             │
│                    │  (bezpečnostní      │                             │
│                    │   vrstvy)           │                             │
│                    └──────────┬──────────┘                             │
│                               │                                         │
│                               ▼                                         │
│                    ┌─────────────────────┐                             │
│                    │    Zero Trust       │                             │
│                    │  (ověřuj vždy)      │◀───┐                        │
│                    └──────────┬──────────┘    │                        │
│                               │               │                        │
│         ┌─────────────────────┼───────────────┼────────────────┐       │
│         │                     │               │                │       │
│         ▼                     ▼               │                ▼       │
│  ┌─────────────┐      ┌─────────────┐        │        ┌─────────────┐ │
│  │ Separation  │      │   Loose     │        │        │    High     │ │
│  │ of Concerns │◀────▶│  Coupling   │────────┘        │  Cohesion   │ │
│  │             │      │             │                 │             │ │
│  └──────┬──────┘      └──────┬──────┘                 └──────┬──────┘ │
│         │                    │                               │        │
│         │                    │    ┌──────────────────────────┘        │
│         │                    │    │                                   │
│         ▼                    ▼    ▼                                   │
│  ┌───────────────────────────────────────┐                            │
│  │          MICROSERVICES                │                            │
│  │    (architektonický výsledek)         │                            │
│  └───────────────────────────────────────┘                            │
│                    │                                                   │
│                    │ vyžaduje                                          │
│                    ▼                                                   │
│         ┌─────────────────────────────────────────────────────┐       │
│         │                                                     │       │
│         ▼                                                     ▼       │
│  ┌─────────────────┐                              ┌─────────────────┐ │
│  │   Idempotency   │                              │   Consistency   │ │
│  │  (bezpečné      │                              │   Trade-offs    │ │
│  │   opakování)    │                              │   (CAP)         │ │
│  └─────────────────┘                              └─────────────────┘ │
│                                                                        │
│  Všechny principy vedou k:                                            │
│  ✅ Bezpečnému systému (Defense in Depth + Zero Trust)               │
│  ✅ Škálovatelnému systému (Loose Coupling + Separation)             │
│  ✅ Udržovatelnému systému (High Cohesion)                           │
│  ✅ Spolehlivému systému (Idempotency + Consistency management)      │
│                                                                        │
└─────────────────────────────────────────────────────────────────────────┘
```

Chceš, abych rozvedl některý z principů ještě více do hloubky, nebo přešel na jiné téma - například konkrétní architektonické patterny (Saga, CQRS, Event Sourcing), PSD2/Open Banking, nebo technologické detaily AWS služeb pro bankovnictví?

# Architektonické patterny pro digitální bankovnictví

---
## 1. Saga Pattern

### Co to je a proč potřebujeme

V monolitické aplikaci můžeš použít databázovou transakci (ACID) pro operace, které ovlivňují více tabulek. V mikroservisní architektuře každá služba má vlastní databázi - tradiční transakce nefungují napříč službami.

Saga je sekvence lokálních transakcí, kde každý krok má definovanou kompenzační akci pro případ selhání.

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    PROČ POTŘEBUJEME SAGA                                │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  MONOLIT (tradiční ACID transakce):                                    │
│  ┌────────────────────────────────────────────────────────┐            │
│  │                    BEGIN TRANSACTION                    │            │
│  │                                                         │            │
│  │  1. Debit Account A    ─┐                              │            │
│  │  2. Credit Account B    ├── Vše nebo nic               │            │
│  │  3. Log Transaction    ─┘                              │            │
│  │                                                         │            │
│  │                    COMMIT (nebo ROLLBACK)               │            │
│  └────────────────────────────────────────────────────────┘            │
│                                                                         │
│  MICROSERVICES (distribuovaný systém):                                 │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐                 │
│  │   Account    │  │   Payment    │  │    Audit     │                 │
│  │   Service    │  │   Service    │  │   Service    │                 │
│  │   ┌─────┐    │  │   ┌─────┐    │  │   ┌─────┐    │                 │
│  │   │ DB  │    │  │   │ DB  │    │  │   │ DB  │    │                 │
│  │   └─────┘    │  │   └─────┘    │  │   └─────┘    │                 │
│  └──────────────┘  └──────────────┘  └──────────────┘                 │
│         │                 │                 │                          │
│         └─────────────────┴─────────────────┘                          │
│                           │                                             │
│              ❌ Žádná společná transakce!                              │
│              ❌ 2PC (Two-Phase Commit) = anti-pattern                  │
│                 (pomalý, blokující, single point of failure)           │
│                                                                         │
│              ✅ Řešení: SAGA PATTERN                                   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### Dva typy Saga

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    CHOREOGRAPHY vs ORCHESTRATION                        │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  CHOREOGRAPHY (Event-driven):                                          │
│  ═══════════════════════════                                           │
│  Každá služba reaguje na eventy a publikuje další eventy.              │
│  Žádný centrální koordinátor.                                          │
│                                                                         │
│  ┌─────────┐  PaymentCreated  ┌─────────┐  FundsReserved  ┌─────────┐ │
│  │ Payment │─────────────────▶│ Account │────────────────▶│  Fraud  │ │
│  │ Service │                  │ Service │                 │ Service │ │
│  └─────────┘                  └─────────┘                 └────┬────┘ │
│       ▲                                                        │      │
│       │                      FraudCheckPassed                  │      │
│       └────────────────────────────────────────────────────────┘      │
│                                                                         │
│  ✅ Výhody: Loose coupling, jednodušší služby, žádný SPOF            │
│  ❌ Nevýhody: Těžší sledovat flow, složitější debugging               │
│              Riziko cyklických závislostí                              │
│                                                                         │
│  Vhodné pro: Jednoduché workflow (3-4 kroky), event-driven systémy    │
│                                                                         │
│  ─────────────────────────────────────────────────────────────────────  │
│                                                                         │
│  ORCHESTRATION (Command-driven):                                       │
│  ═══════════════════════════════                                       │
│  Centrální orchestrátor řídí celý workflow.                           │
│                                                                         │
│                    ┌─────────────────────┐                             │
│                    │      SAGA           │                             │
│                    │   ORCHESTRATOR      │                             │
│                    │  (Step Functions)   │                             │
│                    └──────────┬──────────┘                             │
│                               │                                         │
│          ┌────────────────────┼────────────────────┐                   │
│          │                    │                    │                   │
│          ▼                    ▼                    ▼                   │
│    ┌──────────┐        ┌──────────┐        ┌──────────┐               │
│    │ Account  │        │  Fraud   │        │  Notif.  │               │
│    │ Service  │        │ Service  │        │ Service  │               │
│    └──────────┘        └──────────┘        └──────────┘               │
│                                                                         │
│  ✅ Výhody: Jasný flow, snadný debugging, centrální logika           │
│  ❌ Nevýhody: Tighter coupling, orchestrátor = potential SPOF        │
│                                                                         │
│  Vhodné pro: Komplexní workflow, mnoho kroků, bankovní transakce     │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### Příklad: Platební Saga s Orchestration

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    PAYMENT SAGA - HAPPY PATH                            │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  Kroky platby 10,000 CZK z účtu A na účet B:                           │
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                     SAGA ORCHESTRATOR                            │   │
│  │                    (AWS Step Functions)                          │   │
│  └────────────────────────────┬────────────────────────────────────┘   │
│                               │                                         │
│  Step 1: Reserve Funds        │                                         │
│  ─────────────────────────────┼────────────────────────────────────     │
│                               ▼                                         │
│                        ┌─────────────┐                                 │
│                        │   Account   │  Zablokuj 10,000 CZK            │
│                        │   Service   │  (available -= 10,000)          │
│                        └──────┬──────┘  (reserved += 10,000)           │
│                               │ ✅ Success                              │
│                               ▼                                         │
│  Step 2: Fraud Check          │                                         │
│  ─────────────────────────────┼────────────────────────────────────     │
│                               ▼                                         │
│                        ┌─────────────┐                                 │
│                        │   Fraud     │  Kontrola rizika                │
│                        │   Service   │  (ML model, rules)              │
│                        └──────┬──────┘                                 │
│                               │ ✅ Low Risk                             │
│                               ▼                                         │
│  Step 3: AML Check            │                                         │
│  ─────────────────────────────┼────────────────────────────────────     │
│                               ▼                                         │
│                        ┌─────────────┐                                 │
│                        │    AML      │  Sanctions screening            │
│                        │   Service   │  (PEP check, watchlist)         │
│                        └──────┬──────┘                                 │
│                               │ ✅ Clear                                │
│                               ▼                                         │
│  Step 4: Execute Transfer     │                                         │
│  ─────────────────────────────┼────────────────────────────────────     │
│                               ▼                                         │
│                        ┌─────────────┐                                 │
│                        │   Core      │  Skutečný převod                │
│                        │  Banking    │  (debit A, credit B)            │
│                        └──────┬──────┘                                 │
│                               │ ✅ Executed                             │
│                               ▼                                         │
│  Step 5: Confirm & Notify     │                                         │
│  ─────────────────────────────┼────────────────────────────────────     │
│                               ▼                                         │
│                        ┌─────────────┐                                 │
│                        │   Account   │  Uvolni rezervaci               │
│                        │   Service   │  (reserved = 0)                 │
│                        └──────┬──────┘                                 │
│                               │                                         │
│                               ▼                                         │
│                        ┌─────────────┐                                 │
│                        │   Notif.    │  Push, SMS, Email               │
│                        │   Service   │                                 │
│                        └─────────────┘                                 │
│                                                                         │
│  VÝSLEDEK: Platba úspěšně dokončena ✅                                 │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### Saga - Failure Path s Compensating Transactions

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    PAYMENT SAGA - FAILURE & COMPENSATION                │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  Scénář: AML Check selže (klient na sankční listině)                  │
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ Step                │ Action           │ Compensation            │   │
│  ├─────────────────────┼──────────────────┼─────────────────────────│   │
│  │ 1. Reserve Funds    │ ✅ Done          │ Release reservation     │   │
│  │ 2. Fraud Check      │ ✅ Done          │ (žádná - read only)    │   │
│  │ 3. AML Check        │ ❌ FAILED        │ N/A                     │   │
│  │ 4. Execute Transfer │ ⏸️ Not started  │ N/A                     │   │
│  │ 5. Notify           │ ⏸️ Not started  │ N/A                     │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  COMPENSATION FLOW (v opačném pořadí):                                 │
│                                                                         │
│                    ┌─────────────────────┐                             │
│                    │      SAGA           │                             │
│                    │   ORCHESTRATOR      │                             │
│                    │   (rollback mode)   │                             │
│                    └──────────┬──────────┘                             │
│                               │                                         │
│  Compensate Step 2:           │  (žádná akce - read only)              │
│  ─────────────────────────────┼─────────────────────────               │
│                               │                                         │
│  Compensate Step 1:           │                                         │
│  ─────────────────────────────┼─────────────────────────               │
│                               ▼                                         │
│                        ┌─────────────┐                                 │
│                        │   Account   │  Uvolni rezervaci:             │
│                        │   Service   │  available += 10,000           │
│                        │             │  reserved -= 10,000            │
│                        └──────┬──────┘                                 │
│                               │                                         │
│  Notify Failure:              │                                         │
│  ─────────────────────────────┼─────────────────────────               │
│                               ▼                                         │
│                        ┌─────────────┐                                 │
│                        │   Notif.    │  "Platba zamítnuta"            │
│                        │   Service   │  + důvod (compliance)          │
│                        └─────────────┘                                 │
│                                                                         │
│  DŮLEŽITÉ PRINCIPY:                                                    │
│  ───────────────────                                                   │
│  • Kompenzace MUSÍ být idempotentní                                   │
│  • Kompenzace MUSÍ být vždy možná (design for failure)                │
│  • Některé akce nelze kompenzovat (např. email odeslán)               │
│    → tyto dělej jako poslední                                         │
│  • Log každý krok pro audit trail                                      │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### Implementace Saga v AWS

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    SAGA S AWS STEP FUNCTIONS                            │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  Komponenty:                                                            │
│  ───────────                                                           │
│  • Step Functions State Machine = Saga Orchestrator                    │
│  • Lambda Functions = jednotlivé kroky                                 │
│  • DynamoDB = saga state store                                         │
│  • SQS/SNS = async komunikace se službami                             │
│  • CloudWatch = monitoring & alerting                                  │
│                                                                         │
│  Step Functions Features pro Saga:                                     │
│  ─────────────────────────────────                                     │
│  • Built-in error handling (Catch, Retry)                             │
│  • Timeouts na každý krok                                              │
│  • Parallel execution (kde možné)                                      │
│  • Visual workflow monitoring                                          │
│  • Automatic state persistence                                         │
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                    STATE MACHINE FLOW                            │   │
│  │                                                                  │   │
│  │  ┌─────────────┐                                                │   │
│  │  │   Start     │                                                │   │
│  │  └──────┬──────┘                                                │   │
│  │         ▼                                                        │   │
│  │  ┌─────────────┐     ┌─────────────┐                            │   │
│  │  │   Reserve   │────▶│   Fraud     │                            │   │
│  │  │   Funds     │     │   Check     │                            │   │
│  │  └──────┬──────┘     └──────┬──────┘                            │   │
│  │         │ onError           │ onError                            │   │
│  │         ▼                   ▼                                    │   │
│  │  ┌─────────────┐     ┌─────────────┐                            │   │
│  │  │  Release    │     │  Release    │                            │   │
│  │  │  Funds      │     │  Funds      │                            │   │
│  │  └──────┬──────┘     └──────┬──────┘                            │   │
│  │         │                   │                                    │   │
│  │         └───────────────────┴───────▶ ┌─────────────┐           │   │
│  │                                       │   Failed    │           │   │
│  │                                       │   State     │           │   │
│  │                                       └─────────────┘           │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  Kroky implementace:                                                    │
│  ───────────────────                                                   │
│  1. Definovat state machine (JSON/YAML)                                │
│  2. Implementovat Lambda pro každý krok                                │
│  3. Implementovat Lambda pro každou kompenzaci                         │
│  4. Nastavit error handling a retry politiky                          │
│  5. Nastavit timeouts                                                   │
│  6. Nastavit monitoring a alerting                                     │
│  7. Implementovat idempotency na každém kroku                         │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 2. CQRS (Command Query Responsibility Segregation)

### Co to je

Oddělení operací čtení (Query) od operací zápisu (Command) do samostatných modelů, případně i samostatných databází.

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    TRADIČNÍ vs CQRS                                     │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  TRADIČNÍ (CRUD):                                                      │
│  ═════════════════                                                     │
│  ┌─────────┐         ┌─────────────┐         ┌─────────┐              │
│  │  API    │────────▶│   Service   │────────▶│   DB    │              │
│  │         │◀────────│   (CRUD)    │◀────────│         │              │
│  └─────────┘         └─────────────┘         └─────────┘              │
│                                                                         │
│  Problémy:                                                              │
│  • Stejný model pro čtení i zápis (kompromis)                         │
│  • Read a Write mají různé požadavky na škálování                     │
│  • Komplexní queries zpomalují writes                                  │
│  • Optimalizace pro jedno zhoršuje druhé                              │
│                                                                         │
│  CQRS:                                                                 │
│  ══════                                                                │
│                                                                         │
│  ┌─────────┐         ┌─────────────┐         ┌─────────┐              │
│  │ Command │────────▶│   Write     │────────▶│  Write  │              │
│  │   API   │         │   Model     │         │   DB    │              │
│  └─────────┘         └─────────────┘         └────┬────┘              │
│                                                    │                   │
│                                            Sync (event/CDC)            │
│                                                    │                   │
│  ┌─────────┐         ┌─────────────┐         ┌────▼────┐              │
│  │  Query  │◀────────│   Read      │◀────────│  Read   │              │
│  │   API   │         │   Model     │         │   DB    │              │
│  └─────────┘         └─────────────┘         └─────────┘              │
│                                                                         │
│  Výhody:                                                                │
│  • Optimalizace read modelu pro queries (denormalizace)               │
│  • Optimalizace write modelu pro transakce (normalizace)              │
│  • Nezávislé škálování read vs write                                   │
│  • Read model může být jiná technologie (ElasticSearch, Redis)        │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### Proč v bankovnictví

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    CQRS USE CASES V BANCE                               │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  WRITE SIDE (Commands):                                                │
│  ──────────────────────                                                │
│  • Platební příkazy                                                     │
│  • Změna limitů                                                         │
│  • Blokace karty                                                        │
│  • Založení účtu                                                        │
│                                                                         │
│  Charakteristiky:                                                       │
│  • Nižší objem (tisíce/sec)                                            │
│  • Komplexní business rules                                            │
│  • ACID transakce                                                       │
│  • Strong consistency                                                   │
│                                                                         │
│  READ SIDE (Queries):                                                  │
│  ────────────────────                                                  │
│  • Zobrazení zůstatku                                                  │
│  • Historie transakcí                                                   │
│  • Výpisy z účtu                                                       │
│  • Dashboard, analytics                                                 │
│  • Vyhledávání transakcí                                               │
│                                                                         │
│  Charakteristiky:                                                       │
│  • Vysoký objem (100x více než writes)                                 │
│  • Komplexní queries (filtrování, agregace)                           │
│  • Eventual consistency často OK                                       │
│  • Denormalizovaná data pro rychlost                                   │
│                                                                         │
│  PŘÍKLAD - TRANSACTION HISTORY:                                        │
│  ══════════════════════════════                                        │
│                                                                         │
│  Write Model (normalized):                                             │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐                 │
│  │ transactions │  │   accounts   │  │  categories  │                 │
│  │ ──────────── │  │ ──────────── │  │ ──────────── │                 │
│  │ id           │  │ id           │  │ id           │                 │
│  │ account_id   │──│ number       │  │ name         │                 │
│  │ amount       │  │ currency     │  │              │                 │
│  │ category_id  │──│              │──│              │                 │
│  │ timestamp    │  │              │  │              │                 │
│  └──────────────┘  └──────────────┘  └──────────────┘                 │
│                                                                         │
│  → 3 JOINy pro zobrazení jedné transakce                              │
│  → Pomalé pro velké objemy                                            │
│                                                                         │
│  Read Model (denormalized):                                            │
│  ┌────────────────────────────────────────────────────┐               │
│  │ transaction_view                                    │               │
│  │ ──────────────────────────────────────────────────  │               │
│  │ id, account_number, account_holder_name,           │               │
│  │ amount, currency, formatted_amount,                │               │
│  │ category_name, category_icon,                      │               │
│  │ timestamp, formatted_date,                         │               │
│  │ counterparty_name, counterparty_iban,             │               │
│  │ balance_after                                      │               │
│  └────────────────────────────────────────────────────┘               │
│                                                                         │
│  → Žádné JOINy                                                         │
│  → Předpočítané hodnoty                                               │
│  → Optimalizované indexy pro typické queries                          │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### Synchronizace Read a Write modelu

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    SYNC STRATEGIE                                       │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  STRATEGIE 1: Event-Based Sync (doporučeno)                           │
│  ═══════════════════════════════════════════                           │
│                                                                         │
│  ┌───────────┐    ┌───────────┐    ┌───────────┐    ┌───────────┐     │
│  │  Command  │───▶│  Write    │───▶│  Event    │───▶│   Read    │     │
│  │  Handler  │    │   DB      │    │   Bus     │    │  Updater  │     │
│  └───────────┘    └───────────┘    └───────────┘    └─────┬─────┘     │
│                                                           │            │
│                                     TransactionCreated    │            │
│                                     { id, amount, ... }   ▼            │
│                                                     ┌───────────┐      │
│                                                     │  Read DB  │      │
│                                                     └───────────┘      │
│                                                                         │
│  Implementace AWS:                                                      │
│  • Write DB: RDS Aurora / DynamoDB                                     │
│  • Event Bus: EventBridge / SNS / MSK (Kafka)                         │
│  • Read Updater: Lambda                                                │
│  • Read DB: DynamoDB / ElastiCache / OpenSearch                       │
│                                                                         │
│  STRATEGIE 2: Change Data Capture (CDC)                               │
│  ═══════════════════════════════════════                               │
│                                                                         │
│  ┌───────────┐    ┌───────────┐    ┌───────────┐    ┌───────────┐     │
│  │  Command  │───▶│  Write    │───▶│    CDC    │───▶│   Read    │     │
│  │  Handler  │    │   DB      │    │  Stream   │    │   DB      │     │
│  └───────────┘    └───────────┘    └───────────┘    └───────────┘     │
│                                         │                              │
│                                    DMS / Debezium                      │
│                                    Kinesis / MSK                       │
│                                                                         │
│  Výhody:                                                                │
│  • Neinvazivní (aplikace nemusí publikovat eventy)                    │
│  • Zachytí i přímé změny v DB                                         │
│                                                                         │
│  LATENCE SYNCHRONIZACE:                                                │
│  ═══════════════════════                                               │
│                                                                         │
│  ┌────────────────────────────────────────────────────────────────┐   │
│  │ Metoda              │ Typická latence │ Konzistence             │   │
│  ├─────────────────────┼─────────────────┼─────────────────────────│   │
│  │ Synchronní update   │ < 100ms         │ Strong (ale pomalejší) │   │
│  │ Event-based         │ 100ms - 1s      │ Eventual               │   │
│  │ CDC                 │ 1s - 10s        │ Eventual               │   │
│  │ Batch refresh       │ minuty - hodiny │ Eventual (stale)       │   │
│  └────────────────────────────────────────────────────────────────┘   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### CQRS - Kdy použít, kdy ne

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    ROZHODOVACÍ KRITÉRIA                                 │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  ✅ POUŽIJ CQRS KDYŽ:                                                  │
│  ─────────────────────                                                 │
│  • Read/Write ratio je vysoký (100:1 nebo více)                       │
│  • Read a Write mají velmi odlišné požadavky                          │
│  • Potřebuješ různé technologie pro read (search, cache)              │
│  • Read queries jsou komplexní (agregace, full-text)                  │
│  • Write vyžaduje strong consistency, read může být eventual          │
│  • Potřebuješ nezávisle škálovat read a write                         │
│                                                                         │
│  ❌ NEPOUŽÍVEJ CQRS KDYŽ:                                              │
│  ──────────────────────────                                            │
│  • Jednoduchá CRUD aplikace                                            │
│  • Read/Write ratio je blízko 1:1                                     │
│  • Týmu chybí zkušenosti s distribuovanými systémy                    │
│  • Strong consistency je vyžadována všude                             │
│  • Komplexita není ospravedlněná benefity                             │
│                                                                         │
│  BANKOVNÍ USE CASES:                                                   │
│  ════════════════════                                                  │
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ Use Case              │ CQRS? │ Důvod                           │   │
│  ├───────────────────────┼───────┼─────────────────────────────────│   │
│  │ Transaction history   │  ✅   │ Vysoký read, komplexní queries │   │
│  │ Account statements    │  ✅   │ PDF generování, archivace      │   │
│  │ Analytics dashboard   │  ✅   │ Agregace, různé views          │   │
│  │ Payment execution     │  ❌   │ Write-heavy, strong consistency │   │
│  │ User authentication   │  ❌   │ Jednoduchá logika              │   │
│  │ Balance inquiry       │  🤔   │ Záleží na volumenu            │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 3. Event Sourcing

### Co to je

Místo ukládání aktuálního stavu entity ukládáme sekvenci událostí, které k tomuto stavu vedly. Aktuální stav je odvozen přehráním všech událostí.

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    TRADIČNÍ STATE vs EVENT SOURCING                     │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  TRADIČNÍ (State-based):                                               │
│  ═══════════════════════                                               │
│                                                                         │
│  Účet #12345:                                                          │
│  ┌────────────────────────────────────────┐                            │
│  │ id: 12345                              │                            │
│  │ balance: 15,000 CZK  ← pouze aktuální │                            │
│  │ status: active                         │                            │
│  │ updated_at: 2024-01-15                │                            │
│  └────────────────────────────────────────┘                            │
│                                                                         │
│  ❌ Ztracená historie: Jak se dostal na 15,000?                       │
│  ❌ Nelze rekonstruovat stav k datu v minulosti                       │
│  ❌ Audit trail vyžaduje separátní tabulku                            │
│                                                                         │
│  EVENT SOURCING:                                                       │
│  ════════════════                                                      │
│                                                                         │
│  Event Store pro účet #12345:                                          │
│  ┌────────────────────────────────────────────────────────────────┐   │
│  │ Seq │ Timestamp           │ Event                              │   │
│  ├─────┼─────────────────────┼────────────────────────────────────│   │
│  │  1  │ 2024-01-01 09:00    │ AccountOpened { initial: 0 }      │   │
│  │  2  │ 2024-01-02 10:30    │ FundsDeposited { amount: 20000 }  │   │
│  │  3  │ 2024-01-05 14:15    │ FundsWithdrawn { amount: 3000 }   │   │
│  │  4  │ 2024-01-10 11:00    │ FundsTransferred { amount: 2000 } │   │
│  │  5  │ 2024-01-15 09:45    │ FundsDeposited { amount: 0 }      │   │
│  └────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  Aktuální stav = přehrání eventů:                                      │
│  0 + 20000 - 3000 - 2000 + 0 = 15,000 CZK ✅                          │
│                                                                         │
│  ✅ Kompletní audit trail (built-in)                                   │
│  ✅ Časové dotazy ("jaký byl stav k 5.1.?")                           │
│  ✅ Debugging ("proč je balance záporný?")                            │
│  ✅ Replay pro opravy nebo nové projekce                              │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### Proč je ideální pro bankovnictví

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    EVENT SOURCING V BANKOVNICTVÍ                        │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  REGULATORNÍ DŮVODY:                                                   │
│  ════════════════════                                                  │
│  • Audit trail je POVINNÝ (AML, PSD2, MiFID II)                       │
│  • Musíš prokázat KDO, KDY, CO udělal                                 │
│  • Data retention 5-10 let                                             │
│  • Immutability = nelze změnit historii (compliance)                  │
│                                                                         │
│  BUSINESS DŮVODY:                                                      │
│  ═════════════════                                                     │
│  • Reklamace: "Tuhle platbu jsem neudělal!"                           │
│    → Můžeš přesně ukázat sekvenci událostí                            │
│                                                                         │
│  • Forenzní analýza: "Jak se dostal účet do mínusu?"                  │
│    → Replay eventů krok po kroku                                       │
│                                                                         │
│  • Oprava chyby: "Bug způsobil špatné zůstatky"                       │
│    → Oprav logiku, replay eventy, nový stav                           │
│                                                                         │
│  • Nové reporty: "Chceme nový dashboard"                               │
│    → Vytvoř novou projekci z existujících eventů                      │
│                                                                         │
│  TYPICKÉ EVENTY V BANCE:                                               │
│  ════════════════════════                                              │
│                                                                         │
│  Account Domain:                                                        │
│  ├── AccountOpened                                                     │
│  ├── AccountClosed                                                     │
│  ├── AccountBlocked                                                    │
│  ├── AccountUnblocked                                                  │
│  └── AccountLimitChanged                                               │
│                                                                         │
│  Transaction Domain:                                                   │
│  ├── FundsDeposited                                                    │
│  ├── FundsWithdrawn                                                    │
│  ├── TransferInitiated                                                 │
│  ├── TransferCompleted                                                 │
│  ├── TransferFailed                                                    │
│  └── TransferReversed                                                  │
│                                                                         │
│  Card Domain:                                                           │
│  ├── CardIssued                                                        │
│  ├── CardActivated                                                     │
│  ├── CardBlocked                                                       │
│  ├── PinChanged                                                        │
│  └── LimitUpdated                                                      │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### Event Sourcing architektura

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    EVENT SOURCING ARCHITEKTURA                          │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│                           ┌─────────────────┐                          │
│                           │    Command      │                          │
│                           │    (Deposit)    │                          │
│                           └────────┬────────┘                          │
│                                    │                                    │
│                                    ▼                                    │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                      COMMAND HANDLER                             │   │
│  │                                                                  │   │
│  │  1. Load events for aggregate                                   │   │
│  │  2. Replay events → current state                               │   │
│  │  3. Validate command against state                              │   │
│  │  4. Generate new event(s)                                       │   │
│  │  5. Persist event(s) to Event Store                            │   │
│  │  6. Publish event(s) to Event Bus                              │   │
│  │                                                                  │   │
│  └────────────────────────────────┬────────────────────────────────┘   │
│                                   │                                     │
│              ┌────────────────────┼────────────────────┐               │
│              │                    │                    │               │
│              ▼                    ▼                    ▼               │
│  ┌───────────────────┐  ┌───────────────┐  ┌───────────────────┐      │
│  │    EVENT STORE    │  │   EVENT BUS   │  │    PROJECTIONS    │      │
│  │   (append-only)   │  │  (EventBridge)│  │   (Read Models)   │      │
│  │                   │  │               │  │                   │      │
│  │ ┌───────────────┐ │  │               │  │ ┌───────────────┐ │      │
│  │ │ AccountEvents │ │  │   Publish     │  │ │Balance View   │ │      │
│  │ │ seq: 1,2,3... │ │──│───────────────│──│ │(DynamoDB)     │ │      │
│  │ └───────────────┘ │  │               │  │ └───────────────┘ │      │
│  │                   │  │               │  │                   │      │
│  │ DynamoDB / Kafka │  │               │  │ ┌───────────────┐ │      │
│  │ (immutable log)   │  │               │  │ │History View   │ │      │
│  │                   │  │               │──│ │(OpenSearch)   │ │      │
│  └───────────────────┘  └───────────────┘  │ └───────────────┘ │      │
│                                            │                   │      │
│                                            │ ┌───────────────┐ │      │
│                                            │ │Analytics View │ │      │
│                                            │ │(Redshift)     │ │      │
│                                            │ └───────────────┘ │      │
│                                            └───────────────────┘      │
│                                                                         │
│  PROJEKCE = Materializovaný view z eventů                             │
│  • Každá projekce může mít jiný formát                                │
│  • Rebuild projekce = replay všech eventů                             │
│  • Nová projekce = nový subscriber na event bus                       │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### Event Store implementace

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    EVENT STORE POŽADAVKY                                │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  ZÁKLADNÍ POŽADAVKY:                                                   │
│  ════════════════════                                                  │
│  • Append-only (žádné UPDATE, žádné DELETE)                           │
│  • Ordered (sekvenční číslo/timestamp)                                 │
│  • Optimistic concurrency (expected version)                           │
│  • Stream per aggregate (partition by account_id)                     │
│                                                                         │
│  AWS IMPLEMENTACE:                                                      │
│                                                                         │
│  Option 1: DynamoDB                                                    │
│  ─────────────────────                                                 │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ PK (Partition Key)    │ SK (Sort Key)   │ Attributes            │   │
│  ├────────────────────────┼─────────────────┼───────────────────────│   │
│  │ ACCOUNT#12345          │ EVENT#00001     │ type, data, timestamp │   │
│  │ ACCOUNT#12345          │ EVENT#00002     │ type, data, timestamp │   │
│  │ ACCOUNT#12345          │ EVENT#00003     │ type, data, timestamp │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  Výhody: Serverless, škálovatelné, nízká latence                      │
│  Nevýhody: Limitovaný query (potřebuješ projekce)                     │
│                                                                         │
│  Option 2: Amazon Kinesis / MSK (Kafka)                               │
│  ─────────────────────────────────────────                            │
│  • Stream = partition by aggregate ID                                  │
│  • Retention: Kinesis 7 dní (extended 365), Kafka unlimited           │
│  • Real-time processing s Lambda/KCL                                  │
│                                                                         │
│  Výhody: Native streaming, replay, multiple consumers                 │
│  Nevýhody: Retention limits (Kinesis), operational complexity         │
│                                                                         │
│  Option 3: DynamoDB + Kinesis (hybrid)                                │
│  ───────────────────────────────────────                              │
│  • DynamoDB = primary event store (permanent)                         │
│  • DynamoDB Streams → Kinesis = distribution                          │
│                                                                         │
│  OPTIMISTIC CONCURRENCY:                                               │
│  ════════════════════════                                              │
│                                                                         │
│  Problem: Dva příkazy současně na stejný účet                         │
│                                                                         │
│  ┌─────────┐                              ┌─────────┐                  │
│  │Request A│  Read: version=5             │Request B│                  │
│  │         │◀─────────────────────────────│         │                  │
│  │ Deposit │                              │Withdraw │                  │
│  │  1000   │  Read: version=5             │  500    │                  │
│  │         │◀─────────────────────────────│         │                  │
│  └────┬────┘                              └────┬────┘                  │
│       │                                        │                       │
│       │  Write: expected_version=5             │                       │
│       │  ──────────────────────────────────────│                       │
│       │                                    Write: expected_version=5   │
│       │                                    ❌ CONFLICT! Version is 6   │
│       │                                                                │
│       ▼                                                                │
│  Version 6 (success)                     → Retry s novou verzí        │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### Event Sourcing + CQRS kombinace

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    ES + CQRS = POWER COMBO                              │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  Event Sourcing poskytuje:                                             │
│  • Kompletní historie (audit)                                          │
│  • Source of truth pro všechny projekce                               │
│                                                                         │
│  CQRS poskytuje:                                                       │
│  • Optimalizované read modely                                          │
│  • Různé views z jednoho event streamu                                │
│                                                                         │
│  KOMBINACE:                                                             │
│                                                                         │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │                         WRITE SIDE                                │  │
│  │                                                                   │  │
│  │  Command → Validate → Generate Event → Event Store              │  │
│  │                                                                   │  │
│  └───────────────────────────────┬──────────────────────────────────┘  │
│                                  │                                      │
│                             Event Bus                                   │
│                                  │                                      │
│         ┌────────────────────────┼────────────────────────┐            │
│         │                        │                        │            │
│         ▼                        ▼                        ▼            │
│  ┌─────────────┐          ┌─────────────┐          ┌─────────────┐    │
│  │  Projection │          │  Projection │          │  Projection │    │
│  │  Handler 1  │          │  Handler 2  │          │  Handler 3  │    │
│  └──────┬──────┘          └──────┬──────┘          └──────┬──────┘    │
│         │                        │                        │            │
│         ▼                        ▼                        ▼            │
│  ┌─────────────┐          ┌─────────────┐          ┌─────────────┐    │
│  │   Balance   │          │Transaction  │          │  Analytics  │    │
│  │   View      │          │  History    │          │   View      │    │
│  │ (DynamoDB)  │          │(OpenSearch) │          │ (Redshift)  │    │
│  └─────────────┘          └─────────────┘          └─────────────┘    │
│         │                        │                        │            │
│  └──────┴────────────────────────┴────────────────────────┘            │
│                                  │                                      │
│  ┌───────────────────────────────┴──────────────────────────────────┐  │
│  │                         READ SIDE                                 │  │
│  │                                                                   │  │
│  │  Query → Choose optimal view → Return denormalized data         │  │
│  │                                                                   │  │
│  └──────────────────────────────────────────────────────────────────┘  │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 4. Backend for Frontend (BFF)

### Co to je

Dedikovaná API vrstva pro každý typ klienta (mobilní app, web, partner API), která optimalizuje data a funkcionalitu pro konkrétní potřeby daného kanálu.

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    BEZ BFF vs S BFF                                     │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  BEZ BFF (Generic API):                                                │
│  ══════════════════════                                                │
│                                                                         │
│  ┌─────────┐                                                           │
│  │ Mobile  │─┐     Problém: Mobile potřebuje jiná data než Web        │
│  │  App    │ │     • Mobile: kompaktní, méně polí                     │
│  └─────────┘ │     • Web: detailní, více polí                         │
│              │     • Partner: jiná struktura (PSD2 spec)              │
│  ┌─────────┐ │                                                         │
│  │   Web   │─┼────▶ ┌──────────────────┐                              │
│  │  App    │ │      │   Generic API    │                              │
│  └─────────┘ │      │   (one size      │                              │
│              │      │    fits none)    │                              │
│  ┌─────────┐ │      └──────────────────┘                              │
│  │ Partner │─┘                                                         │
│  │  (TPP)  │       Výsledek:                                          │
│  └─────────┘       • Over-fetching (stahuje se víc než potřeba)       │
│                    • Under-fetching (potřeba více API calls)          │
│                    • Složitá logika na klientovi                       │
│                    • API "zamrzne" - změny rozbijí všechny klienty    │
│                                                                         │
│  S BFF:                                                                │
│  ══════                                                                │
│                                                                         │
│  ┌─────────┐     ┌──────────────┐                                     │
│  │ Mobile  │────▶│  Mobile BFF  │──┐                                  │
│  │  App    │     │ (optimized)  │  │                                  │
│  └─────────┘     └──────────────┘  │    ┌──────────────────┐          │
│                                    │    │                  │          │
│  ┌─────────┐     ┌──────────────┐  ├───▶│ Backend Services │          │
│  │   Web   │────▶│   Web BFF    │──┤    │                  │          │
│  │  App    │     │ (detailed)   │  │    │ • Accounts       │          │
│  └─────────┘     └──────────────┘  │    │ • Payments       │          │
│                                    │    │ • Cards          │          │
│  ┌─────────┐     ┌──────────────┐  │    │ • ...            │          │
│  │ Partner │────▶│ Partner BFF  │──┘    │                  │          │
│  │  (TPP)  │     │ (PSD2 spec)  │       └──────────────────┘          │
│  └─────────┘     └──────────────┘                                     │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### Proč BFF v bankovnictví

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    BFF USE CASES V BANCE                                │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  MOBILE BFF:                                                           │
│  ════════════                                                          │
│  Optimalizace pro:                                                      │
│  • Malá obrazovka → méně dat, kompaktní formát                        │
│  • Pomalá síť → agregace volání, caching                              │
│  • Battery → méně requestů                                             │
│  • Offline → data pro offline mode                                     │
│  • Biometrie → specifické auth flows                                   │
│  • Push notifications → FCM/APNS integrace                             │
│                                                                         │
│  Příklad - Dashboard endpoint:                                          │
│  ┌───────────────────────────────────────────────────────────────┐     │
│  │ Mobile BFF: GET /dashboard (1 call)                           │     │
│  │                                                                │     │
│  │ Agreguje interně:                                              │     │
│  │ • GET /accounts (seznam účtů)                                 │     │
│  │ • GET /accounts/{id}/balance (pro každý účet)                │     │
│  │ • GET /notifications/unread-count                             │     │
│  │ • GET /cards/active                                           │     │
│  │                                                                │     │
│  │ Vrací: kompaktní JSON s vším potřebným pro home screen       │     │
│  └───────────────────────────────────────────────────────────────┘     │
│                                                                         │
│  WEB BFF:                                                              │
│  ═════════                                                             │
│  Optimalizace pro:                                                      │
│  • Velká obrazovka → více detailů, tabulky                            │
│  • Rychlá síť → více dat najednou                                     │
│  • SEO (pro veřejné stránky)                                          │
│  • Session management (cookies)                                        │
│  • File downloads (výpisy PDF)                                        │
│                                                                         │
│  PARTNER BFF (Open Banking):                                           │
│  ════════════════════════════                                          │
│  Optimalizace pro:                                                      │
│  • PSD2/Berlin Group specifikace                                       │
│  • Specifická autentizace (QWAC, QSEAL)                               │
│  • Rate limiting per TPP                                               │
│  • Consent management                                                   │
│  • Regulatory reporting                                                 │
│                                                                         │
│  ADMIN BFF:                                                             │
│  ═══════════                                                           │
│  Optimalizace pro:                                                      │
│  • Interní nástroje                                                     │
│  • Bulk operace                                                         │
│  • Audit a reporting                                                    │
│  • Privilegovaný přístup                                               │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### BFF architektura v AWS

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    BFF IMPLEMENTACE                                     │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                         CLIENTS                                  │   │
│  │  ┌─────────┐    ┌─────────┐    ┌─────────┐    ┌─────────┐      │   │
│  │  │ iOS App │    │Android  │    │   Web   │    │   TPP   │      │   │
│  │  └────┬────┘    └────┬────┘    └────┬────┘    └────┬────┘      │   │
│  └───────┼──────────────┼──────────────┼──────────────┼────────────┘   │
│          │              │              │              │                 │
│          └──────────────┴──────┬───────┴──────────────┘                 │
│                                │                                         │
│                         CloudFront                                       │
│                                │                                         │
│  ┌─────────────────────────────┼───────────────────────────────────┐   │
│  │                     API GATEWAYS                                 │   │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐          │   │
│  │  │ Mobile API   │  │   Web API    │  │ Partner API  │          │   │
│  │  │ (HTTP API)   │  │ (HTTP API)   │  │ (REST API)   │          │   │
│  │  │              │  │              │  │ + WAF        │          │   │
│  │  │ JWT Auth     │  │ JWT Auth     │  │ + mTLS       │          │   │
│  │  │ Low latency  │  │              │  │ + Usage Plans│          │   │
│  │  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘          │   │
│  └─────────┼─────────────────┼─────────────────┼───────────────────┘   │
│            │                 │                 │                        │
│  ┌─────────┼─────────────────┼─────────────────┼───────────────────┐   │
│  │         ▼                 ▼                 ▼                   │   │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐          │   │
│  │  │  Mobile BFF  │  │   Web BFF    │  │ Partner BFF  │          │   │
│  │  │  (Lambda)    │  │  (Lambda)    │  │  (Lambda)    │          │   │
│  │  │              │  │              │  │              │          │   │
│  │  │ • Agregace   │  │ • SSR support│  │ • PSD2 spec  │          │   │
│  │  │ • Komprese   │  │ • PDF export │  │ • Consent    │          │   │
│  │  │ • Caching    │  │ • Full data  │  │ • Rate limit │          │   │
│  │  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘          │   │
│  │         │                 │                 │                   │   │
│  │  BFF LAYER (канál-specifická logika)                           │   │
│  └─────────┼─────────────────┼─────────────────┼───────────────────┘   │
│            │                 │                 │                        │
│            └─────────────────┼─────────────────┘                        │
│                              │                                          │
│                       Internal ALB                                      │
│                              │                                          │
│  ┌───────────────────────────┼─────────────────────────────────────┐   │
│  │                   DOMAIN SERVICES                                │   │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐        │   │
│  │  │ Account  │  │ Payment  │  │  Card    │  │  Notif.  │        │   │
│  │  │ Service  │  │ Service  │  │ Service  │  │ Service  │        │   │
│  │  └──────────┘  └──────────┘  └──────────┘  └──────────┘        │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  ODPOVĚDNOSTI BFF:                                                     │
│  ─────────────────                                                     │
│  • Agregace dat z více služeb                                          │
│  • Transformace do kanálově-specifického formátu                      │
│  • Kanálově-specifická autentizace                                    │
│  • Caching (per-channel strategie)                                    │
│  • Error handling a fallbacks                                          │
│                                                                         │
│  BFF NEOBSAHUJE:                                                       │
│  ────────────────                                                      │
│  • Business logiku (ta je v domain services)                          │
│  • Persistenci dat                                                     │
│  • Transakční logiku                                                   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 5. Anti-Corruption Layer (ACL)

### Co to je

Vrstva, která izoluje moderní systém od legacy systému nebo externího systému s odlišným modelem. Překládá mezi doménami a chrání před "kontaminací" špatným designem.

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    ANTI-CORRUPTION LAYER                                │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  BEZ ACL (přímá integrace):                                            │
│  ══════════════════════════                                            │
│                                                                         │
│  ┌────────────────┐              ┌────────────────┐                    │
│  │ Modern Service │─────────────▶│  Core Banking  │                    │
│  │                │              │  (Mainframe)   │                    │
│  │ Používá model  │              │                │                    │
│  │ core bankingu! │              │ COBOL structs  │                    │
│  │ ❌ Kontaminace │              │ Fixed-width    │                    │
│  └────────────────┘              │ EBCDIC         │                    │
│                                  └────────────────┘                    │
│                                                                         │
│  Problémy:                                                              │
│  • Moderní služba závisí na legacy modelu                             │
│  • Změna v legacy = změna v moderní službě                            │
│  • Legacy koncepty "prosakují" do nového kódu                        │
│  • Těžká údržba a evoluce                                             │
│                                                                         │
│  S ACL:                                                                │
│  ══════                                                                │
│                                                                         │
│  ┌────────────────┐  ┌────────────────┐  ┌────────────────┐           │
│  │ Modern Service │─▶│      ACL       │─▶│  Core Banking  │           │
│  │                │  │                │  │  (Mainframe)   │           │
│  │ Vlastní čistý  │  │ • Translation  │  │                │           │
│  │ doménový model │  │ • Mapping      │  │ COBOL structs  │           │
│  │ ✅ Izolovaný   │  │ • Adaptation   │  │ Fixed-width    │           │
│  └────────────────┘  └────────────────┘  └────────────────┘           │
│                                                                         │
│  ACL zajišťuje:                                                        │
│  • Překlad mezi modely                                                 │
│  • Izolaci od legacy komplexity                                       │
│  • Možnost změnit legacy bez dopadu na moderní služby                │
│  • Postupnou migraci (strangler fig pattern)                          │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### ACL v bankovním kontextu

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    ACL PŘÍKLADY V BANCE                                 │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  PŘÍKLAD 1: Core Banking Integration                                   │
│  ═══════════════════════════════════                                   │
│                                                                         │
│  Modern Model:                      Legacy Model (Mainframe):          │
│  ────────────────                   ─────────────────────────          │
│  {                                  ACCT-RECORD:                       │
│    "accountId": "uuid",               01 ACCT-NO    PIC X(10)          │
│    "balance": {                       01 BAL-AMT    PIC S9(13)V99      │
│      "amount": 1234.56,               01 BAL-SIGN   PIC X(1)           │
│      "currency": "CZK"                01 CCY-CD     PIC X(3)           │
│    },                                 01 ACCT-STAT  PIC X(2)           │
│    "status": "ACTIVE",                01 LAST-TXN   PIC 9(8)           │
│    "lastTransactionDate":           END-RECORD                         │
│      "2024-01-15T10:30:00Z"                                            │
│  }                                                                      │
│                                                                         │
│  ACL transformace:                                                      │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ Input (Mainframe)        │ ACL Transformation                   │   │
│  ├──────────────────────────┼──────────────────────────────────────│   │
│  │ ACCT-NO = "0012345678"   │ Remove leading zeros, add UUID      │   │
│  │ BAL-AMT = 000000123456   │ Divide by 100, parse sign           │   │
│  │ BAL-SIGN = "-"           │ Apply to amount                      │   │
│  │ CCY-CD = "CZK"           │ Map to ISO 4217                      │   │
│  │ ACCT-STAT = "01"         │ Map: 01→ACTIVE, 02→BLOCKED, etc.   │   │
│  │ LAST-TXN = "20240115"    │ Parse YYYYMMDD to ISO 8601          │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  PŘÍKLAD 2: Payment Gateway Integration                               │
│  ═══════════════════════════════════════                               │
│                                                                         │
│  Interní model:                    External Gateway (ISO 8583):       │
│  ────────────────                  ─────────────────────────────      │
│  PaymentRequest {                  Field 2: PAN                        │
│    cardNumber: "****1234"          Field 3: Processing Code           │
│    amount: Money                   Field 4: Amount (cents)            │
│    merchantId: "M123"              Field 42: Merchant ID              │
│    ...                             Field 49: Currency Code (numeric)  │
│  }                                 ...                                 │
│                                                                         │
│  ACL zajišťuje:                                                        │
│  • Mapování polí                                                       │
│  • Konverzi formátů (decimal → cents)                                 │
│  • Maskování citlivých dat                                            │
│  • Protocol translation                                                │
│  • Error code mapping                                                  │
│                                                                         │
│  PŘÍKLAD 3: External Data Provider                                    │
│  ═════════════════════════════════                                     │
│                                                                         │
│  Interní model:                    External Credit Bureau:            │
│  ────────────────                  ─────────────────────────          │
│  CreditScore {                     <CreditReport>                      │
│    score: 750,                       <Score value="750"/>             │
│    riskLevel: "LOW",                 <RiskIndicator>L</RiskIndicator> │
│    factors: [...]                    <Factors>...</Factors>           │
│  }                                 </CreditReport>                     │
│                                                                         │
│  ACL zajišťuje:                                                        │
│  • XML → JSON transformace                                             │
│  • Vendor-specific → standardní model                                 │
│  • Error handling pro nedostupnost                                    │
│  • Caching (kredit skóre se nemění každou minutu)                    │
│  • Circuit breaker pro externí závislost                              │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### ACL implementační vzory

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    ACL IMPLEMENTAČNÍ VZORY                              │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  VZOR 1: Synchronní ACL (API Wrapper)                                  │
│  ═════════════════════════════════════                                 │
│                                                                         │
│  ┌────────────┐    ┌─────────────────────────────┐    ┌────────────┐  │
│  │  Payment   │───▶│           ACL               │───▶│   Core     │  │
│  │  Service   │    │  ┌─────────────────────┐    │    │  Banking   │  │
│  │            │◀───│  │ Request Translator  │    │◀───│            │  │
│  └────────────┘    │  │ Response Translator │    │    └────────────┘  │
│                    │  │ Error Mapper        │    │                    │
│                    │  │ Circuit Breaker     │    │                    │
│                    │  └─────────────────────┘    │                    │
│                    └─────────────────────────────┘                    │
│                                                                         │
│  Použití:                                                              │
│  • Real-time operace (balance check, payment)                         │
│  • Nízká latence požadována                                           │
│  • Synchronní response nutná                                          │
│                                                                         │
│  VZOR 2: Asynchronní ACL (Message Translator)                         │
│  ══════════════════════════════════════════════                        │
│                                                                         │
│  ┌────────────┐    ┌───────────┐    ┌───────────┐    ┌────────────┐  │
│  │  Modern    │───▶│   Queue   │───▶│    ACL    │───▶│   Legacy   │  │
│  │  Service   │    │ (modern   │    │ Processor │    │   System   │  │
│  │            │    │  events)  │    │           │    │            │  │
│  └────────────┘    └───────────┘    └─────┬─────┘    └────────────┘  │
│                                           │                           │
│  ┌────────────┐    ┌───────────┐          │                           │
│  │  Modern    │◀───│   Queue   │◀─────────┘                           │
│  │  Service   │    │ (results) │   Translated response                │
│  └────────────┘    └───────────┘                                      │
│                                                                         │
│  Použití:                                                              │
│  • Batch operace                                                       │
│  • Event synchronizace                                                 │
│  • Když legacy nemá API (file-based)                                  │
│                                                                         │
│  VZOR 3: Database ACL (View/Materialized View)                        │
│  ══════════════════════════════════════════════                        │
│                                                                         │
│  ┌────────────┐    ┌─────────────────────────────────────┐            │
│  │  Modern    │───▶│         Virtuální vrstva            │            │
│  │  Service   │    │  ┌─────────────────────────────┐    │            │
│  │            │    │  │    Materialized Views       │    │            │
│  │  Dotazuje  │    │  │    (modern schema)          │    │            │
│  │  moderní   │    │  └──────────────┬──────────────┘    │            │
│  │  schéma    │    │                 │                    │            │
│  └────────────┘    │  ┌──────────────▼──────────────┐    │            │
│                    │  │    Legacy Database          │    │            │
│                    │  │    (original schema)        │    │            │
│                    │  └─────────────────────────────┘    │            │
│                    └─────────────────────────────────────┘            │
│                                                                         │
│  Použití:                                                              │
│  • Read-only přístup k legacy datům                                   │
│  • Reporting a analytics                                               │
│  • Postupná migrace                                                    │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 6. Circuit Breaker Pattern

### Co to je

Mechanismus pro detekci selhání a zabránění kaskádovému šíření problémů. Když služba selhává, circuit breaker "otevře" a vrací chyby okamžitě místo čekání na timeout.

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    CIRCUIT BREAKER STAVY                                │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  ┌────────────────────────────────────────────────────────────────┐    │
│  │                                                                 │    │
│  │           ┌─────────────┐                                      │    │
│  │           │   CLOSED    │  ← Normální provoz                   │    │
│  │           │             │    Requesty procházejí               │    │
│  │           └──────┬──────┘                                      │    │
│  │                  │                                              │    │
│  │                  │ Failure threshold                            │    │
│  │                  │ exceeded                                     │    │
│  │                  │ (např. 5 failures)                          │    │
│  │                  ▼                                              │    │
│  │           ┌─────────────┐                                      │    │
│  │           │    OPEN     │  ← Služba nefunguje                  │    │
│  │  Reset    │             │    Okamžitý fail (bez čekání)       │    │
│  │  timer    │  Fail fast  │    Šetří resources                   │    │
│  │  expires  └──────┬──────┘                                      │    │
│  │      ▲           │                                              │    │
│  │      │           │ After timeout                                │    │
│  │      │           │ (např. 30s)                                 │    │
│  │      │           ▼                                              │    │
│  │      │    ┌─────────────┐                                      │    │
│  │      │    │ HALF-OPEN   │  ← Test, jestli služba funguje      │    │
│  │      │    │             │    Pustí pár test requestů          │    │
│  │      │    └──────┬──────┘                                      │    │
│  │      │           │                                              │    │
│  │      │    ┌──────┴──────┐                                      │    │
│  │      │    │             │                                      │    │
│  │      │    ▼             ▼                                      │    │
│  │   Success          Failure                                     │    │
│  │      │                │                                        │    │
│  │      ▼                │                                        │    │
│  │  ┌─────────┐          │                                        │    │
│  │  │ CLOSED  │◀─────────┘                                        │    │
│  │  └─────────┘  Back to OPEN                                     │    │
│  │                                                                 │    │
│  └────────────────────────────────────────────────────────────────┘    │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### Proč v bankovnictví

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    CIRCUIT BREAKER V BANCE                              │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  SCÉNÁŘ BEZ CIRCUIT BREAKER:                                           │
│  ════════════════════════════                                          │
│                                                                         │
│  ┌─────────┐    ┌─────────┐    ┌─────────┐    ┌─────────┐            │
│  │ Mobile  │───▶│ Payment │───▶│  Fraud  │───▶│  Core   │            │
│  │  App    │    │ Service │    │ Service │    │ Banking │            │
│  └─────────┘    └─────────┘    └────┬────┘    └─────────┘            │
│       │                             │                                  │
│       │                        ❌ DOWN!                                │
│       │                             │                                  │
│       │         ┌───────────────────┘                                  │
│       │         │                                                      │
│       │         ▼                                                      │
│       │    Payment čeká 30s na timeout...                             │
│       │    Dalších 100 requestů čeká...                               │
│       │    Thread pool vyčerpán...                                    │
│       │    Payment Service přestává odpovídat...                      │
│       │    Mobile App timeoutuje...                                   │
│       │                                                                │
│       └───▶ 💥 KASKÁDOVÉ SELHÁNÍ CELÉHO SYSTÉMU                      │
│                                                                         │
│  SCÉNÁŘ S CIRCUIT BREAKER:                                             │
│  ══════════════════════════                                            │
│                                                                         │
│  ┌─────────┐    ┌─────────────────────┐    ┌─────────┐                │
│  │ Mobile  │───▶│ Payment Service     │───▶│  Fraud  │                │
│  │  App    │    │ ┌─────────────────┐ │    │ Service │                │
│  └─────────┘    │ │ Circuit Breaker │ │    └────┬────┘                │
│       │         │ │    [OPEN]       │ │         │                     │
│       │         │ └─────────────────┘ │    ❌ DOWN!                   │
│       │         └──────────┬──────────┘                               │
│       │                    │                                           │
│       │                    ▼                                           │
│       │         Okamžitá odpověď:                                     │
│       │         "Fraud check temporarily unavailable"                 │
│       │         Fallback: Proceed with risk limits                   │
│       │                                                                │
│       └───▶ ✅ SYSTÉM ZŮSTÁVÁ FUNKČNÍ (degraded mode)                │
│                                                                         │
│  KONFIGURAČNÍ PARAMETRY:                                               │
│  ════════════════════════                                              │
│                                                                         │
│  ┌────────────────────────────────────────────────────────────────┐   │
│  │ Parametr              │ Hodnota    │ Vysvětlení                │   │
│  ├───────────────────────┼────────────┼───────────────────────────│   │
│  │ Failure threshold     │ 5          │ Počet selhání pro OPEN   │   │
│  │ Success threshold     │ 3          │ Úspěchů pro CLOSED       │   │
│  │ Timeout              │ 30s        │ Čas v OPEN stavu         │   │
│  │ Half-open requests    │ 3          │ Test requestů            │   │
│  │ Failure rate window   │ 60s        │ Období pro měření        │   │
│  └────────────────────────────────────────────────────────────────┘   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### Fallback strategie

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    FALLBACK STRATEGIE                                   │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  Když Circuit Breaker otevře, co vrátit?                              │
│                                                                         │
│  STRATEGIE 1: Cached Response                                          │
│  ─────────────────────────────                                         │
│  • Vrať poslední úspěšnou odpověď                                     │
│  • Vhodné pro: Exchange rates, product catalog                        │
│  • Riziko: Stale data                                                  │
│                                                                         │
│  STRATEGIE 2: Default Response                                         │
│  ─────────────────────────────                                         │
│  • Vrať předefinovanou defaultní hodnotu                              │
│  • Vhodné pro: Feature flags, recommendations                         │
│  • Příklad: "Doporučení dočasně nedostupná"                          │
│                                                                         │
│  STRATEGIE 3: Degraded Functionality                                   │
│  ───────────────────────────────────                                   │
│  • Pokračuj s omezenou funkcionalitou                                 │
│  • Vhodné pro: Non-critical services                                  │
│  • Příklad: Platba bez fraud check (s limitem)                        │
│                                                                         │
│  STRATEGIE 4: Alternative Service                                      │
│  ─────────────────────────────────                                     │
│  • Použij záložní službu / provider                                   │
│  • Vhodné pro: Kritické služby s redundancí                           │
│  • Příklad: Záložní SMS gateway                                        │
│                                                                         │
│  STRATEGIE 5: Queue for Retry                                          │
│  ─────────────────────────────                                         │
│  • Ulož request do fronty pro pozdější zpracování                    │
│  • Vhodné pro: Non-time-critical operations                           │
│  • Příklad: Statement generování                                       │
│                                                                         │
│  PŘÍKLADY V BANKOVNICTVÍ:                                              │
│  ════════════════════════                                              │
│                                                                         │
│  ┌────────────────────────────────────────────────────────────────┐   │
│  │ Služba              │ Fallback                                 │   │
│  ├─────────────────────┼──────────────────────────────────────────│   │
│  │ Fraud Detection     │ Proceed with lower limit (1000 CZK)     │   │
│  │ Credit Bureau       │ Use cached score (max 24h old)          │   │
│  │ Exchange Rates      │ Use last known rate + spread            │   │
│  │ Notification        │ Queue for later, log for manual send    │   │
│  │ Core Banking        │ ❌ NO FALLBACK - must fail              │   │
│  │ Balance Check       │ ❌ NO FALLBACK - must fail              │   │
│  └────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  ⚠️ DŮLEŽITÉ: Některé operace NEMOHOU mít fallback!                   │
│     Platby a balance checks musí selhat čistě.                        │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 7. Bulkhead Pattern

### Co to je

Izolace komponent systému do oddělených "přihrádek" (jako lodní trupy), aby selhání jedné části nezpůsobilo selhání celku. Každá komponenta má vyhrazené resources.

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    BULKHEAD PATTERN                                     │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  ANALOGIE - Lodní trup:                                                │
│  ═══════════════════════                                               │
│                                                                         │
│  Bez bulkheads:              S bulkheads:                              │
│  ┌─────────────────┐         ┌──┬──┬──┬──┬──┐                         │
│  │     SHIP        │         │  │  │  │  │  │                         │
│  │                 │         │  │  │  │  │  │                         │
│  │  💧 LEAK       │         │💧│  │  │  │  │                         │
│  │  Celá loď se   │         │  │  │  │  │  │                         │
│  │  potápí!       │         │  │  │  │  │  │                         │
│  └─────────────────┘         └──┴──┴──┴──┴──┘                         │
│         ❌                    ✅ Jen jedna sekce                       │
│                                 zatopena                               │
│                                                                         │
│  V SOFTWARE:                                                           │
│  ════════════                                                          │
│                                                                         │
│  Bez bulkheads:                                                        │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                    SHARED THREAD POOL (100 threads)             │   │
│  │                                                                  │   │
│  │   Payments ──────▶ │░░░░░░░░░░░░░░░░│ ← 50 threads stuck       │   │
│  │   Accounts ──────▶ │░░░░░░░░░░░░░░░░│   waiting for            │   │
│  │   Cards ─────────▶ │░░░░░░░░░░░░░░░░│   slow Payments          │   │
│  │   Notifications ─▶ │░░░░░░░░░░░░░░░░│                          │   │
│  │                                                                  │   │
│  │   ❌ Pomalá Payments služba vyčerpá všechny thready            │   │
│  │   ❌ Všechny ostatní služby přestanou fungovat                 │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  S bulkheads:                                                          │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │   Payments ──────▶ │░░░░░░░░░░│ (30 threads) ← stuck           │   │
│  │                    └──────────┘                                  │   │
│  │   Accounts ──────▶ │░░░░░░░░░░│ (30 threads) ← OK ✅            │   │
│  │                    └──────────┘                                  │   │
│  │   Cards ─────────▶ │░░░░░░░░░░│ (25 threads) ← OK ✅            │   │
│  │                    └──────────┘                                  │   │
│  │   Notifications ─▶ │░░░░░░░░░░│ (15 threads) ← OK ✅            │   │
│  │                    └──────────┘                                  │   │
│  │                                                                  │   │
│  │   ✅ Payments jsou pomalé, ale ostatní fungují                  │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### Typy Bulkhead izolace

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    TYPY IZOLACE                                         │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  1. THREAD POOL ISOLATION                                              │
│  ═════════════════════════                                             │
│  Každá závislost má vlastní thread pool.                              │
│                                                                         │
│  ┌────────────────────────────────────────────────────────────────┐   │
│  │ Služba              │ Thread Pool │ Důvod                       │   │
│  ├─────────────────────┼─────────────┼─────────────────────────────│   │
│  │ Core Banking calls  │ 50 threads  │ Kritické, vysoká priorita  │   │
│  │ Fraud Detection     │ 20 threads  │ ML model, může být pomalý  │   │
│  │ Notification        │ 10 threads  │ Non-critical               │   │
│  │ External APIs       │ 15 threads  │ Nedůvěryhodné závislosti  │   │
│  └────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  2. CONNECTION POOL ISOLATION                                          │
│  ══════════════════════════════                                        │
│  Oddělené database connection pools.                                   │
│                                                                         │
│  ┌────────────────────────────────────────────────────────────────┐   │
│  │ Database            │ Connections │ Použití                    │   │
│  ├─────────────────────┼─────────────┼────────────────────────────│   │
│  │ Read replica        │ 100         │ Queries, reporting         │   │
│  │ Primary (writes)    │ 30          │ Transactions               │   │
│  │ Analytics DB        │ 20          │ Heavy queries              │   │
│  └────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  3. PROCESS/CONTAINER ISOLATION                                        │
│  ═══════════════════════════════                                       │
│  Samostatné procesy/kontejnery s resource limits.                     │
│                                                                         │
│  AWS implementace:                                                      │
│  • ECS Task Definition s CPU/Memory limits                            │
│  • Lambda s reserved concurrency                                       │
│  • Fargate s resource specifications                                   │
│                                                                         │
│  ┌────────────────────────────────────────────────────────────────┐   │
│  │ Service              │ CPU    │ Memory │ Replicas               │   │
│  ├──────────────────────┼────────┼────────┼────────────────────────│   │
│  │ Payment Service      │ 2 vCPU │ 4 GB   │ 5 (min) - 20 (max)    │   │
│  │ Account Service      │ 1 vCPU │ 2 GB   │ 3 (min) - 10 (max)    │   │
│  │ Notification Service │ 0.5 vCPU│ 1 GB  │ 2 (min) - 5 (max)     │   │
│  └────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  4. LAMBDA RESERVED CONCURRENCY                                        │
│  ═══════════════════════════════                                       │
│                                                                         │
│  ┌────────────────────────────────────────────────────────────────┐   │
│  │ Lambda Function      │ Reserved │ Důvod                        │   │
│  ├──────────────────────┼──────────┼──────────────────────────────│   │
│  │ Payment Processor    │ 200      │ Kritické, vysoká priorita   │   │
│  │ Statement Generator  │ 50       │ Batch job, nelimituje ostatní│   │
│  │ Webhook Handler      │ 100      │ External triggers           │   │
│  │ Unreserved (shared)  │ 650      │ Ostatní funkce              │   │
│  │ ─────────────────────│──────────│                              │   │
│  │ TOTAL (account limit)│ 1000     │                              │   │
│  └────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  5. QUEUE/TOPIC ISOLATION                                              │
│  ═════════════════════════                                             │
│  Oddělené fronty pro různé typy zpráv.                                │
│                                                                         │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐                    │
│  │ High Priority│  │   Normal    │  │ Low Priority│                    │
│  │    Queue     │  │   Queue     │  │    Queue    │                    │
│  │ (Payments)   │  │ (General)   │  │ (Reports)   │                    │
│  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘                    │
│         │                │                │                            │
│    5 consumers      3 consumers      1 consumer                        │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 8. Strangler Fig Pattern

### Co to je

Postupná migrace z legacy systému na nový systém. Nový systém "obalí" legacy a postupně přebírá funkcionalitu, až legacy "uškrtí" (jako fíkus strom).

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    STRANGLER FIG PATTERN                                │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  FÁZE 1: Začátek (Legacy dominuje)                                     │
│  ══════════════════════════════════                                    │
│                                                                         │
│  ┌─────────┐     ┌───────────────┐     ┌─────────────────────────┐    │
│  │ Clients │────▶│    Facade     │────▶│      LEGACY SYSTEM      │    │
│  └─────────┘     │   (Router)    │     │                         │    │
│                  └───────────────┘     │  ████████████████████   │    │
│                                        │  ████████████████████   │    │
│                                        │  ████████████████████   │    │
│                                        └─────────────────────────┘    │
│                                                                         │
│  FÁZE 2: Migrace začíná (první moduly)                                │
│  ═════════════════════════════════════                                 │
│                                                                         │
│  ┌─────────┐     ┌───────────────┐     ┌─────────────────────────┐    │
│  │ Clients │────▶│    Facade     │──┬─▶│      LEGACY SYSTEM      │    │
│  └─────────┘     │   (Router)    │  │  │                         │    │
│                  └───────────────┘  │  │  ████████████████████   │    │
│                         │           │  │  ████████████████████   │    │
│                         │           │  └─────────────────────────┘    │
│                         │           │                                  │
│                         │           │  ┌─────────────────────────┐    │
│                         └───────────┴─▶│    NEW MICROSERVICE     │    │
│                                        │    (Accounts Module)    │    │
│                                        │  ░░░░░░                 │    │
│                                        └─────────────────────────┘    │
│                                                                         │
│  FÁZE 3: Postupná migrace (více modulů)                               │
│  ══════════════════════════════════════                                │
│                                                                         │
│  ┌─────────┐     ┌───────────────┐     ┌─────────────────────────┐    │
│  │ Clients │────▶│    Facade     │──┬─▶│      LEGACY SYSTEM      │    │
│  └─────────┘     │   (Router)    │  │  │  ████████████           │    │
│                  └───────────────┘  │  │  (zbývající funkce)     │    │
│                         │           │  └─────────────────────────┘    │
│                         │           │                                  │
│                         │           │  ┌─────────────────────────┐    │
│                         ├───────────┴─▶│    Account Service      │    │
│                         │              └─────────────────────────┘    │
│                         │              ┌─────────────────────────┐    │
│                         ├─────────────▶│    Payment Service      │    │
│                         │              └─────────────────────────┘    │
│                         │              ┌─────────────────────────┐    │
│                         └─────────────▶│    Card Service         │    │
│                                        └─────────────────────────┘    │
│                                                                         │
│  FÁZE 4: Dokončení (Legacy odstraněn)                                 │
│  ════════════════════════════════════                                  │
│                                                                         │
│  ┌─────────┐     ┌───────────────┐     ┌─────────────────────────┐    │
│  │ Clients │────▶│  API Gateway  │──┬─▶│    Account Service      │    │
│  └─────────┘     │               │  │  └─────────────────────────┘    │
│                  └───────────────┘  │  ┌─────────────────────────┐    │
│                                     ├─▶│    Payment Service      │    │
│                                     │  └─────────────────────────┘    │
│                                     │  ┌─────────────────────────┐    │
│                                     └─▶│    Card Service         │    │
│                                        └─────────────────────────┘    │
│                                                                         │
│                  Legacy system decommissioned ✅                       │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### Strangler Fig v bankovním kontextu

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    MIGRACE CORE BANKING                                 │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  Typický scénář: Banka má 30 let starý mainframe core banking          │
│                  Chce modernizovat, ale nemůže vypnout                 │
│                                                                         │
│  STRATEGIE MIGRACE:                                                     │
│  ══════════════════                                                    │
│                                                                         │
│  ┌────────────────────────────────────────────────────────────────┐   │
│  │ Fáze │ Modul              │ Riziko │ Délka │ Poznámka          │   │
│  ├──────┼────────────────────┼────────┼───────┼───────────────────│   │
│  │  1   │ Customer Profile   │ Low    │ 3 měs │ Read-mostly data │   │
│  │  2   │ Notifications      │ Low    │ 2 měs │ Non-critical     │   │
│  │  3   │ Card Management    │ Medium │ 6 měs │ Bounded context  │   │
│  │  4   │ Standing Orders    │ Medium │ 4 měs │ Batch processing │   │
│  │  5   │ Transaction History│ Medium │ 4 měs │ Read-only        │   │
│  │  6   │ Payment Processing │ High   │ 12 měs│ Core business    │   │
│  │  7   │ Account Ledger     │ High   │ 18 měs│ Source of truth  │   │
│  └────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  KLÍČOVÉ PRINCIPY:                                                     │
│  ─────────────────                                                     │
│  1. Začni s low-risk, high-value moduly                               │
│  2. Zachovej zpětnou kompatibilitu                                    │
│  3. Feature flags pro postupné přepínání                              │
│  4. Dual-write období pro validaci                                    │
│  5. Rollback strategie pro každý modul                                │
│                                                                         │
│  FACADE IMPLEMENTACE:                                                  │
│  ════════════════════                                                  │
│                                                                         │
│  ┌──────────────────────────────────────────────────────────────────┐ │
│  │                       API GATEWAY (Facade)                        │ │
│  │                                                                   │ │
│  │  Route: /accounts/*                                              │ │
│  │  ┌─────────────────────────────────────────────────────────────┐ │ │
│  │  │ if (featureFlag.newAccountService.enabled(customerId))      │ │ │
│  │  │     route to: New Account Service                           │ │ │
│  │  │ else                                                         │ │ │
│  │  │     route to: Legacy (via ACL)                              │ │ │
│  │  └─────────────────────────────────────────────────────────────┘ │ │
│  │                                                                   │ │
│  │  Route: /payments/*                                              │ │
│  │  ┌─────────────────────────────────────────────────────────────┐ │ │
│  │  │ route to: Legacy (migration not started)                    │ │ │
│  │  └─────────────────────────────────────────────────────────────┘ │ │
│  │                                                                   │ │
│  └──────────────────────────────────────────────────────────────────┘ │
│                                                                         │
│  DUAL-WRITE VALIDACE:                                                  │
│  ═════════════════════                                                 │
│                                                                         │
│  ┌─────────┐     ┌───────────────┐                                    │
│  │ Request │────▶│   Facade      │                                    │
│  └─────────┘     └───────┬───────┘                                    │
│                          │                                             │
│              ┌───────────┴───────────┐                                │
│              │                       │                                 │
│              ▼                       ▼                                 │
│       ┌─────────────┐         ┌─────────────┐                         │
│       │   Legacy    │         │    New      │                         │
│       │   System    │         │   Service   │                         │
│       └──────┬──────┘         └──────┬──────┘                         │
│              │                       │                                 │
│              └───────────┬───────────┘                                │
│                          ▼                                             │
│                  ┌───────────────┐                                    │
│                  │  Comparator   │                                    │
│                  │  (async)      │                                    │
│                  │               │                                    │
│                  │ Result match? │                                    │
│                  │ Log diff      │                                    │
│                  │ Alert on diff │                                    │
│                  └───────────────┘                                    │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 9. Outbox Pattern

### Co to je

Řešení problému atomického zápisu do databáze a publikování eventu. Místo přímého publish do message broker zapisujeme do outbox tabulky ve stejné transakci, a separátní proces publikuje eventy.

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    PROBLÉM BEZ OUTBOX                                   │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  Scénář: Vytvoření platby + publikování eventu                        │
│                                                                         │
│  ┌──────────────┐                                                      │
│  │   Service    │                                                      │
│  │              │                                                      │
│  │  1. BEGIN    │                                                      │
│  │  2. INSERT   │─────────────▶ Database ✅                           │
│  │  3. COMMIT   │                                                      │
│  │  4. PUBLISH  │─────────────▶ Message Broker                        │
│  │              │                   │                                  │
│  └──────────────┘                   ❌ FAIL (network error)           │
│                                                                         │
│  PROBLÉM:                                                              │
│  • Platba je v DB ✅                                                   │
│  • Event nebyl publikován ❌                                          │
│  • Systém je v nekonzistentním stavu                                  │
│  • Downstream services neví o platbě                                   │
│                                                                         │
│  Opačný scénář:                                                        │
│  │  1. BEGIN    │                                                      │
│  │  2. INSERT   │─────────────▶ Database ✅                           │
│  │  3. PUBLISH  │─────────────▶ Message Broker ✅                     │
│  │  4. COMMIT   │─────────────▶ Database                              │
│  │              │                   │                                  │
│  └──────────────┘                   ❌ FAIL (deadlock)                │
│                                                                         │
│  PROBLÉM:                                                              │
│  • Event publikován ✅                                                 │
│  • Transakce rollback ❌                                              │
│  • Platba neexistuje, ale event byl odeslán!                         │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### Outbox řešení

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    OUTBOX PATTERN                                       │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │                    SINGLE TRANSACTION                             │  │
│  │                                                                   │  │
│  │  1. BEGIN TRANSACTION                                            │  │
│  │                                                                   │  │
│  │  2. INSERT INTO payments (id, amount, ...)                      │  │
│  │     VALUES ('pay-123', 1000, ...)                               │  │
│  │                                                                   │  │
│  │  3. INSERT INTO outbox (id, aggregate_type, aggregate_id,       │  │
│  │                         event_type, payload, created_at)        │  │
│  │     VALUES ('evt-456', 'Payment', 'pay-123',                    │  │
│  │             'PaymentCreated', '{"amount": 1000}', NOW())        │  │
│  │                                                                   │  │
│  │  4. COMMIT                                                       │  │
│  │                                                                   │  │
│  └──────────────────────────────────────────────────────────────────┘  │
│                                                                         │
│  ✅ Obě operace atomické - buď obě nebo žádná                        │
│                                                                         │
│                              │                                          │
│                              ▼                                          │
│                                                                         │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │                    OUTBOX PROCESSOR                               │  │
│  │                    (Separate Process)                             │  │
│  │                                                                   │  │
│  │  LOOP:                                                            │  │
│  │    1. SELECT * FROM outbox                                       │  │
│  │       WHERE processed = false                                    │  │
│  │       ORDER BY created_at                                        │  │
│  │       LIMIT 100                                                  │  │
│  │                                                                   │  │
│  │    2. For each event:                                            │  │
│  │       a. PUBLISH to EventBridge/Kafka/SNS                       │  │
│  │       b. UPDATE outbox SET processed = true                     │  │
│  │          WHERE id = event_id                                     │  │
│  │                                                                   │  │
│  │    3. Sleep / Poll interval                                      │  │
│  │                                                                   │  │
│  └──────────────────────────────────────────────────────────────────┘  │
│                                                                         │
│  OUTBOX TABULKA:                                                       │
│  ════════════════                                                      │
│                                                                         │
│  ┌────────────────────────────────────────────────────────────────┐   │
│  │ id       │ aggregate │ event_type      │ payload    │processed│   │
│  ├──────────┼───────────┼─────────────────┼────────────┼─────────│   │
│  │ evt-456  │ pay-123   │ PaymentCreated  │ {json}     │ false   │   │
│  │ evt-457  │ pay-124   │ PaymentCreated  │ {json}     │ false   │   │
│  │ evt-458  │ pay-123   │ PaymentCompleted│ {json}     │ true    │   │
│  └────────────────────────────────────────────────────────────────┘   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### Outbox implementace v AWS

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    OUTBOX V AWS                                         │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  VARIANTA 1: DynamoDB Streams                                          │
│  ══════════════════════════════                                        │
│                                                                         │
│  ┌────────────┐    ┌─────────────────┐    ┌────────────────┐          │
│  │  Lambda    │───▶│    DynamoDB     │───▶│   DynamoDB     │          │
│  │ (write)    │    │   (main table)  │    │    Streams     │          │
│  └────────────┘    └─────────────────┘    └───────┬────────┘          │
│                                                   │                    │
│                                                   ▼                    │
│                                           ┌────────────────┐          │
│                                           │    Lambda      │          │
│                                           │  (processor)   │          │
│                                           └───────┬────────┘          │
│                                                   │                    │
│                                                   ▼                    │
│                                           ┌────────────────┐          │
│                                           │  EventBridge   │          │
│                                           └────────────────┘          │
│                                                                         │
│  Výhody:                                                                │
│  • Není potřeba explicitní outbox tabulka                             │
│  • DynamoDB Streams = built-in CDC                                    │
│  • Automatické škálování                                               │
│                                                                         │
│  VARIANTA 2: RDS + Polling                                             │
│  ═══════════════════════════                                           │
│                                                                         │
│  ┌────────────┐    ┌─────────────────┐                                │
│  │  Lambda    │───▶│    Aurora/RDS   │                                │
│  │ (write)    │    │  ┌───────────┐  │                                │
│  └────────────┘    │  │  outbox   │  │                                │
│                    │  │  table    │  │                                │
│                    │  └─────┬─────┘  │                                │
│                    └────────┼────────┘                                │
│                             │                                          │
│  ┌──────────────────────────┼──────────────────────────┐              │
│  │  EventBridge Scheduler   │   (every 1 second)       │              │
│  │  or CloudWatch Events    ▼                          │              │
│  └──────────────────────────┼──────────────────────────┘              │
│                             │                                          │
│                    ┌────────▼────────┐                                │
│                    │     Lambda      │                                │
│                    │  (poll & publish)│                               │
│                    └────────┬────────┘                                │
│                             │                                          │
│                             ▼                                          │
│                    ┌────────────────┐                                 │
│                    │  EventBridge   │                                 │
│                    └────────────────┘                                 │
│                                                                         │
│  VARIANTA 3: Transactional Outbox s CDC (Debezium)                   │
│  ══════════════════════════════════════════════════                    │
│                                                                         │
│  ┌────────────┐    ┌─────────────────┐    ┌────────────────┐          │
│  │  Service   │───▶│    Aurora       │───▶│   DMS/Debezium │          │
│  │ (write)    │    │  (with outbox)  │    │   (CDC)        │          │
│  └────────────┘    └─────────────────┘    └───────┬────────┘          │
│                                                   │                    │
│                                                   ▼                    │
│                                           ┌────────────────┐          │
│                                           │     MSK        │          │
│                                           │    (Kafka)     │          │
│                                           └────────────────┘          │
│                                                                         │
│  Výhody:                                                                │
│  • Log-based CDC (neinvazivní)                                        │
│  • Zachycuje všechny změny                                            │
│  • Ordered delivery                                                    │
│                                                                         │
│  DŮLEŽITÉ ASPEKTY:                                                     │
│  ═════════════════                                                     │
│                                                                         │
│  1. Idempotency                                                        │
│     • Outbox processor může zpracovat event vícekrát                  │
│     • Consumers musí být idempotentní                                 │
│     • Použij event ID pro deduplikaci                                 │
│                                                                         │
│  2. Ordering                                                           │
│     • Events pro stejný aggregate musí být in-order                   │
│     • Partition by aggregate_id                                       │
│                                                                         │
│  3. Cleanup                                                            │
│     • Mazání starých processed events (TTL nebo batch job)           │
│     • Archivace pro audit                                              │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## Shrnutí: Kdy použít který pattern

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    DECISION MATRIX                                      │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  ┌────────────────────────────────────────────────────────────────┐   │
│  │ Problém                        │ Pattern                       │   │
│  ├────────────────────────────────┼───────────────────────────────│   │
│  │ Distribuované transakce        │ Saga                          │   │
│  │ Read/Write škálování           │ CQRS                          │   │
│  │ Audit trail, temporal queries  │ Event Sourcing                │   │
│  │ Optimalizace pro kanály        │ Backend for Frontend (BFF)    │   │
│  │ Legacy integrace               │ Anti-Corruption Layer         │   │
│  │ Kaskádové selhání             │ Circuit Breaker               │   │
│  │ Resource izolace               │ Bulkhead                      │   │
│  │ Postupná migrace               │ Strangler Fig                 │   │
│  │ Atomický publish + write       │ Outbox                        │   │
│  └────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  KOMBINACE PRO BANKOVNÍ PLATBY:                                        │
│  ══════════════════════════════                                        │
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────────┐  │
│  │                                                                  │  │
│  │   Mobile App                                                     │  │
│  │       │                                                          │  │
│  │       ▼                                                          │  │
│  │   ┌─────────┐                                                   │  │
│  │   │Mobile BFF│ ◀─── BFF Pattern                                 │  │
│  │   └────┬────┘                                                   │  │
│  │        │                                                         │  │
│  │        ▼                                                         │  │
│  │   ┌─────────┐     ┌─────────┐                                   │  │
│  │   │ Payment │────▶│  Saga   │ ◀─── Saga Pattern                 │  │
│  │   │ Service │     │Orchestr.│                                   │  │
│  │   └────┬────┘     └────┬────┘                                   │  │
│  │        │               │                                         │  │
│  │        │          ┌────┴────┐                                   │  │
│  │        │          ▼         ▼                                   │  │
│  │        │    ┌─────────┐ ┌─────────┐                             │  │
│  │        │    │ Account │ │  Fraud  │ ◀─── Circuit Breaker        │  │
│  │        │    │ Service │ │ Service │      (pro Fraud)            │  │
│  │        │    └────┬────┘ └─────────┘                             │  │
│  │        │         │                                               │  │
│  │        │    ┌────▼────┐                                         │  │
│  │        │    │  Event  │ ◀─── Event Sourcing                     │  │
│  │        │    │  Store  │                                         │  │
│  │        │    └────┬────┘                                         │  │
│  │        │         │                                               │  │
│  │        │    ┌────▼────┐                                         │  │
│  │        │    │ Outbox  │ ◀─── Outbox Pattern                     │  │
│  │        │    │ Table   │                                         │  │
│  │        │    └────┬────┘                                         │  │
│  │        │         │                                               │  │
│  │        │    ┌────▼────┐     ┌─────────┐                         │  │
│  │        │    │Event Bus│────▶│  CQRS   │ ◀─── CQRS               │  │
│  │        │    └─────────┘     │ Projec. │                         │  │
│  │        │                    └─────────┘                         │  │
│  │        │                                                         │  │
│  │        │    ┌─────────┐                                         │  │
│  │        └───▶│   ACL   │────▶ Core Banking ◀─── ACL              │  │
│  │             └─────────┘      (Legacy)                           │  │
│  │                                                                  │  │
│  └─────────────────────────────────────────────────────────────────┘  │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

Chceš, abych rozvedl některý pattern více do detailu, nebo přešel na jiné téma jako PSD2/Open Banking, messaging patterns, nebo observability?

# Architektonické dovednosti pro Solution Architekta

---

## 1. C4 Model a architektonická dokumentace

### Co je C4 Model

C4 model je hierarchický přístup k vizualizaci softwarové architektury vytvořený Simonem Brownem. Název pochází ze čtyř úrovní abstrakce: Context, Container, Component, Code.

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    C4 MODEL - ÚROVNĚ ABSTRAKCE                          │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  ÚROVEŇ 1: SYSTEM CONTEXT                                              │
│  ═════════════════════════                                             │
│  • Nejvyšší úroveň abstrakce                                           │
│  • Systém jako black box                                               │
│  • Ukazuje uživatele a externí systémy                                 │
│  • Audience: Všichni (business, tech, management)                      │
│  • Otázka: "Co děláme a s kým komunikujeme?"                          │
│                                                                         │
│           ┌─────────────────────────────────────────┐                  │
│           │                ZOOM                      │                  │
│           │                  │                       │                  │
│           │                  ▼                       │                  │
│  ÚROVEŇ 2: CONTAINER DIAGRAM                        │                  │
│  ═══════════════════════════                        │                  │
│  • Vysokoúrovňová technická architektura            │                  │
│  • Aplikace, služby, databáze, message brokers      │                  │
│  • Technologie a protokoly                          │                  │
│  • Audience: Architekti, senior developeři          │                  │
│  • Otázka: "Z jakých běžících částí se systém skládá?"                │
│           │                                          │                  │
│           │                  │                       │                  │
│           │                  ▼                       │                  │
│  ÚROVEŇ 3: COMPONENT DIAGRAM                        │                  │
│  ════════════════════════════                       │                  │
│  • Vnitřní struktura kontejneru                     │                  │
│  • Komponenty, jejich zodpovědnosti a interakce     │                  │
│  • Audience: Developeři pracující na kontejneru     │                  │
│  • Otázka: "Z jakých stavebních bloků se kontejner skládá?"           │
│           │                                          │                  │
│           │                  │                       │                  │
│           │                  ▼                       │                  │
│  ÚROVEŇ 4: CODE DIAGRAM (volitelné)                 │                  │
│  ══════════════════════════════                     │                  │
│  • UML class diagramy, ER diagramy                  │                  │
│  • Detailní implementace                            │                  │
│  • Audience: Developeři                             │                  │
│  • Často generováno z kódu (IDE, reverse engineering)                  │
│           │                                          │                  │
│           └─────────────────────────────────────────┘                  │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### Level 1: System Context Diagram

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    SYSTEM CONTEXT - DIGITAL BANKING                     │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│                                                                         │
│    ┌─────────────┐                              ┌─────────────┐        │
│    │   Retail    │                              │  Corporate  │        │
│    │  Customer   │                              │  Customer   │        │
│    │  [Person]   │                              │  [Person]   │        │
│    └──────┬──────┘                              └──────┬──────┘        │
│           │                                            │               │
│           │ Uses mobile/web                            │               │
│           │ banking                                    │               │
│           │                                            │               │
│           ▼                                            ▼               │
│    ┌─────────────────────────────────────────────────────────┐        │
│    │                                                         │        │
│    │              DIGITAL BANKING PLATFORM                   │        │
│    │                   [Software System]                     │        │
│    │                                                         │        │
│    │  Provides internet and mobile banking services          │        │
│    │  for retail and corporate customers                     │        │
│    │                                                         │        │
│    └───────────────────────────┬─────────────────────────────┘        │
│                                │                                       │
│         ┌──────────────────────┼──────────────────────┐               │
│         │                      │                      │               │
│         ▼                      ▼                      ▼               │
│  ┌─────────────┐       ┌─────────────┐       ┌─────────────┐         │
│  │    Core     │       │   Card      │       │   Payment   │         │
│  │   Banking   │       │  Processor  │       │   Gateway   │         │
│  │  [External] │       │  [External] │       │  [External] │         │
│  │             │       │             │       │             │         │
│  │ Mainframe   │       │ Visa/MC     │       │ SEPA/SWIFT  │         │
│  │ legacy      │       │ network     │       │ network     │         │
│  └─────────────┘       └─────────────┘       └─────────────┘         │
│                                                                        │
│         ┌──────────────────────┬──────────────────────┐               │
│         ▼                      ▼                      ▼               │
│  ┌─────────────┐       ┌─────────────┐       ┌─────────────┐         │
│  │   Credit    │       │    AML      │       │    TPP      │         │
│  │   Bureau    │       │   System    │       │  (PSD2)     │         │
│  │  [External] │       │  [External] │       │  [External] │         │
│  └─────────────┘       └─────────────┘       └─────────────┘         │
│                                                                        │
│                                                                        │
│  LEGENDA:                                                              │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐                   │
│  │  [Person]   │  │  [Software  │  │  [External  │                   │
│  │             │  │   System]   │  │   System]   │                   │
│  └─────────────┘  └─────────────┘  └─────────────┘                   │
│                                                                        │
└─────────────────────────────────────────────────────────────────────────┘
```

### Level 2: Container Diagram

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    CONTAINER DIAGRAM - DIGITAL BANKING                  │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  ┌─────────────┐     ┌─────────────┐                                   │
│  │   Mobile    │     │    Web      │                                   │
│  │    App      │     │  Browser    │                                   │
│  │ [Container] │     │ [Container] │                                   │
│  │ iOS/Android │     │    React    │                                   │
│  └──────┬──────┘     └──────┬──────┘                                   │
│         │                   │                                           │
│         │    HTTPS/JSON     │                                           │
│         └─────────┬─────────┘                                           │
│                   ▼                                                     │
│  ┌───────────────────────────────────────────────────────────────────┐ │
│  │                        API GATEWAY                                 │ │
│  │                       [Container]                                  │ │
│  │                    AWS API Gateway                                 │ │
│  │                                                                    │ │
│  │   • Authentication/Authorization                                   │ │
│  │   • Rate limiting                                                  │ │
│  │   • Request routing                                                │ │
│  └───────────────────────────┬───────────────────────────────────────┘ │
│                              │                                          │
│      ┌───────────────────────┼───────────────────────┐                 │
│      │                       │                       │                 │
│      ▼                       ▼                       ▼                 │
│ ┌──────────────┐      ┌──────────────┐      ┌──────────────┐          │
│ │   Account    │      │   Payment    │      │    Card      │          │
│ │   Service    │      │   Service    │      │   Service    │          │
│ │ [Container]  │      │ [Container]  │      │ [Container]  │          │
│ │              │      │              │      │              │          │
│ │ Java/Spring  │      │ Java/Spring  │      │   Node.js    │          │
│ │ ECS Fargate  │      │ ECS Fargate  │      │   Lambda     │          │
│ └──────┬───────┘      └──────┬───────┘      └──────┬───────┘          │
│        │                     │                     │                   │
│        │                     │                     │                   │
│        ▼                     ▼                     ▼                   │
│ ┌──────────────┐      ┌──────────────┐      ┌──────────────┐          │
│ │   Account    │      │   Payment    │      │    Card      │          │
│ │   Database   │      │   Database   │      │   Database   │          │
│ │ [Container]  │      │ [Container]  │      │ [Container]  │          │
│ │              │      │              │      │              │          │
│ │ Aurora MySQL │      │ Aurora MySQL │      │  DynamoDB    │          │
│ └──────────────┘      └──────────────┘      └──────────────┘          │
│                                                                         │
│                              │                                          │
│                              ▼                                          │
│              ┌───────────────────────────────┐                         │
│              │         EVENT BUS             │                         │
│              │        [Container]            │                         │
│              │       Amazon MSK              │                         │
│              │                               │                         │
│              │  Async communication          │                         │
│              │  Event distribution           │                         │
│              └───────────────────────────────┘                         │
│                              │                                          │
│              ┌───────────────┴───────────────┐                         │
│              ▼                               ▼                         │
│       ┌──────────────┐              ┌──────────────┐                  │
│       │ Notification │              │   Audit      │                  │
│       │   Service    │              │   Service    │                  │
│       │ [Container]  │              │ [Container]  │                  │
│       │   Lambda     │              │   Lambda     │                  │
│       └──────────────┘              └──────────────┘                  │
│                                                                         │
│                                                                         │
│  LEGENDA:                                                              │
│  ┌──────────────┐   ┌──────────────┐   ┌──────────────┐               │
│  │ [Container:  │   │ [Container:  │   │ [Container:  │               │
│  │  Application]│   │  Database]   │   │  Message Bus]│               │
│  └──────────────┘   └──────────────┘   └──────────────┘               │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### Level 3: Component Diagram

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    COMPONENT DIAGRAM - PAYMENT SERVICE                  │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                      PAYMENT SERVICE                             │   │
│  │                      [Container]                                 │   │
│  │                                                                  │   │
│  │  ┌────────────────────────────────────────────────────────────┐ │   │
│  │  │                    API LAYER                                │ │   │
│  │  │                                                             │ │   │
│  │  │  ┌─────────────────┐    ┌─────────────────┐                │ │   │
│  │  │  │ Payment         │    │ Standing Order  │                │ │   │
│  │  │  │ Controller      │    │ Controller      │                │ │   │
│  │  │  │ [Component]     │    │ [Component]     │                │ │   │
│  │  │  │                 │    │                 │                │ │   │
│  │  │  │ REST endpoints  │    │ REST endpoints  │                │ │   │
│  │  │  │ for payments    │    │ for recurring   │                │ │   │
│  │  │  └────────┬────────┘    └────────┬────────┘                │ │   │
│  │  │           │                      │                          │ │   │
│  │  └───────────┼──────────────────────┼──────────────────────────┘ │   │
│  │              │                      │                            │   │
│  │              ▼                      ▼                            │   │
│  │  ┌────────────────────────────────────────────────────────────┐ │   │
│  │  │                   DOMAIN LAYER                              │ │   │
│  │  │                                                             │ │   │
│  │  │  ┌─────────────────┐    ┌─────────────────┐                │ │   │
│  │  │  │ Payment         │    │ Validation      │                │ │   │
│  │  │  │ Processor       │───▶│ Service         │                │ │   │
│  │  │  │ [Component]     │    │ [Component]     │                │ │   │
│  │  │  │                 │    │                 │                │ │   │
│  │  │  │ Orchestrates    │    │ IBAN, amount,   │                │ │   │
│  │  │  │ payment flow    │    │ business rules  │                │ │   │
│  │  │  └────────┬────────┘    └─────────────────┘                │ │   │
│  │  │           │                                                 │ │   │
│  │  │           │         ┌─────────────────┐                    │ │   │
│  │  │           ├────────▶│ Fraud Checker   │                    │ │   │
│  │  │           │         │ [Component]     │                    │ │   │
│  │  │           │         │                 │                    │ │   │
│  │  │           │         │ Risk scoring    │                    │ │   │
│  │  │           │         └─────────────────┘                    │ │   │
│  │  │           │                                                 │ │   │
│  │  │           │         ┌─────────────────┐                    │ │   │
│  │  │           └────────▶│ Fee Calculator  │                    │ │   │
│  │  │                     │ [Component]     │                    │ │   │
│  │  │                     │                 │                    │ │   │
│  │  │                     │ Calculates fees │                    │ │   │
│  │  │                     │ and charges     │                    │ │   │
│  │  │                     └─────────────────┘                    │ │   │
│  │  │                                                             │ │   │
│  │  └────────────────────────────────────────────────────────────┘ │   │
│  │              │                                                   │   │
│  │              ▼                                                   │   │
│  │  ┌────────────────────────────────────────────────────────────┐ │   │
│  │  │                 INFRASTRUCTURE LAYER                        │ │   │
│  │  │                                                             │ │   │
│  │  │  ┌─────────────────┐    ┌─────────────────┐                │ │   │
│  │  │  │ Payment         │    │ Core Banking    │                │ │   │
│  │  │  │ Repository      │    │ Client          │                │ │   │
│  │  │  │ [Component]     │    │ [Component]     │                │ │   │
│  │  │  │                 │    │                 │                │ │   │
│  │  │  │ Database access │    │ ACL to legacy   │                │ │   │
│  │  │  │ Aurora MySQL    │    │ mainframe       │                │ │   │
│  │  │  └─────────────────┘    └─────────────────┘                │ │   │
│  │  │                                                             │ │   │
│  │  │  ┌─────────────────┐    ┌─────────────────┐                │ │   │
│  │  │  │ Event           │    │ External        │                │ │   │
│  │  │  │ Publisher       │    │ Payment Gateway │                │ │   │
│  │  │  │ [Component]     │    │ [Component]     │                │ │   │
│  │  │  │                 │    │                 │                │ │   │
│  │  │  │ Kafka producer  │    │ SEPA/SWIFT      │                │ │   │
│  │  │  │ for events      │    │ integration     │                │ │   │
│  │  │  └─────────────────┘    └─────────────────┘                │ │   │
│  │  │                                                             │ │   │
│  │  └────────────────────────────────────────────────────────────┘ │   │
│  │                                                                  │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### Doplňkové diagramy (mimo C4)

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    DOPLŇKOVÉ DIAGRAMY                                   │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  1. DEPLOYMENT DIAGRAM                                                  │
│  ═════════════════════                                                 │
│  Jak jsou kontejnery nasazeny na infrastrukturu.                       │
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                     AWS CLOUD                                    │   │
│  │                                                                  │   │
│  │  ┌─────────────────────────────────────────────────────────┐    │   │
│  │  │                    VPC (10.0.0.0/16)                     │    │   │
│  │  │                                                          │    │   │
│  │  │   ┌─────────────────┐    ┌─────────────────┐            │    │   │
│  │  │   │  Public Subnet  │    │  Public Subnet  │            │    │   │
│  │  │   │   AZ-1 (a)      │    │   AZ-2 (b)      │            │    │   │
│  │  │   │                 │    │                 │            │    │   │
│  │  │   │  ┌───────────┐  │    │  ┌───────────┐  │            │    │   │
│  │  │   │  │    ALB    │  │    │  │    ALB    │  │            │    │   │
│  │  │   │  └───────────┘  │    │  └───────────┘  │            │    │   │
│  │  │   └─────────────────┘    └─────────────────┘            │    │   │
│  │  │                                                          │    │   │
│  │  │   ┌─────────────────┐    ┌─────────────────┐            │    │   │
│  │  │   │ Private Subnet  │    │ Private Subnet  │            │    │   │
│  │  │   │   AZ-1 (a)      │    │   AZ-2 (b)      │            │    │   │
│  │  │   │                 │    │                 │            │    │   │
│  │  │   │  ┌───────────┐  │    │  ┌───────────┐  │            │    │   │
│  │  │   │  │ECS Fargate│  │    │  │ECS Fargate│  │            │    │   │
│  │  │   │  │ Services  │  │    │  │ Services  │  │            │    │   │
│  │  │   │  └───────────┘  │    │  └───────────┘  │            │    │   │
│  │  │   └─────────────────┘    └─────────────────┘            │    │   │
│  │  │                                                          │    │   │
│  │  │   ┌─────────────────┐    ┌─────────────────┐            │    │   │
│  │  │   │   Data Subnet   │    │   Data Subnet   │            │    │   │
│  │  │   │   AZ-1 (a)      │    │   AZ-2 (b)      │            │    │   │
│  │  │   │                 │    │                 │            │    │   │
│  │  │   │  ┌───────────┐  │    │  ┌───────────┐  │            │    │   │
│  │  │   │  │  Aurora   │◀─┼────┼─▶│  Aurora   │  │            │    │   │
│  │  │   │  │ (Primary) │  │    │  │ (Replica) │  │            │    │   │
│  │  │   │  └───────────┘  │    │  └───────────┘  │            │    │   │
│  │  │   └─────────────────┘    └─────────────────┘            │    │   │
│  │  │                                                          │    │   │
│  │  └─────────────────────────────────────────────────────────┘    │   │
│  │                                                                  │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  2. DYNAMIC DIAGRAM (Sequence)                                         │
│  ══════════════════════════════                                        │
│  Jak komponenty spolupracují pro konkrétní use case.                   │
│                                                                         │
│  Payment Flow:                                                          │
│  ┌────────┐  ┌────────┐  ┌────────┐  ┌────────┐  ┌────────┐           │
│  │ Mobile │  │  API   │  │Payment │  │ Fraud  │  │  Core  │           │
│  │  App   │  │Gateway │  │Service │  │Service │  │Banking │           │
│  └───┬────┘  └───┬────┘  └───┬────┘  └───┬────┘  └───┬────┘           │
│      │           │           │           │           │                 │
│      │──POST────▶│           │           │           │                 │
│      │ /payment  │──route───▶│           │           │                 │
│      │           │           │──check───▶│           │                 │
│      │           │           │◀──ok──────│           │                 │
│      │           │           │──execute─────────────▶│                 │
│      │           │           │◀─────────────────ok───│                 │
│      │◀──────────│◀──────────│           │           │                 │
│      │  202      │           │           │           │                 │
│      │           │           │           │           │                 │
│                                                                         │
│  3. LANDSCAPE DIAGRAM                                                  │
│  ════════════════════                                                  │
│  Jak více systémů spolupracuje na úrovni enterprise.                  │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### Pravidla a best practices pro C4

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    C4 BEST PRACTICES                                    │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  OBECNÁ PRAVIDLA:                                                      │
│  ════════════════                                                      │
│                                                                         │
│  1. Každý box obsahuje:                                                │
│     • Název                                                            │
│     • Typ [Person/System/Container/Component]                          │
│     • Technologie (kde relevantní)                                     │
│     • Stručný popis odpovědnosti                                      │
│                                                                         │
│  2. Každá šipka obsahuje:                                              │
│     • Směr komunikace                                                  │
│     • Popis (co se přenáší)                                           │
│     • Protokol/technologie                                             │
│                                                                         │
│  3. Vždy přidej legendu                                                │
│                                                                         │
│  4. Udržuj diagramy aktuální (automatizace kde možné)                 │
│                                                                         │
│  ÚROVEŇ DETAILU:                                                       │
│  ════════════════                                                      │
│                                                                         │
│  ┌────────────────────────────────────────────────────────────────┐   │
│  │ Úroveň       │ Audience           │ Aktualizace               │   │
│  ├──────────────┼────────────────────┼───────────────────────────│   │
│  │ L1 Context   │ Všichni            │ Při změně scope/integrace │   │
│  │ L2 Container │ Architekti, DevOps │ Při přidání/odebrání svc  │   │
│  │ L3 Component │ Dev tým            │ Při větším refactoringu  │   │
│  │ L4 Code      │ Jednotliví devs    │ Generovat automaticky     │   │
│  └────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  NÁSTROJE:                                                             │
│  ═════════                                                             │
│                                                                         │
│  • Structurizr (oficiální nástroj, DSL)                               │
│  • PlantUML + C4 extension                                             │
│  • draw.io s C4 shapes                                                 │
│  • Mermaid diagrams                                                    │
│  • Lucidchart                                                          │
│  • IcePanel                                                            │
│                                                                         │
│  ANTI-PATTERNS:                                                        │
│  ══════════════                                                        │
│                                                                         │
│  ❌ Příliš mnoho detailů na vyšší úrovni                              │
│  ❌ Chybějící popis odpovědností                                      │
│  ❌ Šipky bez popisu                                                   │
│  ❌ Míchání úrovní abstrakce                                          │
│  ❌ Neaktuální diagramy                                               │
│  ❌ Chybějící legenda                                                  │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 2. Architecture Decision Records (ADR)

### Co je ADR

Dokument zachycující významné architektonické rozhodnutí, jeho kontext, zvažované alternativy a důsledky. Slouží jako "decision log" pro budoucí referenci.

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    PROČ ADR                                             │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  PROBLÉMY BEZ ADR:                                                     │
│  ═════════════════                                                     │
│                                                                         │
│  "Proč používáme Kafka místo SQS?"                                     │
│       │                                                                 │
│       ▼                                                                 │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ • Nikdo neví                                                    │   │
│  │ • Ten kdo rozhodl už odešel                                     │   │
│  │ • Rozhodnutí je zpochybňováno každý měsíc                      │   │
│  │ • Noví členové týmu nerozumí kontextu                          │   │
│  │ • Stejné diskuze se opakují dokola                              │   │
│  │ • Riskujeme nekonzistentní rozhodnutí                          │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  VÝHODY ADR:                                                           │
│  ═══════════                                                           │
│                                                                         │
│  ✅ Zachycení kontextu v době rozhodnutí                              │
│  ✅ Onboarding nových členů týmu                                      │
│  ✅ Zamezení opakovaným diskuzím                                      │
│  ✅ Podklad pro future review (je kontext stále platný?)              │
│  ✅ Audit trail pro compliance                                         │
│  ✅ Knowledge sharing across teams                                     │
│                                                                         │
│  CO DOKUMENTOVAT:                                                      │
│  ═════════════════                                                     │
│                                                                         │
│  ┌────────────────────────────────────────────────────────────────┐   │
│  │ Dokumentuj                      │ Nedokumentuj                 │   │
│  ├─────────────────────────────────┼──────────────────────────────│   │
│  │ Volba databáze                  │ Název proměnné              │   │
│  │ Messaging pattern               │ Formátování kódu            │   │
│  │ Authentication strategie        │ Triviální implementace      │   │
│  │ API design (REST vs GraphQL)    │ Bug fixy                    │   │
│  │ Deployment strategie            │ Minor refactoring           │   │
│  │ Third-party integrace           │ Library version updates     │   │
│  │ Data retention politika         │                              │   │
│  │ Security architektura           │                              │   │
│  └────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  PRAVIDLO: Pokud by rozhodnutí bylo těžké změnit nebo má              │
│            významný dopad na systém → dokumentuj jako ADR              │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### ADR šablona

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    ADR TEMPLATE (Michael Nygard format)                 │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  # ADR-XXXX: [Název rozhodnutí]                                        │
│                                                                         │
│  ## Status                                                              │
│  [Proposed | Accepted | Deprecated | Superseded by ADR-YYYY]           │
│                                                                         │
│  ## Date                                                                │
│  YYYY-MM-DD                                                             │
│                                                                         │
│  ## Context                                                             │
│  Jaká je situace, která nás vede k tomuto rozhodnutí?                 │
│  Jaké jsou síly v play? (technické, business, organizační)            │
│  Jaký problém řešíme?                                                  │
│                                                                         │
│  ## Decision                                                            │
│  Jaké rozhodnutí jsme učinili?                                        │
│  Formuluj jako "We will..." nebo "We decided to..."                   │
│                                                                         │
│  ## Alternatives Considered                                            │
│  Jaké alternativy jsme zvažovali?                                     │
│  Proč jsme je zamítli?                                                │
│                                                                         │
│  ## Consequences                                                        │
│  ### Positive                                                           │
│  - Co získáme?                                                         │
│                                                                         │
│  ### Negative                                                           │
│  - Co ztrácíme nebo riskujeme?                                        │
│  - Jaký je trade-off?                                                  │
│                                                                         │
│  ### Risks                                                              │
│  - Jaká rizika přijímáme?                                             │
│  - Jak je budeme mitigovat?                                           │
│                                                                         │
│  ## References                                                          │
│  - Odkazy na dokumentaci, RFC, jiné ADR                               │
│                                                                         │
│  ## Participants                                                        │
│  - Kdo se podílel na rozhodnutí?                                      │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### Příklad ADR pro bankovnictví

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    ADR PŘÍKLAD                                          │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  # ADR-0023: Event Sourcing pro Payment Domain                         │
│                                                                         │
│  ## Status                                                              │
│  Accepted                                                               │
│                                                                         │
│  ## Date                                                                │
│  2024-01-15                                                             │
│                                                                         │
│  ## Context                                                             │
│                                                                         │
│  Payment Service zpracovává kritické finanční transakce. Máme          │
│  následující požadavky:                                                │
│                                                                         │
│  1. Regulatorní: Kompletní audit trail všech změn stavu platby        │
│     (AML directive, PSD2 compliance)                                   │
│  2. Business: Schopnost rekonstruovat stav k libovolnému časovému     │
│     bodu pro reklamace a spory                                         │
│  3. Technické: Debugging production issues vyžaduje pochopení          │
│     sekvence událostí                                                   │
│  4. Analytické: Real-time event streaming pro fraud detection          │
│                                                                         │
│  Aktuálně používáme CRUD model s audit tabulkou, která:               │
│  - Je často nekonzistentní s hlavními daty                            │
│  - Neobsahuje dostatečný detail pro rekonstrukci                      │
│  - Je náročná na údržbu                                                │
│                                                                         │
│  ## Decision                                                            │
│                                                                         │
│  Implementujeme Event Sourcing pro Payment domain s následující        │
│  architekturou:                                                         │
│                                                                         │
│  - Event Store: Amazon DynamoDB s DynamoDB Streams                     │
│  - Event publikace: EventBridge pro async consumers                    │
│  - Snapshots: Každých 100 eventů pro performance                      │
│  - Read Models: CQRS projekce do Aurora pro queries                   │
│                                                                         │
│  Events budou immutable a append-only.                                 │
│                                                                         │
│  ## Alternatives Considered                                            │
│                                                                         │
│  ### 1. Enhanced Audit Logging                                         │
│  Rozšíření stávající audit tabulky.                                   │
│                                                                         │
│  Zamítnuto protože:                                                    │
│  - Audit je second-class citizen, snadno se rozchází                  │
│  - Nelze spolehlivě rekonstruovat stav                                │
│  - Nepodporuje event streaming                                         │
│                                                                         │
│  ### 2. Kafka jako Event Store                                         │
│  Použití Apache Kafka (MSK) pro ukládání eventů.                      │
│                                                                         │
│  Zamítnuto protože:                                                    │
│  - Kafka retention není neomezený (cost prohibitive)                  │
│  - Random access k eventům je neefektivní                             │
│  - Operační komplexita Kafka clusteru                                 │
│                                                                         │
│  ### 3. Specialized Event Store (EventStoreDB)                         │
│  Zamítnuto protože:                                                    │
│  - Další technologie k provozování                                    │
│  - Tým nemá zkušenosti                                                │
│  - AWS managed alternativa neexistuje                                 │
│                                                                         │
│  ## Consequences                                                        │
│                                                                         │
│  ### Positive                                                           │
│  - Kompletní audit trail jako vedlejší efekt (ne extra práce)        │
│  - Temporal queries out of the box                                    │
│  - Event replay pro debugging a recovery                              │
│  - Native event streaming pro real-time processing                    │
│  - Snadné přidávání nových read models                                │
│                                                                         │
│  ### Negative                                                           │
│  - Vyšší komplexita implementace                                      │
│  - Learning curve pro tým                                             │
│  - Eventual consistency pro read models                               │
│  - Event schema evolution vyžaduje pečlivé plánování                 │
│                                                                         │
│  ### Risks                                                              │
│                                                                         │
│  | Risk                    | Probability | Impact | Mitigation        │
│  |─────────────────────────|─────────────|────────|───────────────────│
│  | Event replay too slow   | Medium      | High   | Snapshots         │
│  | Schema evolution issues | Medium      | Medium | Versioned events  │
│  | Team skill gap          | High        | Medium | Training, pairing │
│  | DynamoDB cost growth    | Low         | Medium | TTL for old events│
│                                                                         │
│  ## References                                                          │
│  - ADR-0019: CQRS for Read Optimization                               │
│  - Martin Fowler: Event Sourcing                                       │
│  - Greg Young: CQRS and Event Sourcing                                │
│                                                                         │
│  ## Participants                                                        │
│  - Jan Novák (Lead Architect) - Decision owner                        │
│  - Payment Team                                                        │
│  - Platform Team (infra review)                                        │
│  - Security Team (compliance review)                                   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### ADR Lifecycle a správa

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    ADR LIFECYCLE                                        │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  STAVY ADR:                                                            │
│  ══════════                                                            │
│                                                                         │
│  ┌─────────────┐                                                       │
│  │  PROPOSED   │  Návrh, čeká na review a schválení                   │
│  └──────┬──────┘                                                       │
│         │                                                               │
│         │  Review & Approval                                           │
│         ▼                                                               │
│  ┌─────────────┐                                                       │
│  │  ACCEPTED   │  Schváleno, implementace může začít                  │
│  └──────┬──────┘                                                       │
│         │                                                               │
│         │  Context changed / Better solution found                     │
│         │                                                               │
│         ├──────────────────────────────┐                               │
│         │                              │                               │
│         ▼                              ▼                               │
│  ┌─────────────┐               ┌─────────────────────┐                │
│  │ DEPRECATED  │               │ SUPERSEDED by       │                │
│  │             │               │ ADR-YYYY            │                │
│  │ Již neplatí │               │                     │                │
│  │ ale nikdy   │               │ Nahrazeno novým    │                │
│  │ nebylo      │               │ rozhodnutím        │                │
│  │ superseded  │               │                     │                │
│  └─────────────┘               └─────────────────────┘                │
│                                                                         │
│  ORGANIZACE ADR:                                                       │
│  ═══════════════                                                       │
│                                                                         │
│  repo/                                                                  │
│  ├── docs/                                                             │
│  │   └── adr/                                                          │
│  │       ├── 0001-record-architecture-decisions.md                    │
│  │       ├── 0002-use-postgresql-for-accounts.md                      │
│  │       ├── 0003-use-kafka-for-events.md                             │
│  │       ├── 0004-authentication-with-cognito.md                      │
│  │       └── template.md                                               │
│  └── ...                                                               │
│                                                                         │
│  POJMENOVÁNÍ:                                                          │
│  ════════════                                                          │
│  • Sekvenční čísla: 0001, 0002, ...                                   │
│  • Lowercase s pomlčkami                                               │
│  • Krátký popisný název                                               │
│  • Formát: NNNN-short-title.md                                        │
│                                                                         │
│  REVIEW PROCESS:                                                       │
│  ═══════════════                                                       │
│                                                                         │
│  1. Autor vytvoří ADR jako PR                                         │
│  2. Stakeholdeři review (architekti, affected teams)                  │
│  3. Diskuze v PR comments                                              │
│  4. Approval od required reviewers                                     │
│  5. Merge = Accepted                                                   │
│  6. Implementace může začít                                           │
│                                                                         │
│  NÁSTROJE:                                                             │
│  ═════════                                                             │
│                                                                         │
│  • adr-tools (CLI pro správu ADR)                                     │
│  • Log4brains (web UI pro ADR)                                        │
│  • Markdown + Git (nejjednodušší)                                     │
│  • Confluence/Notion (pokud tým preferuje)                            │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 3. Trade-off Analysis

### Co je Trade-off Analysis

Systematický proces hodnocení alternativních řešení a jejich dopadů. Každé architektonické rozhodnutí má trade-offy - nelze mít vše.

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    TRADE-OFF FUNDAMENTY                                 │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  ZÁKLADNÍ PRAVDA:                                                      │
│  ════════════════                                                      │
│                                                                         │
│  "There are no solutions, only trade-offs."                            │
│                                        - Thomas Sowell                 │
│                                                                         │
│  V architektuře neexistuje "správné" řešení - pouze řešení            │
│  optimální pro daný kontext a priority.                                │
│                                                                         │
│  TYPICKÉ TRADE-OFF DIMENZE:                                            │
│  ═══════════════════════════                                           │
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                                                                  │   │
│  │  Performance ◀────────────────────────────▶ Maintainability     │   │
│  │  (rychlost)                                  (čitelnost kódu)    │   │
│  │                                                                  │   │
│  │  Consistency ◀────────────────────────────▶ Availability        │   │
│  │  (správnost dat)                             (dostupnost)        │   │
│  │                                                                  │   │
│  │  Flexibility ◀────────────────────────────▶ Simplicity          │   │
│  │  (přizpůsobivost)                            (jednoduchost)      │   │
│  │                                                                  │   │
│  │  Security ◀───────────────────────────────▶ Usability           │   │
│  │  (bezpečnost)                                (použitelnost)      │   │
│  │                                                                  │   │
│  │  Cost ◀───────────────────────────────────▶ Quality             │   │
│  │  (náklady)                                   (kvalita)           │   │
│  │                                                                  │   │
│  │  Time to Market ◀─────────────────────────▶ Completeness        │   │
│  │  (rychlost dodání)                           (úplnost řešení)   │   │
│  │                                                                  │   │
│  │  Scalability ◀────────────────────────────▶ Simplicity          │   │
│  │  (škálovatelnost)                            (jednoduchost)      │   │
│  │                                                                  │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  PŘÍKLAD - CAP THEOREM:                                                │
│  ═══════════════════════                                               │
│                                                                         │
│  V distribuovaném systému můžeš mít pouze 2 ze 3:                      │
│                                                                         │
│          Consistency                                                    │
│              ▲                                                          │
│             ╱ ╲                                                         │
│            ╱   ╲                                                        │
│           ╱     ╲                                                       │
│          ╱       ╲                                                      │
│         ╱    ●    ╲        ← Můžeš být TADY (CP)                       │
│        ╱           ╲       ← nebo TADY (CA)                            │
│       ╱             ╲      ← nebo TADY (AP)                            │
│      ▼───────────────▼     ← ale NE všude                              │
│  Availability    Partition                                              │
│                  Tolerance                                              │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### Trade-off Analysis Framework

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    ANALYSIS FRAMEWORK                                   │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  KROK 1: DEFINUJ KONTEXT                                               │
│  ════════════════════════                                              │
│                                                                         │
│  • Jaký problém řešíme?                                                │
│  • Jaké jsou constraints (budget, time, skills)?                       │
│  • Kdo jsou stakeholders a jaké mají priority?                        │
│  • Jaké jsou non-negotiables?                                          │
│                                                                         │
│  KROK 2: IDENTIFIKUJ ALTERNATIVY                                       │
│  ═══════════════════════════════                                       │
│                                                                         │
│  • Jaké jsou možné přístupy?                                          │
│  • Co dělá industrie? (reference architectures)                        │
│  • Co jsme dělali dříve? (lessons learned)                            │
│  • Jaké jsou emerging approaches?                                      │
│                                                                         │
│  KROK 3: DEFINUJ KRITÉRIA                                              │
│  ═════════════════════════                                             │
│                                                                         │
│  ┌────────────────────────────────────────────────────────────────┐   │
│  │ Kritérium              │ Váha │ Důvod                          │   │
│  ├────────────────────────┼──────┼────────────────────────────────│   │
│  │ Performance            │ 25%  │ High-volume payment processing │   │
│  │ Availability           │ 25%  │ 24/7 banking requirement       │   │
│  │ Security               │ 20%  │ Financial data, compliance     │   │
│  │ Operational complexity │ 15%  │ Small ops team                 │   │
│  │ Cost                   │ 10%  │ Budget constrained            │   │
│  │ Time to implement      │  5%  │ Q2 deadline                   │   │
│  └────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  KROK 4: SKÓROVÁNÍ ALTERNATIV                                          │
│  ═════════════════════════════                                         │
│                                                                         │
│  Škála: 1 (špatné) - 5 (výborné)                                      │
│                                                                         │
│  ┌────────────────────────────────────────────────────────────────┐   │
│  │                   │ Option A  │ Option B  │ Option C            │   │
│  │ Kritérium (váha)  │ Lambda    │ ECS       │ EKS                 │   │
│  ├───────────────────┼───────────┼───────────┼─────────────────────│   │
│  │ Performance (25%) │ 4         │ 5         │ 5                   │   │
│  │ Availability (25%)│ 5         │ 4         │ 4                   │   │
│  │ Security (20%)    │ 5         │ 4         │ 4                   │   │
│  │ Ops complex. (15%)│ 5         │ 3         │ 2                   │   │
│  │ Cost (10%)        │ 4         │ 3         │ 2                   │   │
│  │ Time (5%)         │ 5         │ 4         │ 2                   │   │
│  ├───────────────────┼───────────┼───────────┼─────────────────────│   │
│  │ WEIGHTED SCORE    │ 4.55      │ 3.95      │ 3.45                │   │
│  └────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  KROK 5: ANALÝZA RIZIK                                                 │
│  ══════════════════════                                                │
│                                                                         │
│  Pro každou alternativu:                                               │
│  • Jaká rizika přináší?                                               │
│  • Jaká je pravděpodobnost a dopad?                                   │
│  • Jak můžeme rizika mitigovat?                                       │
│  • Jaké jsou unknown unknowns?                                        │
│                                                                         │
│  KROK 6: SENZITIVNÍ ANALÝZA                                            │
│  ════════════════════════════                                          │
│                                                                         │
│  • Co když se změní předpoklady?                                      │
│  • Co když se zdvojnásobí volume?                                     │
│  • Co když přijde nový regulatorní požadavek?                         │
│  • Jak robustní je naše rozhodnutí?                                  │
│                                                                         │
│  KROK 7: DOKUMENTUJ ROZHODNUTÍ                                         │
│  ═════════════════════════════                                         │
│                                                                         │
│  → ADR s kompletním kontextem a zdůvodněním                           │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### Příklad: Database Trade-off Analysis

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    PŘÍKLAD: VOLBA DATABÁZE PRO TRANSAKCE               │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  KONTEXT:                                                              │
│  ═════════                                                             │
│  Payment Service potřebuje databázi pro ukládání transakcí.           │
│  • Očekávaný volume: 10,000 TPS peak                                  │
│  • Retention: 7 let (regulace)                                        │
│  • Query patterns: vysoký write, read history, real-time balance      │
│  • Consistency: Strong pro balance, eventual OK pro history           │
│                                                                         │
│  ALTERNATIVY:                                                          │
│  ═════════════                                                         │
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │             │ Aurora     │ DynamoDB    │ CockroachDB           │   │
│  │             │ PostgreSQL │             │                        │   │
│  ├─────────────┼────────────┼─────────────┼────────────────────────│   │
│  │ Type        │ Relational │ NoSQL       │ Distributed SQL       │   │
│  │             │ Managed    │ Serverless  │ Managed               │   │
│  ├─────────────┼────────────┼─────────────┼────────────────────────│   │
│  │ Write perf  │ Good       │ Excellent   │ Good                  │   │
│  │             │ (vertical) │ (horizontal)│ (horizontal)          │   │
│  ├─────────────┼────────────┼─────────────┼────────────────────────│   │
│  │ Read perf   │ Excellent  │ Good        │ Excellent             │   │
│  │             │ (SQL)      │ (key-based) │ (SQL)                 │   │
│  ├─────────────┼────────────┼─────────────┼────────────────────────│   │
│  │ Consistency │ Strong     │ Eventual*   │ Strong                │   │
│  │             │            │ (or strong) │ (serializable)        │   │
│  ├─────────────┼────────────┼─────────────┼────────────────────────│   │
│  │ Schema      │ Fixed      │ Flexible    │ Fixed                 │   │
│  │             │ (migration)│ (schemaless)│ (migration)           │   │
│  ├─────────────┼────────────┼─────────────┼────────────────────────│   │
│  │ SQL support │ Full       │ Limited     │ Full                  │   │
│  │             │            │ (PartiQL)   │ (PostgreSQL wire)     │   │
│  ├─────────────┼────────────┼─────────────┼────────────────────────│   │
│  │ Cost model  │ Instance   │ Pay per req │ Instance              │   │
│  │             │ based      │ + storage   │ based                 │   │
│  ├─────────────┼────────────┼─────────────┼────────────────────────│   │
│  │ Ops burden  │ Low        │ Very Low    │ Medium                │   │
│  │             │ (managed)  │ (serverless)│ (managed but complex) │   │
│  ├─────────────┼────────────┼─────────────┼────────────────────────│   │
│  │ Team skills │ High       │ Medium      │ Low                   │   │
│  └─────────────┴────────────┴─────────────┴────────────────────────┘   │
│                                                                         │
│  TRADE-OFF MATRIX:                                                     │
│  ═════════════════                                                     │
│                                                                         │
│                       Aurora    DynamoDB   CockroachDB                 │
│                       ┌────┐    ┌────┐     ┌────┐                      │
│  Write Scalability    │ ██ │    │████│     │███ │                      │
│                       └────┘    └────┘     └────┘                      │
│                       ┌────┐    ┌────┐     ┌────┐                      │
│  Query Flexibility    │████│    │ █  │     │████│                      │
│                       └────┘    └────┘     └────┘                      │
│                       ┌────┐    ┌────┐     ┌────┐                      │
│  Strong Consistency   │████│    │ ██ │     │████│                      │
│                       └────┘    └────┘     └────┘                      │
│                       ┌────┐    ┌────┐     ┌────┐                      │
│  Operational Ease     │███ │    │████│     │ ██ │                      │
│                       └────┘    └────┘     └────┘                      │
│                       ┌────┐    ┌────┐     ┌────┐                      │
│  Cost Efficiency      │ ██ │    │███ │     │ ██ │                      │
│  (at 10K TPS)         └────┘    └────┘     └────┘                      │
│                       ┌────┐    ┌────┐     ┌────┐                      │
│  Team Expertise       │████│    │███ │     │ █  │                      │
│                       └────┘    └────┘     └────┘                      │
│                                                                         │
│  ROZHODNUTÍ:                                                           │
│  ════════════                                                          │
│                                                                         │
│  Hybrid approach:                                                       │
│  • Aurora PostgreSQL pro balance a kritické transakce (strong cons.)  │
│  • DynamoDB pro transaction history (vysoký write, eventual OK)       │
│  • Event-driven sync mezi nimi                                         │
│                                                                         │
│  DŮVOD:                                                                │
│  • Využíváme silné stránky obou                                       │
│  • Aurora pro SQL flexibility a strong consistency (balance)          │
│  • DynamoDB pro write scalability (history)                           │
│  • Team má expertise s oběma                                          │
│  • Komplexita je akceptovatelná vzhledem k benefitům                 │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### Vizualizace Trade-offs

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    VIZUALIZAČNÍ TECHNIKY                                │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  1. SPIDER/RADAR DIAGRAM                                               │
│  ═══════════════════════                                               │
│                                                                         │
│                     Performance                                         │
│                         ▲                                               │
│                        ╱│╲                                              │
│                       ╱ │ ╲                                             │
│                      ╱  │  ╲                                            │
│               ●─────●───┼───●─────●                                     │
│              ╱      ╲   │   ╱      ╲                                    │
│  Maintainability ────●──┼──●──── Security                              │
│              ╲      ╱   │   ╲      ╱                                    │
│               ●─────●───┼───●─────●                                     │
│                      ╲  │  ╱                                            │
│                       ╲ │ ╱                                             │
│                        ╲│╱                                              │
│                         ▼                                               │
│                    Scalability                                          │
│                                                                         │
│  ─── Option A (Lambda)                                                 │
│  ─ ─ Option B (ECS)                                                    │
│                                                                         │
│  2. DECISION MATRIX (Pugh Matrix)                                      │
│  ═══════════════════════════════                                       │
│                                                                         │
│  Baseline: Current solution                                            │
│  Scoring: + (better), - (worse), S (same)                              │
│                                                                         │
│  ┌──────────────────┬────────┬────────┬────────┐                      │
│  │ Criteria         │ Base   │ Opt A  │ Opt B  │                      │
│  ├──────────────────┼────────┼────────┼────────┤                      │
│  │ Performance      │   S    │   +    │   +    │                      │
│  │ Cost             │   S    │   +    │   -    │                      │
│  │ Complexity       │   S    │   S    │   -    │                      │
│  │ Time to market   │   S    │   +    │   -    │                      │
│  │ Scalability      │   S    │   ++   │   +    │                      │
│  ├──────────────────┼────────┼────────┼────────┤                      │
│  │ Net Score        │   0    │  +4    │  -1    │                      │
│  └──────────────────┴────────┴────────┴────────┘                      │
│                                                                         │
│  3. QUADRANT ANALYSIS                                                  │
│  ═════════════════════                                                 │
│                                                                         │
│  High Impact                                                           │
│       ▲                                                                 │
│       │    Quick Wins    │   Strategic                                 │
│       │      ●           │      ●                                      │
│       │   (Option A)     │   (Option C)                                │
│       │                  │                                              │
│       ├──────────────────┼──────────────────▶ High Effort             │
│       │                  │                                              │
│       │   Fill-ins       │   Time Sinks                                │
│       │      ●           │      ●                                      │
│       │   (Option D)     │   (Option B)                                │
│       │                  │                                              │
│  Low Impact                                                             │
│                                                                         │
│  4. COST-BENEFIT OVER TIME                                             │
│  ══════════════════════════                                            │
│                                                                         │
│  Value                                                                  │
│    ▲                                                                    │
│    │           Option B ──────────────●                                │
│    │                  ╱               ╱                                 │
│    │                 ╱    ●──────────                                   │
│    │                ╱    ╱  Option A                                   │
│    │               ╱    ╱                                               │
│    │        ●─────●────╱                                                │
│    │       ╱          ╱                                                 │
│    │      ╱          ╱                                                  │
│    │─────●──────────╱──────────────────────▶ Time                      │
│    │     │          │                                                   │
│    │  Initial    Break-even                                            │
│    │  Investment    Point                                               │
│                                                                         │
│  Option A: Nižší počáteční investice, dřívější break-even             │
│  Option B: Vyšší investice, ale lepší long-term value                  │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 4. NFR (Non-Functional Requirements) definice a měření

### Co jsou NFR

Non-Functional Requirements definují JAK se systém chová, ne CO dělá. Jsou kritické pro kvalitu, ale často přehlížené.

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    NFR KATEGORIE                                        │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                                                                  │   │
│  │  RUNTIME QUALITIES (měřitelné za běhu)                          │   │
│  │  ═════════════════════════════════════                          │   │
│  │                                                                  │   │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐             │   │
│  │  │ Performance │  │Availability │  │ Scalability │             │   │
│  │  │             │  │             │  │             │             │   │
│  │  │ Latency     │  │ Uptime      │  │ Throughput  │             │   │
│  │  │ Throughput  │  │ MTBF/MTTR   │  │ Elasticity  │             │   │
│  │  │ Resource    │  │ Failover    │  │ Load        │             │   │
│  │  └─────────────┘  └─────────────┘  └─────────────┘             │   │
│  │                                                                  │   │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐             │   │
│  │  │  Security   │  │ Reliability │  │ Usability   │             │   │
│  │  │             │  │             │  │             │             │   │
│  │  │ AuthN/AuthZ │  │ Fault tol.  │  │ Response    │             │   │
│  │  │ Encryption  │  │ Data integ. │  │ Accessibility│            │   │
│  │  │ Audit       │  │ Recovery    │  │ Learnability│             │   │
│  │  └─────────────┘  └─────────────┘  └─────────────┘             │   │
│  │                                                                  │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                                                                  │   │
│  │  NON-RUNTIME QUALITIES (měřitelné při vývoji/provozu)          │   │
│  │  ═══════════════════════════════════════════════════            │   │
│  │                                                                  │   │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐             │   │
│  │  │Maintainabil.│  │ Testability │  │ Deployabil. │             │   │
│  │  │             │  │             │  │             │             │   │
│  │  │ Modularity  │  │ Coverage    │  │ Automation  │             │   │
│  │  │ Readability │  │ Isolation   │  │ Rollback    │             │   │
│  │  │ Upgradabil. │  │ Mock-able   │  │ Downtime    │             │   │
│  │  └─────────────┘  └─────────────┘  └─────────────┘             │   │
│  │                                                                  │   │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐             │   │
│  │  │ Portability │  │ Compliance  │  │   Cost      │             │   │
│  │  │             │  │             │  │             │             │   │
│  │  │ Platform    │  │ Regulatory  │  │ TCO         │             │   │
│  │  │ Data format │  │ Standards   │  │ OpEx/CapEx  │             │   │
│  │  │ Migration   │  │ Audit       │  │ Licensing   │             │   │
│  │  └─────────────┘  └─────────────┘  └─────────────┘             │   │
│  │                                                                  │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### SMART NFR definice

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    SMART NFR FRAMEWORK                                  │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  NFR musí být SMART:                                                   │
│                                                                         │
│  S - Specific (konkrétní)                                              │
│  M - Measurable (měřitelné)                                            │
│  A - Achievable (dosažitelné)                                          │
│  R - Relevant (relevantní pro business)                                │
│  T - Time-bound (s časovým rámcem)                                     │
│                                                                         │
│  ❌ ŠPATNĚ DEFINOVANÉ NFR:                                             │
│  ═════════════════════════                                             │
│                                                                         │
│  • "Systém musí být rychlý"                                            │
│  • "Aplikace musí být bezpečná"                                        │
│  • "Musíme mít vysokou dostupnost"                                     │
│  • "Systém musí škálovat"                                              │
│                                                                         │
│  ✅ SPRÁVNĚ DEFINOVANÉ NFR:                                            │
│  ══════════════════════════                                            │
│                                                                         │
│  PERFORMANCE:                                                          │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ "API endpoint GET /accounts/{id}/balance musí odpovědět         │   │
│  │  do 200ms pro P95 při zátěži 1000 concurrent users.            │   │
│  │  Měřeno: New Relic APM                                          │   │
│  │  Platí pro: Production environment                              │   │
│  │  Ověření: Load test před každým release"                        │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  AVAILABILITY:                                                         │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ "Payment Service musí mít dostupnost 99.95% měřenou             │   │
│  │  jako successful transactions / total transactions.             │   │
│  │  Měřeno: CloudWatch metrics, měsíční okno                       │   │
│  │  Excludes: Scheduled maintenance (max 4h/měsíc)                 │   │
│  │  SLA penalty: 10% credit za každých 0.1% pod target"           │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  SECURITY:                                                             │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ "Všechny API endpointy musí vyžadovat OAuth 2.0 autentizaci.   │   │
│  │  Session timeout: 15 minut inaktivity.                          │   │
│  │  Token refresh: max 8 hodin.                                    │   │
│  │  Failed login lockout: 5 attempts → 30 min lock.               │   │
│  │  Audit log retention: 7 let.                                    │   │
│  │  Ověření: Penetration test quarterly"                          │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  SCALABILITY:                                                          │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ "Systém musí zvládnout 10x nárůst traffic během 5 minut        │   │
│  │  bez degradace performance (latency < 500ms P99).               │   │
│  │  Baseline: 1000 TPS                                             │   │
│  │  Peak: 10,000 TPS                                               │   │
│  │  Scale-up time: < 5 minut                                       │   │
│  │  Scale-down time: < 15 minut                                    │   │
│  │  Ověření: Chaos engineering test monthly"                       │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### NFR pro bankovnictví - kompletní specifikace

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    NFR KATALOG - DIGITAL BANKING                        │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  1. PERFORMANCE                                                         │
│  ══════════════                                                        │
│                                                                         │
│  ┌────────────────────────────────────────────────────────────────┐   │
│  │ Metrika           │ Target      │ Měření          │ Criticality│   │
│  ├────────────────────┼─────────────┼─────────────────┼────────────│   │
│  │ API Latency P50    │ < 100ms     │ X-Ray, APM      │ High       │   │
│  │ API Latency P95    │ < 300ms     │ X-Ray, APM      │ High       │   │
│  │ API Latency P99    │ < 500ms     │ X-Ray, APM      │ High       │   │
│  │ Page Load Time     │ < 2s        │ RUM, Lighthouse │ Medium     │   │
│  │ Time to Interactive│ < 3s        │ RUM             │ Medium     │   │
│  │ Database query     │ < 50ms      │ Query logs      │ High       │   │
│  │ Throughput         │ 10,000 TPS  │ CloudWatch      │ High       │   │
│  │ Batch processing   │ 1M rec/hour │ Job metrics     │ Medium     │   │
│  └────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  Kontexty:                                                              │
│  • Salary day: 3x normal load                                          │
│  • Month end: 5x batch processing                                      │
│  • Black Friday: 10x peak (pro retail klienty)                        │
│                                                                         │
│  2. AVAILABILITY & RELIABILITY                                         │
│  ══════════════════════════════                                        │
│                                                                         │
│  ┌────────────────────────────────────────────────────────────────┐   │
│  │ Služba              │ Availability │ RPO       │ RTO           │   │
│  ├─────────────────────┼──────────────┼───────────┼───────────────│   │
│  │ Payment Processing  │ 99.99%       │ 0         │ < 5 min       │   │
│  │ Account Inquiry     │ 99.95%       │ < 1 min   │ < 15 min      │   │
│  │ Card Services       │ 99.95%       │ < 1 min   │ < 15 min      │   │
│  │ Statements          │ 99.9%        │ < 1 hour  │ < 4 hours     │   │
│  │ Admin Portal        │ 99.5%        │ < 1 hour  │ < 4 hours     │   │
│  │ Analytics           │ 99.0%        │ < 24 hours│ < 24 hours    │   │
│  └────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  Definice:                                                              │
│  • RPO (Recovery Point Objective): Max data loss                       │
│  • RTO (Recovery Time Objective): Max downtime                         │
│  • MTBF (Mean Time Between Failures): > 720 hours                     │
│  • MTTR (Mean Time To Recovery): < 30 minutes                         │
│                                                                         │
│  3. SECURITY                                                           │
│  ═══════════                                                           │
│                                                                         │
│  ┌────────────────────────────────────────────────────────────────┐   │
│  │ Requirement                              │ Standard/Target      │   │
│  ├──────────────────────────────────────────┼──────────────────────│   │
│  │ Encryption at rest                       │ AES-256             │   │
│  │ Encryption in transit                    │ TLS 1.2+            │   │
│  │ Authentication                           │ OAuth 2.0 + MFA     │   │
│  │ Session timeout (idle)                   │ 15 minutes          │   │
│  │ Session timeout (absolute)               │ 8 hours             │   │
│  │ Password policy                          │ NIST 800-63B        │   │
│  │ Failed login lockout                     │ 5 attempts/30 min   │   │
│  │ API rate limiting                        │ 100 req/min/user    │   │
│  │ Audit log retention                      │ 7 years             │   │
│  │ Vulnerability scan                       │ Weekly              │   │
│  │ Penetration test                         │ Quarterly           │   │
│  │ Security patching                        │ Critical: 24h       │   │
│  │                                          │ High: 7 days        │   │
│  │                                          │ Medium: 30 days     │   │
│  └────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  4. SCALABILITY                                                        │
│  ═══════════════                                                       │
│                                                                         │
│  ┌────────────────────────────────────────────────────────────────┐   │
│  │ Dimension           │ Current    │ Target     │ Growth Rate   │   │
│  ├─────────────────────┼────────────┼────────────┼───────────────│   │
│  │ Registered users    │ 500K       │ 2M         │ 30% YoY       │   │
│  │ Concurrent users    │ 10K        │ 50K        │ 20% YoY       │   │
│  │ Transactions/day    │ 1M         │ 5M         │ 40% YoY       │   │
│  │ Data volume         │ 10TB       │ 50TB       │ 50% YoY       │   │
│  │ API calls/day       │ 50M        │ 200M       │ 30% YoY       │   │
│  └────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  Elasticity requirements:                                              │
│  • Scale out: < 5 minutes                                              │
│  • Scale in: < 15 minutes                                              │
│  • Auto-scaling trigger: CPU > 70% for 2 minutes                      │
│                                                                         │
│  5. COMPLIANCE                                                         │
│  ═════════════                                                         │
│                                                                         │
│  ┌────────────────────────────────────────────────────────────────┐   │
│  │ Regulation      │ Requirement                     │ Audit      │   │
│  ├─────────────────┼─────────────────────────────────┼────────────│   │
│  │ PSD2            │ SCA, Open Banking APIs          │ Annual     │   │
│  │ GDPR            │ Data protection, right to erase │ Annual     │   │
│  │ PCI-DSS         │ Card data handling              │ Annual     │   │
│  │ AML/KYC         │ Transaction monitoring          │ Continuous │   │
│  │ DORA            │ ICT risk management             │ Annual     │   │
│  │ NIS2            │ Cybersecurity measures          │ Annual     │   │
│  └────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  6. OPERATIONAL                                                        │
│  ═══════════════                                                       │
│                                                                         │
│  ┌────────────────────────────────────────────────────────────────┐   │
│  │ Aspect              │ Target                                   │   │
│  ├─────────────────────┼──────────────────────────────────────────│   │
│  │ Deployment frequency│ Daily (with zero-downtime)              │   │
│  │ Lead time for change│ < 1 day                                 │   │
│  │ Change failure rate │ < 5%                                    │   │
│  │ Time to restore     │ < 1 hour                                │   │
│  │ Monitoring coverage │ 100% services                           │   │
│  │ Alert response time │ P1: 15 min, P2: 1 hour, P3: 4 hours    │   │
│  │ Runbook coverage    │ 100% critical paths                     │   │
│  └────────────────────────────────────────────────────────────────┘   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### NFR měření a monitoring

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    NFR MEASUREMENT FRAMEWORK                            │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  MONITORING STACK PRO NFR:                                             │
│  ═══════════════════════════                                           │
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                        OBSERVABILITY                             │   │
│  │                                                                  │   │
│  │   ┌─────────────┐    ┌─────────────┐    ┌─────────────┐        │   │
│  │   │   METRICS   │    │    LOGS     │    │   TRACES    │        │   │
│  │   │             │    │             │    │             │        │   │
│  │   │ CloudWatch  │    │ CloudWatch  │    │   X-Ray     │        │   │
│  │   │ Prometheus  │    │ Logs        │    │   Jaeger    │        │   │
│  │   │ Grafana     │    │ OpenSearch  │    │             │        │   │
│  │   └──────┬──────┘    └──────┬──────┘    └──────┬──────┘        │   │
│  │          │                  │                  │                │   │
│  │          └──────────────────┼──────────────────┘                │   │
│  │                             │                                    │   │
│  │                             ▼                                    │   │
│  │                    ┌─────────────────┐                          │   │
│  │                    │   DASHBOARDS    │                          │   │
│  │                    │                 │                          │   │
│  │                    │ Grafana         │                          │   │
│  │                    │ CloudWatch      │                          │   │
│  │                    │ Dashboards      │                          │   │
│  │                    └────────┬────────┘                          │   │
│  │                             │                                    │   │
│  │                             ▼                                    │   │
│  │                    ┌─────────────────┐                          │   │
│  │                    │    ALERTING     │                          │   │
│  │                    │                 │                          │   │
│  │                    │ CloudWatch      │                          │   │
│  │                    │ Alarms → SNS    │                          │   │
│  │                    │ → PagerDuty     │                          │   │
│  │                    └─────────────────┘                          │   │
│  │                                                                  │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  METRIKY PRO JEDNOTLIVÉ NFR:                                           │
│  ═════════════════════════════                                         │
│                                                                         │
│  PERFORMANCE:                                                          │
│  ┌────────────────────────────────────────────────────────────────┐   │
│  │ Metrika              │ Zdroj           │ Agregace              │   │
│  ├──────────────────────┼─────────────────┼───────────────────────│   │
│  │ request_latency_ms   │ API Gateway     │ P50, P95, P99         │   │
│  │ request_count        │ API Gateway     │ Sum per minute        │   │
│  │ error_rate           │ API Gateway     │ 4xx+5xx / total       │   │
│  │ db_query_time_ms     │ RDS Performance │ Average, Max          │   │
│  │ lambda_duration_ms   │ Lambda metrics  │ P50, P99              │   │
│  │ lambda_cold_start    │ Lambda metrics  │ Count, Percentage     │   │
│  └────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  AVAILABILITY:                                                         │
│  ┌────────────────────────────────────────────────────────────────┐   │
│  │ Metrika              │ Výpočet                                 │   │
│  ├──────────────────────┼─────────────────────────────────────────│   │
│  │ Availability %       │ (total_time - downtime) / total_time   │   │
│  │ Error budget         │ (1 - SLO) × time_period                │   │
│  │ Error budget burn    │ actual_errors / error_budget           │   │
│  │ MTBF                 │ total_uptime / number_of_failures      │   │
│  │ MTTR                 │ total_downtime / number_of_failures    │   │
│  └────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  SLI / SLO / SLA HIERARCHY:                                            │
│  ═══════════════════════════                                           │
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                                                                  │   │
│  │  SLI (Service Level Indicator)                                  │   │
│  │  ─────────────────────────────                                  │   │
│  │  "Co měříme"                                                    │   │
│  │  Příklad: Request latency P99                                   │   │
│  │                                                                  │   │
│  │           ▼                                                      │   │
│  │                                                                  │   │
│  │  SLO (Service Level Objective)                                  │   │
│  │  ──────────────────────────────                                 │   │
│  │  "Jaký je náš interní cíl"                                     │   │
│  │  Příklad: P99 latency < 300ms for 99.9% of requests            │   │
│  │                                                                  │   │
│  │           ▼                                                      │   │
│  │                                                                  │   │
│  │  SLA (Service Level Agreement)                                  │   │
│  │  ──────────────────────────────                                 │   │
│  │  "Co slibujeme zákazníkovi (s penalizací)"                     │   │
│  │  Příklad: P99 latency < 500ms for 99.5% of requests            │   │
│  │           Breach = 10% credit                                   │   │
│  │                                                                  │   │
│  │  Poznámka: SLO je přísnější než SLA (buffer)                   │   │
│  │                                                                  │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  ERROR BUDGET:                                                         │
│  ═════════════                                                         │
│                                                                         │
│  Příklad pro 99.9% availability SLO:                                   │
│                                                                         │
│  ┌────────────────────────────────────────────────────────────────┐   │
│  │ Období       │ Total time │ Allowed downtime │ Error budget   │   │
│  ├──────────────┼────────────┼──────────────────┼────────────────│   │
│  │ Day          │ 1,440 min  │ 1.44 min         │ 86.4 sec       │   │
│  │ Week         │ 10,080 min │ 10.08 min        │ 604.8 sec      │   │
│  │ Month (30d)  │ 43,200 min │ 43.2 min         │ 2,592 sec      │   │
│  │ Quarter      │ 129,600 min│ 129.6 min        │ 7,776 sec      │   │
│  │ Year         │ 525,600 min│ 525.6 min        │ 31,536 sec     │   │
│  └────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  Error budget consumption:                                             │
│  • < 50% consumed → Safe, can take risks                              │
│  • 50-80% consumed → Caution, prioritize reliability                  │
│  • > 80% consumed → Freeze features, focus on stability               │
│  • 100% consumed → Only reliability work until reset                  │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### NFR Testing

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    NFR TESTING STRATEGIE                                │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  ┌────────────────────────────────────────────────────────────────┐   │
│  │ NFR Kategorie    │ Test Type              │ Nástroje           │   │
│  ├──────────────────┼────────────────────────┼────────────────────│   │
│  │ Performance      │ Load testing           │ k6, Gatling, JMeter│   │
│  │                  │ Stress testing         │ Locust             │   │
│  │                  │ Spike testing          │                    │   │
│  │                  │ Endurance testing      │                    │   │
│  ├──────────────────┼────────────────────────┼────────────────────│   │
│  │ Availability     │ Chaos engineering      │ AWS FIS, Gremlin   │   │
│  │                  │ Failover testing       │ Chaos Monkey       │   │
│  │                  │ DR drills              │                    │   │
│  ├──────────────────┼────────────────────────┼────────────────────│   │
│  │ Security         │ Penetration testing    │ OWASP ZAP, Burp    │   │
│  │                  │ Vulnerability scanning │ Qualys, Nessus     │   │
│  │                  │ SAST/DAST              │ SonarQube, Snyk    │   │
│  ├──────────────────┼────────────────────────┼────────────────────│   │
│  │ Scalability      │ Capacity testing       │ k6, custom scripts │   │
│  │                  │ Auto-scaling tests     │                    │   │
│  ├──────────────────┼────────────────────────┼────────────────────│   │
│  │ Reliability      │ Fault injection        │ AWS FIS            │   │
│  │                  │ Recovery testing       │ Litmus             │   │
│  └────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  LOAD TESTING PROFILY:                                                 │
│  ══════════════════════                                                │
│                                                                         │
│  1. BASELINE TEST                                                      │
│     Cíl: Zjistit normální performance                                  │
│                                                                         │
│     Load                                                                │
│       ▲                                                                 │
│       │     ┌──────────────────┐                                       │
│       │     │                  │                                       │
│       │─────┘                  └─────                                  │
│       └─────────────────────────────▶ Time                             │
│             │←─  30 min  ─→│                                           │
│                                                                         │
│  2. STRESS TEST                                                        │
│     Cíl: Najít breaking point                                          │
│                                                                         │
│     Load                                                                │
│       ▲                          ╱                                     │
│       │                        ╱                                       │
│       │                      ╱                                         │
│       │                    ╱                                           │
│       │                  ╱                                             │
│       │                ╱                                               │
│       │              ╱                                                 │
│       └────────────╱─────────────────▶ Time                            │
│                                                                         │
│  3. SPIKE TEST                                                         │
│     Cíl: Testovat náhlý nárůst (salary day, marketing campaign)       │
│                                                                         │
│     Load                                                                │
│       ▲         ┌┐                                                     │
│       │         ││                                                     │
│       │         ││                                                     │
│       │─────────┘└──────────                                           │
│       └─────────────────────────────▶ Time                             │
│                                                                         │
│  4. SOAK TEST (Endurance)                                              │
│     Cíl: Odhalit memory leaks, resource exhaustion                    │
│                                                                         │
│     Load                                                                │
│       ▲                                                                 │
│       │     ┌──────────────────────────────────────┐                   │
│       │     │                                      │                   │
│       │─────┘                                      └───                │
│       └─────────────────────────────────────────────▶ Time             │
│             │←──────────  24 hours  ──────────→│                       │
│                                                                         │
│  CHAOS ENGINEERING EXPERIMENTY:                                        │
│  ═══════════════════════════════                                       │
│                                                                         │
│  ┌────────────────────────────────────────────────────────────────┐   │
│  │ Experiment                    │ Co testuje                     │   │
│  ├───────────────────────────────┼────────────────────────────────│   │
│  │ Kill random instance          │ Auto-recovery, load balancing  │   │
│  │ Network latency injection     │ Timeout handling, circuit break│   │
│  │ CPU stress                    │ Auto-scaling, degradation      │   │
│  │ Memory stress                 │ OOM handling, graceful degrad. │   │
│  │ Database failover             │ Connection retry, read replica │   │
│  │ AZ failure simulation         │ Multi-AZ resilience           │   │
│  │ DNS failure                   │ Caching, fallback              │   │
│  │ Third-party API failure       │ Circuit breaker, fallback      │   │
│  └────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  SECURITY TESTING CADENCE:                                             │
│  ═══════════════════════════                                           │
│                                                                         │
│  ┌────────────────────────────────────────────────────────────────┐   │
│  │ Test Type              │ Frequency    │ Scope                  │   │
│  ├────────────────────────┼──────────────┼────────────────────────│   │
│  │ SAST (Static)          │ Every commit │ Changed code           │   │
│  │ Dependency scan        │ Daily        │ All dependencies       │   │
│  │ Container scan         │ Every build  │ Docker images          │   │
│  │ DAST (Dynamic)         │ Weekly       │ Staging environment    │   │
│  │ Infrastructure scan    │ Weekly       │ AWS resources          │   │
│  │ Penetration test       │ Quarterly    │ Full application       │   │
│  │ Red team exercise      │ Annually     │ Organization-wide      │   │
│  └────────────────────────────────────────────────────────────────┘   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### NFR v CI/CD Pipeline

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    NFR GATES V PIPELINE                                 │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                        CI/CD PIPELINE                            │   │
│  │                                                                  │   │
│  │  ┌─────────┐   ┌─────────┐   ┌─────────┐   ┌─────────┐        │   │
│  │  │  Build  │──▶│  Unit   │──▶│  SAST   │──▶│ Integr. │        │   │
│  │  │         │   │  Tests  │   │  Scan   │   │  Tests  │        │   │
│  │  └─────────┘   └─────────┘   └────┬────┘   └────┬────┘        │   │
│  │                                   │             │              │   │
│  │                              ┌────▼────┐   ┌────▼────┐        │   │
│  │                              │ Quality │   │ Perf.   │        │   │
│  │                              │  Gate   │   │ Gate    │        │   │
│  │                              │         │   │         │        │   │
│  │                              │ • Vuln  │   │ • P99   │        │   │
│  │                              │   count │   │   < 300ms│       │   │
│  │                              │ • Crit  │   │ • Error │        │   │
│  │                              │   = 0   │   │   < 1%  │        │   │
│  │                              └────┬────┘   └────┬────┘        │   │
│  │                                   │             │              │   │
│  │                                   ▼             ▼              │   │
│  │  ┌─────────┐   ┌─────────┐   ┌─────────┐   ┌─────────┐        │   │
│  │  │ Deploy  │──▶│  Smoke  │──▶│ Canary  │──▶│  Full   │        │   │
│  │  │ Staging │   │  Tests  │   │ Deploy  │   │ Deploy  │        │   │
│  │  └─────────┘   └─────────┘   └────┬────┘   └─────────┘        │   │
│  │                                   │                            │   │
│  │                              ┌────▼────┐                       │   │
│  │                              │ Canary  │                       │   │
│  │                              │ Metrics │                       │   │
│  │                              │         │                       │   │
│  │                              │ • Error │                       │   │
│  │                              │   rate  │                       │   │
│  │                              │ • Latency│                      │   │
│  │                              │ • Anom. │                       │   │
│  │                              └────┬────┘                       │   │
│  │                                   │                            │   │
│  │                          Pass?    │    Fail?                   │   │
│  │                    ┌──────────────┼──────────────┐             │   │
│  │                    ▼              │              ▼             │   │
│  │              [Continue]           │         [Rollback]         │   │
│  │                                   │                            │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  QUALITY GATES DEFINICE:                                               │
│  ════════════════════════                                              │
│                                                                         │
│  ┌────────────────────────────────────────────────────────────────┐   │
│  │ Gate           │ Metric                 │ Threshold │ Action   │   │
│  ├────────────────┼────────────────────────┼───────────┼──────────│   │
│  │ Security       │ Critical vulns         │ = 0       │ Block    │   │
│  │                │ High vulns             │ < 5       │ Block    │   │
│  │                │ Medium vulns           │ < 20      │ Warn     │   │
│  ├────────────────┼────────────────────────┼───────────┼──────────│   │
│  │ Performance    │ P99 latency            │ < 300ms   │ Block    │   │
│  │                │ Throughput             │ > 1000 TPS│ Block    │   │
│  │                │ Error rate             │ < 0.1%    │ Block    │   │
│  ├────────────────┼────────────────────────┼───────────┼──────────│   │
│  │ Reliability    │ Test coverage          │ > 80%     │ Warn     │   │
│  │                │ Integration tests pass │ 100%      │ Block    │   │
│  ├────────────────┼────────────────────────┼───────────┼──────────│   │
│  │ Canary         │ Error rate vs baseline │ < 10%     │ Rollback │   │
│  │                │ Latency vs baseline    │ < 20%     │ Rollback │   │
│  └────────────────────────────────────────────────────────────────┘   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## Shrnutí: Architektonické dovednosti checklist

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    SOLUTION ARCHITECT SKILLS CHECKLIST                  │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  C4 MODEL & DOKUMENTACE                                                │
│  ══════════════════════                                                │
│  □ Umím vytvořit Context Diagram pro systém                           │
│  □ Umím vytvořit Container Diagram s technologiemi                    │
│  □ Umím vytvořit Component Diagram pro klíčové služby                 │
│  □ Vím kdy použít kterou úroveň pro které publikum                    │
│  □ Udržuji diagramy aktuální (automation kde možné)                   │
│  □ Používám konzistentní notaci a legendu                             │
│                                                                         │
│  ARCHITECTURE DECISION RECORDS                                         │
│  ═════════════════════════════                                         │
│  □ Dokumentuji významná architektonická rozhodnutí                    │
│  □ Používám strukturovanou šablonu (context, decision, consequences)  │
│  □ Zahrnuji zvažované alternativy a důvody zamítnutí                 │
│  □ Dokumentuji trade-offs a rizika                                    │
│  □ ADR jsou verzované a review-ované                                  │
│  □ Umím najít a odkazovat relevantní historická rozhodnutí           │
│                                                                         │
│  TRADE-OFF ANALYSIS                                                    │
│  ═══════════════════                                                   │
│  □ Umím identifikovat relevantní kritéria pro rozhodnutí             │
│  □ Umím přiřadit váhy kritériím podle business priority              │
│  □ Umím systematicky hodnotit alternativy                             │
│  □ Umím vizualizovat trade-offs (radar, matrix, quadrant)            │
│  □ Umím prezentovat doporučení s jasným zdůvodněním                  │
│  □ Uvědomuji si, že "záleží na kontextu" - není univerzální řešení  │
│                                                                         │
│  NFR DEFINICE A MĚŘENÍ                                                 │
│  ═════════════════════                                                 │
│  □ Umím definovat SMART NFR požadavky                                 │
│  □ Znám klíčové NFR kategorie (performance, availability, security)  │
│  □ Umím navrhnout metriky a SLI/SLO/SLA                              │
│  □ Rozumím error budget konceptu                                      │
│  □ Umím navrhnout NFR testing strategii                              │
│  □ Umím integrovat NFR gates do CI/CD pipeline                       │
│  □ Umím nastavit monitoring a alerting pro NFR                       │
│                                                                         │
│  KOMUNIKAČNÍ DOVEDNOSTI                                                │
│  ═══════════════════════                                               │
│  □ Umím prezentovat architekturu technickému i business publiku      │
│  □ Umím vést architektonické review                                   │
│  □ Umím facilitovat trade-off diskuze se stakeholders                │
│  □ Umím obhájit architektonická rozhodnutí                           │
│  □ Umím vysvětlit technické koncepty netechnickým lidem             │
│  □ Umím psát jasnou a stručnou technickou dokumentaci               │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

Chceš, abych pokračoval s dalšími tématy jako PSD2/Open Banking architektura, messaging patterns, observability, nebo něco jiného?


# Škálování v digitálním bankovnictví na AWS

---

## 1. Co je potřeba škálovat

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    VRSTVY ŠKÁLOVÁNÍ                                     │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                                                                  │   │
│  │  1. COMPUTE (Výpočetní kapacita)                                │   │
│  │     • API servery / Lambda funkce                               │   │
│  │     • Background workers                                         │   │
│  │     • Batch processing                                           │   │
│  │                                                                  │   │
│  │  2. DATABASE (Datová vrstva)                                    │   │
│  │     • Read capacity (čtení)                                     │   │
│  │     • Write capacity (zápis)                                    │   │
│  │     • Storage (úložiště)                                        │   │
│  │                                                                  │   │
│  │  3. MESSAGING (Fronty a eventy)                                 │   │
│  │     • Queue throughput                                           │   │
│  │     • Consumer processing                                        │   │
│  │     • Event bus capacity                                         │   │
│  │                                                                  │   │
│  │  4. CACHING (Cache vrstva)                                      │   │
│  │     • Memory capacity                                            │   │
│  │     • Connection count                                           │   │
│  │                                                                  │   │
│  │  5. NETWORK / API (Síťová vrstva)                              │   │
│  │     • API Gateway throughput                                    │   │
│  │     • Load balancer capacity                                    │   │
│  │     • CDN edge locations                                        │   │
│  │                                                                  │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 2. Horizontální vs Vertikální škálování

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    DVA PŘÍSTUPY                                         │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  VERTIKÁLNÍ (Scale Up)              HORIZONTÁLNÍ (Scale Out)           │
│  ══════════════════════             ════════════════════════           │
│                                                                         │
│  Větší instance                     Více instancí                      │
│                                                                         │
│       ┌───────┐                        ┌───┐ ┌───┐ ┌───┐              │
│       │       │                        │   │ │   │ │   │              │
│       │       │                        └───┘ └───┘ └───┘              │
│       │  BIG  │                        ┌───┐ ┌───┐ ┌───┐              │
│       │       │                        │   │ │   │ │   │              │
│       │       │                        └───┘ └───┘ └───┘              │
│       └───────┘                                                        │
│                                                                         │
│  ✅ Jednodušší                      ✅ Teoreticky neomezené            │
│  ✅ Žádná změna aplikace            ✅ Fault tolerant                  │
│  ❌ Limit velikosti instance        ✅ Cost-effective                  │
│  ❌ Single point of failure         ❌ Aplikace musí podporovat        │
│  ❌ Downtime při změně              ❌ State management složitější    │
│                                                                         │
│  BANKOVNICTVÍ: Preferujeme HORIZONTÁLNÍ (resilience + elasticita)     │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 3. AWS řešení pro každou vrstvu

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    COMPUTE ŠKÁLOVÁNÍ                                    │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  OPTION A: SERVERLESS (Lambda)                                         │
│  ══════════════════════════════                                        │
│                                                                         │
│  • Automatické škálování (0 → 1000s concurrent)                       │
│  • Platíš za execution time                                            │
│  • Reserved concurrency pro izolaci                                    │
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │  API Gateway ──▶ Lambda (auto-scales) ──▶ DynamoDB              │   │
│  │                     │                                            │   │
│  │              Concurrent executions:                              │   │
│  │              0 ──▶ 100 ──▶ 1000 ──▶ 3000 (account limit)       │   │
│  │                                                                  │   │
│  │  Nastavení:                                                      │   │
│  │  • Reserved concurrency: 500 (pro Payment service)              │   │
│  │  • Provisioned concurrency: 50 (eliminace cold starts)         │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  OPTION B: CONTAINERS (ECS/EKS)                                        │
│  ═══════════════════════════════                                       │
│                                                                         │
│  • Auto Scaling Groups / ECS Service Auto Scaling                     │
│  • Target tracking (CPU, memory, custom metrics)                      │
│  • Více kontroly, ale více operations                                 │
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │  ALB ──▶ ECS Service (Fargate)                                  │   │
│  │              │                                                   │   │
│  │         Auto Scaling Policy:                                     │   │
│  │         • Min: 3 tasks                                          │   │
│  │         • Max: 30 tasks                                         │   │
│  │         • Target: CPU 70%                                       │   │
│  │         • Scale out: +3 tasks when CPU > 70% for 2 min         │   │
│  │         • Scale in:  -1 task when CPU < 30% for 10 min         │   │
│  │         • Cooldown: 300 seconds                                 │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────┐
│                    DATABASE ŠKÁLOVÁNÍ                                   │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  AURORA (Relational):                                                  │
│  ════════════════════                                                  │
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                                                                  │   │
│  │  WRITE scaling: Vertikální (větší instance)                    │   │
│  │  ┌──────────────┐                                               │   │
│  │  │   Primary    │  db.r6g.xlarge → db.r6g.4xlarge              │   │
│  │  │   (Writer)   │                                               │   │
│  │  └──────────────┘                                               │   │
│  │                                                                  │   │
│  │  READ scaling: Horizontální (read replicas)                    │   │
│  │  ┌──────────────┐ ┌──────────────┐ ┌──────────────┐            │   │
│  │  │   Replica 1  │ │   Replica 2  │ │   Replica 3  │            │   │
│  │  │   (Reader)   │ │   (Reader)   │ │   (Reader)   │            │   │
│  │  └──────────────┘ └──────────────┘ └──────────────┘            │   │
│  │                                                                  │   │
│  │  Aurora Auto Scaling:                                           │   │
│  │  • Min replicas: 1                                              │   │
│  │  • Max replicas: 5                                              │   │
│  │  • Target: CPU 70% nebo connections 80%                        │   │
│  │                                                                  │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  DYNAMODB (NoSQL):                                                     │
│  ═════════════════                                                     │
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                                                                  │   │
│  │  On-Demand Mode: Automatické, platíš za request                │   │
│  │  • Škáluje se na miliony requests/sec                          │   │
│  │  • Žádná konfigurace                                           │   │
│  │  • Dražší při konstantní zátěži                                │   │
│  │                                                                  │   │
│  │  Provisioned Mode: Nastavíš capacity, auto-scaling            │   │
│  │  ┌────────────────────────────────────────────────────────┐    │   │
│  │  │ Read Capacity:  Min 100 RCU, Max 10000 RCU, Target 70% │    │   │
│  │  │ Write Capacity: Min 50 WCU,  Max 5000 WCU,  Target 70% │    │   │
│  │  └────────────────────────────────────────────────────────┘    │   │
│  │                                                                  │   │
│  │  DAX (Cache): Horizontální škálování nodes                     │   │
│  │                                                                  │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────┐
│                    MESSAGING ŠKÁLOVÁNÍ                                  │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  SQS: Automaticky škáluje (prakticky neomezené)                       │
│  ═══                                                                   │
│  • Bottleneck je consumer, ne queue                                   │
│  • Škáluj consumery podle ApproximateNumberOfMessages                │
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │  Queue depth > 1000 → Scale out consumers                       │   │
│  │  Queue depth < 100  → Scale in consumers                        │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  MSK (Kafka): Škáluj brokers + partitions                             │
│  ═══                                                                   │
│  • Více partitions = více paralelních consumers                       │
│  • Více brokers = více throughput                                     │
│                                                                         │
│  EventBridge: Automaticky škáluje                                     │
│  ════════════                                                          │
│  • Soft limit 10,000 events/sec (zvýšitelné)                         │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────┐
│                    CACHING ŠKÁLOVÁNÍ                                    │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  ElastiCache (Redis):                                                  │
│  ════════════════════                                                  │
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                                                                  │   │
│  │  Vertikální: Větší node type                                   │   │
│  │  cache.r6g.large → cache.r6g.xlarge → cache.r6g.2xlarge        │   │
│  │                                                                  │   │
│  │  Horizontální (Cluster Mode):                                   │   │
│  │  ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐                  │   │
│  │  │Shard 1 │ │Shard 2 │ │Shard 3 │ │Shard 4 │                  │   │
│  │  │Primary │ │Primary │ │Primary │ │Primary │                  │   │
│  │  │Replica │ │Replica │ │Replica │ │Replica │                  │   │
│  │  └────────┘ └────────┘ └────────┘ └────────┘                  │   │
│  │                                                                  │   │
│  │  Každý shard drží část key space (sharding by key hash)        │   │
│  │                                                                  │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────┐
│                    API / NETWORK ŠKÁLOVÁNÍ                              │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  API Gateway: Automaticky škáluje                                      │
│  ════════════                                                          │
│  • Default: 10,000 RPS (zvýšitelné)                                   │
│  • Throttling per client (usage plans)                                │
│                                                                         │
│  ALB: Automaticky škáluje                                              │
│  ════                                                                  │
│  • Pre-warming pro expected spikes                                     │
│                                                                         │
│  CloudFront: Globálně distribuované                                   │
│  ═══════════                                                           │
│  • Edge caching                                                        │
│  • Snižuje load na origin                                             │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 4. Konkrétní příklad: Payment Service při Salary Day

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    SCÉNÁŘ: SALARY DAY                                   │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  KONTEXT:                                                              │
│  • Normální den: 1,000 plateb/min                                     │
│  • Salary day (10. v měsíci): 10,000 plateb/min (10x spike)           │
│  • Spike trvá 2-3 hodiny (9:00 - 12:00)                               │
│  • Požadavek: P99 latency < 500ms, availability 99.99%                │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────┐
│                    ARCHITEKTURA                                         │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│                           ┌─────────────┐                              │
│                           │ CloudFront  │                              │
│                           │   (CDN)     │                              │
│                           └──────┬──────┘                              │
│                                  │                                      │
│                           ┌──────▼──────┐                              │
│                           │ API Gateway │ ← Auto-scales               │
│                           │  (REST)     │   10K RPS default           │
│                           └──────┬──────┘                              │
│                                  │                                      │
│              ┌───────────────────┼───────────────────┐                 │
│              │                   │                   │                 │
│              ▼                   ▼                   ▼                 │
│  ┌───────────────────────────────────────────────────────────────┐    │
│  │                    ECS FARGATE CLUSTER                         │    │
│  │                                                                │    │
│  │    Payment Service (Auto Scaling)                              │    │
│  │    ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐ ... ┌─────┐       │    │
│  │    │Task1│ │Task2│ │Task3│ │Task4│ │Task5│     │TaskN│       │    │
│  │    └─────┘ └─────┘ └─────┘ └─────┘ └─────┘     └─────┘       │    │
│  │                                                                │    │
│  │    Scaling Policy:                                             │    │
│  │    • Min: 5 tasks (baseline)                                   │    │
│  │    • Max: 50 tasks                                             │    │
│  │    • Target: CPU 60%                                           │    │
│  │    • Scheduled: 10. v měsíci 8:30 → min 20 tasks              │    │
│  │                                                                │    │
│  └───────────────────────────────────────────────────────────────┘    │
│              │                                                         │
│              ├─────────────────────────────────────┐                   │
│              │                                     │                   │
│              ▼                                     ▼                   │
│  ┌───────────────────────────┐      ┌───────────────────────────┐     │
│  │      AURORA CLUSTER       │      │      ELASTICACHE          │     │
│  │                           │      │       (Redis)             │     │
│  │  ┌─────────┐ ┌─────────┐  │      │                           │     │
│  │  │ Primary │ │ Replica │  │      │  Session cache            │     │
│  │  │(Writer) │ │ (Reader)│  │      │  Rate limiting            │     │
│  │  └─────────┘ └─────────┘  │      │  Idempotency keys         │     │
│  │              ┌─────────┐  │      │                           │     │
│  │              │ Replica │  │      │  Cluster: 3 shards        │     │
│  │              │ (Reader)│  │      │                           │     │
│  │              └─────────┘  │      │                           │     │
│  │                           │      └───────────────────────────┘     │
│  │  Read Replica Auto Scale: │                                        │
│  │  Min: 2, Max: 5           │                                        │
│  │  Target: CPU 70%          │                                        │
│  │                           │                                        │
│  └───────────────────────────┘                                        │
│              │                                                         │
│              │                                                         │
│              ▼                                                         │
│  ┌───────────────────────────────────────────────────────────────┐    │
│  │                    SQS QUEUE                                   │    │
│  │                 (Payment Processing)                           │    │
│  │                                                                │    │
│  │  Queue auto-scales (unlimited)                                 │    │
│  │  Consumer scaling based on queue depth                         │    │
│  │                                                                │    │
│  └───────────────────────────────────────────────────────────────┘    │
│              │                                                         │
│              ▼                                                         │
│  ┌───────────────────────────────────────────────────────────────┐    │
│  │                PAYMENT PROCESSOR (Lambda)                      │    │
│  │                                                                │    │
│  │  Reserved concurrency: 500                                     │    │
│  │  Provisioned concurrency: 100 (warm)                          │    │
│  │  Auto-scales: 0 → 500 based on SQS messages                   │    │
│  │                                                                │    │
│  └───────────────────────────────────────────────────────────────┘    │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### Scaling konfigurace

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    SCALING POLICIES                                     │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  1. ECS SERVICE AUTO SCALING:                                          │
│  ═════════════════════════════                                         │
│                                                                         │
│  # Target Tracking Policy                                              │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ {                                                               │   │
│  │   "targetValue": 60.0,                                         │   │
│  │   "predefinedMetricSpecification": {                           │   │
│  │     "predefinedMetricType": "ECSServiceAverageCPUUtilization"  │   │
│  │   },                                                            │   │
│  │   "scaleOutCooldown": 60,                                      │   │
│  │   "scaleInCooldown": 300                                       │   │
│  │ }                                                               │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  # Scheduled Scaling (Salary Day)                                      │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ aws application-autoscaling put-scheduled-action \              │   │
│  │   --service-namespace ecs \                                     │   │
│  │   --resource-id service/cluster/payment-service \               │   │
│  │   --scheduled-action-name salary-day-scale-up \                 │   │
│  │   --schedule "cron(30 8 10 * ? *)" \                           │   │
│  │   --scalable-dimension ecs:service:DesiredCount \               │   │
│  │   --scalable-target-action MinCapacity=20,MaxCapacity=50       │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  2. AURORA READ REPLICA AUTO SCALING:                                  │
│  ═════════════════════════════════════                                 │
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ aws application-autoscaling register-scalable-target \          │   │
│  │   --service-namespace rds \                                     │   │
│  │   --resource-id cluster:payment-db-cluster \                    │   │
│  │   --scalable-dimension rds:cluster:ReadReplicaCount \           │   │
│  │   --min-capacity 2 \                                            │   │
│  │   --max-capacity 5                                              │   │
│  │                                                                  │   │
│  │ # Target tracking na CPU                                        │   │
│  │ Target: 70% CPU utilization                                     │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  3. LAMBDA CONCURRENCY:                                                │
│  ═══════════════════════                                               │
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ PaymentProcessor:                                               │   │
│  │   ReservedConcurrentExecutions: 500   # Max concurrent         │   │
│  │   ProvisionedConcurrency: 100          # Always warm           │   │
│  │                                                                  │   │
│  │ Provisioned Concurrency Auto Scaling:                          │   │
│  │   Schedule: 10. v měsíci 8:00 → 200 provisioned               │   │
│  │             10. v měsíci 14:00 → 50 provisioned                │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  4. SQS-BASED SCALING (Lambda):                                        │
│  ═══════════════════════════════                                       │
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ Event Source Mapping:                                           │   │
│  │   BatchSize: 10                                                 │   │
│  │   MaximumBatchingWindowInSeconds: 1                            │   │
│  │   ScalingConfig:                                                │   │
│  │     MaximumConcurrency: 500                                     │   │
│  │                                                                  │   │
│  │ Lambda automaticky škáluje podle queue depth                   │   │
│  │ Více zpráv → více concurrent executions                        │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### Timeline: Salary Day

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    SALARY DAY TIMELINE                                  │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  Load                                                                   │
│    ▲                                                                    │
│    │                     ┌──────────────────┐                          │
│ 10x│                    ╱│                  │╲                         │
│    │                   ╱ │    PEAK LOAD     │ ╲                        │
│    │                  ╱  │   10K req/min    │  ╲                       │
│  5x│                 ╱   │                  │   ╲                      │
│    │                ╱    │                  │    ╲                     │
│    │               ╱     │                  │     ╲                    │
│  1x│──────────────╱      │                  │      ╲──────────────     │
│    │                     │                  │                          │
│    └────────────────────────────────────────────────────────────▶ Time│
│         8:00    9:00    10:00   11:00   12:00   13:00   14:00         │
│                                                                         │
│  EVENTS:                                                               │
│  ════════                                                              │
│                                                                         │
│  08:30 - Scheduled scale-up triggers                                   │
│          • ECS: 5 → 20 tasks (pre-warming)                            │
│          • Lambda: Provisioned concurrency 100 → 200                  │
│          • Aurora: Ensure 3 read replicas ready                       │
│                                                                         │
│  09:00 - Load begins increasing                                        │
│          • Target tracking policies activate                          │
│          • ECS scales: 20 → 30 → 40 tasks                            │
│          • Lambda concurrent executions: 50 → 200 → 400              │
│          • Aurora read replicas: 3 → 4                                │
│                                                                         │
│  10:00 - Peak load                                                      │
│          • ECS: 45 tasks running                                       │
│          • Lambda: 450 concurrent executions                          │
│          • Cache hit rate: 95% (reducing DB load)                     │
│          • Queue depth: stable ~100 messages                          │
│                                                                         │
│  12:00 - Load decreasing                                               │
│          • Scale-in cooldown prevents premature reduction            │
│          • Gradual reduction begins                                    │
│                                                                         │
│  14:00 - Scheduled scale-down                                          │
│          • ECS: min capacity back to 5                                │
│          • Lambda: Provisioned concurrency 200 → 50                  │
│          • Natural scale-in over next 30 minutes                     │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### Monitoring škálování

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    KLÍČOVÉ METRIKY PRO SCALING                          │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  ┌────────────────────────────────────────────────────────────────┐   │
│  │ Component       │ Scaling Metric      │ Threshold │ Action     │   │
│  ├─────────────────┼─────────────────────┼───────────┼────────────│   │
│  │ ECS Service     │ CPUUtilization      │ > 60%     │ Scale out  │   │
│  │                 │ CPUUtilization      │ < 30%     │ Scale in   │   │
│  │                 │ MemoryUtilization   │ > 80%     │ Alert      │   │
│  ├─────────────────┼─────────────────────┼───────────┼────────────│   │
│  │ Aurora          │ CPUUtilization      │ > 70%     │ Add replica│   │
│  │                 │ DatabaseConnections │ > 80%     │ Alert      │   │
│  │                 │ ReadLatency         │ > 20ms    │ Alert      │   │
│  ├─────────────────┼─────────────────────┼───────────┼────────────│   │
│  │ Lambda          │ ConcurrentExecutions│ > 80%     │ Alert      │   │
│  │                 │ Throttles           │ > 0       │ Alert      │   │
│  │                 │ Duration P99        │ > 5s      │ Alert      │   │
│  ├─────────────────┼─────────────────────┼───────────┼────────────│   │
│  │ SQS             │ ApproxNumberOfMsgs  │ > 1000    │ Scale cons.│   │
│  │                 │ ApproxAgeOfOldest   │ > 60s     │ Alert      │   │
│  ├─────────────────┼─────────────────────┼───────────┼────────────│   │
│  │ ElastiCache     │ CPUUtilization      │ > 65%     │ Alert      │   │
│  │                 │ CurrConnections     │ > 80%     │ Alert      │   │
│  │                 │ CacheHitRate        │ < 90%     │ Alert      │   │
│  ├─────────────────┼─────────────────────┼───────────┼────────────│   │
│  │ API Gateway     │ 5XXError            │ > 1%      │ Alert      │   │
│  │                 │ Latency P99         │ > 500ms   │ Alert      │   │
│  │                 │ Count               │ > 8000/s  │ Pre-warm   │   │
│  └────────────────────────────────────────────────────────────────┘   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### Shrnutí

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    SCALING BEST PRACTICES                               │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  1. PREFERUJ HORIZONTÁLNÍ před vertikálním (resilience)               │
│                                                                         │
│  2. KOMBINUJ STRATEGIE:                                                │
│     • Target tracking pro reactive scaling                             │
│     • Scheduled scaling pro předvídatelné peaks                       │
│     • Step scaling pro gradual response                                │
│                                                                         │
│  3. SERVERLESS kde možné (Lambda, DynamoDB on-demand)                 │
│     • Automatické škálování                                            │
│     • Pay-per-use                                                       │
│                                                                         │
│  4. PRE-WARMING pro kritické eventy                                    │
│     • Scheduled scale-up před expected peak                           │
│     • Provisioned concurrency pro Lambda                              │
│     • ALB pre-warming pro massive spikes                              │
│                                                                         │
│  5. BOTTLENECK AWARENESS:                                              │
│     • Database je často bottleneck → read replicas, caching          │
│     • Identifikuj a škáluj celý řetězec                              │
│                                                                         │
│  6. COOLDOWN PERIODS:                                                  │
│     • Scale-out: kratší (rychlá reakce)                               │
│     • Scale-in: delší (prevence flapping)                             │
│                                                                         │
│  7. COST AWARENESS:                                                    │
│     • Reserved capacity pro baseline                                   │
│     • On-demand/Spot pro peaks                                         │
│     • Savings Plans pro predictable workloads                         │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

Chceš, abych rozvedl některou konkrétní část škálování více do detailu, nebo přešel na další téma?