*elaboruj nad certifikáty používanými v šifrované komunikaci. Co to je, jak je získat, kdo je vydává, jak vím že je pravý atd. Nejprve obecně a poté pro konkrétní aplikaci  -  komunikace mezi službami v AWS , internetové a mobilní bankovnictví, https a další. Jaký je vztah mezi certifikáty a klíči používanými k šifrování dat at rest a in transtit v AWS? Elaboruj i nad tím.*

Velké téma, jdeme na to. Rozdělím to na: základy kryptografie, certifikáty a PKI, TLS protokol, AWS implementace, a vztah k šifrování at rest.

## Kryptografické základy

Abychom mluvili o certifikátech, potřebujeme rozumět dvěma typům šifrování:

### Symetrické šifrování

Jeden klíč šifruje i dešifruje. Rychlé, efektivní, vhodné pro velké objemy dat. Problém: jak klíč bezpečně předat druhé straně? Pokud ho pošlete po síti, útočník ho odposlechne.

Algoritmy: **AES-128**, **AES-256** (industry standard), ChaCha20.

### Asymetrické šifrování (public-key cryptography)

**Dva matematicky propojené klíče**: privátní a veřejný. Co zašifrujete jedním, rozšifrujete jen druhým. Veřejný klíč můžete publikovat celému světu, privátní držíte v tajnosti.

Klíčové vlastnosti:

- **Šifrování**: kdokoli zašifruje veřejným klíčem, jen držitel privátního klíče dešifruje
- **Podepisování**: držitel privátního klíče podepíše, kdokoli ověří veřejným klíčem

Algoritmy: **RSA** (2048, 3072, 4096 bit), **ECDSA** (Elliptic Curve, menší klíče se stejnou bezpečností), **EdDSA** (Ed25519).

Asymetrické je pomalé, vhodné jen pro malá data. V praxi se používá **hybridní přístup**: asymetricky se vymění symetrický klíč, dál se komunikuje symetricky. Přesně tohle dělá TLS.

## Problém: jak vím, komu patří veřejný klíč?

Veřejný klíč sám o sobě je jen náhodný řetězec bajtů. Když dostanu veřejný klíč, jak poznám, že skutečně patří bance, a ne útočníkovi, který se za banku vydává?

**Odpověď: certifikát.** Certifikát je **veřejný klíč + metadata + podpis důvěryhodné třetí strany**, která garantuje, že klíč patří tomu, komu má.

## Co je certifikát

**X.509 certifikát** je standardizovaný formát obsahující:

- **Subject** — komu certifikát patří (např. `CN=www.banka.cz, O=Banka a.s., C=CZ`)
- **Subject Public Key** — veřejný klíč vlastníka
- **Issuer** — kdo certifikát vydal (Certificate Authority)
- **Validity** — od kdy do kdy platí (`Not Before`, `Not After`)
- **Serial Number** — unikátní ID v rámci issuer
- **Signature Algorithm** — čím je podepsán (např. SHA256withRSA)
- **Signature** — podpis vydavatele
- **Extensions** — dodatečné informace:
    - **Subject Alternative Names (SAN)** — alternativní doménová jména (`www.banka.cz`, `banka.cz`, `api.banka.cz`)
    - **Key Usage** — k čemu se smí použít (digital signature, key encipherment...)
    - **Extended Key Usage** — specifické použití (server auth, client auth, code signing)
    - **CRL Distribution Points** — URL pro kontrolu zneplatnění
    - **Authority Information Access** — URL pro OCSP

### Formáty certifikátů

Stejný certifikát může existovat v různých souborových formátech:

- **PEM** — Base64 text mezi `-----BEGIN CERTIFICATE-----` a `-----END CERTIFICATE-----`. Nejběžnější pro servery, Linux.
- **DER** — binární varianta, typicky pro Javu, Windows
- **PKCS#12 (.p12, .pfx)** — kontejner obsahující certifikát + privátní klíč, chráněný heslem. Pro import do systémů, které chtějí obojí najednou.
- **PKCS#7 (.p7b)** — kontejner pro certifikát + chain, bez privátního klíče
- **JKS** — Java KeyStore, legacy Java formát

## Certificate Authority (CA) a PKI

**Certificate Authority** je organizace, která **vydává certifikáty**. Svým podpisem garantuje, že veřejný klíč v certifikátu skutečně patří subjektu uvedenému v `Subject`.

### Jak se pozná důvěryhodná CA

Operační systémy a prohlížeče mají **trust store** — seznam předinstalovaných **root certifikátů** CA, kterým důvěřují „ze začátku". Windows má svůj, macOS/iOS svůj (různé), Firefox svůj vlastní, Android svůj. Linux typicky používá ca-certificates balík.

Root CA prošly auditovacími procesy (WebTrust, ETSI) a operační systém/prohlížeč jim důvěřuje. Seznam důvěryhodných CA se periodicky aktualizuje.

### Chain of trust

Root CA přímo nepodepisují koncové certifikáty — to by bylo riskantní (pokud by někdo ukradl root privátní klíč, ohrozilo by to celý systém). Místo toho existuje **hierarchie**:

```
Root CA (self-signed, offline, velmi chráněné)
   ↓ podepisuje
Intermediate CA (online, vydává koncové certifikáty)
   ↓ podepisuje
End-entity certificate (certifikát vašeho serveru)
```

Když prohlížeč dostane certifikát serveru, ověřuje **celý řetězec**:

1. Je certifikát serveru podepsán Intermediate CA? Ano (podpis matematicky sedí).
2. Mám Intermediate CA? Ne přímo v trust store, ale…
3. Je Intermediate CA podepsána Root CA? Ano.
4. Mám Root CA v trust store? Ano.
5. Důvěřuji celému řetězci.

**Server musí poslat nejen svůj certifikát, ale i intermediate certifikáty** (chain). Pokud chain není kompletní, prohlížeč nemůže ověřit důvěru, i když certifikát je technicky platný.

### Self-signed certifikáty

Certifikát podepsaný sám sebou — `Issuer` = `Subject`. Je technicky platný, ale **nikdo mu nedůvěřuje ze začátku**, protože ho nepodepsala žádná CA. Použití: interní testování, vnitřní infrastruktura s vlastní CA, development.

Browser při self-signed certu zobrazí „Your connection is not private" — protože nemá v trust store podpisovatele.

## Jak získat certifikát

### Certificate Signing Request (CSR)

Proces:

1. **Vygenerujete key pair** (privátní + veřejný klíč) na serveru. Privátní nikdy neopustí server.
2. **Vytvoříte CSR** — požadavek obsahující váš veřejný klíč, Subject info, podepsaný vaším privátním klíčem (dokazuje, že máte odpovídající privátní klíč).
3. **Pošlete CSR na CA**.
4. **CA validuje** váš požadavek — různě intenzivně podle typu certifikátu.
5. **CA vydá certifikát** — váš veřejný klíč + Subject + podpis CA.
6. **Instalujete certifikát** na server. Server má teď svůj privátní klíč + vydaný certifikát.

### Typy validace (a tedy typy certifikátů)

**Domain Validation (DV)** — CA ověří, že kontrolujete doménu. Nejčastěji přes:

- DNS challenge (přidat TXT záznam na doménu)
- HTTP challenge (umístit soubor na `http://doména/.well-known/...`)
- Email challenge (poslat email na `admin@doména`)

Automatizovatelné, rychlé (minuty), levné nebo zdarma. **Let's Encrypt** je nejznámější DV CA, zdarma, přes ACME protokol.

**Organization Validation (OV)** — CA navíc ověří existenci organizace (obchodní rejstřík, telefonát). Trvá dny, stojí peníze. Subject obsahuje ověřené jméno organizace.

**Extended Validation (EV)** — rozsáhlá validace organizace podle CA/Browser Forum požadavků. Dříve zobrazoval prohlížeč zelený pruh s názvem firmy, dnes to prohlížeče zrušily. **EV certifikáty ztratily velkou část své praktické hodnoty**, ale pro finanční instituce jsou stále běžné z compliance důvodů.

### Free vs. paid certifikáty

Pro standardní webové servery je **Let's Encrypt zdarma a dostatečný**. DV úroveň je technicky stejná jako placené DV certifikáty. 90denní platnost vynucuje automatizaci, což je feature, ne bug.

Placené certifikáty mají smysl pro:

- OV/EV úroveň (compliance)
- Wildcard certifikáty (`*.firma.cz`) — Let's Encrypt je umí, ale některé corporate prostředí preferují placené
- Záruky a pojištění (CA garantuje odškodné při chybném vydání)
- Specializované certifikáty (code signing, S/MIME, client authentication)

## TLS handshake — jak se certifikáty používají v praxi

Když otevřete `https://www.banka.cz`, proběhne TLS handshake. Popíšu **TLS 1.3** (modern standard, rychlejší a bezpečnější než starší verze):

1. **Client Hello** — prohlížeč pošle: podporované šifry, TLS verze, náhodné číslo, SNI (Server Name Indication — doména, na kterou se připojuje), seznam podporovaných elliptic curves, Client Key Share (veřejná část klienta pro ECDHE).
    
2. **Server Hello** — server odpoví: zvolená šifra, svůj **certifikát** + chain, Server Key Share (veřejná část serveru pro ECDHE), podpis pro ověření, že vlastní privátní klíč odpovídající veřejnému klíči v certifikátu.
    
3. **Klient ověří certifikát**:
    
    - Platnost (`Not Before` < dnes < `Not After`)
    - Subject / SAN odpovídá doméně (`www.banka.cz` je v SAN?)
    - Podpis sedí
    - Chain končí u root CA v trust store
    - Certifikát není revoked (CRL / OCSP)
    - Server dokázal, že má privátní klíč (podepsal handshake)
4. **Key derivation** — obě strany nezávisle spočítají shared secret pomocí ECDHE (Elliptic Curve Diffie-Hellman Ephemeral). Toto je **forward secrecy** — ani únik privátního klíče serveru v budoucnu nerozšifruje starou komunikaci, protože shared secret nebyl nikdy přenášen.
    
5. **Aplikační data** — dál komunikace symetricky šifrovaná odvozeným session key.
    

### Klíčová observace o TLS

Certifikát serveru **neslouží k šifrování dat**. Slouží k:

1. **Autentizaci serveru** — „tento server je skutečně www.banka.cz"
2. **Bezpečné výměně klíčů** — podpis serveru v handshake dokazuje vlastnictví privátního klíče

Samotná data se šifrují **symetrickým klíčem**, který vznikne z Diffie-Hellman výměny. Certifikát je o důvěře, ne o šifrování obsahu.

## Ověření pravosti — jak vím, že certifikát je validní

Klient (prohlížeč, mobilní aplikace) provádí několik kontrol:

**1. Chain validation** — certifikát → intermediate → root, všechny podpisy sedí, root je v trust store.

**2. Platnost v čase** — systémový čas je mezi `Not Before` a `Not After`. Toto znamená, že **špatně nastavený čas zařízení způsobí TLS chyby**.

**3. Hostname verification** — doména v URL odpovídá `Subject Alternative Names` v certifikátu. Wildcardy `*.firma.cz` matchují jednu úroveň (nematchují `api.v2.firma.cz`).

**4. Revocation check** — byl certifikát zneplatněn před expirací?

- **CRL (Certificate Revocation List)** — CA publikuje seznam zneplatněných certifikátů. Klient ho stáhne. Problém: velké soubory, zastaralé.
- **OCSP (Online Certificate Status Protocol)** — klient se zeptá CA na konkrétní certifikát. Problém: privacy (CA vidí, co uživatel navštěvuje), latency.
- **OCSP Stapling** — server periodicky stáhne OCSP response od CA a přikládá ji k handshake. Klient nemusí kontaktovat CA. Moderní standard.
- **CRLite, Must-Staple** — pokročilejší techniky.

V praxi prohlížeče revocation check dělají laxně — při výpadku OCSP typicky „soft fail" (povolí připojení). Pro vysokou bezpečnost se používá **OCSP Must-Staple** extension, která vynucuje přítomnost stapled OCSP response.

**5. Certificate Transparency (CT)** — CA musí publikovat každý vydaný certifikát do veřejných CT logů. Prohlížeč kontroluje, že certifikát je v CT logu. Tím se odhalují chybně vydané certifikáty (útočník, který přesvědčí CA k vydání certifikátu pro cizí doménu, bude odhalen, protože CT log je veřejný).

## Certificate pinning

Standard TLS říká „důvěřuj jakémukoli certifikátu vydanému důvěryhodnou CA". Problém: **existuje stovky CA**, a pokud _kterákoli_ z nich vydá falešný certifikát pro vaši doménu (chyba, kompromitace, státní tlak), útočník může MITM.

**Certificate pinning** říká: „pro moji doménu akceptuju _jen_ tento konkrétní certifikát (nebo tento public key, nebo tuto CA)". Pokud přijde jiný, i když by byl validní, aplikace odmítne spojení.

Použití:

- **Mobilní bankovnictví** — pinuje certifikáty backend API, ochrana proti MITM i když útočník má validní certifikát
- **Desktop aplikace** komunikující s backendem

Neuplatňuje se pro webové prohlížeče (HTTP Public Key Pinning byl deprecated — riziko zablokování vlastní stránky při rotaci klíčů).

**Rizika:** pokud pinujete a rotujete klíče, starší verze aplikace přestanou fungovat. Řešení: pinovat **záložní klíč** navíc k aktivnímu, nebo pinovat na úrovni CA/intermediate.

## Mutual TLS (mTLS)

Normální TLS: server dokazuje identitu klientovi. **mTLS: oba dokazují identitu**. Klient má svůj certifikát + privátní klíč, server ho validuje stejně jako klient validuje server.

Použití:

- **Service-to-service komunikace** v microservices (každá služba má cert)
- **B2B API** kde obě strany jsou firmy
- **Zero trust networking** — identita je ověřena certifikátem, ne jen IP adresou
- **IoT zařízení** — každé zařízení má unikátní cert

Aplikace: service mesh (Istio, Linkerd, AWS App Mesh), AWS IoT Core.

---

Teď konkrétní aplikace.

## Certifikáty v AWS

### AWS Certificate Manager (ACM)

**ACM je managed služba** pro vydávání a správu certifikátů pro AWS resources. Dvě hlavní použití:

**Public certifikáty (zdarma)** — Amazon je sama public CA (Amazon Trust Services). Vydá certifikát pro vaši doménu po DNS validaci (nebo email validaci). Automatická rotace před expirací. **Platí jen pro použití v AWS službách**, nemůžete si stáhnout privátní klíč. To je intentional — privátní klíč nikdy neopustí AWS, což je security feature.

Podporované služby:

- CloudFront
- Application/Network Load Balancer
- API Gateway (Edge: cert musí být v us-east-1; Regional: v libovolném regionu)
- Elastic Beanstalk
- AWS App Runner
- AWS Amplify

**Jak získat public ACM certifikát:**

1. V ACM požádat o cert pro `api.firma.cz` (+ SAN `*.api.firma.cz` pokud chcete)
2. ACM vygeneruje DNS validation CNAME záznamy
3. Přidat CNAME do DNS (Route 53 má one-click, pro externí DNS manuálně)
4. ACM ověří do pár minut
5. Cert je vydán, automaticky se rotuje před expirací

**Private certifikáty (AWS Private CA)** — pro interní use cases. Spravovaná private CA pro vaši organizaci, vydává certifikáty pro vnitřní služby. Stojí peníze ($400/měsíc za CA + per-cert fee). Použití: service mesh s mTLS, interní API, IoT.

**Import externích certifikátů** — pokud máte cert od jiné CA (Let's Encrypt, DigiCert, Sectigo), můžete ho importovat do ACM. Ztrácíte automatickou rotaci, musíte ji řešit sami.

### Kde se certifikáty používají ve vašem BFF setupu

Projdu flow s konkrétními certifikáty:

**1. Mobilní aplikace → Public API Gateway:**

API Gateway potřebuje TLS certifikát pro custom doménu (`api.firma.cz`). Bez custom domény se používá default `execute-api` URL s *AWS-wildcard certem, pro produkci nepoužitelný. S custom doménou:

- Certifikát z ACM (public)
- Pro Edge-optimized API Gateway: cert musí být v **us-east-1** (jde přes CloudFront)
- Pro Regional API Gateway: cert ve stejném regionu jako API
- ACM cert se připojí k **Custom Domain Name** v API Gateway

Mobilní aplikace validuje cert standardním způsobem (chain to root). Pokud používáte **certificate pinning v mobilní appce**, pinujete buď:

- **ACM certifikát** (problém — ACM rotuje certy automaticky, pin se rozbije)
- **ACM intermediate / root** (Amazon Root CA 1) — stabilní, pin přežije rotaci
- **Amazon public CA** obecně — ochrana proti útokům z jiných CA

Doporučený přístup pro banking app: pin na Amazon Root CA 1 + záložní pin na jiný root (pro případ, že byste migrovali pryč z ACM).

**2. Public API Gateway → NLB (přes VPC Link):**

VPC Link používá TLS. NLB má listener TLS:443 s certifikátem — můžete použít ACM cert. API Gateway validuje cert NLB.

Zde vzniká zajímavá otázka: **jaký hostname API Gateway očekává u NLB?** NLB má interní DNS jméno (`internal-xxx.elb.amazonaws.com`). Certifikát musí být validní pro toto jméno, nebo pro custom doménu, kterou na NLB pointujete.

**3. NLB → VPC Endpoint (Interface Endpoint pro execute-api):**

VPC Endpoint má svůj vlastní TLS certifikát. NLB jako L4 **nedělá TLS termination** — to je klíčový detail. Pokud má NLB listener TLS:443, **NLB terminuje TLS a znovu šifruje k targetu** (toto je relativně nová funkcionalita, „TLS passthrough listener" existovala dřív).

Dvě možné konfigurace:

- **TCP listener na NLB** — NLB passthrough, nedívá se do TLS, target (endpoint) termniuje TLS. Endpoint má svůj cert (managed AWS).
- **TLS listener na NLB** — NLB terminuje a znovu šifruje. Potřebuje cert na NLB side, a TLS k targetu je separátní spojení.

Pro execute-api endpoint je typicky TCP passthrough, protože endpoint má svůj managed cert a přímá TLS spojení.

**4. VPC Endpoint → Private API Gateway:**

Interně v AWS, certifikát managed AWS. Standardní TLS.

**5. Private API Gateway → Backend (ALB, Lambda, EC2):**

API Gateway k backendu používá HTTPS. Pokud backend je ALB, ALB má certifikát (typicky ACM). API Gateway validuje cert ALB.

### CloudFront certifikáty

Pokud byste měli CloudFront distribuci před API Gateway:

- Cert **musí být v us-east-1** bez ohledu na distribuci (globální edge network)
- Buď ACM cert, nebo importovaný
- Pro custom doménu v CloudFront se cert připojí k distribuci

### AWS Private CA pro service mesh

Pokud byste šli cestou service mesh (App Mesh, EKS s Istio):

- Private CA vydává **certifikáty pro každou službu**
- Služby mají svůj cert + privátní klíč (typicky v secret manageru)
- mTLS mezi službami — každá validuje druhou pomocí private CA root
- **Automatická rotace** přes AWS App Mesh / ACM integrace nebo přes cert-manager v Kubernetes

**Pro váš BFF setup s API Gateway toto nepotřebujete** — mTLS se řeší přes resource policies a IAM, ne certifikáty.

## Internetové a mobilní bankovnictví — specifika

Banking má přísnější požadavky než typická aplikace:

### Web banking

**EV certifikát** — i když prohlížeče zrušily zelený pruh, EV cert je stále standard v bankovnictví z regulatorních důvodů. Často vydávaný specifickou CA schválenou regulátorem.

**HSTS (HTTP Strict Transport Security)** — hlavička, která říká prohlížeči „komunikuj se mnou jen přes HTTPS, nikdy HTTP, po dobu X". Preloaded HSTS list v prohlížečích obsahuje bankovní domény — prohlížeč nikdy nepovolí HTTP spojení.

**CSP (Content Security Policy)** — která doména může dodávat JS, obrázky atd. Ochrana proti XSS.

**Certificate Transparency monitoring** — banka monitoruje CT logy a alertuje, pokud někdo vystaví cert pro její doménu (signál útoku).

**Certifikát pro klienta** — některé banky používají client certificate authentication pro přihlášení (token s certem, nebo certifikát v prohlížeči). mTLS na úrovni uživatele. Méně časté, ale v korporátním bankovnictví standard.

### Mobilní bankovnictví

**Certificate pinning je povinnost.** Banking apps pinují:

- Typicky **public key pinning** (hash SPKI), ne celý cert — přežije rotaci certu se stejným klíčem
- **Dva piny minimálně** — primární a záložní (pro plánovanou rotaci)
- **Implementace v native kódu**, ne v JS vrstvě (ochrana proti tamperingu)

**Jailbreak/root detection** — aplikace detekuje rooted zařízení, které by umožnilo útočníkovi nainstalovat vlastní root CA. Pinning je pak obcházený.

**SSL Kill Switch ochrana** — pokročilé banking apps detekují pokusy obejít pinning (Frida, Objection hooks).

**End-to-end šifrování citlivých polí** — i přes TLS se PIN nebo heslo šifruje **dodatečně** aplikačním public key banky, aby ani kompromitace TLS nevedla k úniku. Typicky RSA-OAEP nad TLS.

**Device binding** — aplikace při prvním spuštění zaregistruje device key pair u banky. Každý request podepíše device privátním klíčem. Pokud útočník ukradne tokeny, bez device klíče z telefonu je nepoužije.

### Open Banking / PSD2

Evropská regulace vyžaduje **eIDAS certifikáty** pro komunikaci mezi bankami a third-party providers:

- **QWAC (Qualified Website Authentication Certificate)** — pro TLS mezi bankou a TPP
- **QSealC (Qualified Seal Certificate)** — pro podepisování payload (non-repudiation)
- Vydává specifický **Qualified Trust Service Provider** (QTSP) schválený EU regulátorem
- Neobchází standardní PKI — jsou to X.509 certifikáty s dodatečnými attributes identifikujícími instituci

## Šifrování at rest vs. in transit — kde se setkávají

Tohle je poslední část a často zdroj zmatku. Pojďme si rozseparovat pojmy.

### Šifrování in transit

Data přenášená po síti. **TLS** je dominantní mechanismus. Používá **asymetrické certifikáty** pro handshake a **symetrické klíče** pro data. Klíče jsou **efemérní** — existují jen po dobu session.

**Certifikát ≠ šifrovací klíč dat.** Certifikát slouží k ověření identity a k dohodě session key.

### Šifrování at rest

Data uložená v databázi, na disku, v S3. Používá se **symetrické šifrování** (typicky AES-256). Klíč musí někde existovat a být dostupný, když potřebujete data dešifrovat.

**Certifikáty se zde nepoužívají.** Používají se **kryptografické klíče**, spravované key management systémem.

### AWS KMS (Key Management Service)

**KMS je centrální key management v AWS.** Stará se o:

- Generování klíčů
- Bezpečné uložení klíčů (v HSM — Hardware Security Module)
- Řízení přístupu (IAM policies na klíče)
- Rotaci klíčů
- Auditní logy (CloudTrail)

**KMS nikdy nevydá privátní klíč.** Šifrování/dešifrování se provádí **uvnitř KMS** — pošlete data, KMS vrátí šifrovaná data.

### Typy KMS klíčů

**Symetrické klíče (AES-256)** — pro většinu šifrování at rest. Nejběžnější.

**Asymetrické klíče (RSA, ECC)** — pro podepisování nebo asymetrické šifrování. Méně časté, specifické use cases.

**HMAC klíče** — pro generování message authentication codes.

### Envelope encryption — jak KMS šifruje velká data

KMS má limit 4 KB na operaci. Šifrování TB dat v S3 se neřeší přímo přes KMS. Místo toho:

1. Aplikace požádá KMS o **data key** (symetrický klíč)
2. KMS vrátí:
    - **Plaintext data key** (použitelný okamžitě)
    - **Encrypted data key** (zašifrovaný KMS master key)
3. Aplikace šifruje data data keyem (rychlé, lokálně)
4. Aplikace **smaže plaintext data key z paměti**
5. Uloží šifrovaná data + encrypted data key

Při dešifrování:

1. Aplikace pošle encrypted data key do KMS
2. KMS ho dešifruje master keyem, vrátí plaintext data key
3. Aplikace dešifruje data
4. Smaže plaintext data key

Výhoda: master key nikdy neopustí KMS, data keys jsou efemérní.

### AWS služby a jejich šifrování at rest

**S3:**

- **SSE-S3** — AWS spravuje klíče kompletně, zdarma
- **SSE-KMS** — klíče v KMS, můžete používat vlastní CMK (Customer Master Key), auditní logy
- **SSE-C** — klient poskytuje klíč, AWS ho nepersistuje (specialized)
- **Client-side encryption** — klient šifruje před uploadem, AWS vidí jen šifrovaná data

**EBS (disky)** — šifrování KMS klíčem, transparentní. Snapshoty zachovávají šifrování.

**RDS** — šifrování na storage úrovni přes KMS. Automatické backupy a repliky šifrované stejným klíčem.

**DynamoDB** — default AWS-managed encryption, volitelně KMS CMK.

**Secrets Manager / Parameter Store** — šifrování KMS, transparentní pro aplikaci.

**Lambda environment variables** — šifrování KMS (volitelně CMK).

### Vztah mezi certifikáty a KMS klíči

Tady je klíčová observace, která odpovídá přímo na vaši otázku:

**Certifikáty (TLS) a KMS klíče (at rest) jsou oddělené systémy s různými účely:**

|Aspekt|TLS certifikát|KMS klíč|
|---|---|---|
|Účel|Autentizace + key exchange|Šifrování dat|
|Kryptografie|Asymetrická (RSA, ECC)|Typicky symetrická (AES)|
|Životnost|Měsíce (rotuje se)|Roky (rotuje se ročně nebo méně)|
|Kdo validuje|Chain of trust přes CA|IAM policies|
|Správa|ACM, external CA|KMS|
|Použití|In transit (sítí)|At rest (storage)|

**Ale mohou se potkat:**

- **AWS Private CA používá KMS pro uložení CA privátního klíče.** CA má privátní klíč, který ale sám o sobě žije v KMS/HSM. KMS zde chrání základ PKI infrastruktury.
- **ACM managed certifikáty** — jejich privátní klíče jsou chráněné AWS HSM infrastrukturou (ne user-accessible KMS, ale stejný princip).
- **KMS asymetrické klíče pro podepisování** — můžete mít KMS key pair pro podepisování dokumentů, kde KMS drží privátní klíč. Toto překrývá use case s certifikáty, ale bez PKI a distribuce důvěry.
- **Nitro Enclaves** — bezpečné enklávy pro zpracování citlivých dat s attestation certifikáty.

### Konkrétní flow pro vaši BFF architekturu

Pojďme si ukázat, kde všude se kryptografie objevuje při jednom requestu:

**1. Mobilní app → Public API Gateway:**

- TLS s ACM certem (us-east-1, custom doména)
- Certificate pinning v app na Amazon Root CA 1
- Cert validace proti chain

**2. JWT v requestu:**

- Podepsaný RS256 privátním klíčem IdP
- IdP drží privátní klíč (u Cognito v AWS managed infrastruktuře)
- Authorizer validuje podpis veřejným klíčem z JWKS

**3. API Gateway → NLB → VPC Endpoint:**

- TLS na každém skoku (různé managed certy)
- Interní komunikace v AWS, ale stále šifrovaná

**4. Lambda authorizer:**

- Běží v izolovaném prostředí
- Environment variables šifrované KMS
- Logy v CloudWatch šifrované KMS

**5. Data v backend službě:**

- RDS database šifrovaná KMS CMK
- S3 bucket s user data šifrovaný KMS
- Secrets (database password) v Secrets Manager, šifrované KMS
- EBS disky ECS tasků šifrované KMS

**6. Response zpět:**

- Stejná TLS cesta obráceně
- Response payload může obsahovat citlivá data, šifrovaná jen TLS

### Doporučení pro komplexní šifrovací strategii

Pro banking-grade bezpečnost:

1. **TLS 1.2 minimum, TLS 1.3 preferováno** na všech vstupních bodech
2. **ACM certifikáty s automatickou rotací** kde to jde
3. **Certificate pinning** v mobilní aplikaci
4. **Customer-managed KMS keys (CMK)** pro citlivá data, ne AWS-managed
5. **Key rotation policy** — KMS automatická rotace ročně
6. **Separate KMS keys per environment / per sensitivity tier** — dev/prod, veřejná/citlivá data mají různé klíče
7. **KMS grants a IAM** — granulární kontrola, kdo může používat klíč
8. **CloudTrail logování** KMS operací — audit trail pro compliance
9. **Envelope encryption** pro velká data
10. **Field-level encryption** pro nejkritičtější data (PIN, čísla karet) — šifrované aplikačním klíčem nad rámec DB encryption

---

Takto to vidím napojené dohromady. Kam pokračovat?

- Konkrétnější implementace Private CA pro interní mTLS?
- Jak konkrétně implementovat certificate pinning v mobilní aplikaci?
- Rotace klíčů a certifikátů v CDK/Terraform?
- Compliance aspekty (PCI DSS, PSD2, SOC 2) a co vyžadují?
- HSM a Nitro Enclaves pro nejvyšší bezpečnost?
- Cross-account KMS sharing?