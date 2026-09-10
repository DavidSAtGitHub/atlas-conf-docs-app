*elaboruj na JWT a autorizací obecně. Jak fungují Oauth a podobné? Kde se bere klientská identita při otevření aplikace? Jak funguje autentizační a autorizační mechanismus. Všechno popiš obecně a poté konkrétně pro AWS služby.*

Velké téma, pojďme na to systematicky. Nejdřív obecné koncepty, potom konkrétní mechanismy, pak AWS implementace.

## Autentizace vs. autorizace — základní rozdíl

Tyto dva pojmy se plete, ale znamenají různé věci:

**Autentizace (AuthN)** = „kdo jsi?" — ověření identity. Uživatel dokazuje, že je ten, za koho se vydává, typicky heslem, biometrií, MFA tokenem.

**Autorizace (AuthZ)** = „co smíš?" — rozhodnutí, zda daná identita má právo provést konkrétní akci. Uživatel už je autentizovaný, teď se řeší, jestli smí číst tento dokument nebo smazat tento účet.

V praxi tyto dva kroky probíhají odděleně a často i v různých systémech. Identity Provider (IdP) řeší autentizaci, aplikace nebo API Gateway řeší autorizaci na základě informací od IdP.

## JWT — stavební kámen moderní autentizace

**JWT (JSON Web Token)** je formát pro bezpečný přenos informací mezi stranami. Je to **samopodpisový token** — obsahuje data i ověřovací podpis v jednom řetězci.

### Struktura JWT

JWT má tři části oddělené tečkami: `header.payload.signature`

```
eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIxMjM0...5678.SflKxwRJSMeKKF2QT4...
```

Každá část je **Base64URL-encoded JSON**:

**Header** popisuje, jak je token podepsaný:

```json
{ "alg": "RS256", "typ": "JWT", "kid": "key-id-123" }
```

**Payload (claims)** obsahuje data o identitě a kontextu:

```json
{
  "sub": "user-12345",
  "iss": "https://auth.firma.cz",
  "aud": "mobile-app",
  "exp": 1736164800,
  "iat": 1736161200,
  "scope": "read:orders write:orders",
  "email": "jan@example.com"
}
```

Standardní claims:

- `sub` (subject) — identifikátor uživatele
- `iss` (issuer) — kdo token vydal
- `aud` (audience) — pro koho je token určen
- `exp` (expiration) — kdy expiruje (unix timestamp)
- `iat` (issued at) — kdy byl vydán
- `nbf` (not before) — od kdy je platný

**Signature** je kryptografický podpis prvních dvou částí. Ověřuje, že token nebyl změněn a že ho vydal důvěryhodný issuer.

### Jak se JWT podepisuje a ověřuje

Dva hlavní přístupy:

**Symetrický (HS256, HS384, HS512)** — stejný secret podepisuje i ověřuje. Jednoduché, ale všichni, kdo ověřují, musí secret znát. Nepoužívat pro mezi-systémovou komunikaci.

**Asymetrický (RS256, ES256, PS256)** — issuer podepisuje **privátním klíčem**, kdokoli ověří **veřejným klíčem**. Veřejný klíč se publikuje, privátní zůstává jen u issuer. **Standard pro moderní auth systémy.**

Veřejné klíče se publikují přes **JWKS endpoint** (JSON Web Key Set) — veřejně dostupné URL, kde issuer zveřejňuje své aktuální veřejné klíče. Typicky na `https://auth.firma.cz/.well-known/jwks.json`.

Když někdo chce ověřit JWT:

1. Dekóduje header, najde `kid` (key ID)
2. Stáhne JWKS z issuer URL (cachuje)
3. Najde klíč s odpovídajícím `kid`
4. Ověří podpis pomocí tohoto veřejného klíče
5. Zkontroluje claims (`exp`, `iss`, `aud`, scopes...)

### Důležitá vlastnost: JWT je stateless

Validátor nepotřebuje databázi. Ověří podpis matematicky, přečte claims z tokenu. Toto je obrovská výhoda pro škálování — žádný databázový lookup per request — ale i nevýhoda: **token, jednou vydaný, se těžko zruší**. Pokud uživatel udělá logout, token technicky pořád platí, dokud neexpiruje. Řešení později.

## OAuth 2.0 — framework pro delegovanou autorizaci

**OAuth 2.0 není autentizační protokol**, i když se tak často používá. Je to **autorizační framework**, který řeší problém: „Jak může aplikace získat přístup k resource v jiné službě, aniž by uživatel dal své heslo té aplikaci?"

Klasický use case: aplikace chce číst vaše fotky z Google Photos. Starý špatný způsob — dáte aplikaci své Google heslo. OAuth způsob — aplikace dostane od Google token, který ji opravňuje číst fotky, ale ne například mazat účet.

### Klíčové role v OAuth

- **Resource Owner** — uživatel (vlastník dat)
- **Client** — aplikace, která chce přístup (mobilní app, webová app)
- **Authorization Server** — IdP, vydává tokeny (Google, Okta, Cognito, Keycloak)
- **Resource Server** — API, které chrání data (API Gateway, backend)

### OAuth flows (grant types)

Různé scénáře používají různé flows. Nejdůležitější:

**Authorization Code Flow (s PKCE)** — pro mobilní a SPA aplikace. Tento používáte pro svou mobilní app:

1. Aplikace vygeneruje `code_verifier` (náhodný string) a `code_challenge` (SHA256 hash verifier)
2. Aplikace otevře browser na auth server s `code_challenge`
3. Uživatel se přihlásí (heslo, MFA, biometrie) přímo u auth serveru
4. Auth server přesměruje zpět do aplikace s `authorization_code`
5. Aplikace pošle `authorization_code` + `code_verifier` na token endpoint
6. Auth server ověří, že `code_verifier` odpovídá původnímu `code_challenge`, vrátí **access token** + **refresh token** + (volitelně) **ID token**

PKCE (Proof Key for Code Exchange) chrání proti odcizení `authorization_code` — útočník bez původního `code_verifier` ho nemůže vyměnit za tokeny.

**Client Credentials Flow** — pro service-to-service komunikaci, žádný uživatel. Aplikace má `client_id` + `client_secret`, výměnou dostane access token.

**Resource Owner Password Credentials (ROPC)** — aplikace přímo posílá heslo uživatele auth serveru. **Nepoužívat v nových systémech**, porušuje princip OAuth.

**Implicit Flow** — deprecated, token přímo v URL. Nebezpečné, nepoužívat.

**Device Authorization Flow** — pro zařízení bez prohlížeče (smart TV). Uživatel otevře URL na telefonu, zadá kód, zařízení dostane token.

### Typy tokenů v OAuth

**Access token** — krátkodobý token (typicky 15 min - 1 hodina), kterým se autorizuje přístup k API. Může, ale nemusí být JWT (u AWS Cognito je, u jiných IdP může být opaque random string).

**Refresh token** — dlouhodobý token (dny, týdny, měsíce), kterým aplikace získá nový access token bez přihlášení uživatele. **Nikdy se neposílá do API**, jen do auth serveru. Je hodnotnější než access token, musí se pečlivě chránit.

**ID token** — specifický pro OpenID Connect, obsahuje info o uživateli (email, jméno). Tohle je ten „autentizační" kus.

## OpenID Connect (OIDC) — autentizační vrstva nad OAuth

**OAuth řeší autorizaci, ne autentizaci.** OIDC je standardizované rozšíření OAuth 2.0, které přidává autentizaci — způsob, jak aplikace pozná _identitu_ přihlášeného uživatele, ne jen _oprávnění_.

Klíčové dodatky oproti OAuth:

- **ID token** — JWT obsahující identity claims (kdo uživatel je, email, jméno)
- **Standardizovaný `/userinfo` endpoint** — API pro získání dalších informací o uživateli
- **Discovery endpoint** `/.well-known/openid-configuration` — metadata auth serveru (URLs, podporované algoritmy)
- **Standardizované scopes** — `openid`, `profile`, `email`

Když uživatel ve vaší mobilní aplikaci udělá login, backend typicky používá OIDC flow (authorization code + PKCE) a dostane tři tokeny:

- **ID token** — „tohle je Jan Novák, email jan@example.com"
- **Access token** — „nositel smí volat API s těmito scopes"
- **Refresh token** — „s tímto si vyžádej nový access token"

## Kde se bere klientská identita při otevření aplikace

Toto je konkrétní otázka a zaslouží si konkrétní flow. Popíšu standardní mobilní scénář:

### První spuštění aplikace

1. Uživatel otevře aplikaci
2. Aplikace zjistí, že nemá žádné tokeny v secure storage (Keychain na iOS, Keystore na Androidu)
3. Aplikace zobrazí login screen nebo rovnou redirektuje do auth flow
4. Aplikace vygeneruje `code_verifier`, `code_challenge`, `state`
5. Aplikace otevře **system browser** (ne webview!) na URL auth serveru:
    
    ```
    https://auth.firma.cz/authorize?  client_id=mobile-app&  redirect_uri=com.firma.app://callback&  response_type=code&  scope=openid+profile+email+api:read+api:write&  code_challenge=xxx&  code_challenge_method=S256&  state=yyy
    ```
    
6. Uživatel se přihlásí (email + heslo, nebo SSO s Google/Apple, plus MFA)
7. Auth server redirektuje zpět do aplikace přes **custom URL scheme** nebo **universal link**:
    
    ```
    com.firma.app://callback?code=ABC123&state=yyy
    ```
    
8. Aplikace zachytí redirect, ověří `state`, vezme `code`
9. Aplikace zavolá token endpoint (přímé HTTPS, ne přes browser):
    
    ```
    POST https://auth.firma.cz/tokengrant_type=authorization_codecode=ABC123code_verifier=původní_verifierclient_id=mobile-appredirect_uri=com.firma.app://callback
    ```
    
10. Dostane zpět:
    
    ```json
    {  "access_token": "eyJ...",  "refresh_token": "xyz...",  "id_token": "eyJ...",  "token_type": "Bearer",  "expires_in": 3600}
    ```
    
11. Aplikace uloží refresh token do secure storage, access token do paměti nebo secure storage
12. Aplikace začne volat API s `Authorization: Bearer <access_token>`

### Následná spuštění

1. Aplikace se otevře, zkontroluje secure storage
2. Najde refresh token
3. Pokud má access token a ještě neexpiroval, použije ho
4. Pokud access token expiroval nebo neexistuje, použije refresh token:
    
    ```
    POST https://auth.firma.cz/tokengrant_type=refresh_tokenrefresh_token=xyz...
    ```
    
5. Dostane nový access token (a případně nový refresh token)
6. Pokud refresh token je také neplatný (expiroval, byl zrušený, uživatel změnil heslo), aplikace musí zpět do plného login flow

### Průběh API volání

Každý request z aplikace do API:

```
GET /api/orders
Authorization: Bearer eyJhbGci...
```

API Gateway (nebo authorizer) dělá:

1. Vytáhne JWT z `Authorization` headeru
2. Dekóduje header, najde `kid`
3. Stáhne JWKS z issuer (cachuje na hodiny)
4. Najde klíč podle `kid`, ověří podpis
5. Validuje `exp`, `iss`, `aud`
6. Zkontroluje scopes — má tento token `api:read`?
7. Předá identity do backendu (buď JWT dál, nebo jako context claims)

## Řešení logout a token revocation

Jak jsem zmínil, JWT je stateless, takže „zrušení" tokenu je tricky. Několik přístupů:

**Krátká životnost access tokenu** — pokud token žije 15 minut, i při kompromitaci je expozice omezená. Standard v průmyslu.

**Refresh token rotation** — každé použití refresh tokenu vydá nový a zneplatní starý. Pokud útočník ukradne refresh token a použije ho, při dalším pokusu legitimní aplikace dostane error, což signalizuje kompromitaci.

**Token blacklist / denylist** — stateful seznam zneplatněných tokenů, API ho kontroluje. Porušuje stateless princip, ale pro kritické use cases nutné.

**Logout endpoint** na auth serveru — zneplatní refresh token na straně serveru. Access tokeny ale dál fungují do expirace.

**Session ve cookie** místo JWT v headeru — pro webové aplikace alternativa, session lze zrušit serverově. Pro mobilní nepoužitelné.

## Scopes vs. roles vs. claims — autorizační modely

JWT nese informace o uživateli, ale jak se rozhoduje, co smí?

**Scope-based** — token má `scope: "orders:read orders:write users:read"`. Každý endpoint kontroluje, zda token má potřebný scope. OAuth-native přístup, hrubá granularita.

**Role-based (RBAC)** — token má `roles: ["admin", "customer"]`. Aplikace mapuje role na oprávnění. Klasika.

**Attribute-based (ABAC)** — token má claims (attributes), rozhodnutí se dělá na základě kombinace (uživatel + resource + kontext). Např. „může editovat dokument, pokud je jeho vlastník a je v pracovní době". Výraznější, komplexnější.

**Policy-based** — externí policy engine (OPA, AWS Cedar) rozhoduje na základě claims a kontextu. Oddělení auth logiky od aplikačního kódu.

V praxi se kombinují — typicky scopes pro hrubé API-level oprávnění a RBAC/ABAC pro resource-level rozhodnutí v aplikaci.

---

A teď konkrétní AWS implementace.

## AWS Cognito — managed Identity Provider

**Amazon Cognito** má dvě samostatné služby, které se často plete:

### Cognito User Pools

**Full-featured IdP**. Řeší registraci, přihlášení, MFA, zapomenuté heslo, password policies, sociální login (Google, Apple, Facebook), SAML/OIDC federation s korporátními IdP.

User Pool vydává **standardní JWT** přes OAuth 2.0 / OIDC. Dostanete access token, ID token, refresh token. Pro vaši mobilní aplikaci je toto typicky to, co používáte pro auth.

Klíčové features:

- **Hosted UI** — Cognito může hostovat samotnou login stránku, aplikace jen otevře browser na Cognito URL
- **Lambda triggers** — před/po signup, před token generation, custom auth challenges. Umožňuje injektovat custom claims nebo vlastní auth logiku
- **App clients** — rozdělení na public clients (mobilní, bez client_secret) a confidential clients (backend, s client_secret)
- **Groups** — uživatele lze přiřadit do skupin, které se propíší jako claim v tokenu (`cognito:groups`)
- **Custom attributes** — vlastní claims per uživatel

JWKS endpoint Cognito:

```
https://cognito-idp.{region}.amazonaws.com/{userPoolId}/.well-known/jwks.json
```

### Cognito Identity Pools

**Úplně jiná služba** navzdory názvu. Neřeší autentizaci, ale **výměnu tokenů za AWS credentials**. Aplikace pošle Identity Pool JWT (z User Pool, nebo z jiného IdP), Identity Pool vrátí dočasné AWS IAM credentials, kterými lze přímo volat AWS služby (S3, DynamoDB).

Použití: mobilní aplikace přímo nahrává soubory do S3 bez průchodu backendem. Identity Pool vydá credentials s omezenými oprávněními (jen do určitého S3 prefixu).

**Pro váš use case (mobilní → API Gateway) Identity Pool typicky nepotřebujete.** Stačí User Pool a JWT, který API Gateway validuje.

### Integrace Cognito s API Gateway

Tři úrovně integrace:

**Cognito Authorizer** (nativní) — API Gateway má nativní integraci s User Pool. Stačí ukázat na User Pool ARN a API Gateway validuje JWT automaticky. Žádný kód, žádná Lambda. Limitace: málo flexibility, jen základní JWT validace, nemůže dělat custom logiku.

**Lambda Authorizer** — vlastní Lambda funkce, která dostane token a vrátí IAM policy. Flexibilní, umí libovolnou custom logiku (database lookups, dynamic policies, custom claims processing). Víc latence, víc ceny. Toto je varianta, kterou používáte.

**JWT Authorizer** (jen HTTP API) — nativní JWT validace, konfigurovatelná (issuer, audience, scopes). Rychlejší než Lambda authorizer, zdarma. Funguje s jakýmkoli OIDC IdP, ne jen Cognito. Limitace: jen JWT validation, žádná custom logika.

### Co dělá Cognito User Pool pod kapotou

Když uživatel udělá login z mobilní aplikace:

1. Aplikace zavolá Cognito `InitiateAuth` API s username + password (nebo přes hosted UI)
2. Cognito ověří credentials proti User Pool
3. Cognito případně vyvolá MFA challenge
4. Cognito spustí **Pre Token Generation Lambda trigger** (pokud je nastavený) — může modifikovat claims
5. Cognito vygeneruje tři JWT a podepíše je privátním klíčem (Cognito drží klíč)
6. Vrátí tokeny aplikaci

Aplikace pak volá API Gateway s access tokenem. API Gateway (nebo authorizer Lambda) stáhne JWKS z Cognito JWKS endpointu, ověří podpis, zkontroluje claims.

## IAM — autorizace uvnitř AWS

**IAM (Identity and Access Management)** je AWS autorizační systém pro AWS resources, ne pro end-users vaší aplikace. Řeší, co může která AWS identity (user, role, service) dělat s AWS API.

Rozdíl oproti Cognito:

- **Cognito** = koncoví uživatelé vaší aplikace (tisíce, miliony)
- **IAM** = AWS identity (vaši zaměstnanci s AWS konzolí, EC2 instance, Lambda funkce, cross-account access)

V kontextu autorizace API Gateway se IAM objevuje takto:

**IAM Authorizer na API Gateway** — alternativa k JWT. Client podepisuje request AWS SigV4 podpisem (stejně jako volání `aws cli`). API Gateway ověří podpis a IAM policy. Použití: service-to-service komunikace mezi AWS službami, ne pro mobilní aplikace (mobilní app nemá AWS credentials).

**Resource policies na API Gateway** — IAM-style policy na samotné API, kontroluje, kdo může volat API (podle VPC endpoint, IP adresy, source account). Používáte pro Private API Gateway resource policy.

**IAM roles pro Lambda** — Lambda authorizer běží pod IAM rolí, která definuje, co Lambda může dělat (číst Cognito, volat DynamoDB pro custom logiku).

## Jak to sedí dohromady ve vašem BFF setupu

Teď můžu popsat celý flow s vědomím, co se děje na každé vrstvě:

### Login flow (před prvním API voláním)

1. Mobilní aplikace otevře OIDC flow s vaším IdP (Cognito User Pool nebo Auth0 nebo Keycloak...)
2. Uživatel se přihlásí, aplikace dostane access token (JWT), refresh token, ID token
3. Tokeny uloženy v secure storage

### API request flow

1. Mobilní aplikace: `GET /api/orders` + `Authorization: Bearer <JWT>`
2. Request jde na **Public API Gateway** (execute-api URL nebo custom doména)
3. API Gateway invokuje **Lambda authorizer #1**:
    - Lambda vezme JWT z headeru
    - Stáhne JWKS z IdP (cachuje)
    - Ověří podpis, `exp`, `iss`, `aud`
    - Zkontroluje scopes
    - Vrátí IAM policy `Allow` + context (např. `userId`, `tenantId`)
4. API Gateway cachuje výsledek authorizeru (TTL)
5. API Gateway forwarduje request přes VPC Link na NLB
6. NLB forwarduje na VPC Endpoint IP
7. VPC Endpoint forwarduje na **Private API Gateway** v target účtu
8. Private API Gateway **resource policy** kontroluje, že request přišel z očekávaného VPC endpointu
9. Private API Gateway invokuje **Lambda authorizer #2**:
    - Znovu validuje JWT (zero trust)
    - Vrátí policy `Allow`
10. Private API Gateway forwarduje na backend službu
11. Backend zpracuje a vrátí response

### Kde identita „žije" v každém kroku

- **V mobilní aplikaci**: refresh token v Keychain/Keystore, access token v paměti
- **V HTTP requestu**: access token v `Authorization` header
- **V Lambda authorizer**: dekódovaný JWT + extrahované claims
- **V API Gateway context**: to, co authorizer vrátil jako `context` (např. `$context.authorizer.userId`)
- **V backend službě**: buď originální JWT (pokud se propaguje), nebo context headers injektované API Gateway

## Praktické doporučení pro zabezpečení

Věci, které jsou často špatně a stojí za kontrolu:

**1. Mobilní aplikace nesmí používat webview pro login** — musí být system browser (SFSafariViewController na iOS, Chrome Custom Tabs na Androidu). Webview umožňuje aplikaci číst uživatelovo heslo, což je porušení OAuth principu.

**2. PKCE je povinné pro mobilní klienty** — ne volitelné.

**3. Secure storage pro refresh tokeny** — Keychain/Keystore, ne UserDefaults/SharedPreferences.

**4. Certificate pinning** pro API volání — ochrana proti MITM útokům.

**5. Refresh token rotation** — pokud to IdP podporuje, zapnout.

**6. Krátký access token TTL** — 15 minut až 1 hodina, ne dny.

**7. Audience validation** — Lambda authorizer musí validovat `aud`, aby token vydaný pro jinou aplikaci nefungoval.

**8. Issuer validation** — stejně tak `iss`.

**9. Algorithm whitelist** — authorizer musí explicitně kontrolovat, že `alg` je očekávaný (RS256), ne přijmout whatever je v headeru. Známý útok: „alg: none" obchází podpis.

**10. JWKS cache invalidation** — cache JWKS rozumně dlouho (hodiny), ale ne navždy. Když IdP rotuje klíče, staré klíče se časem přestanou publikovat, a pokud cache nikdy neexpiruje, nové tokeny se neověří.

---

Toto je docela hutné téma, takže — chcete na něčem elaborovat detailněji? Třeba:

- Konkrétní implementace Lambda authorizeru a jak vypadá policy, kterou vrací
- Jak konkrétně nastavit Cognito User Pool pro mobilní aplikaci
- Multi-tenant setup (kde každý tenant má vlastní scopes/claims)
- Service-to-service auth mezi backend službami
- Federation s korporátním IdP (SAML/OIDC) pro employee-facing aplikace
- Token propagace napříč microservices (kde každá služba musí znát identitu)
- 