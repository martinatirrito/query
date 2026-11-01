# Ricorsione (punto 2) – COPIA E INCOLLA

Ogni blocco è **completo**: lo incolli nel Model così com'è e cambi solo le righe segnate `# <-- CAMBIA QUI`.
I nodi sono **oggetti** (il controller converte con `idMap[int(dd.value)]`), il grafo è `self._graph`.

Serve in cima al model: `import networkx as nx` (c'è già).

## Indice

0. [Quale blocco uso? (4 domande)](#0-quale-blocco-uso-4-domande)
1. [A. Cammino più lungo / di peso massimo (da un nodo)](#a-cammino-più-lungo--di-peso-massimo-da-un-nodo)
2. [A-bis. Cammino di lunghezza fissa N](#a-bis-cammino-di-lunghezza-fissa-n)
3. [A-ter. Cammino da sorgente a destinazione](#a-ter-cammino-da-sorgente-a-destinazione)
4. [B. Gruppo connesso di N nodi](#b-gruppo-connesso-di-n-nodi)
5. [C. K nodi in componenti connesse diverse](#c-k-nodi-in-componenti-connesse-diverse)
6. [D. K elementi da una lista (stile zaino)](#d-k-elementi-da-una-lista-stile-zaino)
7. [Controller del punto 2](#7-controller-del-punto-2)
8. [Orale: le 6 frasi che valgono per tutte](#8-orale-le-6-frasi-che-valgono-per-tutte)
9. [Se non funziona](#9-se-non-funziona)

---

## 0. Quale blocco uso? (4 domande)

Leggi il testo del punto 2 e rispondi in ordine:

| Il testo dice... | Blocco |
|---|---|
| "**cammino**" / "percorso" / "sequenza di nodi collegati" **da un nodo**, "più lungo" o "peso massimo" | **A** |
| cammino "di **lunghezza N**" / "esattamente N archi" | **A-bis** |
| cammino "**da X a Y**" / "tra due nodi" | **A-ter** |
| "**gruppo / insieme** di N nodi", "ogni nodo **adiacente** ad almeno uno del gruppo" | **B** |
| "K nodi, **ognuno in una componente connessa diversa**" | **C** |
| "scegli **K** elementi" tra quelli validi, l'adiacenza non c'entra | **D** |

Poi guarda il **vincolo** ("strettamente crescente", "paese diverso", "peso ≤ budget"...): in ogni blocco c'è una funzione `_vincolo...` / `_ammissibile...` con le opzioni già scritte. **Ne tieni UNA** (le altre restano commentate).

---

## A. Cammino più lungo / di peso massimo (da un nodo)

"Trovare il cammino semplice più lungo a partire da X tale che..." – nessuna lunghezza fissa: **ogni** cammino è una soluzione, quindi controllo il best a **ogni** chiamata e **non** c'è `return`.

```python
    def getBestCammino(self, source):
        self._bestPath = []
        self._bestPeso = -1
        self._ricCammino([source])
        return self._bestPath, self._bestPeso

    def _ricCammino(self, parziale):
        peso = self._pesoCammino(parziale)

        # è migliore del best? -> tieni UNA sola riga "if"                                  # <-- CAMBIA QUI
        if len(parziale) > len(self._bestPath) or (len(parziale) == len(self._bestPath) and peso > self._bestPeso):  # più lungo, a parità peso max
        # if len(parziale) > len(self._bestPath):                                            # solo più lungo
        # if peso > self._bestPeso:                                                          # solo peso massimo
            self._bestPath = list(parziale)        # COPIA, non parziale
            self._bestPeso = peso

        ultimo = parziale[-1]
        for v in self._graph.neighbors(ultimo):    # ORIENTATO: self._graph.successors(ultimo)   # <-- CAMBIA QUI
            if v not in parziale and self._vincoloCammino(parziale, v):     # "semplice" = niente nodi ripetuti
                parziale.append(v)
                self._ricCammino(parziale)
                parziale.pop()                     # backtracking

    def _pesoCammino(self, path):
        return sum(self._graph[path[i]][path[i + 1]]['weight'] for i in range(len(path) - 1))

    def _vincoloCammino(self, parziale, v):
        ultimo = parziale[-1]
        # tieni UNA sola opzione, commenta le altre                                         # <-- CAMBIA QUI

        # (1) NESSUN VINCOLO
        return True

        # (2) ATTRIBUTO DEL NODO strettamente decrescente (età, fatturato...)  ("non strettamente": <=)
        # return v.valore < ultimo.valore

        # (3) PESO DEGLI ARCHI strettamente crescente  (decrescente: <)   (non decrescente: >=)
        # if len(parziale) < 2:
        #     return True                           # primo arco: non c'è un arco precedente
        # pesoPrec = self._graph[parziale[-2]][ultimo]['weight']
        # pesoNuovo = self._graph[ultimo][v]['weight']
        # return pesoNuovo > pesoPrec

        # (4) PESO TOTALE entro un budget
        # return self._pesoCammino(parziale) + self._graph[ultimo][v]['weight'] <= self._budget
        #   (e in getBestCammino aggiungi il parametro: def getBestCammino(self, source, budget): self._budget = budget)
```

> Numero di archi del cammino = `len(path) - 1`. Stampa il cammino **nell'ordine** in cui è (non ordinarlo).

---

## A-bis. Cammino di lunghezza fissa N

"Cammino di esattamente N archi di peso massimo." C'è un **caso terminale** → confronto e `return`.

```python
    def getBestCamminoN(self, source, N):
        self._bestPath = []
        self._bestPeso = -1
        self._ricCamminoN([source], N)
        return self._bestPath, self._bestPeso

    def _ricCamminoN(self, parziale, N):
        if len(parziale) == N + 1:                 # N archi = N+1 nodi  (se il testo conta i NODI: == N)   # <-- CAMBIA QUI
            peso = self._pesoCammino(parziale)
            if peso > self._bestPeso:
                self._bestPath = list(parziale)
                self._bestPeso = peso
            return                                 # completo: non lo allungo

        ultimo = parziale[-1]
        for v in self._graph.neighbors(ultimo):    # ORIENTATO: successors(ultimo)
            if v not in parziale and self._vincoloCammino(parziale, v):
                parziale.append(v)
                self._ricCamminoN(parziale, N)
                parziale.pop()
```

> Usa `_pesoCammino` e `_vincoloCammino` del blocco A (incollali anche loro).

---

## A-ter. Cammino da sorgente a destinazione

"Cammino da X a Y di peso massimo (o con più nodi)." Il cammino è finito quando arriva a `target`.

```python
    def getBestCamminoST(self, source, target):
        self._bestPath = []
        self._bestPeso = -1                        # MINIMO: float('inf') e sotto  peso < self._bestPeso   # <-- CAMBIA QUI
        self._ricCamminoST([source], target)
        return self._bestPath, self._bestPeso

    def _ricCamminoST(self, parziale, target):
        if parziale[-1] == target:
            peso = self._pesoCammino(parziale)     # "più nodi": usa len(parziale) al posto del peso
            if peso > self._bestPeso:
                self._bestPath = list(parziale)
                self._bestPeso = peso
            return                                 # oltre il target non si va

        ultimo = parziale[-1]
        for v in self._graph.neighbors(ultimo):    # ORIENTATO: successors(ultimo)
            if v not in parziale and self._vincoloCammino(parziale, v):
                parziale.append(v)
                self._ricCamminoST(parziale, target)
                parziale.pop()
```

> Prima conviene controllare `nx.has_path(self._graph, source, target)`: se è False, messaggio "nessun cammino" nel controller.
> Usa `_pesoCammino` e `_vincoloCammino` del blocco A.

---

## B. Gruppo connesso di N nodi

"Gruppo di N nodi che contiene X, ogni nodo aggiunto adiacente ad almeno uno già nel gruppo, [vincolo], massimizzare la somma di ..."
Differenza dal cammino: i candidati sono i vicini di **tutto il gruppo**, non solo dell'ultimo.

```python
    def getBestGruppo(self, source, N):
        self._bestSol = []
        self._bestScore = -1
        self._visti = set()                        # gruppi già esplorati (in qualunque ordine)
        self._ricGruppo([source], N)
        return self._bestSol, self._bestScore

    def _ricGruppo(self, parziale, N):
        chiave = frozenset(parziale)               # stesso insieme in ordine diverso = stessa cosa
        if chiave in self._visti:
            return
        self._visti.add(chiave)

        if len(parziale) == N:
            score = sum(n.valore for n in parziale)          # <-- CAMBIA QUI (n.spesa, n.numBrani...)
            if score > self._bestScore:
                self._bestSol = list(parziale)
                self._bestScore = score
            return

        for c in self._candidatiGruppo(parziale):
            parziale.append(c)
            self._ricGruppo(parziale, N)
            parziale.pop()

    def _candidatiGruppo(self, parziale):
        res = set()                                # set: un vicino di due membri lo conto una volta
        for m in parziale:
            for v in self._graph.neighbors(m):     # ORIENTATO: nx.all_neighbors(self._graph, m)
                if v not in parziale and self._ammissibileGruppo(v, parziale):
                    res.add(v)
        return res

    def _ammissibileGruppo(self, c, parziale):
        # tieni UNA sola opzione                                                          # <-- CAMBIA QUI

        # (1) NESSUN VINCOLO
        return True

        # (2) nessuno dello STESSO ATTRIBUTO (es. paese diverso)
        # return all(c.Country != m.Country for m in parziale)

        # (3) nessuna coppia collegata da un arco di PESO 1
        # return all(not (self._graph.has_edge(c, m) and self._graph[c][m]['weight'] == 1) for m in parziale)

        # (4) adiacente a ESATTAMENTE UNO del gruppo (struttura ad albero)
        # return sum(1 for m in parziale if self._graph.has_edge(c, m)) == 1
```

> Se `self._bestSol` resta `[]` → nel controller "Nessun gruppo valido".

---

## C. K nodi in componenti connesse diverse

"K nodi, ognuno in una componente connessa diversa, massimizzare la somma di ..." (schema del prof).
Scorro le **componenti** una alla volta: per ognuna o la **salto** o **prendo uno dei suoi nodi**. Così due nodi della stessa componente sono impossibili.

```python
    def getBestComponenti(self, K):
        comps = [list(cc) for cc in nx.connected_components(self._graph)]    # ORIENTATO: nx.weakly_connected_components   # <-- CAMBIA QUI
        self._bestSol = []
        self._bestScore = -1                       # MINIMO (range): float('inf')
        if K > len(comps):
            return [], -1                          # impossibile: il controller stampa l'errore
        self._ricComp(comps, K, [], 0)
        return self._bestSol, self._bestScore

    def _ricComp(self, comps, K, parziale, i):
        if len(parziale) == K:
            score = sum(n.valore for n in parziale)          # <-- CAMBIA QUI
            # RANGE MINIMO: valori = [n.valore for n in parziale]; score = max(valori) - min(valori)  e sotto  score < self._bestScore
            if score > self._bestScore:
                self._bestSol = list(parziale)
                self._bestScore = score
            return

        if len(comps) - i < K - len(parziale):     # pruning: componenti rimaste insufficienti
            return

        self._ricComp(comps, K, parziale, i + 1)   # scelta 1: SALTO la componente i
        for n in comps[i]:                         # scelta 2: PRENDO un nodo della componente i
            parziale.append(n)
            self._ricComp(comps, K, parziale, i + 1)
            parziale.pop()
```

**Se il testo dice "il primo è il nodo scelto X"**: cambia solo `getBestComponenti`:

```python
    def getBestComponentiDa(self, source, K):
        comps = [list(cc) for cc in nx.connected_components(self._graph)]    # ORIENTATO: weakly
        altre = [c for c in comps if source not in c]   # la componente di source è già "usata"
        self._bestSol = []
        self._bestScore = -1
        if K > len(comps):
            return [], -1
        self._ricComp(altre, K, [source], 0)            # parto con source già dentro
        return self._bestSol, self._bestScore
```

> **Se è lenta** (somma da massimizzare): in `comps` tieni per ogni componente solo il nodo migliore → `comps = [[max(cc, key=lambda n: n.valore)] for cc in ...]`.
> Orale: "se scambio un nodo con uno migliore della stessa componente i vincoli restano rispettati e la somma sale". **Non** vale per il range minimo.

---

## D. K elementi da una lista (stile zaino)

"Scegli K elementi tra quelli validi che massimizzano la somma di ...". Il grafo serve solo a **filtrare** i candidati.
Indice `i`: dopo aver preso il candidato `j` provo solo quelli **dopo** `j` → ogni insieme generato una volta sola.

```python
    def getBestK(self, K):
        self._candidati = [n for n in self._graph.nodes if self._validoK(n)]   # filtro UNA volta
        self._bestSol = []
        self._bestScore = -1
        self._ricK([], 0, K)
        return self._bestSol, self._bestScore

    def _ricK(self, parziale, i, K):
        if len(parziale) == K:
            score = sum(n.valore for n in parziale)          # <-- CAMBIA QUI
            if score > self._bestScore:
                self._bestSol = list(parziale)
                self._bestScore = score
            return

        for j in range(i, len(self._candidati)):
            c = self._candidati[j]
            if self._compatibileK(c, parziale):
                parziale.append(c)
                self._ricK(parziale, j + 1, K)     # j+1: solo i candidati successivi
                parziale.pop()

    def _validoK(self, n):
        return True                                # es. return self._graph.degree(n) > 0    # <-- CAMBIA QUI

    def _compatibileK(self, c, parziale):
        return True                                # es. esclusione: return all(not self._graph.has_edge(c, m) for m in parziale)   # <-- CAMBIA QUI
```

---

## 7. Controller del punto 2

Schema completo (gruppo/componenti/K): vedi `README-controller.md`, sezione 8 (`handleCammino`).
Per un **cammino** la stampa è diversa: in ordine, con il peso di ogni arco.

```python
        path, peso = self._model.getBestCammino(source)
        if len(path) <= 1:
            self._view.txt_result.controls.append(ft.Text("Nessun cammino trovato"))
            self._view.update_page()
            return
        self._view.txt_result.controls.append(ft.Text(f"Cammino di {len(path) - 1} archi, peso totale {peso}:"))
        for i in range(len(path) - 1):
            w = self._model._graph[path[i]][path[i + 1]]['weight']
            self._view.txt_result.controls.append(ft.Text(f"{path[i]} --{w}--> {path[i + 1]}"))
        self._view.update_page()
```

Per C con `K > numero componenti`: se il model restituisce `[]` → `"K maggiore del numero di componenti"`.

---

## 8. Orale: le 6 frasi che valgono per tutte

1. **Cos'è**: "È una ricerca esaustiva con backtracking: costruisco la soluzione un nodo alla volta, provo tutte le estensioni ammissibili e tengo la migliore."
2. **Caso terminale**: "Quando la soluzione è completa (N nodi, arrivo al target...) calcolo il punteggio, lo confronto con il best e faccio `return`. Nel cammino più lungo non c'è terminale: ogni cammino è valido, quindi confronto a ogni chiamata e si ferma da solo quando non ci sono vicini ammissibili."
3. **append / ricorsione / pop**: "Aggiungo un candidato, esploro tutto quello che viene dopo, poi lo tolgo per riportare la lista com'era e provare il candidato successivo."
4. **Copia**: "`parziale` è una sola lista modificata per tutta la ricorsione; se salvassi il riferimento, alla fine dei pop il best sarebbe vuoto. Con `list(parziale)` salvo una fotografia."
5. **Reset in getBest**: "Il best va azzerato a ogni chiamata, altrimenti al secondo click resta la soluzione vecchia. Parto da -1 così anche un punteggio 0 viene salvato; per un minimo parto da infinito."
6. **Complessità**: "Esponenziale. Per questo B usa il frozenset (non riesploro lo stesso gruppo in un altro ordine), C e D usano l'indice (ogni insieme generato una volta) e C ha il pruning sulle componenti rimaste."

Domande secche:
- **Perché `v not in parziale`?** Cammino semplice = niente nodi ripetuti, e senza il controllo la ricorsione gira all'infinito.
- **Perché `>` e non `>=`?** Con `>` a parità tengo la prima soluzione trovata.
- **Perché `frozenset`?** Un `set` non può stare dentro un altro set (è modificabile); il frozenset è immutabile quindi hashabile.
- **`neighbors` vs `successors`?** Nel grafo orientato seguo solo gli archi uscenti (successors); `neighbors` su un DiGraph dà comunque solo i successori.

---

## 9. Se non funziona

| Sintomo | Causa |
|---|---|
| soluzione vuota o solo `[source]` | best salvato senza copia (`= parziale`) |
| soluzioni assurde / lunghissime | manca `pop()` o è fuori dal `for` |
| `RecursionError` | manca `v not in parziale` |
| al secondo click soluzione vecchia | best non resettato in `getBest...` |
| `KeyError` su idMap | `dd.value` è stringa → `int(...)` |
| `KeyError: 'weight'` | arco creato senza `weight=` |
| si "blocca" | è esponenziale: prova N piccolo (2, 3) |
| minimo sempre vuoto | best parte da -1: per il minimo `float('inf')` e `<` |
| ho lasciato due opzioni nel `_vincolo` | conta solo il primo `return`: tienine UNA |
