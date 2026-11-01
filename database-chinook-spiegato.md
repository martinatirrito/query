# Il database Chinook, spiegato

Solo il database: niente soluzione della simulazione. Tutti i numeri sono stati contati sul dump del repo (`Chinook_MySql.sql`) il 2026-10-18.

---

## 1. Di cosa parla

Chinook simula un **negozio di musica online** (tipo iTunes): un catalogo di brani organizzati in album e artisti, dei clienti che comprano brani, le fatture di quegli acquisti, i dipendenti che seguono i clienti e delle playlist.

Le 11 tabelle si dividono in **tre blocchi**:

| Blocco | Tabelle | Cosa descrive |
|---|---|---|
| **Catalogo** | `Artist`, `Album`, `Track`, `Genre`, `MediaType` | cosa si vende |
| **Vendite** | `Customer`, `Invoice`, `InvoiceLine`, `Employee` | chi compra cosa, e chi lo segue |
| **Playlist** | `Playlist`, `PlaylistTrack` | raccolte di brani |

Il punto di contatto fra i blocchi è sempre **`Track`**: è il centro del database.

## 2. Mappa delle relazioni

```
                        Genre        MediaType
                          │ 1            │ 1
                          │              │
                          │ N            │ N
 Artist ──1───N── Album ──1───N── Track ──1───N── InvoiceLine ──N───1── Invoice ──N───1── Customer ──N───1── Employee
                                    │                                                                         │  ▲
                                    │ 1                                                                       └──┘
                                    │                                                                      ReportsTo
                                    N                                                                     (self-join)
                              PlaylistTrack
                                    N
                                    │
                                    1
                                 Playlist
```

Come si legge: "un Artist ha N Album", "un Album ha N Track", "una Track appare in N InvoiceLine" e così via.

Le due relazioni **molti-a-molti** sono risolte da tabelle ponte:

- **Track ↔ Invoice** tramite `InvoiceLine`: una fattura contiene molti brani e un brano è venduto in molte fatture.
- **Track ↔ Playlist** tramite `PlaylistTrack`: una playlist contiene molti brani e un brano sta in molte playlist.

## 2bis. Ripasso: cosa fa davvero una JOIN

Una **chiave primaria (PK)** è la colonna che identifica una riga in modo univoco, per esempio `Artist.ArtistId = 1` è AC/DC.
Una **chiave esterna (FK)** è una colonna che **contiene la PK di un'altra tabella**, cioè un "puntatore". `Album.ArtistId = 1` vuol dire "questo album è di AC/DC".

La regola d'oro è che **la FK sta sempre dal lato "molti"**. Un artista ha tanti album, e infatti la colonna `ArtistId` sta in `Album`, non il contrario.

Una JOIN **affianca le righe di due tabelle in cui FK e PK coincidono**:

```sql
SELECT ar.Name, al.Title
FROM artist ar, album al
WHERE ar.ArtistId = al.ArtistId      -- condizione di join: PK = FK
```

```
Artist                     Album                                   Risultato
ArtistId | Name            AlbumId | Title             | ArtistId   Name  | Title
1        | AC/DC     ───►  1       | For Those About…  | 1          AC/DC | For Those About…
                     ───►  4       | Let There Be Rock | 1          AC/DC | Let There Be Rock
2        | Accept    ───►  2       | Balls to the Wall | 2          Accept| Balls to the Wall
                     ───►  3       | Restless and Wild | 2          Accept| Restless and Wild
```

Due conseguenze da ricordare:
- **Le righe si moltiplicano**: AC/DC compare 2 volte perché ha 2 album. Se poi aggiungi `Track` compare una volta per ogni brano. Per questo servono `DISTINCT`, oppure `GROUP BY` + `COUNT`/`SUM`.
- **Chi non ha corrispondenze sparisce**: un artista senza album non esce proprio. Con `LEFT JOIN` invece resta, con NULL al posto dell'album.

`FROM a, b WHERE a.x = b.x` (lo stile usato a lezione) e `FROM a JOIN b ON a.x = b.x` sono **la stessa cosa**.

## 2ter. Ogni collegamento, passo per passo, con dati veri

Seguiamo un acquisto reale attraverso tutto il database.

**① `Album.ArtistId → Artist.ArtistId`: "di chi è l'album"** (N album → 1 artista)
```
Album:  AlbumId=1, Title='For Those About To Rock We Salute You', ArtistId=1
Artist: ArtistId=1, Name='AC/DC'
```
Si legge: "l'album 1 è di AC/DC". AC/DC ha 2 album (1 e 4).

**② `Track.AlbumId → Album.AlbumId`: "in quale album sta il brano"** (N brani → 1 album)
```
Track: TrackId=6, Name='Put The Finger On You', AlbumId=1, GenreId=1, MediaTypeId=1, UnitPrice=0.99
```
L'album 1 contiene 10 brani. Per sapere **l'artista di un brano** fai due salti: `Track.AlbumId → Album`, poi `Album.ArtistId → Artist`. In Track non c'è nessun ArtistId.

**③ `Track.GenreId → Genre.GenreId`: "di che genere è il brano"** (N brani → 1 genere)
```
Genre: GenreId=1, Name='Rock'
```
Il brano 6 è Rock. Il genere è un'etichetta **del singolo brano**: "artisti Rock" vuol dire `Genre → Track → Album → Artist`, tre salti.

**④ `Track.MediaTypeId → MediaType.MediaTypeId`: "che formato è il file"** (1 = MPEG audio file). Funziona come il genere.

**⑤ `InvoiceLine.TrackId → Track.TrackId`: "quale brano è stato comprato"** (N righe → 1 brano)
**⑥ `InvoiceLine.InvoiceId → Invoice.InvoiceId`: "in quale fattura"** (N righe → 1 fattura)
```
InvoiceLine: (Id=3, InvoiceId=2, TrackId=6,  UnitPrice=0.99, Quantity=1)
             (Id=4, InvoiceId=2, TrackId=8,  0.99, 1)
             (Id=5, InvoiceId=2, TrackId=10, 0.99, 1)
             (Id=6, InvoiceId=2, TrackId=12, 0.99, 1)
```
La fattura 2 ha 4 righe, cioè 4 brani comprati (tutti di AC/DC). `InvoiceLine` è la **tabella ponte** tra brani e fatture: ogni riga dice "il brano X è stato comprato nella fattura Y".

**⑦ `Invoice.CustomerId → Customer.CustomerId`: "chi ha pagato"** (N fatture → 1 cliente)
```
Invoice:  InvoiceId=2, CustomerId=4, InvoiceDate=2021-01-02, BillingCountry='Norway', Total=3.96
Customer: CustomerId=4, FirstName='Bjørn', LastName='Hansen', Country='Norway', SupportRepId=4
```
`Total = 3.96 = 4 × 0.99`, cioè la somma delle sue righe. Per sapere "**quali brani ha comprato Bjørn**" fai `Customer → Invoice → InvoiceLine → Track`, perché InvoiceLine da sola non conosce il cliente.

**⑧ `Customer.SupportRepId → Employee.EmployeeId`: "quale dipendente lo segue"** (N clienti → 1 dipendente)
```
Employee: EmployeeId=4, 'Margaret Park', Title='Sales Support Agent', ReportsTo=2
```
Il nome della colonna è diverso (`SupportRepId` contro `EmployeeId`), ma è comunque una FK: `c.SupportRepId = e.EmployeeId`.

**⑨ `Employee.ReportsTo → Employee.EmployeeId`: "chi è il suo capo"** (self-join)
Margaret ha `ReportsTo=2`, cioè Nancy Edwards. La tabella punta **a se stessa**, quindi nella query la usi due volte con due alias:
```sql
SELECT e.FirstName, boss.FirstName
FROM employee e, employee boss
WHERE e.ReportsTo = boss.EmployeeId
```

**⑩ `PlaylistTrack.TrackId → Track` e `PlaylistTrack.PlaylistId → Playlist`: "in quali playlist sta il brano"** (molti-a-molti)
```
PlaylistTrack: (PlaylistId=1, TrackId=1), (8, 1), (17, 1)
```
Il brano 1 sta nelle playlist 1 (Music), 8 (Music, il doppione) e 17 (Heavy Metal Classic). È una tabella ponte pura: ha solo le due FK, che insieme formano la PK.

### Il percorso completo in una query

"Quali clienti hanno comprato brani di AC/DC?"
```sql
SELECT DISTINCT c.FirstName, c.LastName
FROM artist ar, album al, track t, invoiceline il, invoice i, customer c
WHERE ar.ArtistId  = al.ArtistId     -- ① album dell'artista
  AND al.AlbumId   = t.AlbumId       -- ② brani dell'album
  AND t.TrackId    = il.TrackId      -- ⑤ righe di vendita del brano
  AND il.InvoiceId = i.InvoiceId     -- ⑥ fattura della riga
  AND i.CustomerId = c.CustomerId    -- ⑦ cliente della fattura
  AND ar.Name = 'AC/DC'
```
(Risultato: 6 clienti.)
**Metodo**: scrivi le tabelle del percorso in fila e metti **una condizione di join per ogni coppia adiacente** (con N tabelle servono N−1 condizioni). Se ne dimentichi una, ottieni un prodotto cartesiano con milioni di righe: è il classico sintomo.

## 3. Le tabelle una per una

### Catalogo

**`Artist`** (275 righe): `ArtistId` (PK), `Name`
- Il nome è unico (nessun doppione).
- **71 artisti non hanno nessun album**, quindi nessun brano. Se parti da `Artist` e fai una JOIN verso `Track`, spariscono.
- Esiste un artista `Various Artists` (Id 21) per le compilation.

**`Album`** (347): `AlbumId` (PK), `Title`, `ArtistId` (FK → Artist)
- Ogni album ha **un solo** artista.
- Gli artisti con più album sono Iron Maiden (21), Led Zeppelin (14) e Deep Purple (11).

**`Track`** (3503): `TrackId` (PK), `Name`, `AlbumId` (FK), `MediaTypeId` (FK), `GenreId` (FK), `Composer`, `Milliseconds`, `Bytes`, `UnitPrice`
- È **la tabella centrale**.
- ⚠️ **Il genere è del brano, non dell'album né dell'artista.** 11 album hanno brani di generi diversi, e 21 artisti hanno brani in più generi (Iron Maiden in 4). "Gli artisti del genere X" significa quindi "gli artisti con almeno un brano di genere X".
- `Composer` è NULL in 977 brani ed è testo libero (es. `"AC/DC"`, nomi separati da `/`), quindi **non** è una FK verso `Artist`.
- `Milliseconds` va da 1 s a circa 88 min (ci sono episodi TV), con una media di circa 6,5 min.
- `UnitPrice` vale solo 0.99 (audio) o 1.99 (video).
- 445 brani hanno un nome uguale a un altro brano. Per identificare un brano usa sempre `TrackId`, mai `Name`.

**`Genre`** (25): `GenreId`, `Name`
- I generi più grandi sono Rock (1297 brani), Latin (579), Metal (374), Alternative & Punk (332) e Jazz (130). Opera ha 1 solo brano.
- Ci sono generi "non musicali": TV Shows, Drama, Sci Fi & Fantasy, Comedy.

**`MediaType`** (5): `MediaTypeId`, `Name`
- È quasi tutto `MPEG audio file` (3034). I file video sono 214 (`Protected MPEG-4 video file`).

### Vendite

**`Customer`** (59): `CustomerId` (PK), nome, indirizzo, `City`, `State`, `Country`, `Email`, … , `SupportRepId` (FK → Employee)
- Vengono da 24 paesi (USA 13, Canada 8, Francia 5, Brasile 5…). `State` e `Company` sono spesso NULL (solo 10 clienti hanno un'azienda).
- Ogni cliente è seguito da un dipendente "support rep" (solo gli Id 3, 4 e 5).

**`Invoice`** (412): `InvoiceId` (PK), `CustomerId` (FK), `InvoiceDate`, `Billing…`, `Total`
- È **una fattura = un acquisto** di un cliente in una data.
- Le date vanno dal 2021-01-01 al 2025-09-07. In MySQL sono `DATETIME`, quindi si filtrano con `YEAR(InvoiceDate) = 2023`; in Python arrivano come oggetti `datetime`.
- Ogni cliente ha 6 o 7 fatture.
- `Total` è coerente: è sempre uguale a Σ `UnitPrice*Quantity` delle sue righe.

**`InvoiceLine`** (2240): `InvoiceLineId` (PK), `InvoiceId` (FK), `TrackId` (FK), `UnitPrice`, `Quantity`
- È **una riga = un brano comprato** dentro una fattura. Ogni fattura ha da 1 a 14 righe.
- ⚠️ `Quantity` vale **sempre 1**, quindi `COUNT(*)` e `SUM(Quantity)` danno lo stesso numero.
- `UnitPrice` qui coincide sempre con `Track.UnitPrice`.
- Dei 3503 brani **solo 1984 sono stati venduti** (al massimo 2 volte ciascuno), e solo 165 artisti su 275 hanno almeno una vendita. Se costruisci i nodi partendo dalle vendite, perdi tutti gli altri.
- ⚠️ **L'acquisto non sa chi è il cliente**: `InvoiceLine` non ha `CustomerId`. Per arrivare al cliente devi passare da `Invoice`.

**`Employee`** (8): `EmployeeId`, nome, `Title`, `ReportsTo` (FK → Employee), `BirthDate`, `HireDate`, …
- È una **gerarchia** tramite self-join:
  ```
  Andrew Adams (General Manager)
  ├── Nancy Edwards (Sales Manager)
  │   ├── Jane Peacock, Margaret Park, Steve Johnson (Sales Support Agent)
  └── Michael Mitchell (IT Manager)
      └── Robert King, Laura Callahan (IT Staff)
  ```
- Il capo ha `ReportsTo` NULL.

### Playlist

**`Playlist`** (18): `PlaylistId`, `Name`
- ⚠️ Ci sono **nomi duplicati**: `Music` (Id 1 e 8), `Movies`, `TV Shows`, `Audiobooks`. Alcune sono vuote (Movies, Audiobooks).
- `Music` contiene 3290 brani, `90's Music` 1477.

**`PlaylistTrack`** (8715): `(PlaylistId, TrackId)` è una PK composta.
- Ogni brano sta in almeno una playlist.

## 4. I percorsi di JOIN

Quasi ogni query d'esame è uno di questi percorsi, eventualmente con un filtro e un `GROUP BY`:

| Domanda | Percorso |
|---|---|
| brani di un artista | `Artist → Album → Track` |
| artisti di un genere | `Genre → Track → Album → Artist` |
| chi ha comprato un brano | `Track → InvoiceLine → Invoice → Customer` |
| cosa ha comprato un cliente (artisti, generi) | `Customer → Invoice → InvoiceLine → Track → Album → Artist` |
| quanto ha speso un cliente | `Customer → Invoice` (`SUM(Total)`) |
| brani di una playlist | `Playlist → PlaylistTrack → Track` |
| chi segue un cliente / chi è il capo | `Customer.SupportRepId → Employee`, `Employee.ReportsTo → Employee` |

Le colonne di aggancio sono sempre gli `…Id` con lo stesso nome (`ar.ArtistId = al.ArtistId`, `al.AlbumId = t.AlbumId`, `t.TrackId = il.TrackId`, `il.InvoiceId = i.InvoiceId`, `i.CustomerId = c.CustomerId`), quindi è difficile sbagliare.

### Come diventa un grafo

Nei temi d'esame tipici:

- **I nodi** sono quasi sempre entità del catalogo (artisti, album, brani) o clienti, spesso **filtrati** con un dropdown (genere, anno, paese, media type).
- **Gli archi** nascono da "hanno qualcosa in comune": stesso cliente, stessa playlist, stesso genere, stesso anno. In SQL si ottengono con un **self-join**, cioè la stessa tabella (o sottoquery) usata due volte con alias diversi, collegata sulla cosa in comune.
- **I pesi** sono conteggi o somme: quanti clienti in comune, quanti brani venduti, durata totale, spesa.

## 5. Trappole da ricordare

1. **Track è il centro**: artista e cliente non sono mai collegati direttamente, devi attraversare `Album` da una parte e `InvoiceLine → Invoice` dall'altra.
2. **Il genere è per brano.** Un filtro sul genere va messo su `Track.GenreId`, e lo stesso artista può comparire in più generi.
3. **INNER JOIN fa sparire chi non ha corrispondenze**: 71 artisti senza album, 1519 brani mai venduti, playlist vuote. Chiediti sempre se i nodi devono includerli.
4. **`DISTINCT` quando la JOIN moltiplica le righe**: un artista con 50 brani compare 50 volte in `Artist ⋈ Album ⋈ Track`.
5. **`Quantity` = 1 sempre**, ma nel codice usa comunque la versione corretta (`SUM`).
6. **Nomi non unici** (brani, playlist): usa gli Id come chiavi dei nodi e i nomi solo per stampare.
7. **Maiuscole nei nomi tabella**: il dump crea `Album`, `InvoiceLine`… Su Mac/Windows MySQL è indifferente, su Linux no.

## 6. Esercizi per prendere confidenza (in DBeaver)

Prova a scrivere tu le query; tra parentesi c'è il risultato atteso per controllarti.

1. Quanti brani ci sono per genere, in ordine decrescente? (Rock 1297 in testa)
2. Quali artisti non hanno album? (71)
3. Quanti artisti diversi hanno almeno un brano Jazz? (10)
4. Quanti brani diversi sono stati venduti almeno una volta? (1984)
5. Quanti brani Rock sono stati venduti in totale? (835)
6. Per ogni cliente, quanti artisti diversi ha comprato? Chi ne ha di più?
7. Quali clienti ha seguito Jane Peacock? (21)
8. Quante fatture ci sono nel 2023 e quanti clienti distinti le hanno fatte? (83 fatture, 47 clienti)
9. Trova le coppie di playlist che hanno almeno un brano in comune, contando quanti. Suggerimento: usa `PlaylistTrack` due volte.
