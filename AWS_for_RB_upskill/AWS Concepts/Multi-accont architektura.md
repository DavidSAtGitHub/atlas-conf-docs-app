
# Multi-account architektura na AWS — detailní pohled

Ano, chápeš to správně — a je to jeden z klíčových konceptů, který lidé zpočátku nechápou. Pojďme to rozebrat od základů.

---

## 1. Proč vůbec víc účtů

AWS účet je **nejsilnější izolační hranice**, kterou AWS nabízí. Silnější než VPC, silnější než IAM v jednom účtu. Důvody, proč rozdělovat:

- **Blast radius** — chyba/kompromitace v jednom účtu se nepřelije do ostatních. IAM role v účtu A nemůže ze své podstaty nic v účtu B, dokud to někdo explicitně nepovolí.
- **Billing a cost allocation** — každý účet má vlastní fakturu. Okamžitě vidíš, co stojí dev vs. prod, nebo jaký tým kolik utrácí.
- **Service Quotas (limity)** — limity jako počet Lambda concurrent executions, VPC, EIP jsou **per účet**. Když prod a dev sdílí účet, dev test ti může vyčerpat prod limit.
- **Compliance a audit** — oddělení produkce od všeho ostatního je často regulatorní požadavek (PCI, HIPAA, ISO).
- **Čistý IAM** — místo složitých ABAC politik s tagy "kdo smí do dev ale ne do prod" prostě dáš roli jen v dev účtu.
- **Rate limiting na API úrovni** — API rate limity (např. EC2 API) jsou per účet; oddělení tě chrání před throttlingem.

---

## 2. Na čem to stojí — Landing Zone a AWS well-architected

Reference architektura se jmenuje **AWS Landing Zone** a je popsaná v dokumentu **"Organizing Your AWS Environment Using Multiple Accounts"** (AWS whitepaper). AWS k tomu nabízí managed službu **Control Tower**, která ji postaví za tebe. Klíčové stavební bloky:

- **AWS Organizations** — "meta" služba, která drží všechny účty pod jednou střechou, umožňuje consolidated billing a centrální policies
- **Organizational Units (OUs)** — hierarchické skupiny účtů, na které aplikuješ policies. Typicky: `Security OU`, `Infrastructure OU`, `Workloads OU` (s pod-OU `Prod`, `Non-Prod`), `Sandbox OU`, `Suspended OU`
- **Service Control Policies (SCPs)** — guardrails na úrovni OU nebo účtu. SCP neudělují oprávnění, jen _omezují_, co jde v účtu udělat. Příklad: "v žádném účtu nesmí nikdo vypnout CloudTrail", "v Sandbox OU nesmí nikdo používat jiný region než eu-central-1".
- **IAM Identity Center (dříve AWS SSO)** — centrální autentikace, jeden login pro všechny účty s různými rolemi
- **CloudTrail organization trail** — centrální audit log ze všech účtů do jednoho S3 bucketu
- **Config aggregator** — centrální view compliance napříč účty

---

## 3. Role jednotlivých účtů (reference model)

Tohle je **AWS doporučená topologie**, ne jediná možná. Rozepíšu kanonickou verzi, v praxi se kombinuje podle velikosti firmy.

### Management (Organizations root) account

- Drží AWS Organizations, SCPs, billing, IAM Identity Center
- **Nic jiného tu neběží** — žádné workloady, žádné data. Je to "hlava" organizace.
- Přístup má jen velmi úzká skupina lidí (typicky 2–3 senior engineers)
- Kompromitace = game over pro celou org, proto extra ochrana (MFA, žádné long-lived keys)

### Log Archive account

- Centrální úložiště immutable logů: CloudTrail, Config, VPC Flow Logs, ALB logs ze všech účtů
- S3 buckety s Object Lock (WORM — write once read many), takže ani admin je nesmaže
- Typicky read-only i pro security tým — čte se přes separátní Audit účet

### Audit / Security account (někdy Security Tooling)

- **Centrální nástroje pro bezpečnost**: GuardDuty delegated admin, Security Hub, Detective, Macie, Inspector
- Security tým sem přistupuje a odsud vidí stav **všech účtů** naráz
- Cross-account read role do ostatních účtů pro incident response
- Často **odděleno** od Log Archive — principle of least privilege (ten kdo čte logy ≠ ten kdo je ukládá)

### Network / Networking account

- **Centrální síťová infrastruktura**: Transit Gateway, Direct Connect, VPN, Route53 resolver, centrální VPC endpoints, Network Firewall
- **Ano, slouží pro všechna prostředí** (dev, uat, prod) — o tom dál
- Network tým jediný má write přístup
- Centralizace snižuje náklady (Transit Gateway, NAT, VPC Endpoints jsou drahé komponenty, nechceš je mít v každém účtu)

### Shared Services account

- Sdílené aplikační zdroje: centrální ECR, Artifact repository, centrální CI/CD runners, Route53 private hosted zones, AD/Directory Services, centrální monitoring nástroje (Grafana, Prometheus pokud běží self-hosted)
- Rozdíl od Network: Network = "síťové dráty", Shared Services = "aplikační patra sdílená mezi workloady"

### Workload accounts (Dev, UAT, Prod)

- Tady teprve běží **tvoje aplikace**
- **Jeden účet per prostředí per workload** je ideál. Pro větší firmy: `team-a-dev`, `team-a-prod`, `team-b-dev`, `team-b-prod`. Pro menší: jen `dev`, `uat`, `prod` sdílené všemi týmy.
- Workload účty **konzumují** zdroje z Network a Shared Services

### Sandbox accounts

- "Hřiště" pro vývojáře, oddělené od všeho důležitého
- SCP zakazují drahé věci, auto-cleanup po X dnech
- Často jeden per vývojář

---

## 4. Jak funguje "jeden Network účet pro všechna prostředí"

Tohle je jádro tvé otázky. **Ano, jeden Network účet typicky obsluhuje dev i prod**, a to přes několik mechanismů:

### Transit Gateway sharing přes RAM (Resource Access Manager)

- V Network účtu postavíš **Transit Gateway (TGW)** — je to L3 router, který propojuje VPC
- Přes **AWS RAM** TGW _nasdílíš_ do Dev, UAT, Prod účtů
- Každý workload účet má vlastní VPC a **připojí se k centrální TGW**
- TGW route tables řeší, kdo s kým může mluvit:
    - Typicky: **Prod VPC ↔ Shared Services ano**, **Prod ↔ Dev NE**, **Dev ↔ Dev ano**
    - Izolace prostředí se tedy řeší **routingem v TGW**, ne tím, že by každé prostředí mělo vlastní TGW

### Centralizované VPC Endpoints

- Interface endpoints (pro S3, DynamoDB, Secrets Manager atd.) stojí ~7 USD/měsíc každý **per AZ per VPC**
- Pokud máš 10 VPC, 3 AZ, 5 endpointů → 1050 USD/měsíc
- Řešení: postavíš endpointy **jednou v Network účtu**, nasdílíš přes Private Hosted Zone + TGW routing. Ušetří to stovky dolarů.

### Centralizovaný egress (NAT Gateway)

- NAT Gateway stojí 32 USD/měsíc + data processing
- Místo NAT GW v každém workload účtu → **jeden centrální egress VPC** v Network účtu, traffic z workload VPC jde přes TGW do egress VPC ven
- **Tradeoff:** cross-AZ data transfer poplatky mohou někdy převážit úspory — je potřeba spočítat

### Centralizovaná DNS (Route53 Resolver)

- Route53 Resolver endpoints v Network účtu, private hosted zones nasdílené přes RAM
- Všechny účty řeší jména konzistentně

### Direct Connect / VPN

- Terminuje v Network účtu, všechny ostatní účty k němu přistupují přes TGW
- Dává ekonomický smysl i bezpečnostní (jedno místo kontroly)

**Takže mechanicky:**

1. **AWS Organizations** spojuje účty do jedné org
2. **RAM** umožňuje sdílet konkrétní zdroje (TGW, subnets, endpointy) napříč účty
3. **TGW** propojuje sítě na L3
4. **Route53 private zones** sdílí DNS
5. **IAM cross-account roles** umožňují operátorům pracovat napříč účty

---

## 5. Tradeoffs — kde to bolí

Nic není zadarmo. Konkrétní bolesti multi-account:

### Komplexita

- **IaC je složitější** — deploy musí rozumět tomu, v jakém účtu co běží. CDK má `Environment`, Terraform `providers` s `assume_role`. Cross-account stacky jsou pain.
- **CI/CD musí umět assume role** do cílového účtu (OIDC tohle hodně zjednodušil, ale stále je to další vrstva).
- **Onboarding nových inženýrů** — "proč musím přepínat role, proč tohle nevidím"

### Debugging napříč účty

- Request jde přes 3 účty (public BFF v Workload → TGW v Network → DB v Shared Services). Trace musí fungovat cross-account, logy musí být centralizované, jinak strávíš hodiny hledáním.
- **X-Ray a CloudWatch cross-account observability** to řeší, ale musíš to nastavit.

### Cena

- **Některé věci zdražují, některé zlevňují.** Centralizace VPC endpointů ušetří, ale cross-account/cross-AZ data transfer přes TGW stojí (0.02 USD/GB) a může překvapit.
- **TGW attachment** sám o sobě stojí ~36 USD/měsíc per VPC attachment. Pro 10 VPC to je 360 USD/měsíc jen za attachmenty.
- Pro malý projekt (jako ten tvůj OrderFlow) může být centralizace _dražší_ než decentralizace.

### Service limity a quoty

- **Počet účtů v Organizations** má defaultní limit (typicky 10, navyšuje se ticketem)
- **RAM sdílení** má limity na počet principals
- **TGW** má limity na počet attachmentů, routes

### Governance overhead

- Kdo schvaluje nový účet? Jak rychle se dá vytvořit? Pokud to trvá týdny, týmy začnou obcházet systém.
- **Control Tower Account Factory** nebo **AFT (Account Factory for Terraform)** tohle automatizuje, ale samy o sobě jsou projekt na zavedení.

### Dependency management

- Když Network tým deployne změnu v TGW routing, může rozbít Workload účty. Potřebuješ **change management proces** a **pre-prod network účet** na testování.

---

## 6. Různé setupy v praxi

Nechci psát "best practice" jako jedinou pravdu — realita je spektrum.

### Setup A: Minimal (startup, solo projekt, learning)

- **2 účty**: Management, Workloads (dev+prod v jednom účtu oddělené přes VPC a tagy)
- Žádný Network/Security účet
- **Kdy:** hobby projekt, MVP, 1–3 vývojáři
- **Proč:** overhead multi-account > přínos
- **Riziko:** při růstu bolestivá migrace

### Setup B: Small team (10–30 lidí)

- **4 účty**: Management, Shared Services (= Network + Logs + CI/CD), Dev, Prod
- Jedna VPC per účet, bez TGW (VPC peering pokud potřeba)
- **Kdy:** SaaS startup po seed
- **Proč:** oddělení prod od dev je must, ale plná Landing Zone je overkill

### Setup C: Mid-size (Landing Zone "lite")

- **6–8 účtů**: Management, Log Archive, Audit, Network, Shared Services, Dev, UAT, Prod
- Control Tower nasazený
- TGW sdílená přes RAM
- **Kdy:** firma s 50–200 inženýry, nějaká regulace
- **Proč:** zralá organizace, security tým existuje, compliance se řeší

### Setup D: Enterprise (full Landing Zone)

- **Desítky až stovky účtů**: výše uvedené + per-team nebo per-workload workload účty, per-developer sandboxy, samostatné účty pro SDLC (Build, Test, Staging, Prod), dedikované účty pro data (Data Lake, Analytics)
- **OU hierarchie** několik úrovní hluboko
- Account vending machine (AFT / Control Tower)
- **Kdy:** enterprise, banky, regulated industries, velké tech firmy
- **Proč:** blast radius, compliance, oddělení týmů, acquisitions integration

### Setup E: Per-customer (SaaS single-tenant)

- **Účet per zákazník** — extrémní izolace pro enterprise B2B SaaS
- Stovky až tisíce účtů, plně automatizovaný vending
- **Kdy:** healthcare SaaS, gov cloud, finance
- **Proč:** zákazník dostává svůj účet, vlastní billing, vlastní compliance perimeter

---

## 7. Doporučení pro tvůj OrderFlow projekt

Pro tvůj učící projekt dává smysl **Setup B** nebo **hybrid mezi A a B**:

- **Management account** (Organizations root, IAM Identity Center)
- **Shared Services account** (ECR, Terraform state, CloudTrail archive bucket, GitHub OIDC provider pokud chceš centrálně)
- **Dev account**
- **Prod account**

**Network account vynech** — pro projekt s pár Lambdami a jednou Aurorou nemá smysl TGW. V každém workload účtu postav jednoduchou VPC přímo. Až bys měl 3+ workloady, které se potřebují propojit, přidáš Network účet.

**Security account také vynech** — GuardDuty a Security Hub zapneš později v Management účtu jako delegated admin.

Tohle ti dá:

- Reálný multi-account pocit (cross-account deploy přes OIDC, SCPs, SSO login)
- Bez zbytečné ceny a složitosti
- Cestu pro růst — přidat Network/Security účet později je přímočaré

---

Chceš teď rozpracovat konkrétně **jak se v tomhle setupu fyzicky vytvoří účty a propojí** (Organizations, Identity Center, OIDC pro GitHub), nebo **jak vypadá IaC struktura** (CDK/Terraform rozvržení pro multi-account), nebo **jak se řeší sdílené síťové zdroje**, kdybys přece jen Network účet chtěl?


# Multi-account v bance bez AWS Organizations

Tahle situace je v enterprise bankingu překvapivě častá. Účty jsou typicky poskytované **centrální cloud platformou / FinOps týmem / Cloud Center of Excellence (CCoE)**, ty jako tým/aplikace dostaneš "hotové" účty s předpřipravenými guardrails, ale **nemáš přístup do Organizations root**, nevidíš OU strukturu, nemůžeš měnit SCPs. Pojďme to rozebrat.

---

## 1. Co to pro tebe znamená prakticky

**Co ztrácíš:**

- Nevidíš Organizations API → nemůžeš programaticky vytvářet účty, listovat OU, číst SCPs
- Nemůžeš nastavit SCPs ani delegated admin pro security služby
- Nemůžeš použít **AWS RAM sharing "přes Organization"** (tj. auto-share všem v org) — musíš sdílet explicitně přes account ID
- Nemůžeš použít fíčury typu "CloudTrail organization trail", "Config aggregator for organization", "GuardDuty organization-wide" — tyhle centrálně vlastní **platform tým**
- Nevidíš **consolidated billing** — dostaneš chargeback report, ne AWS konzoli s billingem

**Co ti typicky platform tým poskytne:**

- Účty s předinstalovanými **baseline** — CloudTrail už loguje do jejich central log archive, Config rules běží, GuardDuty je zapnutý, IAM Identity Center je nakonfigurovaný
- **Permission boundary** — IAM policy, kterou musí mít připnutou každá role/user, kterou v účtu vytvoříš. Omezuje maximum oprávnění, i kdybys roli dal `AdministratorAccess`.
- **SCP guardrails** — zákazy typu "nesmíš vypnout CloudTrail", "jen eu-central-1 a eu-west-1", "žádné public S3 buckety", "jen approved AMI"
- **Předpřipravenou VPC** nebo **VPC blueprint** — často už připojenou k centrální TGW, s definovanými CIDR z IP Address Management (IPAM) systému
- **Egress přes centrální proxy/firewall** — veškerý odchozí traffic jde přes Palo Alto / Zscaler / AWS Network Firewall v Network účtu, který nevlastníš
- **Approved services katalog** — smíš používat jen podmnožinu AWS služeb (často přes **Service Catalog** s předschválenými produkty)
- **Předpřipravené IAM role** pro cross-account operace (break-glass, audit, deployment)

---

## 2. Jak je to architektonicky postavené

Banka obvykle má **platformní tým**, který vlastní Organizations a provozuje "AWS jako interní službu". Typická struktura:

### Vrstva 1: Platform-owned (ty tam nemáš přístup)

- Management account (Organizations root)
- Log Archive, Audit, Security Tooling
- Network account (TGW, Direct Connect do on-prem datacentra, centrální firewall, DNS)
- Shared Services (centrální ECR, artifact repo, CI/CD runners pokud jsou sdílené)
- **Identity account** — často vlastní AWS instance propojená s bankovním **Active Directory / Azure AD / PingFederate**

### Vrstva 2: Tenant accounts (tam žiješ ty)

- Dostaneš sadu účtů per aplikace per prostředí: `app-orderflow-dev`, `app-orderflow-uat`, `app-orderflow-prod`
- Někdy i `app-orderflow-dr` (disaster recovery v jiném regionu)
- Účty jsou ti **přiřazené**, ale patří do banky

### Vrstva 3: Governance mimo AWS

- **ServiceNow / Jira** pro žádosti o nové účty, FW pravidla, nové služby
- **CMDB** (Configuration Management Database), kde jsou účty evidované
- **Schvalovací workflow** (architektura, security, compliance, data privacy)
- **Change Advisory Board (CAB)** pro prod změny

---

## 3. Best practices, které se v téhle situaci používají

### Landing Zone Accelerator (LZA) nebo Control Tower

- Platformní tým typicky používá **AWS Landing Zone Accelerator** (open-source AWS řešení pro regulated industries) nebo **Control Tower** s custom customizations (CfCT)
- LZA je populární v bankách, protože má out-of-box podporu pro NIST, PCI-DSS, HIPAA compliance frameworks
- **Pro tebe jako tenanta** to znamená: dostaneš účet s určitou úrovní "hygieny" a musíš v ní žít

### Permission Boundaries jsou kritické

- I když ti dají `AdministratorAccess` role, **v praxi nemůžeš dělat všechno** — permission boundary omezuje i admina
- Typické restrikce: nesmíš vytvořit IAM user (jen role), nesmíš roli bez boundary, nesmíš měnit CloudTrail, nesmíš dělat IAM na určité service prefixy
- **Musíš to respektovat v IaC** — každá role, kterou tvoříš, potřebuje mít připnutý boundary ARN

### Service Control Policies (SCPs) jsou nad tebou

- SCPs vidíš jen jako "effective denial" — něco prostě nejde a dostaneš `AccessDenied`, i když IAM politika to povoluje
- **Preventivní přístup:** ptej se platform týmu na **seznam aktivních SCPs** nebo alespoň na matici toho, co je zakázané
- Časté zákazy: root user login, regions mimo EU, určité služby (třeba Bedrock v US regionu kvůli data residency)

### Service Catalog pro provisioning

- Místo volného přístupu k AWS API máš často **AWS Service Catalog** s "approved products" — třeba "RDS Postgres s našimi defaults", "Lambda s našimi logs/tracing"
- Pro tebe to může znamenat: **neprovisionuješ RDS přes CDK/Terraform přímo**, ale přes Service Catalog product, který už má všechny hardening defaults
- Tradeoff: míň flexibility, víc bezpečí a compliance

### IaC omezení

- Často nesmíš používat `aws_iam_account_password_policy`, `aws_organizations_*`, `aws_cloudtrail` (už běží), `aws_config_*` (už běží)
- Musíš v Terraform/CDK vědět, co je "platform-managed" a nesahat na to
- Platform tým typicky poskytuje **Terraform moduly** nebo **CDK construct library** s approved patterny

### CI/CD bez Organizations

- GitHub Actions OIDC ti **pořád funguje** — OIDC federation je per account, nepotřebuje Organizations
- Často ale nesmíš do GitHubu.com → musíš používat **GitHub Enterprise Server on-prem** nebo **self-hosted runners** v bankovní síti
- Cross-account deploy z CI/CD do tvých dev/uat/prod účtů je na tobě (standardní assume role chain)

### Centrální networking je fait accompli

- VPC máš "dodanou" s předefinovanými CIDR bloky (dostaneš je z IPAM systému banky)
- Subnets jsou často oddělené: `private-app`, `private-data`, `transit` (pro TGW attachment)
- **Nemůžeš udělat Internet Gateway** — vše jde přes centrální egress
- VPC Endpoints jsou buď nasdílené z Network účtu, nebo si je vytvoříš lokálně (pokud SCP povoluje)

### Logging a monitoring už běží

- CloudTrail, VPC Flow Logs, Config jsou **platform-managed** — logují do Log Archive účtu, který nevidíš
- Ty si zapínáš **application-level logs** do vlastních CloudWatch Log Groups
- Často je vyžadované **posílat logy do SIEM** (Splunk, QRadar) přes Kinesis Firehose / subscription filter — platform tým ti dá endpoint

### Break-glass procedury

- Pro incident response má banka **emergency role** s vyšším oprávněním, ale s povinným approval flow a post-hoc auditem
- Ty jako vývojář do toho nesaháš, ale je dobré vědět, že existuje

### Tagging a cost allocation

- **Povinné tagy** vynucované přes SCP/Config rules: `CostCenter`, `Application`, `Environment`, `DataClassification`, `Owner`
- Bez tagů ti deploy selže, nebo dostaneš compliance violation
- V IaC tohle musíš mít centrálně (aspect/provider default tags)

### Data residency a encryption

- Povinné **CMK (Customer Managed Keys)** v KMS pro všechno (S3, EBS, RDS, Secrets Manager)
- Často vlastní **central KMS účet** s multi-region keys
- **Nesmíš** používat AWS-managed keys pro produkční data
- Data musí zůstat v EU regionech (SCP to vynucuje)

---

## 4. Jak se to prakticky projeví na tvém OrderFlow projektu

Kdybys stavěl OrderFlow v bance, **architektura se nezmění**, ale změní se _jak_ ji postavíš:

|Oblast|V bance navíc/jinak|
|---|---|
|**Účty**|Nedostaneš je ty, požádáš o ně přes ticket. Dostaneš je s předinstalovaným baseline. Trvá to dny až týdny.|
|**VPC**|Nevytváříš, dostaneš připojenou k TGW. CIDR z IPAM.|
|**Internet access**|Public API Gateway OK, ale **odchozí** traffic z Lambdy jde přes centrální egress proxy (musíš konfigurovat HTTP_PROXY env vars)|
|**Cognito**|Může být zakázaný — banky často chtějí **federaci s vlastním IdP** (PingFederate, ForgeRock). Místo Cognito User Pool dostaneš OIDC issuer do API Gateway JWT authorizeru.|
|**Aurora**|Přes Service Catalog product, povinně s KMS CMK, automated backups 35 dní, Multi-AZ v prod. Nemůžeš volit všechny engine verze.|
|**DynamoDB**|Povinně encryption s CMK, PITR zapnuté.|
|**S3**|Povinně block public access, encryption s CMK, versioning, access logging.|
|**GitHub Actions**|Pravděpodobně GitHub Enterprise on-prem se self-hosted runners. OIDC přes vnitřní proxy.|
|**Observability**|CloudWatch + povinné forwardování do Splunk. Metriky jdou do centrálního Grafana/Dynatrace.|
|**Secrets**|Secrets Manager s CMK, rotation povinná, nebo HashiCorp Vault který banka provozuje centrálně.|
|**Change management**|Každý prod deploy = CAB ticket, nebo approved change window, nebo standard change template.|

---

## 5. Tradeoffs téhle situace

**Ve prospěch:**

- Hodně "security plumbing" máš zadarmo — CloudTrail, GuardDuty, Config běží
- Compliance je z velké části vyřešená platformou
- Jasné mantinely, méně prostoru pro "self-inflicted wounds"
- Síť, DNS, ingress/egress máš hotové

**Proti:**

- **Rychlost** — nový účet/FW rule/službu čekáš dny až týdny
- **Flexibilita** — nemůžeš použít nejnovější AWS fíčury, dokud je platform tým neaprovuje
- **Cena** — platíš overhead platformy (TGW attachments, centrální endpointy), i když je nepotřebuješ. Pro malou aplikaci je to disproportional.
- **Debugging cross-account** — když je problém v centrálním egress firewall, potřebuješ tým, který jej vlastní, a ten má vlastní queue
- **Vendor lock-in uvnitř firmy** — když platform tým přepíše modul, migrace je nucená
- **Shadow IT risk** — pokud je platforma moc restriktivní/pomalá, týmy začnou obcházet (vlastní AWS účty mimo org, SaaS služby mimo schválený katalog)

---

## 6. Jak se v tomhle prostředí orientovat — praktický postup

1. **Najdi si svůj platform/CCoE tým** a jejich dokumentaci (Confluence, internal dev portal). 80 % odpovědí je tam.
2. **Zjisti seznam approved services** — co smíš používat bez schvalování, co s approval, co vůbec ne
3. **Zjisti permission boundary ARN** a jak ji applikovat v IaC
4. **Zjisti jejich Terraform/CDK moduly** — skoro jistě existují a měl bys je používat místo raw resources
5. **Zjisti networking model** — jak vypadá egress, jak dostaneš public endpoint (pokud vůbec), jak se řeší DNS
6. **Zjisti observability pipeline** — kam mají jít logy/metriky, jaký formát očekává SIEM
7. **Zjisti change management** — jak vypadá prod deployment, co je standard vs normal change
8. **Zjisti disaster recovery požadavky** — RTO/RPO tiers, jestli je DR účet povinný

---

## 7. Návrh OrderFlow pro bankovní prostředí

Tvoje architektura by se upravila takhle:

```
┌──────────────────────────────────────────────────────────┐
│ Platform-owned (nevidíš):                                │
│   Management, Log Archive, Audit, Network (TGW, Egress), │
│   Shared Services (central ECR, Vault)                   │
└──────────────────────────────────────────────────────────┘
                          ▲
                          │ RAM sharing, TGW routing
                          │
┌──────────────────────────────────────────────────────────┐
│ Tvoje účty (tenant):                                     │
│   orderflow-dev / uat / prod                             │
│                                                          │
│   VPC už připojená k TGW                                 │
│   ├─ Public BFF Lambda → API Gateway (regional, private  │
│   │  custom domain přes central ingress)                 │
│   ├─ Private BFF Lambda → Private API Gateway + VPC EP   │
│   ├─ Worker Lambda (Python) → SQS, Aurora               │
│   ├─ Aurora Serverless v2 (přes Service Catalog)        │
│   ├─ DynamoDB (s CMK)                                   │
│   └─ Secrets v Secrets Manager (s CMK) nebo Vault       │
│                                                          │
│   Auth: Federace s bankovním IdP (ne Cognito)            │
│   Egress: přes centrální proxy                           │
│   Logs: CloudWatch → Firehose → Splunk                   │
└──────────────────────────────────────────────────────────┘
```

---

Chceš, abych navázal na konkrétní téma? Přirozené pokračování jsou **permission boundaries a jak je v IaC aplikovat**, nebo **jak postavit CI/CD, když jsi za bankovní proxy a používáš GitHub Enterprise**, nebo **jak vypadá federace s enterprise IdP místo Cognito** pro JWT flow v API Gateway.

# CIDR a subnety — od základů po enterprise planning

Tohle je téma, kde malá chyba na začátku zabolí za 3 roky. Pojďme postupně.

---

## 1. Co je CIDR a jak ho číst

**CIDR** = Classless Inter-Domain Routing, notace `10.0.0.0/16`. Číslo za lomítkem je **prefix length** — kolik bitů adresy je "network part" (fixní), zbytek jsou "host part" (variabilní = použitelné adresy).

IPv4 má 32 bitů. Takže:

- `/16` = 16 bitů network + 16 bitů host = **65 536 adres**
- `/20` = 20 bitů network + 12 bitů host = **4 096 adres**
- `/24` = 24 bitů network + 8 bitů host = **256 adres**
- `/28` = 28 bitů network + 4 bity host = **16 adres**

**Pravidlo:** každé zvětšení prefixu o 1 = poloviční počet adres. `/17` má polovinu `/16`.

### Rychlá tabulka pro orientaci

|Prefix|Adres celkem|Použitelných v AWS subnetu*|
|---|---|---|
|/16|65 536|65 531|
|/20|4 096|4 091|
|/22|1 024|1 019|
|/24|256|251|
|/26|64|59|
|/27|32|27|
|/28|16|11|

*AWS si v **každém subnetu** bere 5 adres: síťová adresa, VPC router, DNS, budoucí použití, broadcast. Takže `/28` subnet ti dá jen **11 použitelných IP**. To je důležité si pamatovat.

---

## 2. Privátní rozsahy — co smíš použít

Podle **RFC 1918** existují tři privátní rozsahy, v rámci kterých si můžeš zvolit cokoli:

|Blok|CIDR|Velikost|
|---|---|---|
|10.0.0.0/8|10.0.0.0 – 10.255.255.255|16 777 216 adres|
|172.16.0.0/12|172.16.0.0 – 172.31.255.255|1 048 576 adres|
|192.168.0.0/16|192.168.0.0 – 192.168.255.255|65 536 adres|

**Plus RFC 6598 "shared address space":** `100.64.0.0/10` — používá se v AWS pro **secondary CIDR** u VPC, když RFC 1918 dochází. Užitečné pro EKS pod IPs, Fargate tasks, overflow dev prostředí.

**AWS omezení pro VPC CIDR:**

- Velikost mezi `/16` (max 65 536) a `/28` (min 16)
- Nesmí být veřejný IP rozsah (teoreticky jde "BYOIP", ale ignoruj to)
- Nesmí přesahovat do reserved ranges

**Volit si je můžeš jakkoliv** v rámci výše uvedeného, ALE — a tohle je klíčové — **musíš myslet na budoucí propojení**. Jakmile dvě sítě chtějí mluvit (VPC peering, TGW, Direct Connect, VPN do on-prem), **nesmí se CIDR překrývat**. A to je důvod, proč se tohle plánuje centrálně.

---

## 3. Co se v praxi používá — konvence

Neexistuje RFC, který by říkal "použij tohle". Ale vznikly silné konvence:

### Home / hobby / lab

- `192.168.x.x` — domácí routery, malé sítě
- Nepoužívat v cloudu kvůli konfliktu s VPN klienty (VPN do práce typicky používá 10.x, ale VPN do domova 192.168.x, kolize jistá)

### Enterprise / cloud

- **`10.0.0.0/8`** je de facto standard — obrovský prostor, nejvíc flexibility
- `172.16.0.0/12` používá menšina firem, často historicky z důvodu oddělení od korporátní `10.x` sítě
- **Docker a Kubernetes** defaultně používají části z obou — pozor na interní kolize (Docker bridge je `172.17.0.0/16`!)

### AWS default VPC

- AWS ti default VPC vytvoří jako `172.31.0.0/16`. **Nikdy to nepoužívej pro produkci.** Smaž default VPC v každém účtu a postav si vlastní.

---

## 4. Jak se plánuje VPC v enterprise — IPAM přístup

Banky a velké firmy mají **IP Address Management (IPAM)** proces, často podporovaný nástrojem (AWS IPAM service, Infoblox, BlueCat, phpIPAM). Typické rozdělení `10.0.0.0/8`:

```
10.0.0.0/8 (celá firma)
├── 10.0.0.0/12      on-premises datacenters
├── 10.16.0.0/12     AWS production
│   ├── 10.16.0.0/16   eu-central-1 prod workloads
│   ├── 10.17.0.0/16   eu-west-1 prod workloads
│   ├── 10.18.0.0/16   us-east-1 prod workloads
│   └── ...
├── 10.32.0.0/12     AWS non-production (dev/uat/staging)
│   ├── 10.32.0.0/16   eu-central-1 dev
│   ├── 10.33.0.0/16   eu-central-1 uat
│   └── ...
├── 10.48.0.0/12     Azure
├── 10.64.0.0/12     GCP
└── 10.240.0.0/12    sandboxy a experimenty
```

**Logika:**

- Každý **region v každém cloudu** má svůj `/16`
- Uvnitř `/16` se vykrajují `/20` až `/24` pro jednotlivé VPC
- Tím je garantováno, že se nic nepřekrývá a je jasné, co kam patří podle první části IP

Když jako vývojář dostaneš VPC, dostaneš typicky **`/22` nebo `/20`** z tohoto pool. Právě tohle ti platform tým **přidělí z IPAM**, ty si to nevymýšlíš.

---

## 5. Rozdělení VPC na subnety — AWS specifika

Klíčová fakta, která musíš vědět:

### Subnet je vázaný na jednu AZ

- Když chceš Multi-AZ HA, potřebuješ **minimálně jeden subnet per AZ per účel**
- Typicky 3 AZ (v eu-central-1 máš 3: eu-central-1a/b/c)
- Takže pro "private application subnet" potřebuješ **3 subnety** (po jednom v každé AZ)

### Subnet typy podle routingu

- **Public subnet** — má route `0.0.0.0/0 → Internet Gateway`. Instance tady mohou mít veřejnou IP.
- **Private subnet** — `0.0.0.0/0 → NAT Gateway` (odchozí internet přes NAT) nebo žádná default route (fully isolated)
- **Isolated subnet** — žádný internet přístup, jen interní komunikace

To je **logické rozlišení podle route table**, ne fyzický typ subnetu. Subnet sám o sobě je jen range IP.

### Standardní rozvržení VPC

Pro typickou aplikační VPC s `/20` (4 096 adres, 3 AZ):

```
VPC: 10.32.16.0/20  (4096 adres)
│
├── Public subnets     (pro ALB, NAT GW, bastion)
│   ├── 10.32.16.0/24    AZ-a  (256 adres)
│   ├── 10.32.17.0/24    AZ-b
│   └── 10.32.18.0/24    AZ-c
│
├── Private App subnets  (pro Lambda, ECS, EC2)
│   ├── 10.32.20.0/23    AZ-a  (512 adres)  ← větší, potřebuje víc IP
│   ├── 10.32.22.0/23    AZ-b
│   └── 10.32.24.0/23    AZ-c
│
├── Private Data subnets  (pro RDS, ElastiCache)
│   ├── 10.32.26.0/27    AZ-a  (32 adres)   ← málo IP, jen databáze
│   ├── 10.32.26.32/27   AZ-b
│   └── 10.32.26.64/27   AZ-c
│
└── Transit/TGW subnets  (pro TGW attachment, ENI)
    ├── 10.32.26.96/28   AZ-a  (16 adres, doporučeno /28)
    ├── 10.32.26.112/28  AZ-b
    └── 10.32.26.128/28  AZ-c
```

**Logika alokace:**

- **Public** dostávají `/24` — ALB, NAT GW moc IP nepotřebují, ale chceš rezervu
- **App subnets** dostávají největší prostor (`/23` nebo `/22`) — tady žere IP Lambda, ECS, EKS nejvíc
- **Data subnets** jsou malé (`/27`) — 2–5 DB endpoint IP stačí
- **TGW subnets** jsou `/28` — AWS doporučuje přesně tohle, TGW potřebuje jen ENI per AZ

---

## 6. Klíčová zrádnost — Lambda, EKS a ENI hlad

Tohle je důvod, proč ti dev účty dojdou IP adresy rychleji, než bys čekal.

### Lambda ve VPC

- Každá **concurrent execution** Lambdy ve VPC si bere **ENI** = **jednu IP ze subnetu**
- Při burst zátěži (třeba 500 parallel requests) Lambda vezme 500 IP
- AWS sice používá **Hyperplane ENIs** (sdílené ENI od 2019), což problém **výrazně** zmírnilo, ale v Multi-AZ setupech se IP stejně spotřebovávají
- **V praxi:** pro Lambda subnet vždy `/22` nebo větší, nikdy ne `/27`

### EKS / Kubernetes

- **Každý pod dostane vlastní IP z VPC CIDR** (default VPC CNI plugin)
- Node (EC2 instance) rezervuje _předem_ několik IP pro pody, které na ní poběží
- `m5.large` si drží až 29 IP adres, i když na ní neběží žádný pod
- 10 nodes × 29 IP = **290 IP adres jen na rezervu**, i prázdný cluster žere tuny IP
- Tohle je _nejčastější_ důvod, proč organizace přecházejí na **secondary CIDR `100.64.0.0/10`** pro pody

### ECS na Fargate

- Každý task = jedna ENI = jedna IP
- Škáluje se rychle, IP mizí rychle

### VPC Endpoints

- Interface endpoint = ENI = 1 IP per AZ per endpoint
- 5 endpointů × 3 AZ = 15 IP

### RDS / Aurora

- Každá instance = 1 IP, ale při Multi-AZ failover se IP mění v rámci subnet pool
- Aurora má cluster endpoint + instance endpoints, rezervuj víc

---

## 7. Problém vyčerpání v dev účtech — tvůj konkrétní case

Scénář: sdílený dev účet, 20 vývojářů, každý si vytváří vlastní stack (Lambda + API GW + Aurora), někdy zapomenou smazat. VPC má `/20` (4 096 IP). Za půl roku narazíš na `InsufficientFreeAddressesInSubnet`.

**Proč k tomu dochází:**

1. **Orphan ENIs** — smazaná Lambda někdy nechá ENI viset 20–40 minut, občas i dýl (bug)
2. **Neukončené stacky** — vývojář smazal kód, ale CloudFormation stack s VPC resources zůstal
3. **Nesprávná subnet velikost** — někdo deploynul do `/27` subnetu a ten je rázem plný
4. **EKS nodes v dev** — pár experimentálních clusterů sežere `/22`
5. **Multiple VPC endpoints** za každý experiment

### Strategie, jak tomu čelit

#### Strategie A: Generous sizing od začátku

- Pro dev VPC použij **`/18` nebo `/19`** (16k–32k IP) místo `/20`
- Subnets pro apps dělej `/21` nebo `/22`, ne `/24`
- **Prevention is cheaper than migration** — zvětšit VPC CIDR později je možné (secondary CIDR), ale rozdělit subnet na větší nejde, musel bys ho smazat a znovu vytvořit, což znamená downtime pro vše v něm

#### Strategie B: Secondary CIDR `100.64.0.0/10`

- Přidáš k VPC druhý CIDR z shared address space
- Dev/ephemeral workloady (Lambda, EKS pods) pošleš do nových subnetů v tomto rozsahu
- Produkční služby zůstávají v primárním RFC 1918 rozsahu
- **Výhoda:** nemusíš sahat na existující subnety, přidáváš prostor

#### Strategie C: Account-per-developer (sandbox model)

- Místo sdíleného dev účtu dostane **každý vývojář vlastní sandbox account**
- Automatické vytvoření přes Account Factory / AFT
- **Nightly cleanup** skripty, které smažou vše starší než X dní
- Každý sandbox má vlastní `/22` VPC — izolace a žádné sdílení IP
- Pro banku těžké (governance), ale mnoho tech firem to tak má

#### Strategie D: Ephemeral environments přes VPC per PR/feature branch

- CI/CD vytvoří mini-VPC per PR, po merge/close ji smaže
- Funguje jen pokud je account velký a VPC creation není bottleneck

#### Strategie E: Fargate/Lambda bez VPC, kde to jde

- **Lambda mimo VPC** nežere žádné VPC IP
- Pokud Lambda nepotřebuje volat do VPC resources (RDS, ElastiCache), **nedávej ji do VPC**
- Pro DynamoDB, S3, Secrets Manager → volej přes public endpoint s IAM auth, ne přes VPC endpoint (pokud bezpečnostní policy dovolí)

#### Strategie F: Cleanup automation

- **AWS Nuke**, **cloud-nuke**, **aws-delete-vpc** — nástroje, které "vymetou" účet
- CloudCustodian policies: "smaž Lambda bez tagu `owner`", "smaž ENI starší 24h bez attachment"
- Cron Lambda, která každou noc projde a smaže orphan ENI

#### Strategie G: Shared infra vs. disposable infra

- Sdílené zdroje (RDS cluster, Redis) sdílí všichni vývojáři — jeden pool IP
- Ephemeral Lambda stacky na share'd infra nesahají, jen volají přes endpoint
- Méně duplicit = méně spotřebovaných IP

#### Strategie H: IPv6

- AWS plně podporuje dual-stack a IPv6-only subnety
- IPv6 prostor je prakticky neomezený — `/56` per VPC = 4.7 × 10¹⁸ adres
- Downside: ne všechny služby podporují IPv6-only (třeba Lambda ve VPC vyžaduje IPv4 dual-stack), RDS má omezení
- Pro greenfield projekty stále složité, ale **to je směr budoucnosti**

---

## 8. Best practices — plánování VPC a CIDR

### Před prvním deploy

1. **Zjisti, zda máš IPAM** — v bance určitě ano, v menší firmě možná ne. Pokud ne, **zaveď ho** (aspoň v tabulce/Confluence).
2. **Plánuj 5–10 let dopředu** — CIDR se špatně mění
3. **Neber první volné** — mysli na, co přijde vedle. Kdo bude mít sousední rozsah?
4. **Nevol příliš malé VPC** — `/24` VPC je bomba, která tikne za půl roku. Minimum `/20`, ideálně `/16` pro prod.
5. **Vyhýbej se `172.17.0.0/16`** (Docker default) a `169.254.0.0/16` (link-local)

### Subnet strategie

1. **Konzistentní pattern napříč VPC** — pokud v prod máš `a = public, b = app, c = data`, stejně to udělej v dev. Snadněji se to pak reviewuje a automatizuje.
2. **Pojmenuj subnety logicky** — `private-app-eu-central-1a`, ne `subnet-a1b2c3`
3. **Rezervuj prostor pro růst** — nealokuj všechny `/24` uvnitř `/20`. Nech půlku volnou pro "co kdyby".
4. **Vyhraď prostor pro TGW, VPC endpointy, ENI** — zvlášť pokud plánuješ Private Link services

### Multi-region

1. **Každý region jiný CIDR** — `eu-central-1 = 10.16.0.0/16`, `eu-west-1 = 10.17.0.0/16`. Kvůli cross-region peeringu a DR.
2. **DR region často menší** — `/18` stačí, neběží tam plná produkce

### Integrace s on-prem

- On-prem má typicky vlastní RFC 1918 pool (často historicky `10.0.0.0/16` nebo nějaký bordel). **Zmapuj to před AWS plánováním.**
- Direct Connect / VPN route propagation musí vědět, co je kde

### Dokumentace

- **Každý CIDR musí mít vlastníka a účel** v IPAM/wiki
- **Nenech nikoho si jen tak zvolit rozsah** — vždy přes IPAM approval

---

## 9. Konkrétní doporučení pro tvůj OrderFlow

Pokud jsi mimo banku a plánuješ vlastní setup:

```
10.0.0.0/8                                 celý tvůj AWS
├── 10.0.0.0/16     Dev account, eu-central-1
│   └── VPC 10.0.0.0/20                    (4096 IP - dost na experimenty)
│       ├── Public:  10.0.0.0/24, /24, /24   (3 AZ)
│       ├── App:     10.0.4.0/22 /22 /22     (velké - Lambda hlad)
│       ├── Data:    10.0.16.0/27 /27 /27    (malé - Aurora)
│       └── TGW:     10.0.17.0/28 /28 /28    (pro budoucnost)
│
├── 10.1.0.0/16     Prod account, eu-central-1
│   └── VPC 10.1.0.0/20 (stejný pattern)
│
└── 10.255.0.0/16   rezerva pro sandbox / budoucí accounts
```

Necháš si 254 volných `/16` bloků na cokoliv dalšího. Dev vs Prod nepřekrývají = můžeš je v budoucnu propojit přes TGW bez bolesti.

---

Pokud chceš, navážu na **jak konkrétně se IPAM zavádí** (AWS IPAM service vs. externí nástroje), nebo **jak vypadá VPC design pro Lambda-heavy workloady** (kde IP hlad je specifický problém), nebo **jak migrovat VPC na větší CIDR, když jsi se spletl** (což je reálný problém, který hodně týmů potkalo).