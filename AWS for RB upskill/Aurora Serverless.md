Aurora Serverless je z tohoto pohledu potřeba rozdělit na dvě věci, protože každá má jiné možnosti přístupu:

- **Serverless v2** žije ve tvojí VPC úplně stejně jako provisioned Aurora – takže platí klasické síťové vzorce.
- **Data API** je HTTP/IAM rozhraní nad databází, které úplně obchází síťovou vrstvu – a to je pro cross-account scénáře často nejčistší cesta.

Projdu všechny smysluplné možnosti od síťové vrstvy až po vlastní API.

## 1. Přímý síťový přístup (klasické DB spojení přes 5432/3306)

Tohle dává smysl, když konzument je tvoje vlastní služba/účet a chce nativní SQL spojení (connection pooling, transakce, kurzory).

**VPC Peering** – nejjednodušší pro 1:1 nebo malý počet účtů. Propeeruješ VPC, nastavíš routy a security groupy (lze referencovat SG napříč peeringem ve stejné regionu). Nelze tranzitivně řetězit a špatně se škáluje při mnoha účtech.

**Transit Gateway** – hub-and-spoke pro mnoho účtů/VPC. Tohle bych volil, jakmile máš víc než pár konzumentů nebo to roste. Stojí to víc (per-attachment + data processing), ale je to centrálně spravovatelné a typicky to sedí k landing-zone setupu.

**Shared VPC přes AWS RAM** – nasdílíš subnety z centrálního „networking" účtu a ostatní účty do nich nasazují své zdroje (Lambdy, ENI). Konzumenti pak jsou „uvnitř" stejné VPC jako databáze. Výborné, pokud máš org se striktní síťovou centralizací.

**PrivateLink (VPC Endpoint Service)** – nejvíc „SaaS-like" model: před Auroru postavíš NLB, vystavíš ji jako Endpoint Service a konzumenti si v cizím účtu vytvoří interface endpoint. Konzument vidí jen privátní endpoint, nikdy ne tvoji VPC. Háček: Aurora writer endpoint může při failoveru změnit IP, a NLB cílí na IP, takže potřebuješ mechanismus na aktualizaci targetů (typicky Lambda spouštěná z RDS eventů / health-check pattern). Je to nejelegantnější z hlediska izolace, ale provozně nejnáročnější.

## 2. Data API – serverless-native cross-account přístup

Tohle bych pro tvůj serverless-first kontext zvážil jako první. Data API dnes funguje i pro **Serverless v2 a provisioned Auroru** (Postgres i MySQL), nejen pro starou v1.

Princip: žádné perzistentní spojení, žádná VPC, žádný NAT/ENI – je to čistě HTTPS volání autentizované přes **IAM**, credentials se tahají ze Secrets Manageru. Cross-account se tím pádem řeší triviálně: v účtu s databází vytvoříš IAM roli s povolením `rds-data:*` na konkrétní cluster, druhý účet ji `sts:AssumeRole` a volá Data API.

Výhody: nulová síťová konfigurace mezi účty, perfektní pro Lambdy (žádný cold-start penalty z VPC ENI), audit přes CloudTrail. Limity, na které narazíš: vyšší latence než přímé spojení, omezení velikosti výsledku (řádově ~1 MB / počet řádků), není to ideální pro vysokopropustné OLTP nebo streamování velkých výsledků. Pro BFF enrichment endpointy s rozumnou velikostí odpovědi je to ale často přesně ono.

## 3. Vlastní API vrstva (doporučený vzor pro „více uživatelů")

Pokud konzumenti nejsou tvoje infrastruktura, ale různí klienti/týmy, skoro vždy nechceš dávat přímý DB přístup – chceš mezi to dát API, které vlastní doménovou logiku a řeší multi-tenancy.

**API Gateway + Lambda** – Lambda buď ve VPC (přímé spojení, ideálně přes **RDS Proxy** kvůli poolování spojení u serverless), nebo mimo VPC přes Data API. Autorizace cross-account/cross-tenant přes IAM (SigV4), Cognito, nebo Lambda authorizer. Tohle je nejflexibilnější a sedí na tvůj BFF kontext.

**AppSync (GraphQL)** – pokud konzumenti chtějí GraphQL; má nativní resolvery nad RDS Data API.

**ECS/Fargate služba** – když potřebuješ dlouhotrvající spojení, vyšší propustnost nebo netriviální framework, který se do Lambdy špatně vejde.

Klíčová výhoda tohoto vzoru: databáze zůstane plně privátní v jednom účtu a sdílíš jen kontrakt API. Schéma databáze i ownership stavu (což je téma, které u tebe v projektu stejně řešíš) zůstává čistě tvoje.

## 4. Oddělené sdílení dat (bez přístupu k živé DB)

Pokud konzumenti potřebují data, ne živý transakční přístup:

- **Export do S3** + sdílení bucketu přes RAM / cross-account bucket policy (případně přes Athenu/Lake Formation pro řízený přístup po sloupcích/řádcích).
- **Zero-ETL do Redshiftu** a sdílení přes Redshift datashare – pokud konzumenti jsou analytičtí.

Tohle úplně rozpojí transakční a konzumní zátěž, což chrání tvůj hot path.

## 5. Podpůrné kameny

Ať zvolíš cokoli s credentials, **Secrets Manager** umí resource policy pro cross-account čtení tajemství, plus rotaci. A **IAM database authentication** ti umožní generovat krátkodobé tokeny místo statických hesel, opět assume-role napříč účty.

---

Pro tvůj profil bych rozhodovací strom zjednodušil takto: konzument je tvoje vlastní serverless služba v jiném účtu → **Data API + assume role** (nejmíň pohyblivých dílů). Konzument je externí tým/klient → **vlastní API přes API Gateway** (databáze zůstává privátní). Potřebuješ nativní vysokopropustné SQL spojení napříč mnoha účty → **Transit Gateway nebo PrivateLink**.

Chceš, abych některou z těchto variant rozkreslil do diagramu nebo abychom prošli konkrétní access patterny tvého případu a vybrali kombinaci? Pomohlo by vědět, jestli „uživatelé" znamenají tvoje služby v jiných účtech, nebo externí konzumenty mimo tvoji organizaci.


Ano, reader endpoint použít můžeš a pro read-only s nízkou latencí je to ten správný stavební kámen – jen je potřeba si ujasnit, co reader endpoint vlastně řeší a co ne.

## Co reader endpoint dělá (a co ne)

Aurora **reader endpoint** je jedno DNS jméno (`*.cluster-ro-*.rds.amazonaws.com`), které load-balancuje příchozí spojení napříč všemi reader instancemi v clusteru. Load balancing je na úrovni spojení – při novém connectu se rozhodne, na kterou repliku tě pošle, a po dobu spojení tam zůstaneš.

Klíčová věc pro tvůj cíl: Aurora repliky nečtou přes binlog jako klasické RDS, ale ze **sdílené storage vrstvy**. Replication lag je proto typicky jednotky až nízké desítky milisekund, často pod 20 ms. To je přesně ten důvod, proč je reader endpoint dobrý základ pro read-only s minimální latencí – nečekáš na pomalou logickou replikaci.

Co reader endpoint **neřeší**: cross-account konektivitu. Je to pořád jen DNS jméno mířící na privátní IP v VPC databáze. Sám o sobě ti přístup z jiného účtu nedá.

## Cross-account mechanika

Dobrá zpráva: endpointy Aurory jsou **veřejně resolvovatelná DNS jména, která vrací privátní IP**. To znamená, že z cizího účtu endpoint vyresolvuješ bez problému – **nemusíš sdílet Route 53 private hosted zone**. Stačí ti tedy zajistit jen síťovou cestu k té privátní IP:

- **VPC Peering / Transit Gateway** – tohle je pro tvůj případ ideální, protože zachová nativní load balancing reader endpointu. Konzument se připojí přímo na reader endpoint, DNS ho rozhodí mezi repliky, routing jde přes peering/TGW. Přidaná latence v rámci regionu je zanedbatelná (TGW přidá řádově mikrosekundy až nízké jednotky ms).

Důležité: **cross-account read replica v pravém smyslu (replika vlastněná jiným účtem) neexistuje.** Repliky jsou členy clusteru a cluster žije v jednom účtu. Cross-account znamená vždy *přístup* k reader endpointu, ne *vlastnictví* repliky v cizím účtu.

## Latency gotchy, na které narazíš

**Cross-AZ traffic** je hlavní skrytá daň. Reader endpoint tě round-robinem může poslat na repliku v jiné AZ, než je tvůj konzument – to je ~1–2 ms navíc na round-trip plus cross-AZ data transfer cost. Pokud chceš opravdu minimální latenci, máš dvě cesty:

- nechat to být a spolehnout se na to, že 1–2 ms je v pohodě (většinou ano), nebo
- použít **instance endpoints** a připnout se na konkrétní repliku ve stejné AZ jako konzument. Tím ale ztratíš load balancing i HA (když ta instance spadne, endpoint přestane fungovat), takže si to musíš ošéfovat sám.

**PrivateLink se sem hodí špatně.** Kdybys chtěl reader endpoint vystavit přes NLB + Endpoint Service, rozbiješ nativní DNS load balancing (NLB cílí na IP, ne na DNS) a navíc se potýkáš se změnami IP replik. Pro read-only fanout je peering/TGW jednoznačně čistší.

## Doplňky podle scénáře

**Serverless konzumenti (Lambda):** dej před repliky **RDS Proxy s read-only endpointem**. Vyřeší connection pooling (Lambdy jinak repliky zahltí spojeními), endpoint je stabilnější a latenci přidá minimálně.

**Konzumenti v jiných regionech:** tady reader endpoint v původním regionu nestačí – platil bys cross-region RTT (desítky až 100+ ms). Řešení je **Aurora Global Database**: sekundární region má vlastní reader endpoint, replikace pod 1 s, a konzument čte lokálně. Cross-account + cross-region pak pořád potřebuje peering/TGW v tom sekundárním regionu.

**Když potřebuješ sub-ms:** žádná replika ti nedá to, co cache. Pro horké read-only patterny zvaž **ElastiCache (Redis)** před Auroru – cache hit je v řádu mikrosekund až ~1 ms. Dává smysl, pokud máš opakované čtení stejných dat.

## Vynucení read-only

Defense in depth: reader endpoint sám tě pustí jen na repliky (které jsou read-only), ale navrch bych konzumentům dal DB uživatele jen s `SELECT` granty, ideálně přes **IAM database authentication** (krátkodobé tokeny, assume-role z cizího účtu) místo statických hesel.

---

Shrnuto pro „cross-account, read-only, min. latency": **Transit Gateway (nebo peering u malého počtu účtů) + reader endpoint + RDS Proxy read-only endpoint pro serverless konzumenty**, s pohlídáním cross-AZ. Pro geograficky rozprostřené konzumenty navrch Aurora Global Database.

Chceš to rozkreslit do diagramu s tokem spojení napříč účty? A pomohlo by mi vědět, jestli konzumenti sedí ve stejném regionu, nebo je potřeba řešit i cross-region – to dost mění doporučení.


--------- GPT -----------

Pokud chceš zpřístupnit Amazon Aurora Serverless více uživatelům, aplikacím nebo celým AWS účtům, existuje několik architektonických možností. Liší se podle:

* kdo databázi používá (lidé vs aplikace),
* zda jde o read/write nebo read-only přístup,
* bezpečnosti,
* škálovatelnosti,
* governance,
* síťové topologie,
* a zda chceš vystavit SQL přístup nebo pouze business API.

Níže jsou prakticky všechny běžné varianty používané v enterprise AWS architekturách.

⸻

1. Přímé DB připojení přes síť (cross-account access)

Nejklasičtější model.

Jak funguje

Aurora běží v privátní VPC v Account A.
Aplikace nebo uživatelé v jiných účtech se připojují:

* přes VPC peering,
* Transit Gateway,
* PrivateLink,
* VPN,
* Direct Connect,
* případně veřejný endpoint (nedoporučeno).

⸻

Varianta A — VPC Peering

Architektura

Account A
  VPC-A
    Aurora Serverless
      ||
VPC Peering
      ||
Account B
  VPC-B
    ECS/Lambda/EC2

Výhody

* jednoduché,
* nízká latence,
* nativní SQL access,
* funguje pro PostgreSQL/MySQL protokol.

Nevýhody

* mesh complexity,
* routing management,
* CIDR konflikty,
* špatně škáluje na mnoho účtů.

Typické použití

* několik interních AWS účtů,
* shared services model.

⸻

Varianta B — AWS Transit Gateway (enterprise best practice)

AWS Transit Gateway

Architektura

                Transit Gateway
               /      |       \
              /       |        \
         AccountA  AccountB  AccountC
             |
         Aurora

Výhody

* centrální networking,
* dobře škáluje,
* multi-account governance,
* vhodné pro landing zone.

Nevýhody

* vyšší cena,
* složitější governance.

Doporučeno pro

* enterprise,
* více týmů,
* desítky/stovky účtů.

⸻

Varianta C — AWS PrivateLink

AWS PrivateLink

Velmi zajímavá varianta.

ALE:
Aurora sama o sobě nejde přímo publikovat přes PrivateLink.

Musíš vytvořit:

Client -> Interface Endpoint -> NLB -> Proxy -> Aurora

Typicky:

* NLB
* RDS Proxy
* ECS proxy
* custom TCP proxy

Výhody

* žádné peering routy,
* izolace sítí,
* provider/consumer model,
* SaaS pattern.

Nevýhody

* složitější,
* Aurora neumí native PrivateLink endpoint.

Typické použití

* multi-tenant SaaS,
* externí zákazníci,
* secure shared database service.

⸻

2. Přístup přes RDS Proxy (doporučeno)

Amazon RDS Proxy

Velmi důležité u Aurora Serverless.

Proč

Aurora Serverless:

* autoscaluje,
* může měnit connections,
* má connection limits,
* cold starts,
* Lambda může vytvořit connection storm.

RDS Proxy řeší:

* pooling,
* reconnect,
* IAM auth,
* failover.

⸻

Cross-account přístup

RDS Proxy může být:

* v samostatném shared services účtu,
* nebo vystaven přes networking.

⸻

3. IAM Database Authentication

AWS Identity and Access Management

Místo DB hesel používáš IAM tokeny.

Jak funguje

Aplikace:

aws rds generate-db-auth-token

Token platí cca 15 minut.

Aurora ověřuje IAM roli.

⸻

Cross-account model

Account B role

Role in Account B
   ->
AssumeRole
   ->
Role in Account A
   ->
RDS IAM auth

⸻

Výhody

* žádná statická hesla,
* centralizovaná IAM governance,
* audit,
* krátkodobé credentials.

Nevýhody

* některé nástroje neumí IAM auth,
* komplikovanější pro BI tools.

⸻

4. Secrets Manager + klasické DB users

AWS Secrets Manager

Klasický model.

Cross-account sdílení secretu

Secret může být sdílen:

* resource policy,
* RAM,
* AssumeRole.

⸻

Výhody

* kompatibilní se vším,
* jednoduché.

Nevýhody

* rotace,
* stále existují hesla.

⸻

5. Přístup přes vlastní API (doporučený moderní pattern)

Tohle bývá architektonicky nejlepší.

Místo SQL přístupu vystavíš business API.

⸻

Architektura

Consumer
   ->
API Gateway
   ->
Lambda/ECS
   ->
Aurora

Možné komponenty:

* Amazon API Gateway
* AWS Lambda
* Amazon ECS
* GraphQL
* REST
* gRPC

⸻

Výhody

Bezpečnost

Nikdo nevidí DB.

Governance

* validace,
* throttling,
* audit,
* business rules,
* row-level security.

Škálování

DB není přímo exponovaná.

Multi-account

Snadné.

Multi-region

Snadnější evoluce.

⸻

Nevýhody

* vyšší latency,
* více vývoje.

⸻

6. Data API (Aurora Serverless specific)

Amazon RDS Data API

Velmi zajímavá možnost.

Aurora Serverless může být přístupná přes HTTPS API.

Bez persistentních DB connections.

⸻

Architektura

Client
   ->
AWS SDK
   ->
RDS Data API
   ->
Aurora

⸻

Výhody

* žádné connections,
* funguje z Lambda,
* IAM auth,
* cross-account možné,
* serverless-native.

⸻

Nevýhody

* vyšší latency,
* není vhodné pro heavy OLTP,
* omezení SQL features,
* není vhodné pro BI.

⸻

Typické použití

* Lambda,
* event-driven workloads,
* admin operations,
* low/medium throughput.

⸻

7. Read replicas / shared read access

Pokud chceš:

* více consumerů,
* reporting,
* analytiku,
* BI,

můžeš oddělit workload.

⸻

Varianty

Aurora Reader Endpoint

writer endpoint
reader endpoint

Cross-region replica

Logical replication (PostgreSQL)

CDC do jiných DB

Např:

* DMS,
* Kafka,
* Redshift,
* OpenSearch,
* Snowflake.

⸻

8. Database-per-tenant nebo schema-per-tenant

Pokud řešíš multi-account/multi-customer SaaS.

⸻

Možnosti

Shared schema

Nejlevnější.

Schema-per-tenant

Dobrý kompromis.

DB-per-tenant

Nejlepší izolace.

Aurora Serverless v2 to dnes umožňuje relativně dobře.

⸻

9. Zero Trust / enterprise access pattern

Moderní enterprise pattern:

User/App
   ->
IAM Identity Center
   ->
Private API
   ->
Service Layer
   ->
Aurora

NE:

User -> direct SQL

⸻

10. Hybrid access

On-premise:

* VPN,
* Direct Connect,
* Transit Gateway,
* Route53 Resolver.

⸻

Co je dnes nejčastější best practice?

Pro aplikace

Doporučeno

API Gateway
   ->
Lambda/ECS
   ->
RDS Proxy
   ->
Aurora

s:

* IAM auth,
* Secrets Manager,
* private subnets.

⸻

Pro interní microservices

Transit Gateway
+
RDS Proxy
+
IAM auth

⸻

Pro externí consumer služby

PrivateLink
+
API layer

⸻

Co bych téměř nikdy nedoporučil

Public Aurora endpoint

0.0.0.0/0

To je téměř vždy špatně.

⸻

Důležité limity Aurora Serverless

Aurora Serverless není ideální pro:

* tisíce persistent connections,
* BI workloady,
* velké ETL,
* connection-heavy workloads.

Proto je často kritické použít:

* RDS Proxy,
* Data API,
* service layer.

⸻

Doporučené architektury podle use-case

Use-case	                        Doporučení
Interní app v jiném účtu	        TGW + RDS Proxy
Lambda workload	                    Data API nebo RDS Proxy
Externí zákazníci	                API Gateway + service layer
BI/reporting	                    Read replica
SaaS multi-tenant	                API + schema-per-tenant
Enterprise landing zone	            Shared DB account + TGW
Zero trust	                        IAM + API layer
High scale microservices	        RDS Proxy + Aurora v2

⸻

Co je architektonicky nejčistší?

Ve většině moderních enterprise systémů:

Aurora není přímo sdílena.

Místo toho:

Consumers
   ->
API/service layer
   ->
Aurora

Protože:

* DB schema není contract,
* lze měnit interní model,
* governance,
* security,
* throttling,
* observability,
* audit,
* tenant isolation,
* caching,
* CQRS,
* event sourcing,
* rate limiting.

Tohle je dnes dominantní cloud-native pattern.