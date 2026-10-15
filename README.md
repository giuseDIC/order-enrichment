# Order Enrichment API

API REST in TIBCO BusinessWorks 6.9. `POST /orders` riceve un ordine e restituisce il totale in euro (cambio da Frankfurter) e la data di consegna effettiva (festività da Nager.Date).

## Applicazioni e avvio

| Applicazione | Porta | Ruolo |
|---|---|---|
| `OrderEnrichment` | 8080 | l'API |
| `OrderEnrichmentMock` | 8081 | simula Frankfurter e Nager.Date, anche con guasti |

Si avviano insieme da Business Studio (Run → *BusinessWorks Application*, con le due applicazioni selezionate).

## Struttura

I processi sono divisi in tre layer.

| Layer | Processo | Ruolo |
|---|---|---|
| Main | `Main_Orders` | riceve `POST /orders`, assegna il `correlationId`, chiama l'orchestrazione e risponde; contiene il Catch e il Catch All |
| Orchestrazione | `Orc_Orders` | valida l'ordine, chiama cambio e data di consegna in parallelo, calcola i totali |
| Sottoprocessi | `ValidateOrder`, `At_FxRate`, `At_DeliveryDate`, `At_Holidays`, `At_ResolveError` | un compito ciascuno |

Nessuna logica sugli End: ogni valore calcolato sta in un Mapper prima dell'End, e l'End copia soltanto.

## Da mock a reale

Si cambiano solo quattro module property di `OrderEnrichment`, poi si riavvia; i processi restano uguali. La porta va scritta esplicitamente.

| Property | Mock | Reale |
|---|---|---|
| `fx.baseUrl` | `http://localhost:8081` | `https://api.frankfurter.dev:443` |
| `nager.baseUrl` | `http://localhost:8081` | `https://date.nager.at:443` |
| `fx.ssl` | `false` | `true` |
| `nager.ssl` | `false` | `true` |

In Postman si sceglie l'environment con lo stesso nome (`mock` o `reale`). Con `mock` devono essere verdi le cartelle Giorno1 e Giorno3, con `reale` la cartella Giorno2.

Guasti del mock: valuta `JPY` → 500 dal cambio; paese `FR` → 500 dalle festività; paese `ES` → nessuna risposta per 10 s.

## Regola della data di consegna

Si parte dalla data richiesta e si avanza di un giorno finché non si trova un giorno feriale che non sia festivo. Contano solo le festività nazionali (`global = true`, tipo `Public`). Per una data di dicembre si scaricano anche le festività dell'anno dopo.

`deliveryShiftReason`: `NONE` se la data non cambia, `WEEKEND` se cadeva di sabato o domenica (anche se festiva), `HOLIDAY` se cadeva in un festivo infrasettimanale.

## Cache delle festività

Una Module Shared Variable in memoria tiene l'elenco per coppia (paese, anno). La scrittura sta in una Critical Section, così due richieste contemporanee non si sovrascrivono.

## Resilienza

- Le chiamate al cambio e alle festività partono **in parallelo**, dopo la validazione.
- Timeout e retry sono module property: `http.timeoutMs` = 3000, `http.retryDelayMs` = 1000, `http.maxRetries` = 1. Si riprova solo su timeout e 5xx, non su 4xx (gruppo *Repeat on Error*).
- **Frankfurter non disponibile** → 502 `UPSTREAM_FX_UNAVAILABLE`.
- **Nager.Date non disponibile** → 200 con la sola regola del weekend e `warnings` = `["HOLIDAY_CHECK_SKIPPED"]`.

## Errori

Gli errori previsti sono fault `ErrorResponse`, lanciati da `Orc_Orders` (ordine non valido) o da `At_FxRate` (cambio). Arrivano al **Catch** di `Main_Orders`, che chiama `At_ResolveError`:

- un **catalogo** (un Choice sul `code`) dà il codice interno e lo status HTTP;
- il log riporta il codice interno con il `correlationId` e il messaggio tecnico di BW;
- il sottoprocesso restituisce la risposta pulita per il client, e il Catch sceglie la Reply in base allo status.

Il **Catch All** prende gli errori imprevisti e resta autonomo (log `OE-999` e risposta 500 con valori fissi).

| Codice interno | Status | `code` | Quando |
|---|---|---|---|
| `OE-001` | 400 | `VALIDATION_ERROR` | ordine non valido; `details` elenca i campi |
| `OE-002` | 400 | `INVALID_CURRENCY` | Frankfurter risponde 4xx (valuta inesistente) |
| `OE-003` | 502 | `UPSTREAM_FX_UNAVAILABLE` | Frankfurter non risponde o risponde 5xx |
| `OE-999` | 500 | `INTERNAL_ERROR` | qualsiasi altro errore |

## Correlation id

Si usa l'header `X-Correlation-Id` della richiesta; se manca se ne genera uno (`OE-<data e ora>-<JobId>`). Compare in ogni riga di log, nell'header `X-Correlation-Id` di tutte le risposte e nel campo `correlationId` delle risposte d'errore.

## Test

Nella cartella `postman/`: la collection (cartelle Giorno1, Giorno2, Giorno3 ed E2E), gli environment `mock` e `reale`, e gli export del Collection Runner con i test verdi.
