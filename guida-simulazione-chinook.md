# Simulazione d'esame Chinook (19-20/05/2026): guida

Fonte: repo `martinatirrito/Lab11-Simulazione-esame-chinook-` (clonato il 2026-10-18). Il repo è lo **scheletro di partenza** (DAO, Model e Controller vuoti) + dump SQL + PDF del testo. Le query qui sotto le ho **eseguite sul dump** (convertito in SQLite, stessa logica SQL): i numeri di Rock, Jazz e Latin coincidono con gli screenshot del PDF.

---

## 1. Struttura del progetto

```
main.py                 # crea Model, View, Controller e lancia Flet
database/
  connector.cnf         # credenziali MySQL (letto dal pool)
  DB_connect.py         # pool di connessioni (già pronto, non si tocca)
  DAO.py                # VUOTO: qui scrivi le query
  Chinook_MySql.sql     # dump da importare in DBeaver
model/model.py          # VUOTO: qui costruisci il grafo (networkx)
UI/view.py              # GUI Flet (già pronta: 2 dropdown + 2 bottoni + ListView)
UI/controller.py        # handler vuoti: fillDDGenre, handleCreaGrafo, handleCammino
Requirements.txt        # flet~=0.22, networkx~=3.2, mysql-connector-python~=8.3
```

**Flusso MVC** (sempre lo stesso a ogni esame):

```
click bottone (View) → Controller.handleX → Model.metodo → DAO.query → DB
                                         ← oggetti/dict ←
Controller scrive il risultato in view.txt_result e chiama view.update_page()
```

- **View**: solo widget. `self._ddGenre`, `self._ddArtist`, `self.txt_result` (ListView). Nota: nel template `self._btnCreaGrafo` è riassegnato due volte (anche per "Trova Cammino"): innocuo, ma se ti serve il secondo bottone rinominalo.
- **Controller**: legge i valori dei dropdown, valida, chiama il Model, stampa. Nel template `handleCreaGrafo` è definito due volte (il secondo sovrascrive il primo): cancellane uno.
- **Model**: tiene il grafo `self._graph = nx.DiGraph()`, mappe `idMap` id → oggetto, algoritmi (ricorsione).
- **DAO**: metodi `@staticmethod` che aprono una connessione, eseguono la query, restituiscono liste/dict, chiudono.

## 2. Connessione al DB

`DB_connect.py` crea **un pool** (`MySQLConnectionPool`) leggendo `connector.cnf`. Non si istanzia la classe: si usa `DBConnect.get_connection()`.

⚠️ **Prima cosa da fare all'esame**: in `connector.cnf` c'è `database=test`. Va cambiato nel nome del DB importato (`chinook`) e vanno sistemati user/password del tuo MySQL locale. Se sbagli, `get_connection()` stampa l'errore e **ritorna `None`**: gestiscilo (`if cnx is None`), altrimenti crash su `cnx.cursor()`.

Pattern DAO standard:

```python
from database.DB_connect import DBConnect

class DAO():
    @staticmethod
    def getAllGenres():
        cnx = DBConnect.get_connection()
        result = []
        if cnx is None:
            print("Connessione fallita")
            return result
        cursor = cnx.cursor(dictionary=True)
        cursor.execute("SELECT GenreId, Name FROM genre ORDER BY Name")
        for row in cursor:
            result.append(row)          # oppure Genre(**row) con una dataclass
        cursor.close()
        cnx.close()                     # restituisce la connessione al pool
        return result
```

Parametri sempre con `%s` e tupla: `cursor.execute(query, (genreId,))`. Mai f-string nella query.

**Case delle tabelle**: il dump crea `Album`, `Artist`, `InvoiceLine`… con la maiuscola. Su macOS/Windows MySQL è case-insensitive sui nomi tabella, su Linux no. Scrivi i nomi come li vedi in DBeaver e sei a posto.

## 3. Schema Chinook

| Tabella | PK | FK | Righe |
|---|---|---|---|
| `Artist` | ArtistId | – | 275 |
| `Album` | AlbumId | ArtistId → Artist | 347 |
| `Track` | TrackId | AlbumId → Album, GenreId → Genre, MediaTypeId → MediaType | 3503 |
| `Genre` | GenreId | – | 25 |
| `MediaType` | MediaTypeId | – | 5 |
| `Playlist` | PlaylistId | – | 18 |
| `PlaylistTrack` | (PlaylistId, TrackId) | → Playlist, → Track | 8715 |
| `Customer` | CustomerId | SupportRepId → Employee | 59 |
| `Employee` | EmployeeId | ReportsTo → Employee (self-join) | 8 |
| `Invoice` | InvoiceId | CustomerId → Customer | 412 |
| `InvoiceLine` | InvoiceLineId | InvoiceId → Invoice, TrackId → Track | 2240 |

Fatti utili:
- `InvoiceLine.Quantity` vale **sempre 1** → `SUM(Quantity)` = `COUNT(*)`.
- `Track.AlbumId` e `Track.GenreId` sono nullable nello schema (in pratica pieni).
- L'artista **non** è collegato direttamente alla traccia: passa sempre da `Album`.

### Catene di join da sapere a memoria

```
Artist ─ Album ─ Track ─ Genre                      (artisti di un genere)
Artist ─ Album ─ Track ─ InvoiceLine ─ Invoice ─ Customer   (chi ha comprato cosa)
Track ─ PlaylistTrack ─ Playlist                     (brani nella stessa playlist)
Customer ─ Employee (SupportRepId), Employee ─ Employee (ReportsTo)
```

Idee tipiche di grafo su Chinook: nodi = artisti/album/brani/clienti; arco se "stesso cliente ha comprato entrambi", "stessa playlist", "stesso genere"; peso = n. clienti in comune, popolarità, durata (`Milliseconds`), spesa (`UnitPrice*Quantity`).

## 4. Il testo della simulazione

**Punto 1**: scelto un genere, grafo **orientato e pesato**.
- **Nodi**: artisti con almeno un brano del genere.
- **Popolarità** di un artista = n. brani acquistati (InvoiceLine) di quell'artista.
- **Arco** tra A e B se almeno un cliente ha comprato brani di entrambi. Verso: da più popolare a meno popolare; a pari popolarità, due archi.
- **Peso** = pop(A) + pop(B).
- Output: n. nodi, n. archi, artista con influenza max (`Σ peso uscenti − Σ peso entranti`), top 5 archi per peso.

**Interpretazione verificata**: sia la popolarità sia gli acquisti in comune vanno **filtrati sul genere scelto**. Solo così tornano i numeri del PDF (ho provato tutte le 8 combinazioni):

| Genere | Nodi | Archi | Più influente | Top arco |
|---|---|---|---|---|
| Rock | 51 | 492 | U2 (4093) | U2 → Led Zeppelin : 178 |
| Jazz | 10 | 11 | Spyro Gyra (75) | Miles Davis → Gene Krupa : 34 |
| Latin | 28 | 141 | Os Paralamas Do Sucesso (1017) | → Chico Buarque : 72 |

Senza filtro sul genere Rock dà 563 archi o Iron Maiden come più influente: sbagliato. Jazz ha 10 nodi ma 2 senza vendite (popolarità 0, isolati): i nodi vanno presi da Track, **non** da InvoiceLine.

### Query (sintassi MySQL)

```sql
-- Q1 dropdown generi
SELECT GenreId, Name FROM genre ORDER BY Name;

-- Q2 nodi: artisti con almeno un brano del genere
SELECT DISTINCT ar.ArtistId, ar.Name
FROM artist ar, album al, track t
WHERE ar.ArtistId = al.ArtistId AND al.AlbumId = t.AlbumId AND t.GenreId = %s;

-- Q3 popolarità per artista nel genere
SELECT al.ArtistId, SUM(il.Quantity) AS pop
FROM invoiceline il, track t, album al
WHERE il.TrackId = t.TrackId AND t.AlbumId = al.AlbumId AND t.GenreId = %s
GROUP BY al.ArtistId;

-- Q4 coppie di artisti con almeno un cliente in comune (nel genere)
SELECT a1.ArtistId AS a1, a2.ArtistId AS a2
FROM (SELECT DISTINCT i.CustomerId, al.ArtistId
      FROM invoice i, invoiceline il, track t, album al
      WHERE i.InvoiceId = il.InvoiceId AND il.TrackId = t.TrackId
        AND t.AlbumId = al.AlbumId AND t.GenreId = %s) a1,
     (SELECT DISTINCT i.CustomerId, al.ArtistId
      FROM invoice i, invoiceline il, track t, album al
      WHERE i.InvoiceId = il.InvoiceId AND il.TrackId = t.TrackId
        AND t.AlbumId = al.AlbumId AND t.GenreId = %s) a2
WHERE a1.CustomerId = a2.CustomerId AND a1.ArtistId < a2.ArtistId
GROUP BY a1.ArtistId, a2.ArtistId;
```

Trucco chiave di Q4: **self-join sulla stessa sottoquery** (cliente, artista) e `a1.ArtistId < a2.ArtistId` per avere ogni coppia una sola volta. Il verso dell'arco lo decidi poi in Python confrontando le popolarità. Q4 richiede `(genreId, genreId)` come parametri.

### Model

```python
import networkx as nx
from database.DAO import DAO

class Model:
    def __init__(self):
        self._graph = nx.DiGraph()
        self._idMap = {}     # ArtistId -> nome (o oggetto Artist)
        self._pop = {}

    def buildGraph(self, genreId):
        self._graph.clear()
        self._idMap = {r["ArtistId"]: r["Name"] for r in DAO.getArtistsByGenre(genreId)}
        self._graph.add_nodes_from(self._idMap.keys())
        self._pop = DAO.getPopularity(genreId)          # dict ArtistId -> pop
        for a, b in DAO.getCoPurchasedPairs(genreId):
            pa, pb = self._pop.get(a, 0), self._pop.get(b, 0)
            w = pa + pb
            if pa >= pb:
                self._graph.add_edge(a, b, weight=w)
            if pb >= pa:                                  # pari → entrambi i versi
                self._graph.add_edge(b, a, weight=w)

    def getMostInfluent(self):
        best, bestVal = None, None
        for n in self._graph.nodes:
            out_w = sum(d["weight"] for _, _, d in self._graph.out_edges(n, data=True))
            in_w = sum(d["weight"] for _, _, d in self._graph.in_edges(n, data=True))
            if bestVal is None or out_w - in_w > bestVal:
                best, bestVal = n, out_w - in_w
        return self._idMap[best], bestVal

    def getTopEdges(self, k=5):
        return sorted(self._graph.edges(data="weight"), key=lambda e: e[2], reverse=True)[:k]
```

(Con `out_degree(n, weight="weight") - in_degree(n, weight="weight")` fai l'influenza in una riga.)

### Punto 2: cammino semplice più lungo con pesi strettamente crescenti

Ricorsione classica (backtracking) a partire dall'artista scelto:

```python
def getPath(self, source):
    self._best = []
    self._ricorsione([source], float("-inf"))
    return self._best

def _ricorsione(self, parziale, lastW):
    if len(parziale) > len(self._best):
        self._best = list(parziale)
    for n in self._graph.successors(parziale[-1]):
        w = self._graph[parziale[-1]][n]["weight"]
        if n not in parziale and w > lastW:
            parziale.append(n)
            self._ricorsione(parziale, w)
            parziale.pop()
```

⚠️ **Sorpresa verificata**: su questo grafo il cammino migliore ha **sempre lunghezza 1**. Motivo: se A → B allora pop(A) ≥ pop(B), e l'arco successivo B → C ha peso pop(B)+pop(C) ≤ pop(A)+pop(B), quindi non può essere strettamente maggiore. Su Rock, da U2/Led Zeppelin/Queen ottieni un solo arco. All'esame scrivi comunque la ricorsione corretta (è quella che valutano), ma se esce lunghezza 1 non è un bug. Se il testo vero dicesse "decrescente", lo stesso codice con `w < lastW` e partenza `float("inf")` dà cammini lunghi.

## 5. Controller: cose da non dimenticare

- `fillDDGenre`: `self._view._ddGenre.options.append(ft.dropdown.Option(key=str(g["GenreId"]), text=g["Name"]))`. Il `value` del dropdown è **una stringa**: `int(self._view._ddGenre.value)`.
- Dopo "Crea grafo" popola `_ddArtist` con i nodi del grafo (punto 2a) e fai `update_page()`.
- Validazioni richieste dal testo ("non sono ammesse eccezioni"): genere non scelto, grafo non ancora creato, artista non scelto, DB che restituisce `None`. Ognuna → messaggio in `txt_result` o `create_alert`, poi `return`.
- Svuota `txt_result.controls.clear()` prima di stampare.
- Il template Flet usa `ft.UserControl` (esiste in flet 0.22, rimosso nelle versioni nuove): installa **esattamente** le versioni del `Requirements.txt`.

## 6. Checklist operativa all'esame

1. Importa il dump in DBeaver, controlla il nome del DB.
2. Sistema `connector.cnf` (database, user, password).
3. Scrivi e **prova le query in DBeaver** prima di metterle nel DAO.
4. DAO → Model `buildGraph` → stampa n. nodi/archi → confronta con gli esempi del testo.
5. Solo dopo: influenza, top 5, ricorsione.
