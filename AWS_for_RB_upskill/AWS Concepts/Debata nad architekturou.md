
# Potřebuji vysvětlit koncepty a služby AWS networkingu. Začneme obecně, poté budeme pokračovat s konkrétní architekturou BFF s více účty.

Jasně, pojďme na to. Začnu obecným přehledem AWS networkingu a pak se můžeme ponořit do konkrétní BFF (Backend for Frontend) architektury napříč více účty.

## Základní stavební kameny AWS networkingu

**VPC (Virtual Private Cloud)** je izolovaná virtuální síť v AWS, kde běží vaše zdroje. Je vázaná na region a definujete v ní rozsah IP adres (CIDR blok, např. `10.0.0.0/16`). VPC je základ — všechno ostatní se staví uvnitř ní nebo ji propojuje s okolím.

**Subnety** rozdělují VPC na menší segmenty vázané na konkrétní Availability Zone. Dělí se na:

- _Public subnet_ — má routu do Internet Gateway, zdroje můžou mít veřejnou IP
- _Private subnet_ — žádná přímá cesta na internet, typicky pro aplikační a databázové vrstvy
- _Isolated subnet_ — bez jakékoliv cesty ven, jen vnitřní komunikace

**Route tables** určují, kam se posílá provoz z daného subnetu. Každý subnet má přiřazenou route table s pravidly typu „pro `0.0.0.0/0` jdi přes NAT Gateway".

**Security Groups vs. NACLs** — oba řídí provoz, ale jinak. Security Group je stavový firewall na úrovni instance/ENI (pamatuje si navázaná spojení, stačí povolit inbound). NACL je bezstavový firewall na úrovni subnetu (musíte povolit inbound i outbound zvlášť). Security Groups se používají primárně, NACLs jako druhá vrstva nebo pro explicitní deny.

## Brány do světa a mezi sítěmi

**Internet Gateway (IGW)** umožňuje obousměrnou komunikaci mezi VPC a internetem. Připojí se k VPC a v route table se na něj směruje `0.0.0.0/0`.

**NAT Gateway** umožňuje zdrojům v privátních subnetech iniciovat spojení ven (stahovat updaty, volat externí API), ale zvenčí k nim nikdo nepřistoupí. Běží v public subnetu a je placená za hodinu + za zpracovaná data.

**VPC Endpoints** umožňují přístup k AWS službám bez průchodu internetem:

- _Gateway Endpoint_ — zdarma, jen pro S3 a DynamoDB, funguje přes route table
- _Interface Endpoint (PrivateLink)_ — placené, vytváří ENI v subnetu, funguje pro většinu AWS služeb a vlastní služby

## Propojování VPC a účtů

**VPC Peering** je přímé 1:1 spojení dvou VPC. Jednoduché, ale netranzitivní — pokud propojíte A-B a B-C, neznamená to, že A mluví s C. Při mnoha VPC vzniká mesh, který se špatně spravuje.

**Transit Gateway (TGW)** je hub-and-spoke router, který centralizuje propojení. Připojíte k němu VPC z různých účtů, VPN, Direct Connect, a TGW rozhoduje routing pomocí route tabulek. Škáluje lépe než peering pro větší množství VPC.

**AWS PrivateLink** exponuje službu z jedné VPC do jiné (nebo jiného účtu) bez peeringu. Poskytovatel vytvoří _VPC Endpoint Service_ za NLB, konzument vytvoří _Interface Endpoint_. Provoz je jednosměrný (konzument → poskytovatel), rozsahy IP se nemusí řešit a je to velmi bezpečné.

**AWS RAM (Resource Access Manager)** umožňuje sdílet zdroje (subnety, TGW, Route 53 resolver rules) mezi účty v organizaci. Typicky se používá při modelu centrální síťové účty.

## Load balancing a aplikační vrstva

**ALB (Application Load Balancer)** pracuje na vrstvě 7 (HTTP/HTTPS), umí routovat podle path, hostname, headers, dělá TLS termination a integruje se s Cognito, WAF, Lambda targets.

**NLB (Network Load Balancer)** pracuje na vrstvě 4 (TCP/UDP/TLS), zachovává zdrojovou IP, má extrémně nízkou latenci a je nutný pro PrivateLink service endpoints.

**API Gateway** je managed služba pro REST/HTTP/WebSocket API — autentizace, throttling, mapování requestů. Může být veřejná nebo privátní (přes interface endpoint).

**CloudFront** je CDN s edge lokacemi po celém světě. Cachuje obsah, terminuje TLS blízko uživateli, integruje se s WAF a Shield, umí invokovat Lambda@Edge nebo CloudFront Functions.

## Multi-account landing zone

V praxi se používá **AWS Organizations** s více účty — typicky odděleně produkce, staging, dev, plus specializované účty (security, logging, networking, shared services). Výhoda je silná izolace blast radius, jasné billing boundaries a granularní IAM.

**Centralizovaný síťový účet** drží Transit Gateway, Route 53 Resolver, někdy i centrální firewall (AWS Network Firewall) a sdílí subnety/TGW do workload účtů přes RAM.

---

Máte už představu o tom, jaká je vaše BFF architektura? Konkrétně by mi pomohlo vědět:

- Kolik účtů plánujete a jak jsou rozdělené (per-service, per-environment, per-team)?
- Kde běží frontend (CloudFront + S3, nebo něco jiného) a kde BFF vrstva (ECS, Lambda, EKS)?
- Jsou backendové služby za BFF v jiných účtech? Komunikuje BFF s externími systémy (on-prem, třetí strany)?
- Máte už rozhodnuto o TGW vs. PrivateLink, nebo to chcete probrat?

Podle toho půjdu do detailu relevantních částí — nechci zahltit věcmi, které pro vaši architekturu nebudou relevantní.


# Popiš rozdíly mezi public a private API gateway.

## Public vs. Private API Gateway

API Gateway v AWS má tři typy endpointů: **Edge-optimized**, **Regional** a **Private**. První dva jsou „public" varianty, třetí je „private". Pojďme si je projít.

### Public API Gateway (Edge-optimized a Regional)

**Edge-optimized** je defaultní typ pro REST API. Request od klienta jde nejdřív do nejbližší CloudFront edge lokace (AWS to dělá automaticky pod kapotou) a odtud do regionu, kde API běží. Vhodné pro globálně distribuované klienty.

**Regional** endpoint je dostupný přímo v daném regionu bez CloudFront vrstvy. Hodí se, když:

- Klienti jsou ve stejném regionu
- Chcete vlastní CloudFront distribuci před API (s vlastním cachováním, WAF konfigurací)
- Potřebujete latency-based routing přes Route 53 do více regionů

Oba typy mají **veřejně dostupný DNS název** typu `abc123.execute-api.eu-central-1.amazonaws.com` a jsou přístupné z internetu. Autorizace se řeší přes IAM, Cognito, Lambda authorizers nebo API keys.

### Private API Gateway

Private API je dostupné **pouze z vaší VPC** (nebo propojených VPC) přes **Interface VPC Endpoint** pro službu `execute-api`. Z internetu se k němu nedostanete vůbec — DNS sice existuje, ale resolvuje se na privátní IP adresy endpoint ENI uvnitř VPC.

Architektura vypadá takto: klient ve VPC → Interface Endpoint (ENI s privátní IP) → API Gateway service → backend integrace.

### Klíčové rozdíly

|Aspekt|Public (Edge/Regional)|Private|
|---|---|---|
|Dostupnost|Z internetu|Pouze z VPC přes Interface Endpoint|
|DNS resolution|Veřejné IP|Privátní IP z VPC endpoint|
|Typický use case|Veřejné API, mobilní/webové klienty|Interní mikroservices, BFF mezi účty|
|Řízení přístupu|IAM, Cognito, authorizers, API keys, WAF|Resource policy + IAM + VPC endpoint policy|
|Custom domain|Ano (ACM cert)|Ano, ale komplikovanější (od 2023 s `disableExecuteApiEndpoint` a Route 53 private hosted zone)|
|WAF|Ano|Ne přímo (musí být před tím něco jiného)|
|Edge caching|Ano (Edge-optimized má CloudFront)|Ne|
|Cena endpointu|Žádný extra náklad|Platí se Interface Endpoint (~$0.01/h za AZ + data)|

### Specifika Private API

**Resource policy je povinná** — musí explicitně povolit přístup, typicky podle `aws:SourceVpc` nebo `aws:SourceVpce`. Bez ní API odmítne všechno. Příklad:

```json
{
  "Effect": "Allow",
  "Principal": "*",
  "Action": "execute-api:Invoke",
  "Resource": "execute-api:/*",
  "Condition": {
    "StringEquals": {
      "aws:SourceVpce": "vpce-1234567890abcdef0"
    }
  }
}
```

**Volání z jiného účtu** — Interface Endpoint může být v jiném účtu, než kde je API Gateway. Stačí, aby resource policy povolila daný `vpce-id` (nebo VPC, nebo account). Toto je důležité pro multi-account BFF setup.

**Sdílení endpointu** — jeden Interface Endpoint může obsluhovat víc Private API v rámci stejného regionu. Endpoint se vytváří jednou, jednotlivá API se rozlišují podle host headeru.

**Custom doména pro Private API** je možná od roku 2023, ale potřebujete:

- Custom domain name resource v API Gateway s typem `PRIVATE`
- Route 53 private hosted zone, která mapuje doménu na endpoint DNS
- TLS certifikát v ACM

### Kdy co použít

**Public Edge-optimized** — veřejná API s globálním publikem, statické weby s API komponentou.

**Public Regional** — když chcete vlastní CloudFront, multi-region setup s Route 53, nebo víte, že klienti jsou v regionu.

**Private** — interní API, BFF vrstvy konzumující backend služby, komunikace mezi účty bez vystavení na internet, compliance požadavky („nic ven z VPC"). V multi-account architektuře je to typický způsob, jak BFF v jednom účtu volá doménové služby v jiných účtech bez peeringu/TGW — jen přes Interface Endpoint.

---

Chcete pokračovat s tím, jak konkrétně poskládat Private API Gateway napříč více účty pro BFF, nebo nejdřív probrat alternativy (PrivateLink + ALB/NLB, App Mesh, atd.)?

# V API gateway vidím položku VPC links. co to je?

## VPC Links

**VPC Link** je opačný směr propojení než Private API Gateway. Zatímco Private API řeší „jak se _klient z VPC_ dostane do API Gateway", VPC Link řeší „jak se _API Gateway_ dostane k backendu uvnitř VPC".

### Problém, který řeší

API Gateway je managed služba, která běží mimo vaši VPC. Když chce volat backend, který je veřejně dostupný (např. veřejné HTTPS API), žádný problém — jde to přes internet. Ale co když je váš backend v privátním subnetu bez veřejné IP? Třeba ECS service za interním ALB, nebo NLB před EC2 instancemi v privátní síti?

VPC Link je most, který umožní API Gateway integraci dosáhnout do privátní VPC, aniž byste backend vystavovali na internet.

### Dva typy VPC Links

Existují dvě varianty podle toho, jaký typ API máte:

**VPC Link pro REST API** (původní, starší):

- Cílí na **Network Load Balancer (NLB)**
- NLB musí být v té VPC, kam chcete dosáhnout
- Pod kapotou používá AWS Hyperplane / PrivateLink technologii
- Backend za NLB může být cokoliv (EC2, ECS, on-prem přes Direct Connect)

**VPC Link pro HTTP API** (novější, od HTTP API v2):

- Cílí na **Application Load Balancer (ALB)**, **Network Load Balancer (NLB)** nebo přímo na **Cloud Map service** (service discovery)
- Flexibilnější — ALB umožňuje routování na vrstvě 7
- Pro většinu nových projektů preferovaná varianta

### Jak to funguje

1. Vytvoříte VPC Link a přiřadíte mu subnety (a security groups u HTTP API varianty) ve vaší VPC
2. AWS vytvoří síťové rozhraní (ENI) ve vašich subnetech, které slouží jako vstupní bod
3. V API Gateway integraci nastavíte typ na _HTTP_PROXY_ (nebo _HTTP_) a uvedete VPC Link ID + URL backendu (privátní DNS NLB/ALB)
4. Když přijde request na API, gateway ho pošle přes VPC Link na váš load balancer

### Konkrétní příklad pro BFF kontext

Řekněme, že máte:

- **Public REST API** v API Gateway, kde frontend volá `/users`, `/orders` atd.
- BFF logiku v ECS Fargate za interním ALB v privátní VPC

Bez VPC Link byste museli ALB udělat veřejný (nebo dát před něj NAT, public subnet atd.). S VPC Link:

- ALB zůstane interní, v privátních subnetech
- API Gateway přes VPC Link routuje requesty na ALB
- Frontend → veřejný API Gateway → VPC Link → interní ALB → ECS tasks

### VPC Link vs. Private API vs. PrivateLink — častý zmatek

Tyto tři věci se pletou, ale řeší různé směry:

|Komponenta|Směr provozu|Použití|
|---|---|---|
|**Private API Gateway**|VPC klient → API Gateway|Interní API přístupné jen z VPC|
|**VPC Link**|API Gateway → VPC backend|Public API volá privátní backend|
|**PrivateLink (Endpoint Service)**|Cross-account/VPC → vaše služba|Sdílení vlastní služby s jinými VPC bez peeringu|

Můžete je i kombinovat — třeba _Private API Gateway_ (přístupné jen z VPC) s _VPC Link_ na backend (běžící v jiné VPC za NLB).

### Cena a omezení

- VPC Link sám o sobě **není zpoplatněn**, platíte jen za API Gateway requesty a za NLB/ALB
- HTTP API VPC Link podporuje až 10 security groups a více subnetů
- Throughput škáluje automaticky, ale prochází přes infrastrukturu API Gateway (latence ~10-30 ms navíc oproti přímému volání)
- Backend musí být dostupný přes privátní DNS nebo IP — VPC Link nedělá DNS resolution mimo VPC

### Kdy ne VPC Link

- Pokud klient i backend jsou ve stejné VPC, použijte přímo ALB/NLB bez API Gateway
- Pokud chcete jen interní API mezi službami, zvažte App Mesh, Service Connect (ECS) nebo přímé volání
- Pokud potřebujete obousměrný/persistentní spojení (WebSocket s vlastní logikou), zvažte jiné architektury

---

Pro vaši BFF architekturu — má smysl probrat, jestli BFF má být veřejné API (public + VPC Link na backend), nebo interní (Private API + VPC Link), nebo úplně bez API Gateway (jen ALB)?

# současný stav a cesta je

1. mobilní aplikace kontaktuje public api gateway
2. autorizace lambda funkcí a přeposlání na VPC link konkrétní aplikace podle cesty
3. NLB míří na 2 různé VPC endpointy
4. dále na private api gateway.

Co je dobře a špatně na tomto setupu a proč?

Dobře, mám obrázek. Pojďme si ten flow rozebrat krok po kroku — co dává smysl a kde jsou problémy nebo věci k zamyšlení.

## Rekapitulace toku

```
Mobilní app
   ↓ HTTPS (internet)
Public API Gateway
   ↓ Lambda authorizer (autorizace)
   ↓ routing podle path
VPC Link → NLB
   ↓ podle target group
VPC Endpoint A  nebo  VPC Endpoint B
   ↓
Private API Gateway (v jiném účtu / VPC)
```

## Co je dobře

**Public API Gateway jako vstupní bod** je správná volba pro mobilní klienty. Dostanete throttling, TLS termination, integraci s WAF, CloudWatch metriky, možnost custom domény s ACM. Mobilní app nepotřebuje nic vědět o vnitřní topologii.

**Lambda authorizer pro autorizaci** je standardní pattern. Centralizuje auth logiku na hraně, takže backend služby nemusí řešit ověření tokenu. API Gateway navíc cachuje výsledek authorizeru (default 5 minut podle policy klíče), takže to není drahé.

**Oddělení BFF služeb do samostatných VPC / účtů** za různými endpointy je správný směr — izolace blast radius, nezávislé deploymenty, jasné ownership boundaries. Toto je zdravý multi-account design.

**Private API Gateway na backendu** dává smysl, pokud chcete v každé doménové službě těžit z features API Gateway (throttling per-service, request validation, mapping templates, vlastní authorizery pro service-to-service auth).

**PrivateLink (VPC Endpoints) místo peering / TGW** je čisté řešení pro cross-account přístup. Žádné překrývání CIDR, jednosměrný provoz, granulární kontrola.

## Co je problematické nebo k zamyšlení

### 1. NLB jako router mezi VPC endpointy je podivný mezičlánek

Toto je nejvíc red flag v celém setupu. Pojďme rozebrat proč:

NLB pracuje na vrstvě 4 (TCP), takže **nedělá routing podle path**. Jediné, jak může NLB rozlišit, kam poslat provoz, je podle **portu** (listener) nebo podle **target group** připojené k danému listeneru. To znamená, že buď:

- **Routing už proběhl v API Gateway** (různé integrace pro různé paths volají různé VPC Linky / různé NLB porty) a NLB jen forwarduje. Pak ale NLB s dvěma target groups nemá smysl — každý path už ví, kam jde.
- **NLB má víc listeneru na různých portech**, kde každý port = jiný backend. To funguje, ale je to neelegantní — port-based routing je křehký a špatně se dokumentuje.

**Lepší alternativy:**

- **ALB místo NLB** — umí path-based a host-based routing na vrstvě 7. Můžete mít jeden ALB s pravidly „`/users/*` → target group A, `/orders/*` → target group B". HTTP API Gateway VPC Link umí cílit přímo na ALB.
- **Více VPC Linků** — jeden VPC Link per backend služba, žádný NLB uprostřed. API Gateway routuje podle path na správný VPC Link, který míří přímo na endpoint cílové služby. Čistší, méně komponent, méně latence.

### 2. Dvojí API Gateway (public + private) přidává latenci a náklady

Každý request prochází **dvěma API Gateway instancemi**, což znamená:

- 2× zpracování requestu, 2× autentizace/autorizace logika (pokud private API taky validuje)
- 2× cena za request (API Gateway účtuje per request — $1.00-3.50 za milion)
- 2× latence (typicky +20-50 ms navíc)
- 2× throttling limity, které musíte koordinovat

**Otázka:** Co private API Gateway přináší, co by neudělal přímo ALB + ECS/Lambda za ním? Pokud odpověď není jasná (request validation per service, service-level rate limiting, mapping templates), tak je to zbytečná vrstva.

**Kdy private API Gateway dává smysl:**

- Doménové týmy chtějí vlastní API management nezávisle (vlastní throttling, vlastní authorizery pro M2M auth, vlastní swagger)
- Backend služba je Lambda a chcete API Gateway pro nativní integraci
- Plánujete service-to-service volání mimo BFF flow, kde private API funguje jako stable contract

**Kdy je to overkill:**

- BFF je jediný konzument backendu
- Backend je containerizovaná služba (ECS/EKS) — pak ALB stačí

### 3. Lambda authorizer — pozor na latenci a cache

Lambda authorizer přidává **cold start** (50-500 ms) a **warm invocation** (~5-30 ms). Mitigace:

- **Provisioned concurrency** na Lambda authorizer pro odstranění cold startů (stojí to peníze, ale pro produkční mobilní backend často nutné)
- **Authorizer caching** — `authorizerResultTtlInSeconds` (default 300s). Klíč cache musí být dobře zvolený. Pokud klíč = `Authorization` header (token), tak každý unikátní token hitne cache jen pro daného uživatele. Pokud potřebujete invalidaci (logout, revocation), je to problém.
- **JWT authorizer místo Lambda** — pokud používáte JWT (Cognito, Auth0, Okta), HTTP API Gateway má **nativní JWT authorizer**, který je rychlejší a zdarma. Pokud autorizační logika je „validuj JWT podpis a ověř scopes", nepotřebujete Lambda.

### 4. Public API Gateway typ — Edge nebo Regional?

Pro mobilní aplikaci s globálními uživateli je **Edge-optimized** typicky lepší (využívá CloudFront edge sítě pro TLS handshake blízko uživateli). Pokud jste na Regional, ztrácíte tuto optimalizaci. Pokud jste na Edge, zase nemáte vlastní kontrolu nad CloudFront (vlastní cache policies, vlastní WAF rules). Záleží, jestli jste si tu volbu udělali vědomě.

### 5. Chybí mi v popisu několik věcí, které jsou kritické

- **WAF** před public API Gateway? Pro mobilní backend by mělo být povinné (rate limiting per IP, protection proti běžným útokům, geo blocking pokud relevantní).
- **CloudWatch / X-Ray tracing** napříč celým flow? Bez distributed tracing je debugging cross-account flow noční můra.
- **Custom doména** na public API Gateway (něco jako `api.firma.cz`)? Default `execute-api` URL je nepoužitelná pro produkci.
- **Versioning strategy** — jak řešíte breaking changes pro mobilní klienty, kteří se neaktualizují?
- **Throttling nastavení** — default 10k req/s může být moc nebo málo. Per-method throttling pro citlivé endpointy?

### 6. VPC Endpoint per backend — náklady

Každý Interface Endpoint stojí **~$0.01/h per AZ** + data processing (~$0.01/GB). Při 3 AZ je to $21.6/měsíc per endpoint, jen za to, že existuje. Při hodně backendech a hodně účtech to roste. Není to drama, ale je dobré vědět — někdy je levnější TGW pro hodně cross-account provozu.

## Shrnutí — priorita fixů

1. **Zpochybnit NLB mezi API Gateway a VPC endpointy** — buď to nahradit více VPC Linky (přímo na cílové endpointy), nebo ALB s path routingem. NLB jako „router" je pravděpodobně architektonický artefakt.
2. **Zpochybnit nutnost private API Gateway** — pokud nepřináší konkrétní hodnotu (request validation, per-service throttling, M2M auth), nahradit přímou integrací s ALB/Lambda.
3. **Zvážit JWT authorizer místo Lambda** — pokud používáte standardní JWT.
4. **Provisioned concurrency** na Lambda authorizer, pokud zůstává.
5. **Ověřit WAF, custom doménu, tracing, throttling** — pokud chybí.

---

Můžete mi říct víc o tom NLB kroku? Konkrétně:

- Routuje podle portu (listener per service), nebo má víc target groups na jednom listeneru?
- Je NLB v stejném účtu jako public API Gateway, nebo jinde?
- Proč tam vznikl — historický důvod, nebo aktivní designové rozhodnutí?

A ohledně private API Gateway — co konkrétně dělá, co by neudělal přímo backend za ALB?

# public api gateway prý slouží pouze jako proxy a k autorizaci. NLB zatím nevím.

Dobře, to je užitečná informace. Pojďme si to rozebrat, protože to otevírá zajímavé otázky o designu.

## Public API Gateway jen jako proxy + auth

Pokud public API Gateway **nedělá nic jiného než autorizaci a forwarding**, je to validní pattern, ale stojí za to si uvědomit, co z toho plyne:

### Co to znamená architektonicky

Public API Gateway v tomto setupu hraje roli **„API edge"** nebo **„auth gateway"** — jediný vstupní bod z internetu, kde se řeší:

- TLS termination
- Autentizace/autorizace (Lambda authorizer)
- Throttling a rate limiting
- WAF integrace
- Custom doména
- Logging/metriky pro všechen traffic

A pak předává request dál, aniž by ho transformoval. Žádné mapping templates, žádná request validation, žádná business logika. Tohle je **legitimní a často doporučovaný pattern** — říká se mu různě: _API edge_, _authentication gateway_, _unified entry point_.

### Výhody tohoto přístupu

- **Jedno místo pro auth** — všechny mobilní requesty procházejí stejnou auth logikou, nelze ji obejít
- **Jednoduchá public API Gateway konfigurace** — méně věcí co se může rozbít
- **Backend služby se nestarají o auth** — dostanou už ověřený request s identity info v headerech (authorizer může přidat `x-user-id`, `x-tenant-id` atd. přes `context`)
- **Centralizované WAF, throttling, logging** na jednom místě

### Nevýhody / věci k zamyšlení

**1. API Gateway je drahý „jen jako proxy"**

API Gateway REST API stojí **$3.50 za milion requestů**, HTTP API **$1.00 za milion**. Pokud děláte jen auth + proxy, je otázka, jestli by nestačila levnější varianta:

- **HTTP API místo REST API** — 3.5× levnější, rychlejší, má nativní JWT authorizer. Pokud nepotřebujete specifické features REST API (request validation, SDK generation, API keys, usage plans, mapping templates), HTTP API je lepší volba právě pro tento use case.
- **ALB s Cognito/OIDC autorizací** — pokud je auth JWT-based, ALB umí auth nativně (levnější při vysokém trafficu, ale méně flexibilní)
- **CloudFront + Lambda@Edge** pro auth — pro extrémní škálu, ale komplikovanější

**Doporučení:** Pokud jste na REST API, zvažte migraci na HTTP API. Rozdíl v ceně je při mobilním provozu znatelný a HTTP API má lepší VPC Link (podporuje ALB, ne jen NLB).

**2. Pokud API Gateway nic netransformuje, proč za ním další API Gateway?**

Tohle je to, na co jsem narážel u private API Gateway. Pokud public API Gateway dělá jen auth + forward, a private API Gateway dělá... co vlastně? Dvě možnosti:

- **Private API Gateway dělá něco navíc** — request validation per service, service-specific throttling, mapping templates, M2M auth pro jiné konzumenty. Pak má smysl.
- **Private API Gateway je tam historicky / „protože to tak dělají všichni"** — pak je to dead weight. Backend tým si možná myslel, že „potřebuje API Gateway", ale reálně by stačil ALB přímo před ECS/Lambda.

**Otázka pro backend týmy:** Co by se rozbilo, kdyby místo private API Gateway byl přímo interní ALB před jejich službou?

**3. Lambda authorizer vs. JWT authorizer**

Pokud je autorizace založená na JWT (což u mobilní app obvykle je — Cognito, Auth0, vlastní Identity Provider), tak:

- **REST API** má jen Lambda authorizer nebo Cognito authorizer
- **HTTP API** má **nativní JWT authorizer** — validuje podpis, `iss`, `aud`, scopes, bez Lambda invokace, bez cold startů, **zdarma**

Pokud Lambda authorizer dělá něco navíc (lookup do databáze, custom business logika, revocation check), tak musí zůstat Lambda. Ale pokud jen validuje JWT a extrahuje claims, je JWT authorizer jednoznačně lepší.

**4. Identita se musí nějak propsat dál**

Důležitý detail: když authorizer ověří uživatele, jak se ta identita dostane k backendu? Několik patternů:

- **Context injection** — authorizer vrátí `context` objekt, API Gateway ho přidá jako headery do požadavku (`$context.authorizer.userId` → `x-user-id`). Backend věří hlavičkám.
- **Přeposlání původního JWT** — backend si ho revaliduje (belt-and-suspenders, ale duplikace)
- **Podepsaný interní token** — authorizer vygeneruje krátkodobý interní JWT, backend validuje jeho podpis

Pokud backend službám věříte headerům z API Gateway (což je v uzavřeném VPC flow OK), context injection je nejjednodušší. Ale musí existovat garance, že **nikdo jiný než API Gateway se k private API Gateway nedostane** — tj. resource policy musí být správně nastavená na `aws:SourceVpce`.

## NLB — spekulace, proč tam je

Bez znalosti konkrétního setupu můžu hádat důvody, proč tam NLB je:

### Pravděpodobný scénář A: NLB jako statický IP endpoint

VPC Link pro REST API **vyžaduje NLB**. Nejde tam dát ALB ani přímo VPC endpoint. Takže pokud má setup REST API (ne HTTP API), NLB **musí** tam být. Pak je NLB technická nutnost, ne designové rozhodnutí.

Pokud je to tento případ, pak NLB target group míří na **IP adresy VPC Endpointů** (PrivateLink endpoint vytvoří ENI s privátními IP v subnetech, NLB je má jako targets). Toto je známý pattern — NLB jako „bridge" z VPC Link na Interface Endpoint.

### Pravděpodobný scénář B: NLB pro cross-account routing

NLB je v „networking" nebo „shared services" účtu, kde sedí VPC Link. Target groups pak míří na VPC endpointy, které odkazují do různých produkčních účtů. NLB funguje jako centrální routing bod mezi jedním VPC Linkem a více cílovými účty.

### Proč je to podezřelé

Obě varianty **mohou být validní**, ale přinášejí komplikaci:

- **Port-based nebo target-group-based routing na L4** — nemůže routovat podle path, takže rozhodnutí „kam jde request" se muselo udělat už v API Gateway (různé integrace pro různé paths)
- **NLB s VPC Endpoint jako target** má specifika — target type musí být **IP** (ne instance), a IP adresy endpoint ENI se můžou měnit (AWS je sice obvykle nemění, ale garance není). Některé týmy to řeší Lambdou, která periodicky resolvuje DNS a aktualizuje target group.
- **Další failure point** — NLB může mít problémy s health checks proti VPC Endpointům (endpoint neposkytuje HTTP health check endpoint přímo, takže se dělá TCP health check na port 443, což neověří funkčnost API za ním)

### Co zjistit

Zeptejte se platformního / síťového týmu:

1. **Je to REST API nebo HTTP API?** Pokud REST, NLB je vynucený. Pokud HTTP, NLB je volba.
2. **Kolik listeneru má NLB a na jakých portech?** Jeden listener = routing se děje v API Gateway (víc integrací míří na různé NLB porty). Víc listenerů = port-based routing.
3. **Co je v target groups?** IP adresy VPC Endpointů? ENI přímo?
4. **Kdo vlastní NLB?** Je v síťovém účtu, nebo v účtu s API Gateway?
5. **Proč ne víc VPC Linků?** Při HTTP API můžete mít víc VPC Linků (limit 10 per account per region), každý mířící na jiný ALB/NLB/Cloud Map — bez NLB uprostřed.

## Zjednodušený target architektury

Pokud by to bylo greenfield a víme, že flow je „mobilní → auth → backend služba", minimalistická varianta by byla:

```
Mobilní app
   ↓
HTTP API Gateway (JWT authorizer, WAF, custom doména)
   ↓ VPC Link per služba
Interní ALB (v účtu dané služby, path-based routing na microservices)
   ↓
ECS / Lambda / EKS
```

Žádné NLB uprostřed, žádné private API Gateway. Méně komponent = méně selhání, méně latence, nižší náklady.

Výjimky, kdy by private API Gateway nebo NLB zůstaly:

- Backend tým aktivně používá features private API Gateway (request validation, per-service throttling)
- Cross-account setup přes PrivateLink vyžaduje NLB architekturu z nějakého důvodu (compliance, network team policy)
- Existují jiní konzumenti než jen BFF (service-to-service z jiných účtů)

---

Dává smysl se teď zaměřit na to, proč tam NLB je, a zvlášť na to, **jestli je to REST nebo HTTP API** — to rozhodne, jestli je NLB nutný, nebo jen volba. Můžete to zjistit?

# Popis 
1. jedná se o REST api
2. 1 listener TLS:443, navázaný na 2 AZ ve stejném regionu. Load balancerů je několik, každý míří na jinou Target group
3. Target type je IP,  vždy jsou 2 IP každá v jiné AZ
4. NLB je v účtu s API gateway

Kvůli zero trust setupu druhá API gateway opět autorizuje JWT

Výborně, teď už vidím celý obrázek jasně. Pojďme si to projít — některé věci dávají víc smysl, než se zdálo, a některé zůstávají diskutabilní.

## Revize setupu s novými informacemi

```
Mobilní app
   ↓ HTTPS
Public REST API Gateway (Lambda authorizer #1 — JWT)
   ↓ VPC Link → NLB (TLS:443, IP targets)
NLB per služba, každý se dvěma IP targets (2 AZ)
   ↓ IP targets = ENI VPC Endpointu v jiném účtu
Private REST API Gateway (authorizer #2 — znovu JWT)
   ↓
Backend služba
```

## Co teď dává smysl

### REST API vynucuje NLB

Potvrzeno — **REST API VPC Link může cílit jen na NLB**, nejde ALB ani Cloud Map. Takže NLB tam **musí** být, není to designová volba, je to technické omezení. Beru zpět svoji kritiku „NLB jako zbytečný mezičlánek" — v kontextu REST API je to jediná cesta.

### IP targets = ENI VPC Endpointů

Toto je standardní a dokumentovaný pattern pro „API Gateway v účtu A volá Private API Gateway v účtu B". Funguje to takto:

1. V účtu A vznikne Interface VPC Endpoint pro `execute-api` v účtu B (přes PrivateLink, přes share nebo cross-account endpoint service)
2. Endpoint vytvoří ENI v subnetech účtu A s privátními IP adresami (jedna per AZ)
3. NLB má tyto IP jako target type `IP`
4. NLB forwarduje TLS:443 na endpoint ENI
5. VPC Endpoint forwarduje na Private API Gateway v účtu B

Proč dvě IP ve dvou AZ: high availability — pokud vypadne jedna AZ, druhá ENI přebírá provoz. Toto je správně.

### Víc NLB = víc backend služeb

Každá backend služba / účet má vlastní NLB s vlastními IP targets. API Gateway routuje přes **různé VPC Link integrace** (jedna integrace per path/služba) na různé NLB. Toto je konzistentní pattern — neukládá se routing na L4 do jednoho NLB, ale rozhoduje se v API Gateway, který NLB použít.

To znamená, že moje dřívější podezření na „NLB jako router" bylo špatné — NLB neroutuje, je to jen bridge na konkrétní VPC endpoint. Routing se děje v API Gateway výběrem VPC Linku.

### Double JWT validation = zero trust

Pokud je to **vědomé rozhodnutí v rámci zero trust modelu**, tak je to legitimní. Zero trust říká „never trust, always verify" — každá síťová hranice ověřuje nezávisle, nikdo se nespoléhá na to, že ten před ním udělal auth správně.

V tomto kontextu:

- Public API Gateway ověří JWT (obrana proti anonymním útokům z internetu)
- Private API Gateway ověří JWT znovu (obrana proti kompromitovanému public API Gateway nebo lateral movement v AWS)

Toto je validní **pokud jsou backend služby citlivé** (finanční data, PII, healthcare) a compliance / security tým explicitně vyžaduje defense in depth. Pro méně citlivé systémy je to overkill.

## Co zůstává diskutabilní

### 1. REST API místo HTTP API — proč?

**Cena:** REST API je 3.5× dražší než HTTP API ($3.50 vs $1.00 za milion). Při mobilní aplikaci s miliony requestů měsíčně je rozdíl reálný.

**Features:** REST API má víc features (request validation, API keys, usage plans, SDK generation, WAF integrace na samotné API Gateway), ale pokud použití je „auth + proxy", většinu z toho nevyužijete.

**Limitace HTTP API:** HTTP API ale **nepodporuje private endpoints** — HTTP API je vždy public. Takže **private API Gateway v backend účtech musí zůstat REST**, pokud má být private.

**Doporučení:**

- **Public API Gateway** by mohl být HTTP API — pokud nepoužíváte features jako mapping templates, request validation, API keys
- **Private API Gateway** musí zůstat REST (HTTP API neumí privátní)

Ale pozor: HTTP API má **VPC Link jen pro ALB/NLB/Cloud Map v rámci VPC**, ne pro cross-account Interface Endpoint přes NLB. Takže migrace public na HTTP API by vyžadovala přehodnocení celého downstream flow. Nemusí se to vyplatit.

### 2. Dvojitý Lambda authorizer = dvojitá latence + cena

Každý Lambda authorizer znamená:

- Invokaci Lambda (warm: ~5-20 ms, cold: 100-500 ms)
- Cenu za invokaci + za GB-sekundy

Dva authorizery = dvojnásobek. Mitigace:

**Authorizer caching** je kritický. Cache key by měl být `Authorization` header (celý JWT). TTL 300s (default) znamená, že pro daného uživatele se Lambda volá maximálně jednou za 5 minut, pak se výsledek cachuje. U mobilní aplikace, kde uživatel dělá desítky requestů v řadě, to šetří hodně.

**Provisioned concurrency** pro authorizer Lambdu — eliminuje cold starty. Pro produkční mobilní backend obvykle nutné.

**Lambda authorizer vs. něco chytřejšího:** Pokud obě API Gateway validují stejný JWT stejným způsobem, je duplikace logiky. Možnosti:

- **Shared Lambda layer** s auth logikou, obě Lambda funkce z ní čtou (není to stejná Lambda, ale kód je sdílený)
- **Second gateway validuje jen podpis**, ne full business logic (první dělá „může tento uživatel vůbec dovnitř?", druhý jen „je JWT platný a podepsaný naší autoritou?")

Ale pokud zero trust říká „každá vrstva musí validovat plně nezávisle", tak je to by design.

### 3. Cena vrstev

Spočítejme si hrubý odhad per milion requestů:

|Komponenta|Cena|
|---|---|
|Public REST API Gateway|$3.50|
|Lambda authorizer #1 (s cachem ~20% hit miss)|~$0.40|
|Data processing přes NLB|$6.00 za GB|
|VPC Endpoint data processing|$0.01 za GB|
|Private REST API Gateway|$3.50|
|Lambda authorizer #2|~$0.40|

Jen API Gateway + authorizery: **~$7.80 per milion** requestů. Při 100M requestů měsíčně = **$780**. NLB + endpoint data processing přidává podle objemu.

Pro porovnání, kdyby to bylo „HTTP API + JWT authorizer + ALB direkt":

- HTTP API: $1.00
- JWT authorizer: zdarma
- ALB: ~$0.008/hodinu + $0.008 per LCU

To by bylo **~$1.50 per milion** requestů — 5× levnější. Ale znovu, zero trust a private API Gateway features to ospravedlňují, pokud jsou reálně potřeba.

### 4. NLB health checks

NLB dělá health check na IP targets. U VPC Endpointu nemůžete dělat HTTP health check na specifický path (Endpoint neposkytuje HTTP health endpoint). Typicky se dělá **TCP health check na port 443**, což ověří, že endpoint naslouchá, ale ne, že Private API Gateway za ním funguje.

Důsledek: pokud Private API Gateway má problém, NLB to neví a dál routuje traffic na „zdravý" endpoint. Chyby se projeví až jako 5xx z API Gateway, ne jako odstranění targetu z LB.

Mitigace:

- CloudWatch alarm na 5xx metriku z public API Gateway
- Možná custom health check endpoint v API Gateway, ale to je komplikace

### 5. Ownership a blast radius NLB účtu

NLB je v účtu s public API Gateway. To znamená, že **tento účet drží kritickou síťovou infrastrukturu** pro všechny backend flow. Pokud dojde ke kompromitaci nebo misconfiguraci v tomto účtu, je postiženo všechno.

Alternativa: mít **networking účet** oddělený od aplikačních účtů, kde sedí NLB + VPC Linky. Public API Gateway pak může být v separátním „edge" účtu. Rozšiřuje blast radius napříč účty, ale zvyšuje komplexitu.

Záleží na tom, jestli je account strategie „jeden účet per tier" nebo „jeden účet per doména". Pokud to první, current setup je OK.

## Shrnutí — co se mi teď líbí a nelíbí

**Dává smysl:**

- NLB je technická nutnost pro REST API VPC Link (zpět beru kritiku)
- IP targets na VPC Endpoint ENI je standardní cross-account pattern
- Separátní NLB per služba = čisté rozdělení, routing v API Gateway
- Double auth v zero trust kontextu je legitimní

**Stojí za přehodnocení:**

- **Cena** — dvě REST API Gateway + dvě Lambda authorizers je drahé. Je vědomý trade-off za security?
- **HTTP API pro public layer** — pokud features nejsou využity, přímá úspora
- **Authorizer caching a provisioned concurrency** — jsou nastavené? Pokud ne, quick win pro latenci
- **NLB health checks** — jsou jen TCP? Je monitoring 5xx na public API?
- **Propagace identity** — jak se uživatelská identita dostává z public authorizeru do backendu? Přes JWT (druhý authorizer ji znovu vyextrahuje) nebo přes context headers?

**Zbývající otázky, které by stálo za to zodpovědět:**

1. Jaký je objem requestů (aspoň řádově)? To rozhodne, jestli má smysl optimalizovat cenu.
2. Je backend skutečně citlivý natolik, aby ospravedlnil double JWT validaci? Nebo je to cargo cult zero trust?
3. Plánuje se do budoucna další konzument Private API Gateway než jen BFF? Pokud ano, private API má smysl. Pokud ne, dá se zjednodušit na ALB + ECS.
4. Je Lambda authorizer #1 a #2 stejný kód, nebo dělá každý něco jiného?

Chcete projít některý z těch bodů detailněji, nebo je dál něco konkrétního, co vás zajímá — třeba jak to vypadá z pohledu Terraform / IaC, jak se to testuje, nebo jak řešit specifický problém v tomto setupu?


# Stav

1. 40 milionů na Proxy Api Gateway
2. je to zero trust
3. Jsou dva typy private účtů - sdílený pro více aplikací a jen pro jednu konkrétní aplikaci. Aplikací bude přibývat
4. Nevím

Dobře, teď máme dost informací na konkrétní doporučení. Pojďme si to shrnout a zaměřit se na to, co má největší smysl.

## Ekonomika při 40M requestů / měsíc

Hrubý odhad nákladů jen na API Gateway + authorizer vrstvu:

|Položka|Měsíčně|
|---|---|
|Public REST API Gateway (40M × $3.50/M)|$140|
|Lambda authorizer #1 (s cachem, odhad 20% miss rate)|~$15-25|
|Private REST API Gateway (40M × $3.50/M)|$140|
|Lambda authorizer #2|~$15-25|
|NLB (per LB, fixní ~$16/měsíc + LCU)|~$25-40 per NLB|
|VPC Endpointy (~$22/měsíc per endpoint × počet AZ)|podle počtu|
|**Celkem API + auth vrstva**|**~$350-450/měsíc**|

Plus data processing, NAT Gateway pokud je, Lambda invokace. Realisticky **$500-800/měsíc** za celou tuto vrstvu. Při 40M requestů to není bolestivé číslo — je to přibližně $0.0001-0.0002 per request. Pro mobilní backend s auth vrstvou rozumné.

**Závěr:** Při 40M/měsíc **nemá smysl přehnaně optimalizovat cenu**. Kdyby to bylo 4 miliardy, řešil bych HTTP API migraci. Při 40M je cena API Gateway zanedbatelná oproti provozním nákladům týmu. Nechte to být a řešte jiné věci.

## Zero trust — co znamená správně implementované

Zero trust není jen „dvakrát ověřit JWT". Pokud chcete mít ten setup skutečně zero trust, stojí za to projít checklist:

**Validace JWT na obou vrstvách musí být skutečně nezávislá:**

- Obě Lambda funkce ověřují **podpis** proti JWKS (public klíče Identity Provideru), ne jen dekódují token
- Obě validují `iss`, `aud`, `exp`, `nbf` — ne jen claims
- Obě kontrolují scopes/permissions relevantní pro daný endpoint
- Ideálně každá vrstva má **vlastní cache JWKS** (neshared), aby kompromitace jedné neotrávila druhou

**Síťová izolace:**

- Private API Gateway resource policy musí povolit **jen konkrétní VPC Endpoint** (`aws:SourceVpce`), ne celou VPC ani širší konstrukce
- Security groups na NLB a endpoint ENI omezují provoz jen na očekávaný zdroj
- Žádný backend endpoint není dostupný z internetu jinou cestou (auditovat!)

**Identity propagace mezi vrstvami:**

Toto je bod, kde jste řekl „nevím", a je to důležité. Jsou dvě možnosti:

_Varianta A:_ Public authorizer validuje JWT a **přeposílá ho beze změny** do private API Gateway. Private authorizer ho validuje znovu. Toto je konzistentní zero trust — druhá vrstva nevěří ničemu, co přišlo před ní, validuje originální token z IdP.

_Varianta B:_ Public authorizer validuje JWT, extrahuje identity, vloží ji jako context headers (`x-user-id`, `x-tenant-id`) a JWT zahodí. Private authorizer ověří něco jiného — například interní podpisový token. Toto **není čistý zero trust**, protože druhá vrstva věří tomu, co jí řekla první.

Zjistit, která varianta to je, je kritické. Pokud varianta B, tak „zero trust" je spíš marketingový nálepka než reálná architektura.

**Doporučuju se zeptat backend/security týmu:** _„Jaký token dostává Lambda authorizer #2? Originální JWT od klienta, nebo něco, co vygeneroval #1?"_

## Multi-aplikační škálování — to je teď hlavní téma

Řekl jste, že aplikací přibývá a existují dva typy účtů:

- **Shared backend účet** pro více aplikací
- **Dedicated backend účet** pro jednu konkrétní aplikaci

Toto je klíčový bod, protože current setup se škáluje lineárně a začne být problematický.

### Co se děje s každou novou aplikací

Pro každou novou backend aplikaci v dedicated účtu musíte:

1. Vytvořit Private API Gateway v target účtu
2. Vytvořit VPC Endpoint ve zdrojovém (proxy) účtu mířící na service v target účtu
3. Vytvořit nový NLB (nebo přidat listener na existující) v proxy účtu
4. Nastavit target group s IP adresami endpoint ENI
5. Vytvořit VPC Link v API Gateway
6. Přidat routing pravidla v public API Gateway (integrace per path)
7. Nastavit resource policy na private API Gateway (jen správný VPC endpoint)
8. Propojit Lambda authorizer #2 v target účtu
9. Terraform / CloudFormation template pro všechny tyto kroky
10. Monitoring, alarmy, dashboardy pro novou službu

To je **hodně manuálních kroků napříč dvěma účty**. Každá aplikace = týdny onboardingu. Při 5 aplikacích manageable, při 50 noční můra.

### Tři možné strategie škálování

**Strategie 1: Standardizace a automatizace současného setupu**

Přijmete složitost, ale zabalíte ji do reusable modulů. Typicky:

- **Terraform modul „onboard new service"** — jeden modul, který deployne private API Gateway v target účtu, VPC Endpoint a NLB setup v proxy účtu, vše propojené
- **Naming conventions** — deterministické pojmenování (např. `svc-{app-name}-api`, `svc-{app-name}-nlb`), aby se nemuselo všude sledovat ID
- **Service catalog / internal platform** — backend team přijde, vyplní formulář (název služby, account ID, path prefix), pipeline vygeneruje PR s potřebnými resources

**Kdy zvolit:** Pokud zero trust a account izolace jsou nedotknutelné, a jste OK s tím, že onboarding trvá, ale je standardizovaný. Většina velkých enterprise prostředí jde touto cestou.

**Strategie 2: Service mesh / Platform layer**

Místo ad-hoc propojení per service postavíte platformní vrstvu. Varianty:

- **AWS App Mesh** — sidecar proxy (Envoy) v každé službě, centrální control plane, mTLS mezi službami. Zero trust na service level bez potřeby API Gateway mezi každou dvojicí.
- **Service Connect (ECS)** nebo **Cloud Map** — service discovery a mTLS nativně
- **Centralizovaný API Gateway hub** v networking účtu, který sdílí routing layer, ale authorizace per service zůstává

**Kdy zvolit:** Pokud plánujete 20+ služeb a už teď víte, že onboarding friction je blocker. App Mesh ale přináší vlastní komplexitu (Envoy debugging, control plane management).

**Strategie 3: Zjednodušit bariéru mezi proxy a backend**

Kacířská myšlenka: **druhý API Gateway a NLB je možná nadbytečný**, pokud:

- Síťová izolace je zajištěna PrivateLink (už je)
- Autorizace je zajištěna validací JWT v backend službě přímo (ne v API Gateway)

Potom by backend běžel za interním ALB s Cognito/OIDC auth nebo s library, která validuje JWT v aplikaci. Proxy API Gateway by mířil přes VPC Link → NLB → VPC Endpoint → interní ALB v target účtu.

**Ušetří to:** Druhý API Gateway, druhý Lambda authorizer, dvojí konfigurace.

**Ztratíte:** API Gateway features v backend vrstvě (pokud je využíváte), konzistenci „všechno je API Gateway".

**Kdy zvolit:** Pokud „zero trust" v druhé vrstvě reálně znamená jen „validuj JWT", a ne „plná API Gateway functionality". Potom library v aplikaci dělá totéž a je levnější na provoz.

### Shared vs. dedicated účty — kdy co

Z toho, co popisujete, vzniká potřeba rozhodovacího kritéria:

**Shared backend účet** (více aplikací):

- Nižší overhead onboardingu (jen nová služba, ne nový účet)
- Sdílená monitoring, logging, IAM baseline
- Menší blast radius izolace — jedna aplikace může ovlivnit druhou (resource limits, IAM misconfiguration)
- Vhodné pro: malé/střední aplikace, podobný security tier, stejný ownership team

**Dedicated účet** (jedna aplikace):

- Plná izolace — resource limits, IAM, billing, blast radius
- Vlastní VPC, vlastní compliance boundaries
- Vyšší overhead onboardingu a provozu
- Vhodné pro: vysoká kritičnost, regulace (PCI, HIPAA, finanční), jiný ownership, výrazně jiný security profile

**Doporučení:** Mít **explicitní kritéria**, kdy aplikace jde do shared a kdy do dedicated, ne case-by-case rozhodnutí. Například: „Aplikace s PII/finančními daty → dedicated, ostatní → shared". Jinak bude každý tým lobovat za dedicated „pro jistotu" a počet účtů exploduje.

## Moje konkrétní doporučení pro váš kontext

Na základě všeho, co jsem slyšel:

1. **Cenu neoptimalizujte** — při 40M je to irelevantní ve srovnání s provozními náklady
2. **Zjistěte, jak funguje identity propagace** mezi authorizer #1 a #2 — pokud varianta B, není to zero trust a stojí za to to buď opravit, nebo přestat tomu tak říkat
3. **Investujte do automatizace onboardingu** — Terraform modul pro „nová backend služba" s jasným API pro backend týmy. Toto je největší ROI při škálování.
4. **Stanovte kritéria shared vs. dedicated účet** — dokumentované, ne ad-hoc
5. **Monitorujte 5xx na obou API Gateway vrstvách** — NLB health check nevidí problémy v Private API Gateway
6. **Nastavte provisioned concurrency** na authorizer Lambdy, pokud už není — při 40M requestů cold starty bolí uživatelské UX
7. **Authorizer cache TTL** — ověřte, že je nastavený a na rozumnou hodnotu (300s je default)

Body 1, 2, 4 jsou „nebolí, ale stojí za to je mít pořádně". Body 3, 5, 6, 7 jsou konkrétní akce s měřitelným dopadem.

---

Chcete se teď zaměřit na některý z těchto bodů detailněji? Třeba jak by vypadal ten Terraform modul pro onboarding, nebo jak konkrétně ověřit a nastavit identity propagaci, nebo jak monitorovat celý flow end-to-end?


# upřesnení

Dle devops týmu umožňuje aktuální setup - inicializace nového aplikačního  private účtu je v CDK šabloně - rychlé vystavění prostředí pro novou aplikaci. Proxy účet je také v CDK, takže umožňuje rychlé vystavení pro testovací účely

Dobře, to je důležitý kontext a mění to hodně z mých předchozích obav. Pojďme si to přehodnotit.

## Co tato informace mění

Pokud máte **CDK šablonu pro dedicated aplikační účet** a **CDK šablonu pro proxy účet**, tak většina friction bodů, které jsem předpokládal, neexistuje:

- Nová aplikace = `cdk deploy` s parametry, ne týdny manuální práce
- Testovací prostředí = stejný mechanismus, paralelní deployment
- Konzistence napříč účty je vynucená kódem, ne disciplínou týmů
- Onboarding nové aplikace je operační záležitost, ne architektonický projekt

To je **silný argument pro zachování současného setupu**. Infrastructure-as-Code, který umí rychle stvořit plné izolované prostředí včetně zero trust networkingu, je reálně cenná věc, kterou mnoho organizací nemá.

## Revize mých dřívějších doporučení

**Strategie 1 (standardizace a automatizace):** Už ji máte. CDK šablona plní roli, kterou by jinak plnil Terraform modul. Hotovo.

**Strategie 2 (service mesh):** Nepotřebné. Benefit service mesh je právě odstranění onboarding friction, který u vás CDK řeší jinou cestou. Přechod na App Mesh by znamenal vyhodit funkční řešení za nefunkční komplexitu.

**Strategie 3 (zjednodušit na ALB):** Bylo by to levnější a jednodušší v runtime, ale ztratili byste konzistenci „všechno vypadá stejně" a musel byste migrovat existující aplikace. ROI je nízké.

**Zjednodušené doporučení: nechat setup být, je dobrý.**

## Co zbývá za reálné věci k řešení

Vyřazuju všechno ostatní a zaměřím se jen na to, co má smysl při znalosti existence CDK:

### 1. Identity propagace (stále otevřené)

Pořád stojí za to zjistit, jak identity teče z authorizer #1 do authorizer #2. Pokud druhý authorizer validuje originální JWT od klienta — zero trust je real. Pokud validuje něco, co vygeneroval první — zero trust je marketing.

Toto není kritika setupu, je to spíš _kontrola vlastního chápání_. Architektura může být dobře, ale pokud nikdo v týmu neumí odpovědět, jak propagace funguje, je to znalostní mezera, která potenciálně zahřeje v budoucnu.

### 2. Testability celé flow

Když CDK vystavuje prostředí rychle, otevírá to otázku **end-to-end testů**:

- Má tým automatizovaný test, který po `cdk deploy` ověří, že mobilní request prochází celým flow až do backendu?
- Testuje se, že resource policy Private API Gateway skutečně odmítá requesty z neočekávaných VPC endpointů?
- Testuje se, že Lambda authorizer #2 odmítne request, který nemá valid JWT (třeba pokus o přímé volání s fake tokenem)?

Pokud CDK deploy je rychlý, smoke test suite po deployi by měl být taky. Hodnota CDK se násobí automatizovanou verifikací.

### 3. Drift mezi CDK verzí a existujícími účty

Pokud CDK šablona se vyvíjí, jak se propisují změny do **existujících** aplikačních účtů? Tři scénáře:

- **Continuous sync** — CI pipeline periodicky deployuje aktuální šablonu do všech existujících účtů. Nejčistší, ale vyžaduje, aby šablona byla vždy zpětně kompatibilní.
- **On-change deploy** — šablona se deployuje jen když se mění infrastruktura dané aplikace. Starší účty zaostávají.
- **Manuální** — drift se ignoruje, dokud nezabolí.

Pokud druhá nebo třetí varianta, po roce provozu budete mít „archeologické vrstvy" různých verzí setupu napříč účty, které se těžko debuggují.

### 4. Observability napříč účty

Při multi-account setupu je nativní CloudWatch view per-account. Distributed tracing přes X-Ray napříč účty vyžaduje explicitní setup:

- **X-Ray tracing zapnutý** na obou API Gateway (public i private)
- **Sdílený pozorovací účet** (observability account) kam se streamují logy a metriky z aplikačních účtů
- **Trace ID propagace** přes NLB (NLB je L4, neupravuje headers, takže by to mělo fungovat nativně, ale ověřit)

Bez toho je debugging flow „mobilní → public API → NLB → private API → backend" detektivní práce. S tím je to trace v jednom UI.

### 5. Limit scaling

AWS má soft limity, které při škálování narazí:

- **VPC Links na API Gateway**: default 20 per account per region
- **VPC Endpointy**: default 50 per VPC
- **Target groups na NLB**: 3000 per region (ne problém)
- **Private API Gateway**: 600 per region

Pokud plánujete desítky nebo stovky aplikací v proxy účtu, stojí za to si spočítat, kdy narazíte na limity a jestli je stačí zvednout supportem, nebo je potřeba strukturální změna (víc proxy účtů per doména).

### 6. Disaster recovery a region failover

Setup je v jednom regionu. Pokud mobilní aplikace musí být odolná proti výpadku celého regionu:

- Je CDK šablona multi-region ready?
- Public API Gateway custom doména má failover na sekundární region přes Route 53?
- Private API Gateway v backend účtech má repliky v jiném regionu?

Pokud to není požadavek (SLA umožňuje delší výpadek), tak good. Pokud to požadavek je, teď je čas ověřit.

## Závěrečné shrnutí

Váš setup **není nad-inženýrovaný**, pokud platí, že:

1. Zero trust je reálný požadavek (ne cargo cult)
2. CDK skutečně dělá onboarding rychlým
3. Účtů bude víc než pár

Za těchto podmínek je to **rozumná architektura pro enterprise mobilní backend** s jasnou cestou pro škálování. Kritiky, které jsem dřív vznesl (NLB, dvojí API Gateway, cena), jsou buď vyvrácené (NLB je vynucený REST API, cena při 40M je nevýznamná), nebo ospravedlněné zero trust požadavkem a CDK automatizací.

**Co opravdu stojí za pozornost:**

1. **Zjistit, jak funguje identity propagace** — zero trust check
2. **Smoke testy po CDK deployi** — pokud ještě neexistují
3. **Strategie driftu šablon** napříč existujícími účty
4. **Distributed tracing přes X-Ray** napříč účty, pokud ještě není
5. **Spočítat AWS limity** vzhledem k plánovanému počtu aplikací
6. **Multi-region / DR strategie** pokud je požadavek

Toto jsou operační a provozní body, ne architektonické. Což znamená, že architektura je OK a řešíte běžné enterprise operations questions — to je dobrá pozice.

---

Na kterém z těch bodů by dávalo smysl zapracovat jako první, nebo je ještě něco jiného, co vás zajímá?