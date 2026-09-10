# Druhy testů při vývoji a nasazování infrastruktury a aplikací v cloudu

**Testovací pyramida** – u základny je mnoho rychlých a levných testů (unit), směrem nahoru jich ubývá, ale jsou komplexnější, pomalejší a nákladnější (integrační, end-to-end).
## 1. Unit testy (jednotkové testy)

Unit test ověřuje nejmenší izolovanou jednotku kódu – typicky jednu funkci, metodu nebo třídu. Cílem je potvrdit, že daná jednotka se chová správně pro různé vstupy, včetně hraničních případů a chybových stavů. Závislosti na okolí (databáze, HTTP volání, souborový systém) se nahrazují tzv. mocky nebo stuby, aby test zůstal rychlý a deterministický.

Vytvářejí se souběžně s kódem, ideálně podle principu TDD (Test-Driven Development), kdy se nejprve napíše test a pak implementace. V CI/CD běží unit testy v úplně první fázi pipeline – obvykle hned po checkout kódu a instalaci závislostí. Musí být extrémně rychlé (milisekundy až jednotky sekund na test), protože jich bývají tisíce.

Měří se také **code coverage** (pokrytí kódu testy), typicky se cílí na 70–90 %, ale samotná metrika neříká, že testy jsou kvalitní – jen že kód je procházen.

**Co se stane když neprojde:** Pipeline se okamžitě zastaví, commit nelze mergnout. Vývojář dostane zpětnou vazbu v řádu minut a opravuje buď kód, nebo test. Unit testy jsou „první obrannou linií" – pokud selžou, nemá smysl pokračovat k dražším testům.

## 2. Integrační testy

Zatímco unit testy ověřují jednotky izolovaně, integrační testy ověřují, že více komponent spolu správně **komunikuje**. Může jít o interakci služby s databází, s cache, s frontou zpráv, nebo mezi dvěma mikroslužbami. Zde se už typicky nepoužívají mocky – testuje se proti reálné databázi (třeba v Docker kontejneru), reálné Redis instanci nebo reálnému Kafka brokeru.

V CI/CD se spouštějí po unit testech. Pipeline obvykle nastartuje závislosti přes `docker-compose` nebo testcontainers, naplní je testovacími daty, spustí testy a prostředí zase zlikviduje. Jsou pomalejší než unit testy (sekundy až minuty), takže jich bývá řádově méně.

Cílem je odhalit problémy, které unit testy z principu nevidí: špatně napsané SQL dotazy, chybné mapování mezi objekty a databázovými schématy, problémy s transakcemi, nesprávnou serializaci dat mezi službami.

**Co se stane když neprojde:** Pipeline se zastaví, build není promován dál. Často odhalí chybu, kterou vývojář nemohl vidět lokálně, například problém s konkrétní verzí databáze nebo s časováním asynchronních operací.

## 3. Kontraktní testy (contract testing)

Tento druh testu řeší specifický problém mikroslužbové architektury: jak zajistit, že změna v jedné službě nerozbije službu, která ji používá, bez nutnosti spouštět obě dohromady. Používají se nástroje jako **Pact** nebo **Spring Cloud Contract**.

Funguje to tak, že konzument (klient API) definuje očekávaný kontrakt – jak vypadá request, jaká přijde odpověď. Tento kontrakt se publikuje do sdíleného úložiště (Pact Broker) a producent (server) ho používá ve svých testech, aby ověřil, že kontrakt dodržuje. Když producent udělá breaking change, jeho testy selžou ještě před nasazením.

V CI/CD běží na obou stranách – u konzumenta generuje kontrakt, u producenta ho ověřuje. Je to klíčové pro týmy, které deployují nezávisle a potřebují jistotu, že se vzájemně nerozbijí.

**Co se stane když neprojde:** Producent nemůže nasadit změnu, dokud neuprovede migrační strategii (nová verze API, deprecation) nebo dokud konzumenti neaktualizují svůj kontrakt.

## 4. End-to-end (E2E) testy

E2E testy ověřují celý systém z pohledu uživatele – od kliknutí na tlačítko ve webovém rozhraní přes API gateway, backend, databázi až po odpověď zpět. Používají se nástroje jako **Playwright**, **Cypress** nebo **Selenium** pro webové aplikace, respektive Postman/Newman pro čistě API testy.

V CI/CD obvykle neběží na každém commitu, protože jsou pomalé (minuty) a křehké – snadno „flakují" kvůli timingu, síťovým problémům nebo změnám v UI. Typicky se spouštějí po deploymentu do testovacího prostředí (staging), často v noci (nightly builds) nebo před releasem.

Cílem je ověřit **kritické uživatelské scénáře** – registrace, přihlášení, provedení platby, dokončení objednávky. E2E testů by nemělo být mnoho, protože údržba je drahá; pokrývají hlavní „happy paths" a nejrizikovější flows.

**Co se stane když neprojde:** Nasazení do produkce se blokuje. Protože E2E testy občas selhávají z falešných důvodů, často existuje mechanismus retry a manuální review selhání před rozhodnutím o rollbacku.

## 5. Smoke testy

Smoke test je rychlá, povrchní kontrola, že **nasazená aplikace vůbec běží a základní funkce fungují**. Název pochází z hardwarového testování – „zapneš to a koukáš, jestli se kouří". Typicky obsahuje několik málo kontrol: je health endpoint dostupný, vrací databáze data, odpovídá hlavní stránka se statusem 200, funguje login.

Běží **ihned po deploymentu**, ať už do jakéhokoliv prostředí. Trvá sekundy až jednotky minut. V cloudovém prostředí se často implementují jako součást deployment pipeline – třeba v Kubernetes přes readiness/liveness probe plus extra smoke test skript, v AWS přes Lambda volající klíčové endpointy po nasazení přes CodeDeploy.

Smoke testy jsou **klíčové pro strategie jako blue-green deployment nebo canary release** – pokud smoke test na nové verzi selže, automaticky se provede rollback na předchozí verzi a uživatelé ani nezjistí, že se něco dělo.

**Co se stane když neprojde:** Automatický rollback, nebo přinejmenším pozastavení rollout procesu a alert na on-call inženýra. Smoke test je poslední záchranná brzda před tím, než se rozbitá verze dostane k uživatelům.

## 6. Regresní testy

Regresní testy nejsou technicky jiný typ testu – je to spíš způsob použití existujících testů. Jde o sadu testů (obvykle kombinace unit, integračních a E2E), která se spouští před každým releasem, aby se ověřilo, že nové změny **nerozbily dříve fungující funkcionalitu**. Když se najde produkční bug, přidá se na něj regresní test, aby se nikdy neopakoval.

## 7. Performance a load testy

Tato kategorie zahrnuje několik podtypů:

**Load test** ověřuje chování systému při očekávané zátěži – třeba 1000 requestů za sekundu. Cílem je potvrdit, že systém zvládá produkční provoz s přijatelnou latencí.

**Stress test** tlačí systém za hranici únosnosti, aby se zjistilo, kde je bod zlomu a jak se systém chová při přetížení – jestli graceful degradation nebo úplný pád.

**Spike test** simuluje náhlý nárůst provozu (třeba při marketingové kampani) a testuje autoscaling.

**Soak test** (nebo endurance test) běží dlouhodobě (hodiny až dny) a hledá problémy jako memory leaky nebo postupnou degradaci výkonu.

Používají se nástroje jako **k6**, **Gatling**, **JMeter** nebo **Locust**. V CI/CD se obvykle nespouštějí na každý commit – jsou příliš drahé a pomalé. Běží typicky před releasem, v dedikovaném výkonnostním prostředí, které co nejvíc odpovídá produkci. Definují se SLO (Service Level Objectives), například „99 % requestů pod 200 ms" a test buď projde, nebo neprojde.

**Co se stane když neprojde:** Release se odkládá, tým hledá bottleneck – může jít o databázové indexy, N+1 dotazy, nedostatečně nadimenzované instance, chybějící cache.

## 8. Bezpečnostní testy

Bezpečnost se testuje na několika úrovních a tvoří disciplínu zvanou **DevSecOps** – integrace bezpečnosti do celé pipeline.

**SAST (Static Application Security Testing)** analyzuje zdrojový kód a hledá bezpečnostní vzorce – SQL injection, XSS, hardcoded hesla, nebezpečné kryptografické funkce. Nástroje: SonarQube, Semgrep, Checkmarx. Běží v CI jako statická analýza bez spouštění kódu.

**DAST (Dynamic Application Security Testing)** testuje běžící aplikaci zvenčí jako útočník – zkouší injekce, scanuje zranitelnosti v HTTP hlavičkách. Nástroje: OWASP ZAP, Burp Suite. Běží po deploymentu do testovacího prostředí.

**SCA (Software Composition Analysis)** kontroluje závislosti (npm, pip, maven balíčky) proti databázi známých zranitelností (CVE). Nástroje: Snyk, Dependabot, Trivy, OWASP Dependency-Check. Běží v CI a často i průběžně – nová CVE může být zveřejněna kdykoliv.

**Container scanning** kontroluje Docker images na zranitelnosti v OS vrstvě i v aplikačních balíčcích. Nástroje: Trivy, Grype, Clair. Běží po buildu image, před pushnutím do registry.

**Secret scanning** hledá v kódu a commit historii nechtěně commitnuté klíče, tokeny, hesla. Nástroje: Gitleaks, TruffleHog.

**Penetrační testy** – manuální test bezpečnostních expertů, typicky jednou za kvartál nebo před velkým releasem, není součástí automatizované pipeline.

**Co se stane když neprojde:** Záleží na závažnosti. Kritické CVE nebo nalezené tajemství obvykle blokuje merge/deployment. Méně závažné nálezy se tiketují a řeší v rámci sprintu.

## 9. Testy infrastruktury jako kódu (IaC)

V cloudu se infrastruktura definuje kódem (Terraform, Pulumi, CloudFormation, Bicep) a jako kód se musí i testovat.

**Statická analýza a linting** – nástroje jako `terraform validate`, `tflint`, `checkov`, `tfsec`, `kube-score` kontrolují syntaxi, best practices a bezpečnostní problémy (veřejně přístupný S3 bucket, security group otevřená do internetu, chybějící šifrování).

**Policy as Code** – nástroje jako **OPA/Rego**, **Sentinel** (Terraform Cloud) nebo **Kyverno** (Kubernetes) umožňují definovat politiky typu „žádný resource nesmí být bez tagu Owner", „všechny databáze musí mít zapnuté šifrování", „produkční prostředí nesmí používat instance menší než určitá velikost". Pipeline tyto politiky vynucuje.

**Plan review** – v Terraform pipeline se generuje `terraform plan`, který ukazuje, co se změní. V CI se tento plán zveřejňuje jako komentář v pull requestu a často vyžaduje manuální schválení, zvlášť pro produkční prostředí.

**Testování aplikace infrastruktury** – nástroje jako **Terratest**, **kitchen-terraform** nebo **Pulumi testing framework** umožňují skutečně provedení infrastruktury v testovacím cloud účtu, ověření že vznikla správně (ping, DNS lookup, connectivity test) a následnou likvidaci. Je to pomalé a nákladné, takže se spouští selektivně.

**Compliance testy** – ověřují, že infrastruktura splňuje regulační rámce jako HIPAA, PCI-DSS, SOC2, GDPR. Nástroje: AWS Config Rules, Azure Policy, InSpec.

**Co se stane když neprojde:** Infrastrukturní změna se neaplikuje. To je kritické, protože chyba v infrastruktuře může znamenat výpadek celé služby, ztrátu dat nebo bezpečnostní incident.

## 10. Chaos engineering

Chaos engineering záměrně **zanáší poruchy do systému**, aby se ověřila jeho odolnost. Vypíná náhodně instance, přidává síťovou latenci, simuluje výpadek celé AZ (Availability Zone), zvyšuje CPU zátěž. Nástroje: **Chaos Monkey** (Netflix), **Gremlin**, **LitmusChaos** pro Kubernetes, **AWS Fault Injection Simulator**.

Není to klasický CI/CD test – běží v produkci nebo ve staging prostředí, často jako plánovaná „game day" nebo kontinuálně v kontrolovaném režimu. Cílem je najít skryté předpoklady („předpokládali jsme, že databáze je vždy dostupná") a ověřit, že autorecovery mechanismy (retry, circuit breaker, failover) skutečně fungují.

**Co se stane když se najde problém:** Tikety na zlepšení odolnosti, revize architektury. Samo o sobě to neblokuje deployment, ale utváří dlouhodobou spolehlivost.

## 11. Acceptance testy (UAT)

**User Acceptance Testing** ověřuje, že funkcionalita splňuje byznys požadavky z pohledu zákazníka nebo product ownera. Může být automatizované (BDD nástroje jako **Cucumber** s Gherkin syntaxí „Given-When-Then") nebo manuální.

V CI/CD automatizovaná část běží na staging prostředí po deploymentu. Manuální UAT probíhá paralelně – product owner nebo QA tým prochází připravené scénáře.

**Co se stane když neprojde:** Feature se vrací k dopracování, release se odkládá.

## 12. Observability testy a synthetic monitoring

Po nasazení do produkce testování nekončí. **Synthetic monitoring** (Datadog Synthetics, Pingdom, New Relic Synthetics) průběžně simuluje uživatelské požadavky z různých lokalit světa a měří dostupnost a výkon. Detekuje problémy dřív, než je nahlásí skuteční uživatelé.

**Canary analysis** – při canary releasu se nová verze pustí jen pro malé procento provozu (třeba 5 %) a automaticky se porovnávají metriky (error rate, latence) se starou verzí. Nástroje jako **Flagger** nebo **Argo Rollouts** to dělají automaticky – pokud nová verze vykazuje zhoršení, rollout se zastaví a provede se rollback.

---

## Jak to všechno zapadá do typické CI/CD pipeline

Typická pipeline v cloudovém prostředí vypadá zhruba takto. Po commitu se spustí nejlevnější a nejrychlejší kontroly – linting, statická analýza, secret scanning a unit testy. Pokud projdou, následuje build artefaktů (Docker image), jejich scan na zranitelnosti a push do registry. Pak se spustí integrační a kontraktní testy proti dočasně vytvořenému prostředí. Po úspěchu se aplikace nasadí do staging prostředí, kde proběhnou E2E testy, DAST scan a případně performance testy. Před nasazením do produkce často čeká manuální schválení. Deployment do produkce se pak dělá postupně (canary nebo blue-green) s průběžným vyhodnocováním smoke testů a metrik, s automatickým rollbackem při problémech. Po nasazení pokračuje synthetic monitoring a případné chaos experimenty.

Klíčový princip je **shift-left** – posouvat testování co nejvíce doleva, tedy do dřívějších fází vývoje, kde je oprava nejlevnější. Chyba nalezená v IDE stojí minuty, chyba nalezená v produkci může stát hodiny práce mnoha lidí, peníze a důvěru zákazníků.