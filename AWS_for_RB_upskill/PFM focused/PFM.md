*Jsi datový architect. V bance je core databáze (oracle), transakce z ní se posílají do aplikace (Postgres), která je obohacuje, ukládá u sebe a obohacené posílá zpět do core systému a do dalších systémů. Core databáze je poté vystavuje do read copy pro čtení internetovým bankovnictvím. Core obohacené transakce také posílá do DWH pro analýzu na konci dne. a co je špatně a co dobře na tomto uspořádání? Jak toto uspořádání efektivně změnit a read copy přesunout z core databáze a kam? Dává to celé smysl? Jak by to řešil datový architekt?*


Tohle je skvělé téma k rozebrání. Pojďme na to systematicky.

## Co máš dnes

Shrnu si nejdřív tok dat, jak jsi ho popsal: Core (Oracle) → Obohacovací aplikace (Postgres) → zpět do Core → Core vystavuje read copy pro IB → Core posílá do DWH na konci dne.

## Co je na tom dobře

**Centrální zdroj pravdy existuje** — Core Oracle funguje jako "system of record" pro transakce, což je v bance správně. Regulátor chce vědět, kde je autoritativní zdroj.

**Oddělení čtení a zápisu** — to, že existuje read copy pro internetové bankovnictví, je správný instinkt. IB by nemělo šahat přímo do transakčního jádra.

**Obohacení mimo core** — vyčlenění obohacovací logiky do samostatné aplikace je rozumné. Core systémy bývají rigidní a drahé na změny.

## Co je na tom špatně

Tady je to zajímavější a problémů je víc:

**1. Core je přetížený rolemi.** Dnes plní čtyři věci najednou — transakční zpracování, příjem obohacených dat zpět, vystavování read copy a feed do DWH. To je příliš mnoho odpovědností pro jeden systém. Každá z těch rolí má jiný profil zátěže a jiné SLA.

**2. Ping-pong pattern.** Transakce jde z Core do Postgresu, obohatí se a vrací se zpět do Core, aby se pak teprve distribuovala dál. Proč musí obohacená data projít zpět přes Core? Core se stává zbytečným prostředníkem — bottleneckem a single point of failure.

**3. Read copy na Core je drahá.** Oracle licence se platí per-core a read replika na Oracle infrastruktuře žere licence úplně stejně jako produkční instance. Navíc read copy zatěžuje Core resources (síť, storage, CPU na replikaci).

**4. DWH feed na konci dne je zastaralý vzor.** Batch na konci dne znamená, že analytici a risk pracují s daty starými až 24 hodin. V moderním bankovnictví chceš near-real-time data pro fraud detection, AML, liquidity reporting.

**5. Těsné provázání (tight coupling).** Pokud Postgres obohacovač spadne, Core nedostane obohacená data zpět, IB nemá kompletní view, DWH nedostane kompletní feed. Kaskádový efekt.

## Jak by to řešil datový architekt

Klíčová změna: **zavést event-driven architekturu s dedikovanou datovou distribuční vrstvou.**

Cílový stav by vypadal zhruba takhle:

**Core (Oracle)** — zůstává system of record, ale POUZE pro transakční zpracování. Publikuje eventy ("transakce vytvořena") do message brokeru (Kafka).

**Kafka jako páteř** — každá změna v Core se publikuje jako event. Kafka slouží jako distribuční vrstva, která odděluje producenty od konzumentů.

**Obohacovací služba (Postgres)** — konzumuje z Kafky, obohacuje, publikuje obohacené eventy zpět do Kafky. Neposílá nic přímo zpět do Core, pokud to Core nepotřebuje pro svou transakční logiku.

**Read store pro IB** — tohle je ta klíčová odpověď na tvou otázku. Read copy přesuneš z Core do **dedikované read-optimalizované databáze**. Kandidáti:

- **PostgreSQL** (pokud už ho máte, znáte ho, umíte ho provozovat)
- **MongoDB / Couchbase** (pokud IB potřebuje flexibilní document model)
- **Elasticsearch** (pokud je důležité fulltextové vyhledávání v transakcích)

Tenhle read store konzumuje obohacené eventy přímo z Kafky. Nepotřebuje Core jako prostředníka.

**DWH** — konzumuje taky z Kafky, ideálně near-real-time streaming místo batch feedu na konci dne. Můžeš použít Kafka Connect do DWH přímo.



## Shrnutí změnTeď ti nakreslím oba modely — současný stav a cílový stav — a pak shrnu klíčové principy.A teď cílový stav s event-driven architekturou:## Klíčové principy cílové architektury

**Kam přesunout read copy:** Pryč z Oracle do dedikovaného read store — nejpravděpodobněji PostgreSQL (pokud ho tým už zná z obohacovací služby) nebo MongoDB (pokud IB potřebuje flexibilnější model). Tento read store konzumuje obohacené transakce přímo z Kafky, takže je vždy aktuální a nepotřebuje Core jako prostředníka.

**Proč Kafka jako páteř:** Kafka řeší hlavní strukturální problém — tight coupling. Dnes když přidáš nového konzumenta (nový systém, který potřebuje transakce), musíš měnit Core. S Kafkou nový konzument jen čte z topicu. Core o něm nemusí vědět. Navíc Kafka drží historii zpráv, takže nový systém si může "přehrát" historické eventy.

**CDC (Change Data Capture):** Aby se nemusela měnit aplikační logika v Core systému, nasadíš Debezium, který čte Oracle redo logy a automaticky publikuje změny do Kafky. Core nemusí "vědět", že do Kafky posílá — Debezium to dělá transparentně.

**Co to přinese konkrétně:**

- Ušetříš Oracle licence za read repliku (to může být v řádu stovek tisíc korun ročně)
- DWH dostane data near-real-time místo jednou za den
- Výpadek obohacovací služby nezablokuje IB — read store má vlastní data
- Přidání nového konzumenta (mobilní appka, risk engine, AML) je otázka dní, ne měsíců

**Rizika a co zvážit:** Kafka přidává infrastrukturní složitost — potřebuješ tým, který ji umí provozovat (nebo managed Kafka jako Confluent Cloud). Musíš vyřešit schema management (Avro + Schema Registry), monitoring lag konzumentů, a eventual consistency — read store bude za Core o milisekundy až sekundy, ne synchronně. Pro bankovnictví to ale typicky stačí.

Dává to celé smysl? Ano, současná architektura je funkční, ale má v sobě dva zásadní dluhy — Core jako bottleneck a batch DWH. Přechod na event-driven model je v bankách dnes standardní cesta (většina tier-1 bank v EU na tom pracuje nebo to má hotové).