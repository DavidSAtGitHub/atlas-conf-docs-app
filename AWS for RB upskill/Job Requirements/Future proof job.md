V cloudu bude „future-proof“ pro data inženýra/solution architekta hlavně to, co AI jen těžko nahradí: rozhodování v kontextu firmy, trade-offy, zodpovědnost za rizika (security, compliance), navrhování platforem a standardů, a schopnost dovést změnu do produkce napříč týmy. AI ti vezme rutinu (boilerplate kód, části ETL, generování IaC), ale nezmizí potřeba někoho, kdo garantuje správnou architekturu, provozní kvalitu a dopad na byznys.

  

  

Kam směřovat fokus (aby tě AI spíš „násobila“ než nahrazovala)

  

  

  

1) Data platforma jako produkt (Platform Engineering pro data)

  

  

Nejstabilnější bývá role, která buduje interní data platformu:

  

- self-service onboarding dat, šablony, standardy, guardrails
- CI/CD pro data, automatizace, observability, reliability
- katalog, lineage, data contracts, governance „by design“  
    Tohle je těžké outsourcovat i automatizovat do nuly, protože je to kombinace tech + procesů + lidí.

  

  

Kompetence: lakehouse/streaming architektura, workflow orchestrace, IaC, runtime bezpečnost, datová kvalita, SLO/SLA pro data, developer experience.

  

  

2) Security, privacy, governance a compliance pro data

  

  

Regulace a rizika porostou rychleji než schopnost firem to zvládat. Kdo umí propojit data architekturu s:

  

- IAM, encryption, key management, network segmentation
- PII, retention, audit, GDPR
- policy-as-code (guardrails), klasifikace dat  
    …ten má velmi silnou pozici.

  

  

Kompetence: cloud security baseline, data access patterns, row/column level security, anonymizace/pseudonymizace, auditovatelnost.

  

  

3) AI-ready data (kvalita, provenance, metadata) + MLOps/LLMOps „na úrovni platformy“

  

  

Firmy nebudou řešit jen modely, ale hlavně spolehlivá data a provoz AI:

  

- feature store / embedding store, vektorové vyhledávání
- governance pro trénovací data, provenance, evaluace
- monitoring driftu, cost controls
- integrace do produktů (RAG, agenti) bezpečně a auditovatelně

  

  

Kompetence: data pro AI pipeline, LLM bezpečnost (prompt injection, data exfil), evaluace, observability, cost/perf trade-offs.

  

  

4) Nákladová disciplína (FinOps pro data/AI)

  

  

AI a data umí rozpočty „sežrat“. Architekt, který umí navrhnout výkon × cenu × spolehlivost, bude klíčový:

  

- storage tiering, compute separation, workload management
- optimalizace dotazů a pipeline, kvóty, chargeback/showback

  

  

  

  

  

O jaké pozice usilovat (prakticky)

  

  

  

Nejvíc „stability“: seniorní architektura + platforma

  

  

- Principal/Staff Data Architect (Cloud) – standardy, target architecture, governance, review
- Data Platform Architect / Data Platform Engineering Lead – platforma jako produkt, self-service, guardrails
- Cloud/Data Security Architect (zaměření na data) – přístupová politika, compliance, threat model

  

  

  

Hodně perspektivní: AI enablement přes data

  

  

- AI/ML Platform Engineer (MLOps/LLMOps) – platforma pro trénink/inferenci, evaluace, observability
- Analytics/Data Product Architect – data produkty, data contracts, doménové rozhraní (data mesh v praxi)

  

  

  

Pokud tě baví leadership a dopad

  

  

- Head of Data Engineering / Data Platform Manager – strategická stabilita, protože jde o organizaci a provoz, ne jen kód
- Enterprise Architect (Data/AI) – nejvíc „politiky“, ale velmi odolné vůči automatizaci

  

  

  

  

  

Jaká role je „nejvíc future-proof“?

  

  

Neexistuje jedna magická, ale nejstabilnější kombinace bývá:

  

„Data Platform / Data Security / AI Enablement Architect na principal/staff úrovni“

= člověk, který nese odpovědnost za platformu, rizika a standardy, ne za psaní jednotlivých pipeline.

  

Jinými slovy: posuň se od „builder“ (dělám pipeline) k „owner“ (navrhuju systém, který umožní stovkám pipeline vznikat bezpečně a levně).

  

  

  

  

Konkrétní roadmap (12–18 měsíců, bez ohledu na konkrétní cloud)

  

  

1. Zvednout platformové základy

  

  

  

- IaC (Terraform/Pulumi), CI/CD, policy-as-code
- observability (logs/metrics/traces) pro data pipeline + data quality checks

  

  

  

2. Zformulovat „reference architecture“

  

  

  

- batch + streaming + governance + security baseline
- standard pro data contracts, naming, ownership, SLO

  

  

  

3. Přidat AI-ready vrstvu

  

  

  

- metadata/lineage, katalog, PII klasifikace
- vektorové use-cases (RAG) s bezpečností a evaluací

  

  

  

4. Udělat z toho produkt

  

  

  

- onboarding nový tým do 1–2 dnů
- golden paths, šablony repo, guardrails, dokumentace

  

  

  

  

  

Rychlý test, jestli jsi na správné trajektorii

  

  

Jsi „future-proof“, pokud většina tvé hodnoty je v tom, že:

  

- umíš říct ne špatnému řešení a obhájit to daty/risks/costem,
- nastavíš standard, který zrychlí 10 týmů, ne jen jeden projekt,
- řešíš spolehlivost, bezpečnost, governance, ne jen transformace dat.

  

  

  

  

Pokud mi napíšeš (stručně) tvůj aktuální stack (AWS/Azure/GCP + Databricks/Snowflake/BigQuery… + streaming/BI) a jak velká je firma/týmy, navrhnu ti 2–3 nejvhodnější cílové role a konkrétní „skill gap“ plán přesně pro tvůj kontext.


Solution architekt je pořád velmi dobrá „future-proof“ volba – jen se ta role posouvá. AI ti vezme část práce (návrhy, diagramy, boilerplate, porovnání služeb), ale nezmizí potřeba člověka, který nese odpovědnost za rozhodnutí a trade-offy a umí to prosadit napříč týmy.

  

  

Proč je SA stabilní i při AI

  

  

SA dělá věci, které AI typicky neumí spolehlivě:

  

- Rozhodování v kontextu firmy (rizika, rozpočet, time-to-market, schopnosti týmu)
- Zodpovědnost: bezpečnost, compliance, data privacy, audit
- Vyjednávání mezi stakeholdery (produkt, security, ops, finance)
- End-to-end ownership: návrh → delivery → provozní kvalita (SLO, incidenty)

  

  

  

Co SA nejvíc ohrožuje (a čemu se vyhnout)

  

  

Nejmíň stabilní je „powerpoint architekt“:

  

- jen kreslí diagramy, píše high-level dokumenty
- nemá vliv na delivery, neměří dopady, neřeší provoz  
    Tohle AI + šablony + reference architektury časem zlevní.

  

  

  

Jak vypadá „future-proof“ Solution Architect v cloudu

  

  

  

1) Platform/Cloud Solution Architect (s dopadem na standardy)

  

  

- vytváří reference architektury a guardrails
- definuje „golden paths“ pro týmy (onboarding, CI/CD, IaC, security baseline)
- rozhoduje o platformních stavebních blocích

  

  

Stabilita: velmi vysoká, protože to mění efektivitu celé organizace.

  

  

2) Security-minded Solution Architect (cloud + data)

  

  

- IAM, network segmentation, encryption, secrets, zero trust
- threat modeling, compliance, auditability, PII
- policy-as-code a prevence driftu

  

  

Stabilita: extrémně vysoká (regulace a rizika porostou).

  

  

3) Data/AI Solution Architect (AI enablement)

  

  

- návrh datových toků tak, aby byly použitelné pro AI (kvalita, lineage, contracts)
- RAG/agentní architektury bezpečně (hodnocení, guardrails, monitoring)
- cost/perf řízení AI workloadů

  

  

Stabilita: vysoká, pokud jsi „praktický“ a držíš provoz.

  

  

Kam směřovat fokus, aby ses odlišil

  

  

Pokud jsi data inženýr/SA, nejvíc se vyplatí posílit tyhle 4 oblasti:

  

1. Operational excellence: SLO, observability, incident postmortems, chaos/DR
2. Security & governance by design: PII, přístupy, audit, policy-as-code
3. FinOps: nákladové modely, chargeback/showback, optimalizace cloud/Databricks/Snowflake apod.
4. Platform thinking: self-service, šablony, standardy, developer experience

  

  

Tohle jsou „multiplikátory“, které AI nezmění tak snadno.

  

  

Jaké pozice cílit (konkrétně pro SA)

  

  

- Senior/Lead Solution Architect (Cloud/Data Platform) – nejčastější sweet spot
- Principal/Staff Solution Architect – když chceš být autor standardů a mít governance mandát
- Cloud Security Architect nebo Security-focused SA – pokud chceš maximální stabilitu
- AI/ML Platform Architect (ne „prompt engineer“) – pokud chceš být u AI, ale s platformovou odpovědností

  

  

  

Praktický checklist: jak poznáš, že máš správný „SA level“

  

  

Jsi na správné trajektorii, pokud:

  

- umíš obhájit rozhodnutí přes trade-off (cena / výkon / riziko / delivery)
- máš vliv na landing zone, baseline security, CI/CD, IaC
- řešíš produkční provoz a ne jen návrh
- zavádíš standardy, které zrychlí více týmů

  

  

  

  

Pokud chceš, napiš jen 3 body: (1) AWS/Azure/GCP, (2) jestli řešíš spíš data platformu nebo aplikační integrace, (3) velikost firmy/týmů – a já ti doporučím nejvhodnější „typ SA“ + konkrétní skill gap a projekty, které si dát do portfolia.


Jsem data inženýr, ale byl jsem zaměřený na onprem sql databáze, bi nástroje a tvorbu reportingu. Mám zkušenosti s hadoopem a poté jsem to samé dělal v aws. Mám pocit, že mě diskvalifikuje slabší znalost pythonu, nezkušenost s moderními platformami databrics a snowflake, chybějící zkušenosti s realtime nebo fast daty a kontejnery. Znám devops a iac z pohledu data inženýra. Vychází mi z toho, že pro role zaměřené na platformu nemám dost zkušeností s SRE, pro data mám omezený toolset a na architekturu mi schází business language, stakeholder management a širší záběr

Tohle tě nediskvalifikuje. Máš solidní základ (SQL + BI + on-prem → Hadoop → AWS + IaC/DevOps mindset). To, co popisuješ, je typický „přechod do moderního data/cloud světa“ – a dá se udělat cíleně tak, aby ses za 6–12 měsíců prodal buď jako Senior Data Engineer (cloud), nebo jako Data Platform / Solution Architect (data).

  

Níže je nejkratší cesta, která ti maximalizuje šance a nevyžaduje „umět všechno“.

  

  

1) Co si vybrat jako cílový profil (2 realistické tracky)

  

  

  

Track A: Senior/Lead Data Engineer → Data Platform Engineer

  

  

Nejlepší, pokud chceš zůstat hands-on.

Tvoje výhoda: SQL, datové modelování, provozní zkušenost, AWS, IaC.

  

Co doplnit, aby to bylo „moderní“:

  

- Python „na úrovni DE“ (ne data science)
- streaming základy (Kafka/Kinesis) + CDC (Debezium/DMS) aspoň konceptuálně
- orchestrace a testing kvality dat
- observability/SLO pro pipeline (lehčí SRE)

  

  

  

Track B: Data/Cloud Solution Architect (data doména)

  

  

Nejlepší, pokud tě láká směr architektury, standardů a návrhů.

Tvoje výhoda: šířka technologií + zkušenost s migracemi a reportingem = business kontext.

  

Co doplnit:

  

- „reference architectures“ a trade-off rozhodování
- security/cost/operability argumentace
- stakeholder management a psaní (1–2 stránky místo 20 slidů)

  

  

Stabilita: obě jsou stabilní; nejvíc stabilní bývá platforma + security/cost mindset.

  

  

  

  

2) Tvoje „gapy“ – co je skutečně kritické vs. nice-to-have

  

  

  

Kritické (řeš jako první)

  

  

Python: nebudeš psát 10k řádků, ale musíš umět:

  

- číst/psát data (pandas, pyarrow), pracovat s API/SDK
- testy (pytest), packaging, struktura projektu
- jednoduché transformace, validace, parsování, práce se soubory  
    Tohle ti otevře 80 % DE rolí.

  

  

Moderní „table format“ a lakehouse principy

Nemusíš hned Databricks, ale musíš rozumět:

  

- partitioning, compaction, schema evolution
- ACID tabulky v datalake (Delta/Iceberg/Hudi) – koncepty

  

  

Orchestrace + kvalita dat

  

- Airflow/Dagster (stačí jedno), plus data quality (Great Expectations / Soda) konceptuálně a prakticky

  

  

  

Důležité, ale dá se dohnat rychle

  

  

Snowflake/Databricks: firmy často berou „cloud DE“ a tool naučí.

  

- Nauč se 1 z nich „do rozhovoru + mini projekt“ (viz níže)

  

  

  

Nice-to-have (neblokuje nástup)

  

  

Containers/Kubernetes: pro data role často stačí:

  

- Docker (build/run), základní znalost k8s (deploy, helm koncept)  
    Reálně se to dá dohnat až v práci.

  

  

Realtime/fast data: hodně rolí je stále batch. Stačí:

  

- rozumět rozdílu event-time vs processing-time, okna, at-least-once vs exactly-once
- mít jeden „mini streaming“ use case v portfoliu

  

  

  

  

  

3) Nejkratší upgrade plán (8–12 týdnů), který nejvíc zvedne employability

  

  

  

Týdny 1–3: Python pro data engineering

  

  

Cíl: cítit se komfortně na coding interview a v PR.

  

- 1 malý projekt: ingest z API → ukládání do Parquet → transformace → testy
- pytest, typing basics, logging, struktura repa

  

  

  

Týdny 4–6: Orchestrace + data quality

  

  

- jednoduchý Airflow/Dagster pipeline (lokálně)
- přidat data checks (nully, unikátnost, referenční integrita)
- přidat retry, idempotenci, basic observability (metriky/logy)

  

  

  

Týdny 7–9: Lakehouse / moderní storage

  

  

- Parquet + partitioning + incremental loads
- Delta/Iceberg (stačí jeden) – merge/upsert, schema evolution

  

  

  

Týdny 10–12: “Modern platform flavor”

  

  

Vyber si podle trhu kolem tebe:

  

- Snowflake: data modeling, tasks, streams, storage/compute separation
- Databricks: Spark + Delta + jobs/workflows  
    A udělej mini use case + cost/perf trade-off popis (1 strana).

  

  

Tím získáš: Python + orchestrace + lakehouse + „znám Snowflake/DBX“ = silný moderní profil.

  

  

  

  

4) Jak překlopit „chybí mi SRE“ do „mám provozní kvalitu“

  

  

Platform roles často nevyžadují hardcore SRE, ale SRE principy pro data:

  

- definovat SLO pro pipeline (freshness, completeness, latency)
- alerting na data incidents (nejen infra)
- backfills, incident playbook, postmortem kultura
- cost guardrails

  

  

Stačí, když o tom umíš mluvit a máš 1 ukázku v projektu (např. freshness check + alert).

  

  

  

  

5) Architektura: business language a stakeholder management – jak to dohnat rychle

  

  

Nejrychlejší způsob je změnit formu výstupu:

  

  

Nauč se „1-pager“ rozhodnutí (Architecture Decision Record / memo)

  

  

Struktura:

  

- Kontext (proč to řešíme, jaké jsou cíle)
- Varianty (A/B/C)
- Trade-offy (cena, riziko, time-to-market, provoz)
- Doporučení + next steps

  

  

Tohle je přesně „business language“, kterou chtějí SA role.

  

  

Nauč se mluvit v dopadech

  

  

Místo „použijeme X“ říkej:

  

- „zkrátíme time-to-insight o X“
- „snížíme riziko úniku PII díky…“
- „náklady budou předvídatelné, protože…“
- „operace zlevní, protože onboarding je self-service“

  

  

  

  

  

6) Jaké role cílit hned teď (reálně s tvým backgroundem)

  

  

- Senior Data Engineer (AWS / SQL-heavy / ETL/ELT) – s plánem na Python
- Analytics Engineer (pokud je ti blízké BI + modeling) – často dobrý mezikrok
- Cloud Data Engineer (batch, lakehouse) – streaming není povinný
- Data Platform Engineer (junior/mid) v týmu, kde tě vytáhnou (často stačí IaC + cloud + data základy)

  

  

A jako přechod k architektuře:

  

- Data Solutions Architect (delivery-oriented) – pokud budeš mít 1–2 reference arch návrhy + ADR styl

  

  

  

  

  

7) Co dát do CV/LinkedIn, aby to „vypadalo moderně“ i před upgradem toolsetu

  

  

Přepiš zkušenosti na výsledky + principy:

  

- “navrhl a provozoval datové pipeline se SLA/freshness”
- “automatizace nasazování (IaC), standardizace prostředí”
- “migrace on-prem → AWS (rizika, bezpečnost, náklady)”
- “optimalizace dotazů / nákladů / performance”  
    A přidej sekci „Modern data stack exposure“ (i kdyby z projektu): Airflow + Delta/Iceberg + dbt nebo GE.

  

  

  

  

Pokud chceš, napiš mi:

  

1. Jestli chceš spíš track A (platform/data eng) nebo track B (SA)
2. Který cloud je pro tebe prioritní (zůstáváš u AWS?)

  

  

A já ti navrhnu konkrétní „portfolio“ 2 projektů (takové, které skutečně fungují na pohovorech) + přesný seznam témat na rozhovory podle cílové role.