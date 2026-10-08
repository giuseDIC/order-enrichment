# Order Enrichment API

API REST su TIBCO BusinessWorks 6.9: riceve un ordine (`POST /orders`) e restituisce il totale in euro (tasso da Frankfurter) e la data di consegna effettiva (festività da Nager.Date).

## Le due applicazioni

| Applicazione | Porta | Cosa fa |
|---|---|---|
| `OrderEnrichment` | 8080 | l'API |
| `OrderEnrichmentMock` | 8081 | finge di essere Frankfurter e Nager.Date, con risposte fisse |

**Avvio** (Business Studio): *Run → Run Configurations…* → configurazione BusinessWorks → tab *Applications* → spuntare `OrderEnrichment.application` e `OrderEnrichmentMock.application` → *Run*.
**Test**: collection Postman in `postman/`, variabile `baseUrl` = `http://localhost:8080`.

## Mock o servizi reali

Si cambiano solo le module properties di `OrderEnrichment` (*Module Descriptors → Module Properties*), poi si riavvia. Nessun processo cambia.

| Property | Mock | Reale |
|---|---|---|
| `fx.baseUrl` | `http://localhost:8081` | `https://api.frankfurter.dev:443` |
| `nager.baseUrl` | `http://localhost:8081` | `https://date.nager.at:443` |
| `fx.ssl` | `false` | `true` |
| `nager.ssl` | `false` | `true` |

- La porta è esplicita perché la Send HTTP Request riceve host e porta separati, ricavati dall'URL.
- `fx.ssl` e `nager.ssl` attivano *Default Confidentiality* (SSL) nei due HTTP Client.
- Con il mock passano i test della cartella `Giorno1`, con i servizi reali quelli di `Giorno2`.

## Regola della data di consegna

Si parte dalla data richiesta e si avanza di un giorno finché il giorno è feriale e non è una festività del paese. Contano solo le festività con `global` = true e `Public` tra i `types`. Se la data richiesta è a dicembre, si scaricano anche le festività dell'anno dopo, per il cambio d'anno.

`deliveryShiftReason`:
- `NONE` se la data non cambia;
- `HOLIDAY` se la data richiesta è un giorno feriale festivo;
- `WEEKEND` se cade di sabato o domenica, anche quando è anche festa (es. sabato 26/12/2026).

## Cache delle festività

Una Module Shared Variable (`holidaysCache`) tiene una voce per ogni coppia (paese, anno): si legge prima di chiamare Nager.Date e si aggiorna dentro una Critical Section, così due richieste contemporanee non si sovrascrivono. È in memoria, senza risorse esterne, e si svuota al riavvio: le festività di un anno non cambiano. Gli errori non entrano in cache.

## Errori

- `400 VALIDATION_ERROR`: request non valida; `details` elenca tutti i campi sbagliati.
- `400 INVALID_CURRENCY`: Frankfurter risponde 4xx, cioè la valuta non esiste.
