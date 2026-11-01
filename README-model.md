# Model – libreria metodi grafo (TdP)

Codice copiato **identico** dal mio `model.py`. Ogni sezione = un metodo da copiare.

## Indice

1. [Import e classe Model (`__init__`)](#import-e-classe-model-__init__)
2. [Costruzione grafo (`buildGraph`)](#costruzione-grafo-buildgraph)
3. [Numero nodi e archi (`getGraphDetails`)](#numero-nodi-e-archi-getgraphdetails)
4. [Top N archi ordinati](#top-n-archi-ordinati)
5. [Top N archi non ordinati](#top-n-archi-non-ordinati)
6. [Maggior numero archi uscenti](#maggior-numero-archi-uscenti)
7. [Maggior numero archi entranti](#maggior-numero-archi-entranti)
8. [Grado](#grado)
9. [Somma pesi uscenti più alta](#somma-pesi-uscenti-più-alta)
10. [Somma pesi entranti più alta](#somma-pesi-entranti-più-alta)
11. [Somma pesi incidenti](#somma-pesi-incidenti)
12. [Componente più grande: per ogni nodo il peso massimo dei suoi archi incidenti](#componente-più-grande-per-ogni-nodo-il-peso-massimo-dei-suoi-archi-incidenti)
13. [Numero componenti debolmente connesse e la più grande componente connessa](#numero-componenti-debolmente-connesse-e-la-più-grande-componente-connessa)
14. [Componente connessa che contiene un nodo](#componente-connessa-che-contiene-un-nodo)
15. [Tutte le componenti con più di un nodo, ordinate per dimensione decrescente](#tutte-le-componenti-con-più-di-un-nodo-ordinate-per-dimensione-decrescente)
16. [Miglior influenza: peso archi uscenti − peso archi entranti](#miglior-influenza-peso-archi-uscenti--peso-archi-entranti)
17. [Top N nodi più profittevoli](#top-n-nodi-più-profittevoli)
18. [Successori di un nodo ordinati per peso](#successori-di-un-nodo-ordinati-per-peso)
19. [Predecessori di un nodo ordinati per peso (solo ORIENTATO)](#predecessori-di-un-nodo-ordinati-per-peso-solo-orientato)
20. [Tutti i vicini (entranti + uscenti, senza duplicati)](#tutti-i-vicini-entranti--uscenti-senza-duplicati)
21. [Vicini/adiacenti ordinati per peso (NON ORIENTATO)](#viciniadiacenti-ordinati-per-peso-non-orientato)
22. [Nodi raggiungibili](#nodi-raggiungibili)
23. [Visite BFS / DFS](#visite-bfs--dfs)
24. [Nodi isolati (vale per ORIENTATO e NON ORIENTATO)](#nodi-isolati-vale-per-orientato-e-non-orientato)
25. [Archi con peso sopra la media](#archi-con-peso-sopra-la-media)
26. [Archi con peso sopra la mediana](#archi-con-peso-sopra-la-mediana)
27. [Archi di peso massimo (tutti, anche a parimerito)](#archi-di-peso-massimo-tutti-anche-a-parimerito)
28. [Nodi pozzo: nessun arco uscente, almeno uno entrante (solo ORIENTATO)](#nodi-pozzo-nessun-arco-uscente-almeno-uno-entrante-solo-orientato)
29. [Nodi sorgente: nessun arco entrante, almeno uno uscente (solo ORIENTATO)](#nodi-sorgente-nessun-arco-entrante-almeno-uno-uscente-solo-orientato)
30. [Archi sopra/sotto una soglia](#archi-soprasotto-una-soglia)
31. [Cammino minimo tra due nodi](#cammino-minimo-tra-due-nodi)
32. [Top N nodi per peso medio degli archi uscenti](#top-n-nodi-per-peso-medio-degli-archi-uscenti)
33. [Lista per dropdown (`getAllEmployee`, `getClienti`)](#lista-per-dropdown-getallemployee-getclienti)

## Import e classe Model (`__init__`)

```python
import statistics

import networkx as nx

from database.DAO import DAO


class Model:
    def __init__(self):
        self._graph = nx.DiGraph()
        self._idMapC = {}
```

## Costruzione grafo (`buildGraph`)

```python
    def buildGraph(self, employee):
        self._graph.clear()
        self._idMapC.clear()

        nodi = DAO.getAllNodes(employee)
        for c in nodi:
            self._idMapC[c.CustomerId] = c
        self._graph.add_nodes_from(nodi)

        edges = DAO.getAllEdges(employee, self._idMapC)
        for e in edges:
            self._graph.add_edge(e.c1, e.c2, weight = e.peso)
```

**Alternativa SENZA PESO (archi come tuple):**

```python
        edges = DAO.getAllEdges(employee, self._idMapC)
        for e1,e2 in edges:      #DATO CHE E' UNA TUPLA DEVO FARE COSì OPPURE FACCIIO ADD EDGES FROM(EDGES)
            self._graph.add_edge(e1, e2)
```
Va bene se nella tupla ci sono già gli oggetti nodo (il DAO usa l'idMap). Equivalente in una riga: `self._graph.add_edges_from(edges)`.
Se invece il DAO restituisce gli id `(id1, id2)`: `self._graph.add_edge(self._idMapC[e1], self._idMapC[e2])`.

## Numero nodi e archi (`getGraphDetails`)

```python
    def getGraphDetails(self):
        return len(self._graph.nodes), len(self._graph.edges)
```

## Top N archi ordinati

QUESTO METODO VA BENE SOLO SE IL GRAFO NON è ORIENTATO

```python
    """ 
    def getTop5Archi(self):
        if len(self._graph.edges) == 0:
            return []

        edges_with_weights = []
        for u, v in self._graph.edges:
            u, v = sorted((u, v), key=lambda c: c.FirstName)
            edges_with_weights.append((u, v, self._graph[u][v]['weight']))

        edges_sorted = sorted(edges_with_weights,
                              key=lambda x: (-x[2], x[0].FirstName, x[1].FirstName))

        return edges_sorted[:5]
    """
    #GRAFO ORIENTATO:
    def getTop5Archi(self):
        if len(self._graph.edges) == 0:
            return []

        edges_with_weights = []
        for u, v, d in self._graph.edges(data=True):
            edges_with_weights.append((u, v, d['weight']))

        edges_sorted = sorted(edges_with_weights,
                              key=lambda x: (-x[2], x[0].FirstName, x[1].FirstName))

        return edges_sorted[:5]
```

## Top N archi non ordinati

```python
    def getTop3Archi(self):
        return sorted(self._graph.edges(data=True), key=lambda x: x[2]["weight"], reverse=True)[:3]
```

## Maggior numero archi uscenti

```python
    def getBestUscenti(self):
        if len(self._graph.nodes) == 0:
            return None, 0

        max_degree = -1
        best_node = None
        for node, degree in self._graph.out_degree():
            if degree > max_degree:
                max_degree = degree
                best_node= node
        return best_node, max_degree
```

## Maggior numero archi entranti

```python
    def getBestEntranti(self):
        if len(self._graph.nodes) == 0:
            return None, 0

        max_degree = -1
        best_nodes = None
        for node, degree in self._graph.in_degree():
            if degree > max_degree:
                max_degree = degree
                best_nodes = [n for n, d in self._graph.in_degree() if d == max_degree]  #COSI HO PIU NODI SE C'E' PARIMERITO
        return best_nodes, max_degree

    #ARCHI USCENTI ED ENTRANTI LI POSSONO AVERE SOLO GRAFI ORIENTATI
```

## Grado

NEL CASO DI GRAFI NON ORIENTATI PUO' ESSERE CHIESTO SOLO IL GRADO
QUESTO METODO VA BENE PER TROVARE IL GRADO ANCHE DI UN GRAFO ORIENTATO QUANDO NON VIENE PRECISATO ENTRANTE O USCENTE

```python
    """
    def getBestNodoDegree(self):
        if len(self._graph.nodes) == 0:
            return None, 0

        max_degree = -1
        best_artist = None
        for node, degree in self._graph.degree():
            if degree > max_degree:
                max_degree = degree
                best_artist = node  #se non voglio più di uno metto la lista come fatto sopra
        return best_artist, max_degree
    """
```

## Somma pesi uscenti più alta

```python
    def getMaxPesoUscente(self):
        if len(self._graph.nodes) == 0:
            return [], 0

        max_peso = max(self._graph.out_degree(n, weight='weight') for n in self._graph.nodes)
        best_nodes = [n for n in self._graph.nodes
                      if self._graph.out_degree(n, weight='weight') == max_peso]
        return best_nodes, max_peso
```

## Somma pesi entranti più alta

```python
    def getMaxPesoEntrante(self):
        if len(self._graph.nodes) == 0:
            return [], 0

        max_peso = max(self._graph.in_degree(n, weight='weight') for n in self._graph.nodes)
        best_nodes = [n for n in self._graph.nodes
                      if self._graph.in_degree(n, weight='weight') == max_peso]
        return best_nodes, max_peso
```

## Somma pesi incidenti

pesi incidenti si calcolano in un grafo non orientato perchè non ha pesi uscenti e entranti

```python
    """
    questo va solo su non orientato perchè usa neighbours
    restituisco solo un nodo quindi in caso li voglio tutti metto lista best nodes come prima 

    def getSommaPesiIncidenti(self):
        if len(self._graph.nodes) == 0:
            return None, 0

        max_sum = -1
        best_artist = None
        for node in self._graph.nodes:
            weight_sum = sum(
                self._graph[node][neighbor]['weight']
                for neighbor in self._graph.neighbors(node)
            )
            if weight_sum > max_sum:
                max_sum = weight_sum
                best_artist = node
        return best_artist, max_sum
    """
    # versione universale
    # invece questo posso usarlo anche in uno orientato e mi da entranti + uscenti
    def getMaxPesoIncidente(self):
        if len(self._graph.nodes) == 0:
            return [], 0

        max_peso = max(self._graph.degree(n, weight='weight') for n in self._graph.nodes)
        best_nodes = [n for n in self._graph.nodes
                      if self._graph.degree(n, weight='weight') == max_peso]
        return best_nodes, max_peso
```

## Componente più grande: per ogni nodo il peso massimo dei suoi archi incidenti

Per ogni nodo della componente connessa più grande,
stampare il peso massimo tra i suoi archi incidenti, in ordine decrescente.”

```python
    def getLargestPesoMaxIncidente(self):
        if len(self._graph.nodes) == 0:
            return []
        largest = max(nx.weakly_connected_components(self._graph), key=len)
        # NON ORIENTATO: nx.connected_components

        lista = []
        for n in largest:
            # ORIENTATO: archi uscenti + entranti
            pesi = [d['weight'] for u, v, d in self._graph.out_edges(n, data=True)]
            pesi += [d['weight'] for u, v, d in self._graph.in_edges(n, data=True)]
            # NON ORIENTATO: pesi = [d['weight'] for u, v, d in self._graph.edges(n, data=True)]
            lista.append((n, max(pesi) if pesi else 0))

        lista.sort(key=lambda x: (-x[1], x[0].FirstName))
        return lista
        # peso MINIMO: min(pesi) e sort con +x[1] se lo chiede crescente
```

## Numero componenti debolmente connesse e la più grande componente connessa

weakly ignora direzione dell'arco
strongly ogni nodo deve raggiungere ogni altro seguendo le frecce, è molto più restrittiva

```python
    def getComponentiConnesse(self):
        if len(self._graph.nodes) == 0:
            return 0, [], [], []
        # nx.number_connected_components(self._graph) anche cosi per numero componente connessa

        # per grafo non orientato basta togliere weakly

        components = list(nx.weakly_connected_components(self._graph))  # len --> numero componenti connesse
        num_components = len(components)
        largest = max(components,key=len)  # usa la dimensione della componente per decidere qual è la più grande. Mi da tutti i nodi
        # len largest mi da quanti ce ne sono

        # STAMPARE NODI DI LARGEST ORDINATI IN SENSO DECRESCENTE DI GRADO DI NODI
        largestOrdinata = sorted(largest, key=lambda n: self._graph.degree(n), reverse=True)

        #STAMPARE IN ORDINE PER ARCHI ENTRANTI decrescente, a parimerito alfabetico (solo ORIENTATO)
        listaEntranti = [(n, self._graph.in_degree(n)) for n in largest]
        listaEntranti.sort(key=lambda x: (-x[1], x[0].FirstName))

        # VARIANTI (cambi solo la funzione dentro la lista, il sort resta uguale):
        #   archi uscenti:   self._graph.out_degree(n)
        #   peso entrante:   self._graph.in_degree(n, weight='weight')
        #   peso uscente:    self._graph.out_degree(n, weight='weight')
        #   peso incidente:  self._graph.degree(n, weight='weight')
        #   NON ORIENTATO:   self._graph.degree(n)  (non esistono in/out)

        return num_components, largest, largestOrdinata, listaEntranti
```

## Componente connessa che contiene un nodo

“Dato un cliente selezionato, stampare la dimensione della componente connessa a cui appartiene e i suoi nodi.”

NEL CONTROLLER comp = self._model.getComponenteDiNodo(source), poi len(comp) per la dimensione.

```python
    def getComponenteDiNodo(self, source):
        # ORIENTATO (debole): cerco la componente che contiene source
        for c in nx.weakly_connected_components(self._graph):
            if source in c:
                return c
        return set()
        # NON ORIENTATO (più semplice): return nx.node_connected_component(self._graph, source)
        # FORTE: stesso for con nx.strongly_connected_components
```

## Tutte le componenti con più di un nodo, ordinate per dimensione decrescente

```python
    def getComponentiNonBanali(self):
        components = list(nx.strongly_connected_components(self._graph))
        # DEBOLI (ORIENTATO): nx.weakly_connected_components
        # NON ORIENTATO:      nx.connected_components

        lista = [c for c in components if len(c) > 1]  # scarta quelle da 1 nodo
        lista.sort(key=lambda c: len(c), reverse=True)
        return lista
        # numero: len(lista); dimensione i-esima: len(lista[i])
```

## Miglior influenza: peso archi uscenti − peso archi entranti

```python
    def getBestInfluenza(self):
        if len(self._graph.nodes) == 0:
            return None, 0
        listNodesPesata = []
        for n in self._graph.nodes:
            score = 0
            for a in self._graph.out_edges(n, data=True):  # arco uscente  #data = true : dammi gli attributi
                score += a[2]["weight"]  # il peso perchè è una tupla
            for a in self._graph.in_edges(n, data=True):  # entranti
                score -= a[2]["weight"]
            listNodesPesata.append((n, score))

        #listNodesPesata.sort(key=lambda x: (-x[1], x[0].FirstName)) alfabetico
        listNodesPesata.sort(key=lambda x: x[1], reverse=True)
        return listNodesPesata[0]
```

## Top N nodi più profittevoli

```python
    def getNodiPiuProfittevoli(self):
        # deve per ogni nodo associare la somma dei pesi uscenti meno la somma dei pesi entranti
        listNodesPesata = []
        for n in self._graph.nodes:
            score = 0
            for a in self._graph.out_edges(n, data=True):  # arco uscente  #data = true : dammi gli attributi
                score += a[2]["weight"]  # il peso perchè è una tupla
            for a in self._graph.in_edges(n, data=True):  # entranti
                score -= a[2]["weight"]
            listNodesPesata.append((n, score))

        listNodesPesata.sort(key=lambda x: x[1], reverse=True)

        return listNodesPesata[0:5]
```

## Successori di un nodo ordinati per peso

```python
    def getSuccessori(self, source):
        #source = self._idMapC[int(idSource)]
        lista = [(n, self._graph[source][n]['weight']) for n in self._graph.successors(source)]
        lista.sort(key=lambda x: (-x[1], x[0].FirstName))
        return lista
    # entranti: predecessors(source) e peso self._graph[n][source]['weight']
    # NON ORIENTATO: neighbors(source) al posto di successors
```

## Predecessori di un nodo ordinati per peso (solo ORIENTATO)

```python
    def getPredecessori(self, source):
        lista = [(n, self._graph[n][source]['weight']) for n in self._graph.predecessors(source)]
        # attenzione: l'arco va da n a source, quindi [n][source] e non [source][n]
        lista.sort(key=lambda x: (-x[1], x[0].FirstName))
        return lista
```

## Tutti i vicini (entranti + uscenti, senza duplicati)

```python
    def getVicini(self, source):
        vicini = set(self._graph.successors(source)) | set(self._graph.predecessors(source))
        # NON ORIENTATO: list(self._graph.neighbors(source))
        return sorted(vicini, key=lambda n: n.FirstName)
```

## Vicini/adiacenti ordinati per peso (NON ORIENTATO)

```python
    def getViciniPesati(self, source):
        lista = [(v, self._graph[source][v]['weight']) for v in self._graph.neighbors(source)]
        lista.sort(key=lambda x: x[1], reverse=True)
        return lista
        # ORIENTATO: per gli uscenti usa getSuccessori, per gli entranti getPredecessori
```

## Nodi raggiungibili

"numero di nodi raggiungibili da un nodo" (sorgente ESCLUSA)

```python
    def getNumRaggiungibili(self, source):
        return len(nx.descendants(self._graph, source))  # ORIENTATO
        # NON ORIENTATO: len(nx.node_connected_component(self._graph, source)) - 1 #-1 per escludere la sorgente
```

## Visite BFS / DFS

VISITE: stessi nodi, ordine diverso
se non viene specificato ampiezza e profondita i nodi raggiungibili è uguale farlo con nc.descendants o con bfs/dfs
BFS = in ampiezza (per livelli), DFS = in profondità

```python
    def getBFSNodesFromEdges(self, source):  # sorgente ESCLUSA
        return [v for u, v in nx.bfs_edges(self._graph, source)]

    def getDFSNodesFromEdges(self, source):  # sorgente ESCLUSA
        return [v for u, v in nx.dfs_edges(self._graph, source)]

    def getBFSNodesFromTree(self, source):  # sorgente INCLUSA
        return list(nx.bfs_tree(self._graph, source).nodes())

    def getDFSNodesFromTree(self, source):  # sorgente INCLUSA
        return list(nx.dfs_tree(self._graph, source).nodes())
```

## Nodi isolati (vale per ORIENTATO e NON ORIENTATO)

```python
    def getNodiIsolati(self):
        return list(nx.isolates(self._graph))
        # equivalente a mano: [n for n in self._graph.nodes if self._graph.degree(n) == 0]

    # "numero di nodi isolati"
    def getNumIsolati(self):
        return nx.number_of_isolates(self._graph)
```

## Archi con peso sopra la media

Raccolgono tutti i pesi in una lista.
Calcolano la media, che è sum/len, oppure la mediana.
Rileggono gli archi e tengono solo quelli col peso sopra la soglia.

```python
    def getArchiSopraMedia(self):
        if len(self._graph.edges) == 0:
            return 0, []

        pesi = [d['weight'] for u, v, d in self._graph.edges(data=True)]
        media = sum(pesi) / len(pesi)

        archi = [(u, v, d['weight']) for u, v, d in self._graph.edges(data=True)
                    if d['weight'] > media]
        archi.sort(key=lambda x: -x[2])  # dal più pesante
        return media, archi
        # quanti sono: len(archi)
        # "maggiore o uguale" -> usa >=
#questi due metodi sopra e sotto valgono per ORIENTATI E NON
```

## Archi con peso sopra la mediana

```python
    def getArchiSopraMediana(self):
        if len(self._graph.edges) == 0:
            return 0, []

        pesi = [d['weight'] for u, v, d in self._graph.edges(data=True)]
        mediana = statistics.median(pesi)

        archi = [(u, v, d['weight']) for u, v, d in self._graph.edges(data=True)
                 if d['weight'] > mediana]
        archi.sort(key=lambda x: -x[2])
        return mediana, archi
```

## Archi di peso massimo (tutti, anche a parimerito)

```python
    def getArchiPesoMax(self):
        if len(self._graph.edges) == 0:
            return 0, []

        max_peso = max(d['weight'] for u, v, d in self._graph.edges(data=True))
        archi = [(u, v, d['weight']) for u, v, d in self._graph.edges(data=True)
                 if d['weight'] == max_peso]
        return max_peso, archi
        # peso MINIMO: min(...) al posto di max(...)
```

## Nodi pozzo: nessun arco uscente, almeno uno entrante (solo ORIENTATO)

```python
    def getNodiPozzo(self):
        return [n for n in self._graph.nodes
                if self._graph.out_degree(n) == 0 and self._graph.in_degree(n) > 0]
```

## Nodi sorgente: nessun arco entrante, almeno uno uscente (solo ORIENTATO)

```python
    def getNodiSorgente(self):
        return [n for n in self._graph.nodes
                if self._graph.in_degree(n) == 0 and self._graph.out_degree(n) > 0]
```

## Archi sopra/sotto una soglia

```python
    def getArchiSoglia(self, soglia):
        sopra = 0
        sotto = 0
        for u, v, d in self._graph.edges(data=True):
            if d['weight'] > soglia:
                sopra += 1
            elif d['weight'] < soglia:
                sotto += 1
            # uguali alla soglia: non contati (leggi bene la traccia: >= / <=)
        return sopra, sotto
    """ nel controller:
    sogliaStr = self._view._txtSoglia.value
    try:
        soglia = float(sogliaStr)      # int(...) se la traccia dice "intero"
    except ValueError:
        self._view.create_alert("Inserire un numero valido.")
        return
    sopra, sotto = self._model.getArchiSoglia(soglia)
    """
```

## Cammino minimo tra due nodi

“Selezionati due clienti A e B, stampare il cammino più breve da A a B (numero di archi) e il suo peso totale.”

(ORIENTATO e NON ORIENTATO)

```python
    def getCamminoMinimo(self, source, target):
        if not nx.has_path(self._graph, source, target):
            return [], 0

        # minimo NUMERO DI ARCHI (ignora i pesi)
        path = nx.shortest_path(self._graph, source, target)
        # minimo PESO TOTALE: nx.shortest_path(self._graph, source, target, weight='weight')

        peso = sum(self._graph[path[i]][path[i + 1]]['weight'] for i in range(len(path) - 1))
        return path, peso
        # numero archi = len(path) - 1
```

## Top N nodi per peso medio degli archi uscenti

Per ogni nodo con almeno un arco uscente, calcolare il peso medio degli archi uscenti. Stampare i primi 5 in ordine decrescente

```python
    def getTopPesoMedio(self, n=5):
        lista = []
        for nodo in self._graph.nodes:
            k = self._graph.out_degree(nodo)  # quanti archi
            if k > 0:  # evita divisione per zero
                somma = self._graph.out_degree(nodo, weight='weight')
                lista.append((nodo, somma / k))
        lista.sort(key=lambda x: (-x[1], x[0].FirstName))
        return lista[:n]
        # ENTRANTI: in_degree   NON ORIENTATO: degree
```

## Lista per dropdown (`getAllEmployee`, `getClienti`)

```python
    def getAllEmployee(self):
        return DAO.getAllEmployee()

    def getClienti(self):
        return list(self._graph.nodes)
```
