## Co je RACI
Zkratka **RACI** představuje čtyři typy rolí:

### **R – Responsible (zodpovědný za provedení)**

- Ten, kdo **reálně vykonává práci**
- Může jich být více
- „Kdo to udělá?“
### **A – Accountable (konečně odpovědný / schvalující)**

- Ten, kdo **nese finální odpovědnost za výsledek**
- **Vždy jen jeden člověk!**
- „Kdo to schvaluje / za koho to padá?“
### **C – Consulted (konzultovaný)**

- Lidé, kteří poskytují **vstupy, rady, expertízu**
- Komunikace je **obousměrná**
- „Koho se musíme zeptat?“
### ** I – Informed (informovaný)**

- Lidé, kteří mají být **informováni o průběhu nebo výsledku**
- Komunikace je **jednosměrná**
- „Kdo to potřebuje vědět?“

| **Úkol / Role**       | **Projektový manažer** | **Vývojář** | **Produktový manažer** | **CEO** |
| --------------------- | ---------------------- | ----------- | ---------------------- | ------- |
| Analýza požadavků     | A                      | R           | C                      | I       |
| Vývoj funkce          | I                      | R           | C                      | I       |
| Schválení release     | R                      | C           | A                      | I       |
| Komunikace zákazníkům | R                      | I           | A                      | I       |
# **Důležitá pravidla RACI**

- Každý úkol musí mít:
    - **alespoň jedno R**
    - **právě jedno A**
        
- Jedna osoba může mít více rolí (např. R i A), ale:
    - pozor na přetížení
        
- Vyhni se:
    - „všichni jsou odpovědní“ → pak není odpovědný nikdo
    - příliš mnoha C → zpomaluje rozhodování
---
# RACI matice a artefakty — migrace BFF do AWS

## Kontext projektu

Migrace stávajícího on-prem BFF (Backend For Frontend) pro mobilní a internetové bankovnictví do AWS, včetně:

- migrace aplikační logiky a databáze,
- zachování integrací: core banking systém, 3rd party obohacení dat, Kafka (consumer/producer),
- nová funkcionalita: nápočty nad daty + persistence do aplikační DB.

V matici používám role: **BA** (Business Analyst), **SA** (Solution Architect), **CA** (Cloud/AWS Architect). Doplňuji další role, které u takového projektu prakticky nelze obejít: **PO/PM** (Product Owner / Project Manager), **SecArch** (Security Architect / InfoSec), **DBA**, **DevTeam Lead**, **SRE/Ops**, **Enterprise Architect (EA)**, **Compliance** (regulatoric/risk — u banky nezbytné).

Legenda: **R** = Responsible (dělá to), **A** = Accountable (zodpovídá, schvaluje, jeden na úkol), **C** = Consulted (konzultuje se před rozhodnutím), **I** = Informed (informován o výsledku).

---
## Doporučení specificky pro váš kontext

1. **Banka = Compliance je rovnocenný hráč.** U BFF pro bankovnictví v AWS si dejte pozor na DORA (od 2025 plně účinné), výběr regionu (EU), data residency, klíče v KMS pod vaší kontrolou (BYOK/CloudHSM zvažte), audit trail. Compliance musí být u stolu od týdne 1, ne na konci.
    
2. **Kafka v AWS — explicitní rozhodnutí.** SA + CA musí brzy rozhodnout: zůstává Kafka on-prem a BFF se k ní připojuje přes Direct Connect? Nebo migrace na MSK / MSK Serverless? Toto rozhodnutí ovlivní latenci a celý design integrace — patří do ADR.
    
3. **Nápočty = nový pattern, vyžaduje vlastní pozornost.** Není to triviální rozšíření. SA by měl explicitně rozhodnout: synchronně v request-response cyklu BFF? Asynchronně přes Kafka events? Periodicky (scheduled job)? Materialized views v DB? Každá varianta má jiné dopady na DB design, náklady i SLA.
    
4. **Core banking integrace = nejrizikovější bod.** Latence, dostupnost a propustnost core systému typicky určují celkovou SLA BFF. Doporučuji explicitní artefakt „Core Integration Resilience Design" — circuit breakers, bulkheads, caching strategie, degraded mode pro mobil/web.
    
5. **Pro hladký handoff** mezi BA → SA → CA si ustavte tři checkpointy s definovanou „definition of ready":
    
    - **Requirements baseline** (BA → SA): NFR jsou kvantifikované, ne „rychlé a bezpečné".
    - **Solution baseline** (SA → CA): logická architektura schválena, integrační vzory zvoleny, ADR pro klíčová rozhodnutí existují.
    - **Build baseline** (CA → DevLead): IaC kostra existuje, landing zone běží, dev prostředí je k dispozici.

Pokud chcete, můžu z toho udělat **Word dokument** nebo **Excel s RACI maticí** k použití v projektu — stačí říct, jaký formát preferujete.


# RACI matice a artefakty — aktualizováno pro váš tým

## Konsolidované role

|Role v týmu|Pokrývá odpovědnosti|
|---|---|
|**BA**|Business Analyst|
|**SA**|Solution Architect + Cloud/AWS Architect + Product Owner + DBA + Enterprise Architect|
|**PM**|Project Manager|
|**DevLead**|Development Lead + Security Architect + SRE/Ops|
|**Compliance**|Regulatory, risk, audit|

⚠️ **Upozornění na rizikové kombinace** (vrátím se k nim na konci):

- **SA má extrémní záběr** — od produktové vize přes architekturu, cloud, databázi až po enterprise standardy. Toto je nejvíce přetížená role v týmu.
- **DevLead kombinuje vývoj + bezpečnost + provoz** — klasický konflikt zájmů (kdo staví ≠ kdo kontroluje bezpečnost ≠ kdo provozuje). U banky obzvlášť citlivé.

---

## RACI matice

### 1. Discovery a požadavky

| Aktivita / artefakt                                 | BA      | SA      | PM      | DevLead | Compliance |
| --------------------------------------------------- | ------- | ------- | ------- | ------- | ---------- |
| Byznysový case, vize, cíle migrace                  | R       | **A**   | R       | I       | C          |
| Project charter, rozpočet, timeline                 | C       | C       | **R/A** | C       | I          |
| Funkční požadavky + user stories (vč. nápočtů)      | **R**   | **A**   | C       | C       | C          |
| Prioritizace backlogu, scope rozhodnutí             | C       | **R/A** | C       | C       | C          |
| Nefunkční požadavky (SLA, RTO/RPO, výkon)           | R       | **R/A** | C       | C       | C          |
| Mapování as-is procesů a integrací                  | **R/A** | C       | I       | C       | I          |
| To-be procesní design                               | **R/A** | C       | I       | C       | C          |
| Compliance & regulatory checklist (ČNB, GDPR, DORA) | C       | C       | I       | C       | **R/A**    |
| Akceptační kritéria                                 | **R**   | **A**   | I       | C       | C          |

### 2. Architektura a design

|Aktivita / artefakt|BA|SA|PM|DevLead|Compliance|
|---|---|---|---|---|---|
|As-is architecture document|C|**R/A**|I|C|I|
|Cílová logická architektura (to-be)|C|**R/A**|I|C|C|
|Migrační strategie (rehost/replatform/refactor)|I|**R/A**|C|C|I|
|AWS landing zone, sítě, VPC, konektivita on-prem|I|**R/A**|I|C|I|
|Volba AWS služeb (compute, DB, messaging, observability)|I|**R/A**|I|C|I|
|Datový model a DB design (aplikační DB)|C|**R/A**|I|C|I|
|Návrh nápočtů (batch vs. stream, kde počítat)|C|**R/A**|I|C|I|
|Integrační design — core banking|R|**R/A**|I|C|C|
|Integrační design — 3rd party obohacení|R|**R/A**|I|C|C|
|Integrační design — Kafka (topiky, schémata, DLQ)|C|**R/A**|I|C|I|
|API kontrakty pro mobilní/web klienty|R|**R/A**|I|C|I|
|Bezpečnostní architektura (IAM, KMS, šifrování, secrets)|I|C|I|**R/A**|C|
|Threat modeling|I|C|I|**R/A**|C|
|HA, DR, multi-AZ strategie|I|**R/A**|C|R|C|
|Observability design (logy, metriky, tracing, alerting)|I|C|I|**R/A**|I|
|FinOps — odhad provozních nákladů|I|**R/A**|C|C|I|
|ADR (Architecture Decision Records)|I|**R/A**|I|C|I|
|Solution Design Document (SDD)|C|**R/A**|I|C|C|

### 3. Migrace dat

| Aktivita / artefakt                                | BA    | SA      | PM  | DevLead | Compliance |
| -------------------------------------------------- | ----- | ------- | --- | ------- | ---------- |
| Strategie migrace dat (DMS, cutover vs. paralelní) | C     | **R/A** | C   | R       | C          |
| Mapování zdrojové → cílové DB                      | C     | **R/A** | I   | C       | I          |
| Plán validace migrovaných dat                      | **R** | **A**   | I   | C       | C          |
| Reconciliation report                              | R     | **A**   | I   | C       | C          |

### 4. Implementace, testování, nasazení

|Aktivita / artefakt|BA|SA|PM|DevLead|Compliance|
|---|---|---|---|---|---|
|Detailní design komponent|I|C|I|**R/A**|I|
|Implementace (kód, IaC)|I|C|I|**R/A**|I|
|CI/CD pipeline|I|C|I|**R/A**|I|
|Testovací strategie (unit/integration/perf)|C|C|C|**R/A**|C|
|Performance & load testing|C|R|I|**R/A**|I|
|UAT akceptace|**R**|**A**|C|C|C|
|Penetration testing & security review|I|C|I|**R/A**|C|
|Compliance audit / regulatorní review|C|C|I|C|**R/A**|
|Cutover plan + rollback|C|R|**R/A**|R|C|
|Go/No-go rozhodnutí|C|C|**R/A**|C|C|
|Provoz po nasazení (run-book, on-call)|I|C|I|**R/A**|I|
|Status reporting směrem k vedení|I|C|**R/A**|I|I|
|Risk management projektu|C|C|**R/A**|C|C|

---

## Seznam artefaktů — kdo dodává

### Discovery fáze

1. **Business Case & Project Charter** — _PM_ (s vstupem SA a BA)
2. **Mapa as-is procesů a systémů** — _BA_
3. **Katalog funkčních požadavků + user stories** — _BA_ (akceptuje SA jako produktový vlastník)
4. **Katalog NFR** — _BA + SA_
5. **Compliance & regulatory checklist** (ČNB, GDPR, DORA, data residency EU) — _Compliance_
6. **Product backlog & roadmapa BFF** — _SA_ (v roli PO)

### Architektura a design

7. **As-is Architecture Document** — _SA_
8. **Solution Design Document (SDD)** — _SA_ (zastřešuje logickou + cloudovou + datovou architekturu, protože SA pokrývá SA+CA+DBA+EA)
9. **Migration Strategy Document** — _SA_
10. **AWS Landing Zone & Network Design** — _SA_ (review DevLead kvůli security)
11. **AWS Service Selection & Low-Level Design** — _SA_
12. **Data Model & Database Design** — _SA_
13. **Integration Design Specs** (3 dokumenty: core banking, 3rd party, Kafka) — _SA_
14. **API Contracts** (OpenAPI specs) — _SA + DevLead_
15. **Computation Design pro nápočty** — _SA_
16. **Security Architecture Document** — _DevLead_ (review SA, schvaluje Compliance)
17. **Threat Model** — _DevLead_
18. **HA & DR Plan** — _SA + DevLead_
19. **Observability Design** — _DevLead_
20. **FinOps / Cost Estimate** — _SA_
21. **ADR Repository** — _SA_

### Migrace dat

22. **Data Migration Plan** — _SA_
23. **Source-to-Target Mapping** — _SA_
24. **Data Validation & Reconciliation Report** — _BA + SA_

### Implementace a nasazení

25. **Detailed Component Designs / Tech specs** — _DevLead_
26. **IaC repozitář** (Terraform/CDK) — _DevLead_
27. **CI/CD pipeline definition** — _DevLead_
28. **Test Strategy & Test Plans** — _DevLead_
29. **Security Test Report / Pentest Report** — _DevLead_ (ideálně externí pentest, viz risk níže)
30. **Compliance Audit Report** — _Compliance_
31. **Cutover Plan + Rollback Plan** — _PM + SA + DevLead_
32. **Run-book & Operations Handbook** — _DevLead_
33. **Status reports** — _PM_
34. **RAID log** (Risks, Assumptions, Issues, Dependencies) — _PM_
35. **Go-Live Acceptance & Sign-off** — _PM_ (vstupy od SA, BA, DevLead, Compliance)

---

## ⚠️ Rizika a doporučení k vašemu setupu

Toto rozdělení je realistické pro menší tým, ale má **konkrétní rizika**, která byste si měli pojmenovat a vědomě řídit. V bance obzvlášť.

### 1. SA je přetížený a má příliš mnoho „klobouků"

SA u vás dělá: produktovou vizi (PO) + řešení (SA) + cloud (CA) + databázi (DBA) + enterprise standardy (EA). To je **5 rolí v jedné**. Důsledky:

- **Konflikt zájmů PO vs. SA**: PO tlačí na rychlé doručení byznys hodnoty, SA tlačí na technickou kvalitu a udržitelnost. Když je to jeden člověk, jedno z toho prohrává — typicky kvalita.
- **Bottleneck**: SA bude úzké hrdlo všeho. Každé rozhodnutí čeká na něj.
- **Žádný „peer review"** architektonických rozhodnutí — SA si schvaluje sám sobě.

**Mitigace:**

- Zaveďte **architekturní review** (alespoň jednou za sprint) s někým mimo projekt — třeba enterprise architektem z jiného týmu nebo externím konzultantem.
- Pro klíčová rozhodnutí o produktu (scope, prioritizace) **přizvěte byznys stakeholdery** přímo — at SA nerozhoduje sám o tom, co je důležité pro mobilní/internet banking.
- Když je rozhodnutí mezi „rychle" a „správně", vědomě ho **logujte do ADR** s odůvodněním — později uvidíte, kam se to ohnulo.

### 2. DevLead = vývoj + bezpečnost + provoz = konflikt zájmů

V bance je tohle **regulatorně problematické**. DORA i interní auditní pravidla typicky vyžadují **separation of duties** mezi:

- kdo staví (development),
- kdo kontroluje bezpečnost (security review, pentest),
- kdo provozuje (operations).

Když je to jeden člověk, vzniká riziko, že:

- bezpečnostní problémy se „odkládají na později", protože tlačí dodávka,
- pentest dělaný interně vlastním týmem najde méně problémů než externí,
- incidenty v provozu se řeší „z pozice viníka" — DevLead opravuje to, co sám postavil.

**Mitigace (silně doporučuji):**

- **Pentest zadejte externě** — buď jiný tým v bance, nebo externí firma. Není to nice-to-have, je to v bance prakticky povinnost.
- **Security review klíčových artefaktů** (Threat Model, Security Architecture Document) by měl dělat **někdo mimo projekt** — typicky bankovní InfoSec/CISO tým. Compliance by to měla vyžadovat.
- **Produkční přístupy a deploymenty** by ideálně neměl mít stejný člověk, který kód píše. Pokud to personálně nejde, alespoň zaveďte **4-eyes princip** (peer approval na PR a deploy).
- Otevřeně komunikujte s Compliance, že DevLead pokrývá tyto tři role — Compliance by měla aktivně kompenzovat audit trailem a externí kontrolou.

### 3. Compliance jako jediná „brzda"

Compliance je u vás jediná role mimo realizační linku. To je dobře (nezávislost), ale znamená to, že **veškerá regulatorní odpovědnost leží na ní**. Doporučuji:

- Zapojit Compliance **od týdne 1**, ne až před go-live.
- Compliance by měla mít **právo veta** na klíčové artefakty (Security Architecture, Threat Model, Cutover Plan, Go-Live).
- Zvažte, zda Compliance pokrývá i **DORA-specifické věci** (operational resilience, ICT third-party risk pro AWS jako kritického dodavatele) — tohle bývá speciální disciplína.

### 4. Co s BA

BA má v této sestavě **silnou a jasnou roli** — je jediný, kdo se plně věnuje byznys stránce. Doporučuji:

- BA by měl být **úzce navázaný na SA jako PO** — protože když SA nestíhá produktovou roli, BA může část přebrat (hlavně backlog grooming a komunikaci s byznysem).
- BA by měl mít **R nebo A na všem, co se týká byznys hodnoty a akceptace** — bez toho hrozí, že tým postaví technicky krásné řešení, které neřeší byznys problém.

### 5. Praktický návrh: pár handoff bodů

Pro tento konkrétní tým bych doporučil tyto **formální checkpointy**, kde se role potkávají a vzájemně si schvalují výstupy:

1. **Requirements baseline** — BA + SA + Compliance schvalují, že požadavky a NFR jsou připraveny pro design.
2. **Architecture baseline** — SA prezentuje SDD, DevLead a Compliance dělají review (toto kompenzuje absenci samostatného CA a SecArch).
3. **Security baseline** — DevLead prezentuje Security Architecture + Threat Model, Compliance + externí reviewer schvalují.
4. **Pre-cutover gate** — všichni schvalují cutover plan, externí pentest report je dostupný, Compliance dává regulatorní OK.
5. **Go-live gate** — PM orchestruje, ale veto má kdokoliv z SA / DevLead / Compliance.

---

Chcete, abych z toho udělal **Word dokument** s formátovanou maticí a artefakty pro sdílení v týmu, nebo **Excel** kde si můžete RACI dál upravovat za pochodu?