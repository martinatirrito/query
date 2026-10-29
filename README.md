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
