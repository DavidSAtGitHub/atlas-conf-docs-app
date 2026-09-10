
Příklad: Kategorizace transakcí v mobilním bankovnictví
Feature overview
Název: Automatická kategorizace transakcí s možností úpravy uživatelem
Business cíl: Uživatel vidí přehled útrat podle kategorií; kategorie přiřazuje BFF pomocí obohacení daty 3. strany (merchant enrichment service); uživatel může kategorii přepsat a oprava se promítne do budoucích podobných transakcí.

Epic → User stories
EPIC-101 Kategorizace transakcí
	•	US-101.1 Jako uživatel vidím u každé transakce kategorii a ikonu
	•	US-101.2 Jako uživatel mohu kategorii transakce změnit
	•	US-101.3 Jako uživatel vidím měsíční přehled útrat po kategoriích
	•	US-101.4 Jako systém obohatím transakci o merchant data ze 3. strany

US-101.2: Editace kategorie transakce – rozpad po vrstvách
Klientská vrstva (mobilní app)
REQ-MOB-101.2.1 — Detail transakce, editace kategorie
Popis: Na detailu transakce uživatel tapne na kategorii, otevře se bottom sheet se seznamem kategorií, vybere novou, sheet se zavře, kategorie se okamžitě překreslí (optimistic update).
Akceptační kritéria:
	•	Given uživatel je na detailu transakce When tapne na řádek “Kategorie” Then se otevře bottom sheet se seznamem 12 hlavních kategorií + “Vlastní”
	•	Given bottom sheet je otevřený When uživatel vybere kategorii Then se sheet zavře do 200 ms a UI okamžitě zobrazí novou kategorii
	•	Given API volání selže When přijde error response Then se UI vrátí na původní kategorii a zobrazí se toast “Kategorii se nepodařilo změnit, zkuste to znovu”
	•	Given uživatel je offline When změní kategorii Then se změna uloží do fronty a synchronizuje při obnovení připojení
UX: Figma odkaz, bottom sheet komponent z design systému, ikony 24×24 dp.
Telemetrie: event transaction_category_changed s atributy from_category, to_category, transaction_id_hash, source: manual.
A11y: VoiceOver/TalkBack popisek “Kategorie: Jídlo a pití, dvojklikem upravit”.

Aplikační vrstva (orchestrace mobile + BFF)
REQ-APP-101.2.2 — Flow editace kategorie
Popis: Uživatelská změna kategorie se propaguje do BFF, BFF aktualizuje transakci v core systému a zároveň posílá feedback do enrichment služby pro učení.
Flow:
	1.	App volá PATCH /transactions/{id} s { "categoryId": "FOOD_DRINK" }
	2.	BFF validuje vstup, ověří vlastnictví transakce (user ID z JWT vs. owner transakce)
	3.	BFF zapíše override do transaction-store (vlastní override má vždy přednost před auto-kategorií)
	4.	BFF asynchronně publikuje event CategoryOverridden na message bus pro enrichment službu (učení)
	5.	BFF vrátí 200 s aktualizovaným objektem transakce
	6.	App invaliduje cache pro list transakcí a měsíční souhrn
Akceptační kritéria:
	•	Given validní request When uživatel je vlastník transakce Then BFF vrátí 200 do 500 ms (P95)
	•	Given uživatel není vlastník When přijde request Then BFF vrátí 403
	•	Given transakce neexistuje When přijde request Then BFF vrátí 404
	•	Given enrichment služba je nedostupná When se publikuje event Then flow nepadá, event jde do dead-letter queue
	•	Idempotence: opakovaný PATCH se stejnou hodnotou je no-op a vrátí 200
Error handling: mapování backend chyb na user-friendly hlášky v appce (tabulka error code → text).

Backend vrstva (BFF + enrichment + core)
REQ-BFF-101.2.3 — API endpoint PATCH /transactions/{id}
Kontrakt (OpenAPI výňatek):

paths:
  /transactions/{id}:
    patch:
      parameters:
        - name: id
          in: path
          required: true
          schema: { type: string, format: uuid }
      requestBody:
        content:
          application/json:
            schema:
              type: object
              properties:
                categoryId:
                  type: string
                  enum: [FOOD_DRINK, GROCERIES, TRANSPORT, ...]
                note:
                  type: string
                  maxLength: 200
      responses:
        '200': { $ref: '#/components/schemas/Transaction' }
        '400': validation error
        '401': unauthorized
        '403': not owner
        '404': not found
        '409': conflict (concurrent edit)
        '429': rate limited


Nefunkční požadavky:
	•	Latence P95 < 500 ms, P99 < 1 s
	•	Dostupnost 99,9 % měsíčně
	•	Rate limit 60 req/min/user
	•	Audit log každé změny (kdo, kdy, z čeho na co) s retencí 5 let (regulatorní)
	•	Šifrování v klidu i v přenosu, PII maskování v logu
REQ-BFF-101.2.4 — Enrichment pipeline (asynchronní obohacení)
Popis: Při příchodu nové transakce z core systému BFF obohatí transakci o merchant data ze 3. strany (např. Mastercard MRS, Plaid, vlastní enrichment provider).
Flow:
	1.	Core systém publikuje TransactionCreated event
	2.	Enrichment worker čte event, extrahuje merchant string a MCC kód
	3.	Worker volá 3rd party API POST /enrich s payloadem
	4.	Provider vrátí { merchantName, merchantLogo, category, confidence }
	5.	Worker uloží enrichment do enrichment-store (klíč = transaction ID)
	6.	Pokud existuje user override pro daného merchanta, použije se kategorie z override
	7.	Mobile app při fetchi transakcí dostává už obohacenou verzi
Akceptační kritéria:
	•	Given nová transakce When je publikován event Then enrichment proběhne do 5 s v 95 % případů
	•	Given 3rd party je nedostupné When worker volá API Then retry s exponential backoff (3 pokusy), pak fallback na default kategorii UNCATEGORIZED
	•	Given confidence < 0,7 When přichází odpověď Then kategorie se označí jako low_confidence a v UI se zobrazí ikona “?”
	•	Given uživatel dříve override kategorie pro merchanta When přijde nová transakce stejného merchanta Then použije se uložená user preference
Nefunkční:
	•	Throughput 1000 transakcí/s peak
	•	Cache enrichment výsledků 30 dní (úspora volání 3. strany, snížení nákladů)
	•	Circuit breaker na 3rd party API (otevře se po 50 % chybách za 1 min)
	•	Smlouva s providerem: SLA 99,5 %, max 100 ms latence, GDPR DPA podepsaná
	•	PII odesílaná do 3. strany: pouze merchant string a MCC, žádné jméno klienta nebo částka v plain textu

Křížové požadavky (napříč vrstvami)
Bezpečnost: OAuth 2.0 + PKCE pro mobile, JWT s krátkou expirací (15 min) + refresh token, certificate pinning v appce, mTLS mezi BFF a core.
Observability: distributed tracing (trace ID propaguje z mobilu přes BFF do enrichment), structured logy, metriky transaction_category_change_total, enrichment_latency_seconds, enrichment_3rd_party_errors_total.
Testování: unit (každá vrstva), kontraktové (Pact mezi mobile a BFF), integrační (BFF + enrichment + mock 3rd party), E2E (vybrané happy paths v staging), performance (k6 na BFF endpointy).

Šablona user story (k znovupoužití)

ID: US-XXX.Y
Title: 
As a [role] I want [capability] so that [benefit]

Acceptance criteria (Gherkin):
  Given... When... Then...

Layer breakdown:
  Client (mobile):  REQ-MOB-...
  Application:      REQ-APP-...
  Backend (BFF):    REQ-BFF-...

Non-functional:
  Performance:  ...
  Security:     ...
  Observability:...

Dependencies:
  - upstream/downstream systems
  - design assets (Figma)
  - API contract (OpenAPI link)

Definition of Done:
  - kód mergnutý, code review
  - unit + kontraktové testy zelené
  - dokumentace API aktualizovaná
  - telemetrie ověřená v dashboardu
  - PO acceptance


Chceš to celé hodit do markdown souboru, nebo rozpracovat další story (třeba US-101.4 enrichment flow detailněji, nebo měsíční souhrn s agregacemi)?​​​​​​​​​​​​​​​​