# Ricorsione TdP (punto 2), spiegata

Solo scheletri riutilizzabili e varianti: niente soluzioni di temi specifici. Tutti gli esempi di codice sono stati provati su un grafo giocattolo con networkx (2026-10-23).

---

## 0. Cosa chiedono davvero (29 temi di TdP-esami)

Ho letto il punto 2 di **29 temi** dell'organizzazione GitHub `TdP-esami`: 2019 e 2020 (Java), 2025 e 2026 (Python). L'organizzazione ne contiene 102, ma da qui non riesco a elencare gli altri 73 (soprattutto 2021–2024): se ti servono, mandami i nomi e li aggiungo. Le soluzioni ufficiali (branch `sol`) usano esattamente gli scheletri qui sotto.

| Famiglia | Cosa chiede | Quante volte (su 29) | Temi |
|---|---|---|---|
| **A. Cammino semplice ottimo** | cammino senza nodi ripetuti, da un nodo (o tra due nodi), più lungo o di peso max/min, con un vincolo sui passi | **12** (41%) | 2019-06-27 A/B/C, 2019-09-03, 2020-06-29 M/P, 2025-07-10 A/B, 2025-11-10, Sim. 2026-05-19, Sim. 2026-05-26, 2026-06-29-A |
| **D. Sottoinsieme di K elementi** | scegli K nodi da una lista (stile zaino), con un filtro o un'esclusione, massimizza una somma | **6** (21%) | 2025-06-26 A/B/C, 2020-06-03, 2025-01-13 (Python e Java) |
| **C. K nodi da componenti connesse diverse** | un nodo per componente, massimizza una somma o minimizza un range | **4** (14%) | 2025-09-15-A, 2026-01-12, 2026-06-29-B, 2026-07-14-B |
| **B. Gruppo connesso di N nodi** | N nodi dal nodo scelto, ognuno adiacente al gruppo, vincoli su coppie | **3** (10%) | 2026-06-29-C, 2026-07-14-A, 2026-09-14-A |
| **E. Simulazione a eventi** | non è ricorsione: coda di eventi, agenti/stazioni, probabilità | **4** (14%) | 2019-06-05, 2019-07-12 A/B, 2020-06-10 |

👉 **Su cosa lavorare, in ordine:**
1. **A (cammini)**: è la più frequente in assoluto e c'è in quasi ogni anno. Priorità massima.
2. **C e B**: sono **le ultime 6 prove** (2026), quindi è il formato attuale del prof. Seconda priorità.
3. **D (K su N)**: è la stessa ricorsione di C senza le componenti, la impari "gratis" insieme a C.
4. **E (simulazione)**: compare solo negli anni Java (2019–2020) e **mai dal 2025 in poi**. Puoi saltarla.

Nota sulle sessioni: i temi A/B/C dello stesso giorno sono quasi sempre **la stessa ricorsione** con un dettaglio diverso (2019-06-27: stesso cammino tra due nodi, cambia solo l'obiettivo).

---

## 1. Lo scheletro generico (impara questo e basta)

Ogni ricorsione dell'esame ha **due metodi** nel `Model`:

```python
import copy

class Model:
    def __init__(self):
        self._graph = nx.Graph()
        self._idMap = {}
        # stato della ricorsione: lo resetto a ogni chiamata, non qui
        self._bestSol = []
        self._bestScore = 0

    # ① metodo "pubblico": lo chiama il controller
    def getBestSol(self, source, N):
        self._bestSol = []            # reset: se l'utente preme il bottone 2 volte non deve vedere la soluzione vecchia
        self._bestScore = -1          # -1 così anche una soluzione con punteggio 0 viene salvata
        parziale = [source]           # la soluzione parte SEMPRE dal nodo scelto dall'utente
        self._ricorsione(parziale, N)
        return self._bestSol, self._bestScore

    # ② metodo ricorsivo: esplora tutte le estensioni di "parziale"
    def _ricorsione(self, parziale, N):
        # (a) CASO TERMINALE: la soluzione è completa?
        if len(parziale) == N:
            score = self._calcolaScore(parziale)
            if score > self._bestScore:                  # è migliore della migliore trovata finora?
                self._bestScore = score
                self._bestSol = copy.deepcopy(parziale)  # salvo una COPIA, non "parziale"
            return                                       # completa: non ha senso allungarla

        # (b) ESPANSIONE: provo ogni candidato ammissibile
        for c in self._candidati(parziale):
            parziale.append(c)                # scelta
            self._ricorsione(parziale, N)     # esploro tutto quello che viene dopo questa scelta
            parziale.pop()                    # annullo la scelta (backtracking) e provo il candidato dopo
```

### Perché ogni riga serve (le domande tipiche dell'orale)

- **Perché `copy.deepcopy(parziale)` e non `self._bestSol = parziale`?**
  `parziale` è **una sola lista** che viene modificata per tutta la ricorsione (append/pop). Se assegni il riferimento, `_bestSol` punta alla **stessa** lista: quando la ricorsione finisce e fa tutti i `pop`, ti ritrovi `_bestSol == [source]` (o vuota). Serve una **fotografia** della lista in quel momento.
  Nota: `copy.deepcopy` copia anche gli oggetti dentro (i nodi `Regista`, `Artist`...). Funziona, ma basta una copia superficiale: `list(parziale)` o `parziale.copy()` (copia la lista, gli oggetti restano gli stessi del grafo). Se all'orale ti chiedono la differenza: *deepcopy duplica anche gli elementi, la copia semplice duplica solo il contenitore*. Usa quella che preferisci, ma sappi spiegarla.

- **Perché il `pop()` dopo la chiamata?**
  È il **backtracking**: dopo aver esplorato tutte le soluzioni che iniziano con `[..., c]`, tolgo `c` per riportare `parziale` com'era e provare il candidato successivo. Senza `pop`, la lista si allunga all'infinito con tutti i candidati uno dopo l'altro e le soluzioni sono sbagliate.

- **Perché confronto con `self._bestScore`?**
  La ricorsione genera **tutte** le soluzioni ammissibili, una dopo l'altra. Tengo solo la migliore vista finora: alla fine è la migliore in assoluto (ricerca esaustiva).

- **Perché il reset in `getBestSol` e non in `__init__`?**
  Perché il metodo può essere chiamato più volte (altro nodo, altro N, grafo ricostruito). Se non resetti, una soluzione vecchia con score alto "vince" su quella nuova.

- **Perché `>` e non `>=`?**
  Con `>` tieni la **prima** soluzione trovata a parità di punteggio. Con `>=` l'ultima. Di solito il testo non chiede nulla sui pareggi: va bene `>`.

- **Complessità?** Esponenziale: nel caso peggiore provi tutte le sequenze di nodi. Per questo si testa con parametri piccoli (N = 2, 3, 4) e grafi piccoli.

### Le 3 cose che cambiano da tema a tema

1. **Caso terminale**: `len(parziale) == N` (gruppo di N) oppure "nessun terminale fisso" (cammino più lungo, vedi §2).
2. **`_candidati(parziale)`**: chi posso aggiungere (vicini dell'ultimo? vicini di tutto il gruppo? nodi di altre componenti?) + i vincoli.
3. **`_calcolaScore(parziale)`**: numero di archi, somma dei pesi, somma di un attributo dei nodi.

---


## 2. Famiglia A: cammino semplice ottimo (12 temi su 29)

"Trovare un cammino semplice di **lunghezza massima** a partire dal nodo X tale che..."

Differenza chiave rispetto allo scheletro: **non c'è una lunghezza fissa**. Ogni cammino parziale è già una soluzione valida, quindi **controllo il best a ogni chiamata** e **non faccio `return`** (il cammino potrebbe allungarsi ancora). La ricorsione si ferma da sola quando non ci sono più vicini ammissibili.

```python
def getCammino(self, source):
    self._bestPath = []
    self._ricorsioneCammino([source])
    return self._bestPath

def _ricorsioneCammino(self, parziale):
    # ogni cammino parziale è ammissibile: è il più lungo visto finora?
    if len(parziale) > len(self._bestPath):
        self._bestPath = copy.deepcopy(parziale)

    ultimo = parziale[-1]
    for v in self._graph.neighbors(ultimo):      # DiGraph: successors(ultimo)
        if v not in parziale and self._vincolo(parziale, v):   # "semplice" = niente nodi ripetuti
            parziale.append(v)
            self._ricorsioneCammino(parziale)
            parziale.pop()
```


### Mappa delle varianti A viste negli esami

| Variante | Obiettivo | Vincolo | Temi |
|---|---|---|---|
| A1 | più lungo | attributo del nodo monotono | Sim. 2026-05-26 (età strett. decrescente), 2026-06-29-A (fatturato ≤) |
| A2 | più lungo / peso max | peso degli archi monotono | Sim. 2026-05-19 (strett. crescente), 2025-11-10 (peso max, strett. decrescente) |
| A3 | peso max | – | si combina con le altre |
| A4 | peso max | lunghezza esattamente N | 2019-09-03, 2025-07-10 A/B |
| A5 | peso max / min, più nodi | da sorgente a destinazione | 2019-06-27 A/B/C, 2020-06-29-P, 2025-07-10 A/B |
| A6 | più lungo | somma dei pesi ≤ budget | 2020-06-29-M |
| A7 | peso min | deve toccare **tutti** i nodi | 2019-06-27-A |
| A8 | peso max | coppie di nodi vietate | 2020-06-29-P |

### Variante A1: attributo del nodo monotono (età, fatturato, popolarità...)

```python
def _vincolo(self, parziale, v):
    ultimo = parziale[-1]
    return v.eta < ultimo.eta            # strettamente decrescente
    # return v.fatturato <= ultimo.fatturato   # "decrescente" non stretto (<=): attenta alla parola "strettamente"
```

### Variante A2: peso degli archi monotono

Serve il peso dell'**ultimo arco** percorso, quindi gli ultimi **due** nodi. Al primo passo (un solo nodo) non c'è arco precedente: qualunque arco va bene.

```python
def _vincolo(self, parziale, v):
    ultimo = parziale[-1]
    if len(parziale) < 2:
        return True                                   # primo arco: nessun vincolo
    pesoPrec = self._graph[parziale[-2]][ultimo]['weight']
    pesoNuovo = self._graph[ultimo][v]['weight']
    return pesoNuovo > pesoPrec                       # "strettamente decrescente": <
```

💡 Nella soluzione ufficiale del 2025-11-10 il prof fa il **primo passo fuori dalla ricorsione**: in `getBestPath` cicla sui vicini del nodo di partenza, fa `append` e chiama `_ricorsione`. Così dentro la ricorsione `parziale` ha sempre almeno 2 nodi e `parziale[-2]` esiste sempre. È equivalente al mio `if len(parziale) < 2`: scegli quello che sai spiegare meglio.

### Variante A3: cammino di peso massimo (non di lunghezza massima)

Cambia solo il confronto con il best: invece di `len`, la somma dei pesi.

```python
def _pesoCammino(self, path):
    return sum(self._graph[path[i]][path[i+1]]['weight'] for i in range(len(path) - 1))

# nella ricorsione, al posto del controllo sulla lunghezza:
peso = self._pesoCammino(parziale)
if peso > self._bestScore:
    self._bestScore = peso
    self._bestPath = copy.deepcopy(parziale)
```

### Variante A4: cammino di lunghezza fissa N

Torna lo scheletro generico con terminale e `return`:
```python
if len(parziale) == N + 1:          # N archi = N+1 nodi
    ...confronto con il best...
    return
```
⚠️ "Lunghezza" di un cammino di solito = numero di **archi**. Nella soluzione ufficiale del 2025-07-10 però il prof scrive `len(parziale) == lungh`, cioè conta i **nodi**. Se il testo è ambiguo, guarda gli esempi di soluzione nel PDF e scrivi nel codice un commento su cosa hai scelto.

### Variante A5: cammino tra sorgente e destinazione

Il cammino è "finito" quando arriva al target: lì confronto e mi fermo (oltre il target non si va).

```python
def _ricorsione(self, parziale, target):
    if parziale[-1] == target:
        score = self._pesoCammino(parziale)        # oppure len(parziale) se chiede "più vertici"
        if score > self._bestScore:
            self._bestScore = score
            self._bestPath = copy.deepcopy(parziale)
        return
    for v in self._graph.neighbors(parziale[-1]):
        if v not in parziale:
            parziale.append(v)
            self._ricorsione(parziale, target)
            parziale.pop()
```
Con anche la lunghezza fissa (2025-07-10): terminale `len(parziale) == N + 1`, e salvo **solo se** `parziale[-1] == target`.

### Variante A6: somma dei pesi entro un budget

Mi porto dietro il peso accumulato come parametro, così non lo ricalcolo a ogni passo:

```python
def _ricorsione(self, parziale, pesoTot, c):
    if len(parziale) > len(self._bestPath):
        self._bestPath = copy.deepcopy(parziale)
    ultimo = parziale[-1]
    for v in self._graph.neighbors(ultimo):
        w = self._graph[ultimo][v]['weight']
        if v not in parziale and pesoTot + w <= c:      # aggiungo v solo se resto nel budget
            parziale.append(v)
            self._ricorsione(parziale, pesoTot + w, c)  # passo il nuovo totale: niente da "annullare"
            parziale.pop()
```
Perché il totale passato come parametro non serve `pop`: ogni chiamata ha **il suo** valore di `pesoTot`, i numeri non sono condivisi come la lista.

### Variante A7: cammino che tocca tutti i vertici (peso minimo)

È A5 con un controllo in più nel terminale: `parziale[-1] == target and len(parziale) == len(self._graph.nodes)`. Per il **minimo**: `self._bestScore = float('inf')` e confronto con `<`. Esplode velocemente: provala su grafi piccoli.

### Variante A8: coppie di nodi vietate

Il vincolo va nel `_vincolo(parziale, v)`: per esempio "v non deve essere una partita tra le stesse due squadre del nodo di partenza" → `return not stesseSquadre(parziale[0], v)`. Sono sempre controlli sul nuovo nodo rispetto a quelli già nel cammino.

---

## 3. Famiglia B: gruppo connesso di N nodi (3 temi su 29)

"Un insieme di N nodi, il primo è quello scelto, **ogni nodo aggiunto dopo il primo deve essere adiacente ad almeno uno dei nodi già presenti**, non possono esserci due nodi collegati da un arco di peso 1, massimizzare la somma di X."

Lo scheletro è quello del §1 (terminale `len == N`). Cambia **chi sono i candidati**: non i vicini dell'**ultimo** nodo (quello è un cammino), ma i vicini di **tutto il gruppo**.

```python
def _candidati(self, parziale):
    res = set()                                   # set: un nodo vicino a due membri lo conto una volta
    for m in parziale:
        for v in self._graph.neighbors(m):
            if v not in parziale and self._ammissibile(v, parziale):
                res.add(v)
    return res

def _ammissibile(self, c, parziale):
    # vincolo "nessuna coppia collegata da arco di peso 1"
    for m in parziale:
        if self._graph.has_edge(c, m) and self._graph[c][m]['weight'] == 1:
            return False
    return True

def _calcolaScore(self, parziale):
    return sum(n.numBrani for n in parziale)     # l'attributo cambia: brani, incasso, n. film...
```

### Variante B1: adiacente a **esattamente uno** (struttura ad albero)

```python
def _ammissibile(self, c, parziale):
    nVicini = sum(1 for m in parziale if self._graph.has_edge(c, m))
    return nVicini == 1
```
(Il ciclo in `_candidati` garantisce già "almeno uno", qui stringi a "esattamente uno".)

### Ottimizzazione utile: non ricalcolare lo stesso gruppo

Il gruppo è un **insieme**: `[A, B, C]` e `[A, C, B]` sono la stessa soluzione, ma la ricorsione le genera entrambe (permutazioni). Con N = 4 o 5 diventa lentissima. Rimedio semplice e facile da spiegare: ricordo i gruppi già visti.

```python
def getBestSol(self, source, N):
    self._bestSol = []
    self._bestScore = -1
    self._visti = set()                 # insiemi già esplorati
    self._ricorsione([source], N)
    return self._bestSol, self._bestScore

def _ricorsione(self, parziale, N):
    chiave = frozenset(parziale)        # frozenset: insieme immutabile, quindi può stare in un set
    if chiave in self._visti:
        return                          # questo gruppo (in un altro ordine) l'ho già esplorato
    self._visti.add(chiave)
    ...                                 # resto identico allo scheletro
```

All'orale: *"lo stesso insieme raggiunto in ordini diversi dà le stesse estensioni, quindi lo esploro una volta sola"*.

---

## 4. Famiglia C: K nodi da componenti connesse diverse (4 temi su 29)

"N nodi, il primo è quello scelto, **ognuno in una componente connessa diversa**, massimizzare la somma di X."

Qui i nodi **non devono essere vicini**, anzi. Due passi:

**① Prima della ricorsione**, calcolo una volta sola la componente di ogni nodo:
```python
self._comp = {}                                         # nodo -> indice della sua componente
for i, cc in enumerate(nx.connected_components(self._graph)):       # DiGraph: nx.weakly_connected_components
    for n in cc:
        self._comp[n] = i
```

**② Ricorsione "a sottoinsiemi" con indice.** Visto che l'ordine non conta, uso una lista di candidati fissa e un indice `i`: dopo aver scelto il candidato `j`, provo solo quelli **dopo** `j`. Così ogni insieme viene generato **una sola volta** (niente permutazioni, niente `frozenset`).

```python
def getSelezione(self, source, N):
    # ... calcolo self._comp come sopra ...
    self._candidati = [n for n in self._graph.nodes if self._comp[n] != self._comp[source]]
    self._bestSol = []
    self._bestScore = -1
    self._ricorsioneSel([source], {self._comp[source]}, 0, N)
    return self._bestSol, self._bestScore

def _ricorsioneSel(self, parziale, compUsate, i, N):
    if len(parziale) == N:
        score = sum(n.avg_rating for n in parziale)
        if score > self._bestScore:
            self._bestScore = score
            self._bestSol = copy.deepcopy(parziale)
        return

    for j in range(i, len(self._candidati)):
        c = self._candidati[j]
        if self._comp[c] in compUsate:         # componente già presa: scarto
            continue
        parziale.append(c)
        compUsate.add(self._comp[c])
        self._ricorsioneSel(parziale, compUsate, j + 1, N)   # j+1: solo candidati successivi
        parziale.pop()                          # backtracking su ENTRAMBE le strutture
        compUsate.remove(self._comp[c])
```

- Il backtracking va fatto su **tutto lo stato** che modifichi: qui sia `parziale` sia `compUsate`.
- **Pruning facile se è lenta**: per massimizzare una somma, da ogni componente conviene prendere solo il suo nodo **migliore**. Puoi mettere in `self._candidati` solo il nodo con X massimo di ogni componente: lo spazio crolla. All'orale si giustifica in una frase: *"se scambio un nodo con uno migliore della stessa componente, i vincoli restano rispettati e la somma sale"*.

Lo **schema a indice** vale per qualunque "scegli k elementi da una lista con vincolo" (stile zaino): è il tuo scheletro per i sottoinsiemi quando l'adiacenza non c'entra.


### Variante C1: minimizzare un range (max − min) invece di una somma

2025-09-15-A e 2026-01-12: "K elementi in componenti diverse tali che la differenza tra la data di nascita più vecchia e quella più giovane sia **minima**". Cambiano due cose:

```python
self._bestScore = float('inf')                 # cerco un MINIMO: parto da +infinito

# nel terminale:
valori = [n.dob for n in parziale]
score = max(valori) - min(valori)               # con le date: (max - min).days per avere i giorni
if score < self._bestScore:                     # < invece di >
    ...
```
Qui non c'è un nodo di partenza scelto dall'utente: parti da `parziale = []`. Il trucco "prendo solo il migliore di ogni componente" **non** vale più (il migliore dipende dagli altri scelti).

### Variante C2: lo schema "per componente" del prof (soluzione ufficiale 2026-01-12)

Invece di scorrere i nodi, scorre le **componenti** con un indice. Per ogni componente ho due scelte: **la salto** oppure **prendo uno dei suoi nodi**. Così è impossibile prendere due nodi della stessa componente, e non serve `compUsate`.

```python
comps = [list(cc) for cc in nx.connected_components(self._graph)]

def _ricorsione(self, comps, K, parziale, i):
    if len(parziale) == K:
        ...confronto con il best...
        return
    if len(comps) - i < K - len(parziale):     # pruning: componenti rimaste insufficienti
        return
    self._ricorsione(comps, K, parziale, i + 1)          # scelta 1: salto la componente i
    for n in comps[i]:                                    # scelta 2: prendo un nodo della componente i
        parziale.append(n)
        self._ricorsione(comps, K, parziale, i + 1)
        parziale.pop()
```
All'orale: *"il pruning taglia i rami in cui, anche prendendo un nodo da ogni componente rimasta, non arriverei a K"*.

---

## 5. Famiglia D: sottoinsieme di K elementi (6 temi su 29)

"Scegli K elementi tra quelli validi che massimizzano la somma di un indice." Il grafo serve solo per **filtrare** i candidati (per esempio quelli della componente più grande, o con almeno M edizioni). È lo **schema a indice** della famiglia C, senza le componenti:

```python
def getBestSet(self, K):
    self._candidati = [n for n in self._graph.nodes if self._valido(n)]   # filtro UNA volta, fuori dalla ricorsione
    self._bestSol = []
    self._bestScore = -1
    self._ricorsione([], 0, K)
    return self._bestSol, self._bestScore

def _ricorsione(self, parziale, i, K):
    if len(parziale) == K:
        score = sum(self._indice(n) for n in parziale)
        if score > self._bestScore:
            self._bestScore = score
            self._bestSol = copy.deepcopy(parziale)
        return
    for j in range(i, len(self._candidati)):
        c = self._candidati[j]
        if self._compatibile(c, parziale):      # se il testo ha esclusioni, altrimenti togli l'if
            parziale.append(c)
            self._ricorsione(parziale, j + 1, K)
            parziale.pop()
```

Varianti viste:
- **Nessun vincolo tra elementi** (2025-06-26 A/B/C): togli l'`if`. Nota onesta: senza vincoli la soluzione ottima sono semplicemente i K con indice più alto. Il testo chiede però la ricorsione, quindi va scritta.
- **Esclusione** (2020-06-03, dream team): "se prendo X, i nodi che X ha battuto non possono entrare" → `_compatibile` controlla che `c` non sia un successore di nessun membro e che nessun membro sia un successore di `c`.
- **Massima cardinalità + spareggio** (2025-01-13): non c'è K, vuoi la lista **più lunga** (nodi in ordine crescente di id, tutti della stessa categoria). Il best si aggiorna a ogni chiamata come nella famiglia A. A parità di lunghezza vince la lista con meno componenti connesse: `nx.number_connected_components(self._graph.subgraph(parziale))`.

```python
# spareggio: prima la lunghezza, poi il punteggio secondario
if (len(parziale) > len(self._bestSol) or
        (len(parziale) == len(self._bestSol) and self._score(parziale) < self._bestScore)):
    self._bestSol = copy.deepcopy(parziale)
    self._bestScore = self._score(parziale)
```

⚠️ Nelle soluzioni ufficiali del 2025 il prof usa un set `rimanenti` con `rimanenti.remove(c)` **dentro** il `for c in rimanenti`. Modificare la collezione su cui stai ciclando è un errore (`RuntimeError` con i set, elementi saltati con le liste). Lo schema a indice qui sopra evita il problema.

---

## 6. Famiglia E: simulazione (solo 2019–2020)

In 4 temi Java il punto 2 era una **simulazione a eventi** (agenti di polizia, stazioni di una cucina, interviste del produttore): coda di priorità (`heapq`) ordinata per tempo, un loop che estrae l'evento e ne genera di nuovi. **Dal 2025 non è mai uscita**: tutti i temi Python chiedono una ricorsione. Non ci perderei tempo.

---


## 7. Collegarla al controller (flet)

### Popolare il dropdown dopo `buildGraph`

Nel `handleCreaGrafo`, dopo aver costruito il grafo:
```python
self._view._ddNodo.options.clear()
for n in sorted(self._model.getAllNodes(), key=lambda x: x.name):
    self._view._ddNodo.options.append(ft.dropdown.Option(key=n.id, text=n.name))
self._view.update_page()
```
⚠️ `dd.value` restituisce la **key**, sempre come **stringa**. Se gli id sono interi (Chinook: `ArtistId`) serve `int(...)`; con IMDB (`'nm...'`, `'tt...'`) no.

### L'handler del bottone

```python
def handleRicorsione(self, e):
    self._view._txt_result.controls.clear()

    # 1. grafo costruito?
    nNodi, nArchi = self._model.getGraphDetails()
    if nNodi == 0:
        self._view.create_alert("Creare prima il grafo.")
        return

    # 2. nodo selezionato?
    if self._view._ddNodo.value is None:
        self._view.create_alert("Selezionare un nodo di partenza.")
        return
    source = self._model._idMap[self._view._ddNodo.value]     # int(...) se l'id è numerico

    # 3. N valido? (solo per famiglie B e C)
    try:
        N = int(self._view._txtInN.value)
    except (ValueError, TypeError):
        self._view.create_alert("N deve essere un intero.")
        return
    if N <= 0 or N > nNodi:
        self._view.create_alert(f"N deve essere tra 1 e {nNodi}.")
        return

    # 4. ricorsione
    sol, score = self._model.getBestSol(source, N)
    if not sol:
        self._view._txt_result.controls.append(ft.Text("Nessuna soluzione ammissibile."))
        self._view.update_page()
        return

    # 5. stampa (il testo chiede quasi sempre "ordinata alfabeticamente")
    for n in sorted(sol, key=lambda x: x.name):
        self._view._txt_result.controls.append(ft.Text(f"{n.name} - {n.numBrani}"))
    self._view._txt_result.controls.append(ft.Text(f"Totale nodi: {len(sol)}"))
    self._view._txt_result.controls.append(ft.Text(f"Punteggio ottimo: {score}"))
    self._view.update_page()
```

- Nella famiglia A (cammino) **non** ordinare: il cammino va stampato **nell'ordine in cui è percorso**.
- Ricorda: `len(sol) < N` non può succedere con il terminale `== N`; se non esiste nessun gruppo di N nodi, `sol` resta `[]` → messaggio "nessuna soluzione" (il testo dice "non sono ammesse eccezioni").

---

## 8. Errori tipici (controlla questi prima di chiamare il prof)

| Sintomo | Causa |
|---|---|
| La soluzione stampata è `[source]` o vuota | `self._bestSol = parziale` senza copia |
| Soluzioni lunghissime/assurde | manca il `pop()`, o è indentato fuori dal `for` |
| `RecursionError` | manca `v not in parziale` |
| Al secondo click esce la soluzione del click prima | best non resettato in `getBest...` |
| Grafo orientato: cammini che vanno "contromano" | usato un grafo non orientato o `predecessors` invece di `successors` |
| Gruppo non connesso / candidati sbagliati | candidati presi solo dai vicini dell'**ultimo** nodo invece che di tutto il gruppo (famiglia B) |
| `KeyError` su `self._idMap[...]` | `dd.value` è una stringa e l'id è un int |
| `KeyError: 'weight'` | arco creato senza `weight=` oppure scritto `'peso'` |
| "Strettamente" ignorato | usato `<=` al posto di `<` (o viceversa) |
| Il programma "si blocca" | non è bloccato, è esponenziale: prova con N piccolo, aggiungi il `frozenset` (B) o il pruning (C) |
| Score 0 mai salvato | `_bestScore = 0` con `>`: inizializza a `-1` |
| Ricorsione su un set che "salta" elementi o dà `RuntimeError` | stai facendo `remove` sulla collezione del `for`: usa lo schema a indice |
| Cerchi un minimo e la soluzione resta vuota | `_bestScore` parte da `0` o `-1`: per il minimo usa `float('inf')` e `<` |
| Cammino A5 che non arriva mai al target | manca il `return` dopo il target, oppure salvi il best anche quando `parziale[-1] != target` |


---

## 9. Cosa scriverti nel tuo readme (da copiare all'esame)

1. Lo **scheletro del §1** con i commenti (getBest + ricorsione).
2. Il **cammino** del §2 con `_vincolo` e `_pesoCammino`, più le righe di A4/A5 (lunghezza fissa e target).
3. Lo **schema a indice** (§4 e §5): è lo stesso per C e D.
4. I **candidati del gruppo** (§3) con il `frozenset`.
5. Le righe utili: `self._graph[u][v]['weight']`, `has_edge(u, v)`, `nx.connected_components` / `nx.weakly_connected_components`, `float('inf')` per i minimi.
6. Il **controller** del §7.

Per esercitarti: riprendi i temi in cui il punto 1 ti funziona già e fai solo il punto 2. Ordine consigliato, una famiglia alla volta:
- **A**: Lab11 / Sim. 2026-05-19 (A2) → 2026-06-29-A (A1) → 2025-07-10-A (A4 + A5, grafo orientato)
- **C**: 2026-06-29-B → 2026-01-12 (C1, minimo)
- **B**: 2026-06-29-C → 2026-09-14-A (B1)
- **D** se avanza tempo: 2025-06-26-A
