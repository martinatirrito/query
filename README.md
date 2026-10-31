# Query TdP - Chinook

## Archi NON orientati con peso = somma nelle due direzioni (LEFT JOIN + COALESCE)

### Quando si usa
Solo quando la relazione ha un **verso** (da A a B) ma il grafo richiesto è **non orientato** con peso = A→B + B→A.

**Frasi spia nella traccia:**
- "il peso è la somma dei voli/spostamenti/passaggi **in entrambe le direzioni**"
- "da A verso B **e** da B verso A"
- "numero di tratte tra A e B, **indipendentemente dal verso**"

Se la relazione è già simmetrica ("A e B condividono un album/genere/playlist...") **NON serve**: basta il solito self-join con `c1.id < c2.id`.

### Come si ragiona
1. Scrivo la **query normale** che restituisce 3 colonne: `a` (da), `b` (a), `n` (quante volte A→B).
2. La incollo **due volte** nello stampo: `t1` = A→B, `t2` = la coppia al contrario B→A.
3. **LEFT JOIN**: se B→A non esiste, la riga di t1 non sparisce, ma resta con t2 = NULL.
4. **COALESCE(t2.n, 0)**: se t2.n è NULL usa 0 (perché `numero + NULL = NULL`).
5. **WHERE** `t1.a < t1.b`: tengo una sola riga per coppia (A,B e B,A avrebbero lo stesso peso).
   `OR t2.a is null`: salvo le coppie che esistono in una sola direzione.

### Stampo
```sql
select t1.a, t1.b, t1.n + coalesce(t2.n, 0) as peso
from ( QUERY_NORMALE ) t1
left join ( QUERY_NORMALE ) t2
     on t1.a = t2.b and t1.b = t2.a
where t1.a < t1.b or t2.a is null
```

### Esempio Chinook: passaggi tra generi
**Nodi:** tutti i generi.
**Passaggio A→B:** nella stessa fattura, una riga di genere A è seguita subito dalla riga successiva di genere B.
**Peso arco A–B:** passaggi A→B + passaggi B→A.

**Query normale (passaggi orientati)**: atteso 91 righe, in testa `4 | 1 | 45`
```sql
select tr1.GenreId as a, tr2.GenreId as b, count(*) as n
from invoiceline il1, invoiceline il2, track tr1, track tr2
where il1.InvoiceId = il2.InvoiceId                -- stessa fattura
  and il2.InvoiceLineId = il1.InvoiceLineId + 1    -- il2 è la riga subito dopo il1
  and tr1.TrackId = il1.TrackId                    -- genere della prima riga
  and tr2.TrackId = il2.TrackId                    -- genere della seconda riga
  and tr1.GenreId <> tr2.GenreId                   -- generi diversi
group by tr1.GenreId, tr2.GenreId
```

**Query finale (non orientato)**: atteso 64 righe, massimo `1 | 4 | 74` (Rock – Alternative & Punk)
```sql
select t1.a, t1.b, t1.n + coalesce(t2.n, 0) as peso
from (select tr1.GenreId as a, tr2.GenreId as b, count(*) as n
      from invoiceline il1, invoiceline il2, track tr1, track tr2
      where il1.InvoiceId = il2.InvoiceId
        and il2.InvoiceLineId = il1.InvoiceLineId + 1
        and tr1.TrackId = il1.TrackId and tr2.TrackId = il2.TrackId
        and tr1.GenreId <> tr2.GenreId
      group by tr1.GenreId, tr2.GenreId) t1
left join
     (select tr1.GenreId as a, tr2.GenreId as b, count(*) as n
      from invoiceline il1, invoiceline il2, track tr1, track tr2
      where il1.InvoiceId = il2.InvoiceId
        and il2.InvoiceLineId = il1.InvoiceLineId + 1
        and tr1.TrackId = il1.TrackId and tr2.TrackId = il2.TrackId
        and tr1.GenreId <> tr2.GenreId
      group by tr1.GenreId, tr2.GenreId) t2
  on t1.a = t2.b and t1.b = t2.a          -- attacco la coppia "al contrario"
where t1.a < t1.b or t2.a is null         -- una riga per coppia + salvo quelle senza ritorno
```

**Nel model:** grafo `nx.Graph()` (non orientato), `add_edge(a, b, weight=peso)`.

## Archi con peso = valore nodo 1 + valore nodo 2 (4 sottoquery) - Esercizio 8
**Traccia:** nodi = generi comprati dai clienti di un Paese; valore = tracce comprate (SUM Quantity);
arco se un cliente del Paese li ha comprati entrambi; peso = valore1 + valore2.

**Schema a 4 blocchi:** p1, p2 = ponte (chi è collegato a cosa) / v1, v2 = valori dei nodi.
Il filtro del parametro va in **TUTTE** le sottoquery. `distinct` perché più clienti in comune = righe doppie.

```sql
select distinct p1.GenreId as g1, p2.GenreId as g2, v1.tot + v2.tot as peso
from (select distinct t.GenreId, i.CustomerId
      from track t, invoiceline il, invoice i, customer c
      where t.TrackId = il.TrackId and il.InvoiceId = i.InvoiceId
        and i.CustomerId = c.CustomerId and c.Country = %s) p1,
     (select distinct t.GenreId, i.CustomerId
      from track t, invoiceline il, invoice i, customer c
      where t.TrackId = il.TrackId and il.InvoiceId = i.InvoiceId
        and i.CustomerId = c.CustomerId and c.Country = %s) p2,
     (select t.GenreId, sum(il.Quantity) as tot
      from track t, invoiceline il, invoice i, customer c
      where t.TrackId = il.TrackId and il.InvoiceId = i.InvoiceId
        and i.CustomerId = c.CustomerId and c.Country = %s
      group by t.GenreId) v1,
     (select t.GenreId, sum(il.Quantity) as tot
      from track t, invoiceline il, invoice i, customer c
      where t.TrackId = il.TrackId and il.InvoiceId = i.InvoiceId
        and i.CustomerId = c.CustomerId and c.Country = %s
      group by t.GenreId) v2
where p1.CustomerId = p2.CustomerId      -- ponte: stesso cliente
  and p1.GenreId < p2.GenreId            -- non orientato
  and v1.GenreId = p1.GenreId            -- attacco i valori
  and v2.GenreId = p2.GenreId
```
**DAO:** `cursor.execute(query, (country, country, country, country))`, con un parametro per ogni %s.
**Atteso (Brazil):** 13 nodi, 56 archi, max 134 (Rock-Latin).

## Soglia sui nodi con HAVING + IN - Esercizio 11
**Traccia:** nodi = album con durata dei brani del MediaType scelto > X minuti;
arco se una playlist contiene brani di entrambi; peso = numero di playlist distinte in comune.

**HAVING** = filtro su un'aggregazione (SUM, COUNT, AVG). Se il filtro non è su un'aggregazione, va nel WHERE.
La query dei nodi la riuso dentro gli archi con `IN (...)`: deve restituire **una sola colonna**, altrimenti errore 1241.
X è in minuti, Milliseconds no: X * 60000.

**Nodi:**
```sql
select t.AlbumId
from track t
where t.MediaTypeId = %s
group by t.AlbumId
having sum(t.Milliseconds) > %s * 60000
```
**Archi:**
```sql
select t1.AlbumId as a1, t2.AlbumId as a2, count(distinct p1.PlaylistId) as peso
from playlisttrack p1, track t1, playlisttrack p2, track t2
where p1.PlaylistId = p2.PlaylistId                       -- ponte: stessa playlist
  and p1.TrackId = t1.TrackId and p2.TrackId = t2.TrackId
  and t1.MediaTypeId = %s and t2.MediaTypeId = %s
  and t1.AlbumId < t2.AlbumId                             -- non orientato
  and t1.AlbumId in (select t.AlbumId from track t where t.MediaTypeId = %s
                     group by t.AlbumId having sum(t.Milliseconds) > %s * 60000)
  and t2.AlbumId in (select t.AlbumId from track t where t.MediaTypeId = %s
                     group by t.AlbumId having sum(t.Milliseconds) > %s * 60000)
group by t1.AlbumId, t2.AlbumId
```
**Peso con COUNT(DISTINCT):** due album hanno più tracce nella stessa playlist, quindi `count(*)` conterebbe la playlist più volte.
**Atteso (MediaType 1, X = 60):** 88 nodi, 3828 archi, max 4.

## Archi senza ponte: confronto tra valori - Esercizio 9
**Traccia:** nodi = album con tracce di un genere; valore = durata media in secondi;
arco se |durata1 - durata2| <= D; peso = differenza arrotondata.

Non c'è un'entità in comune: faccio il self-join del **blocco valori** con se stesso.

```sql
select a1.AlbumId as a1, a2.AlbumId as a2, round(abs(a1.durata - a2.durata)) as peso
from (select a.AlbumId, avg(t.Milliseconds / 1000) as durata
      from album a, track t, genre g
      where a.AlbumId = t.AlbumId and t.GenreId = g.GenreId and g.Name = %s
      group by a.AlbumId) a1,
     (select a.AlbumId, avg(t.Milliseconds / 1000) as durata
      from album a, track t, genre g
      where a.AlbumId = t.AlbumId and t.GenreId = g.GenreId and g.Name = %s
      group by a.AlbumId) a2
where a1.AlbumId < a2.AlbumId
  and abs(a1.durata - a2.durata) <= %s     -- WHERE, non HAVING: non è un'aggregazione
```
**Atteso (Jazz, D = 10):** 13 nodi, 4 archi, max 8.

## Grafo ORIENTATO con filtro date - Esercizio 10
**Traccia:** nodi = clienti di un Paese con acquisti tra anno1 e anno2;
arco se hanno comprato lo stesso artista; c1 -> c2 se fatture(c1) >= fatture(c2);
peso = numero di artisti distinti in comune.

Il filtro anni va in **TUTTE** le sottoquery. `<>` invece di `<` perché è orientato.
`>=` dà l'arco in **entrambe** le direzioni quando i valori sono uguali.

```sql
select b1.CustomerId as c1, b2.CustomerId as c2, count(distinct b1.ArtistId) as peso
from (select distinct i.CustomerId, al.ArtistId
      from customer c, invoice i, invoiceline il, track t, album al
      where c.CustomerId = i.CustomerId and i.InvoiceId = il.InvoiceId
        and il.TrackId = t.TrackId and t.AlbumId = al.AlbumId
        and c.Country = %s and year(i.InvoiceDate) between %s and %s) b1,
     (select distinct i.CustomerId, al.ArtistId
      from customer c, invoice i, invoiceline il, track t, album al
      where c.CustomerId = i.CustomerId and i.InvoiceId = il.InvoiceId
        and il.TrackId = t.TrackId and t.AlbumId = al.AlbumId
        and c.Country = %s and year(i.InvoiceDate) between %s and %s) b2,
     (select i.CustomerId, count(*) as nFatt
      from customer c, invoice i
      where c.CustomerId = i.CustomerId
        and c.Country = %s and year(i.InvoiceDate) between %s and %s
      group by i.CustomerId) v1,
     (select i.CustomerId, count(*) as nFatt
      from customer c, invoice i
      where c.CustomerId = i.CustomerId
        and c.Country = %s and year(i.InvoiceDate) between %s and %s
      group by i.CustomerId) v2
where b1.ArtistId = b2.ArtistId              -- ponte: stesso artista
  and b1.CustomerId <> b2.CustomerId         -- orientato
  and v1.CustomerId = b1.CustomerId
  and v2.CustomerId = b2.CustomerId
  and v1.nFatt >= v2.nFatt                   -- direzione
group by b1.CustomerId, b2.CustomerId
```
**Atteso (Germany, 2022-2023):** 4 nodi (Niklas Schröder isolato), 6 archi, max 2 (Hannah <-> Fynn).
**Nel model:** `nx.DiGraph()`.
