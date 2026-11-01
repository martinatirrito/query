# Controller – schemi pronti (TdP)

Schemi da copiare e adattare: cambi solo i nomi dei controlli della view (`_ddGenre`, `_ddCliente`, ...) e dei metodi del model.

## Indice

1. [Import e classe Controller](#1-import-e-classe-controller)
2. [fillDD: riempire i dropdown](#2-filldd-riempire-i-dropdown)
3. [Leggere i valori: stringa, id con idMap, numero](#3-leggere-i-valori-stringa-id-con-idmap-numero)
4. [DatePicker (se nella traccia ci sono le date)](#4-datepicker-se-nella-traccia-ci-sono-le-date)
5. [handleCreaGrafo (sempre uguale)](#5-handlecreagrafo-sempre-uguale)
6. [handleDettagli: stampe senza for e con for](#6-handledettagli-stampe-senza-for-e-con-for)
7. [Stampe tipiche del prof (componenti, archi con trattini, cammino)](#7-stampe-tipiche-del-prof-componenti-archi-con-trattini-cammino)
8. [handleRicorsione (punto 2)](#8-handlericorsione-punto-2)

---

## 1. Import e classe Controller

```python
import traceback

import flet as ft


class Controller:
    def __init__(self, view, model):
        # the view, with the graphical elements of the UI
        self._view = view
        # the model, which implements the logic of the program and holds the data
        self._model = model
```

---

## 2. fillDD: riempire i dropdown

**A) Dropdown di STRINGHE / ANNI** (generi, paesi, anni): il value sarà la stringa stessa.

```python
    def fillDDGenre(self):
        generi = self._model.getAllGenre()
        for g in generi:
            self._view._ddGenre.options.append(ft.dropdown.Option(g))
        self._view.update_page()

    def fillDDYears1(self):
        anni = self._model.getAllYears()
        for a in anni:
            self._view._ddAnnoDa.options.append(ft.dropdown.Option(a))
        self._view.update_page()
```

**B) Dropdown di OGGETTI** (clienti, album, artisti...): `key` = id come stringa, `text` = quello che si vede.
Il value del dropdown sarà la **key** (l'id come stringa) → poi lo converti con l'idMap (sezione 3).

```python
    def fillDDClienti(self):
        clienti = self._model.getAllClienti()
        for c in clienti:
            self._view._ddCliente.options.append(ft.dropdown.Option(key=str(c.CustomerId), text=str(c)))
        self._view.update_page()
```

**C) Dropdown dei NODI del grafo** (quello che serve per dettagli/ricorsione): si riempie **dopo** `buildGraph`, dentro `handleCreaGrafo`.

```python
    def fillDDNodi(self):
        self._view._ddCliente.options.clear()          # se ricrei il grafo non duplica le opzioni
        self._view._ddCliente.value = None
        for n in sorted(self._model.getClienti(), key=lambda x: x.FirstName):   # getClienti = list(self._graph.nodes)
            self._view._ddCliente.options.append(ft.dropdown.Option(key=str(n.CustomerId), text=str(n)))
        self._view.update_page()
```

> Se `text=str(c)` stampa una cosa brutta tipo `Customer(CustomerId=...)`, usa `text=f"{c.FirstName} {c.LastName}"` oppure definisci `__str__` nel dataclass.

---

## 3. Leggere i valori: stringa, id con idMap, numero

```python
        # STRINGA (genere, paese): lo uso così com'è
        genere = self._view._ddGenre.value

        # ANNO da dropdown: arriva come STRINGA -> se devo confrontarlo o fare conti, int()
        anno1 = self._view._ddAnnoDa.value
        anno2 = self._view._ddAnnoA.value
        if anno1 is None or anno2 is None:
            ...  # messaggio errore + update_page + return
        if int(anno1) > int(anno2):
            ...  # errore range

        # NODO INIZIALE (dettagli) -> value = key = id stringa -> int -> idMap -> OGGETTO
        if self._view._ddCliente.value is None:
            ...  # errore + return
        cliente = self._model._idMapC[int(self._view._ddCliente.value)]

        # NODO DELLA RICORSIONE (secondo dropdown, es. target / nodo di partenza P2): stessa conversione
        if self._view._ddTarget.value is None:
            ...
        target = self._model._idMapC[int(self._view._ddTarget.value)]

        # NUMERO da TextField (N, K, soglia): try / except
        try:
            n = int(self._view._txtN.value)          # float(...) se la soglia può essere decimale
        except (ValueError, TypeError):
            self._view.txt_result.controls.append(ft.Text("Inserire un numero intero", color="red"))
            self._view.update_page()
            return
```

> Convenzione: ai metodi del model passo sempre l'**OGGETTO nodo**, la conversione `idMap[int(...)]` la faccio qui nel controller.
> Se l'id nel DB è una stringa (non numero) niente `int()`: `self._model._idMap[self._view._dd.value]`.

---

## 4. DatePicker (se nella traccia ci sono le date)

**Nella view** (di solito è già fatto dal prof): un `ft.DatePicker` + un bottone che lo apre.

```python
        self._dp1 = ft.DatePicker(first_date=datetime.datetime(2021, 1, 1),
                                  last_date=datetime.datetime(2025, 12, 31),
                                  on_change=self._controller.handleDate1)
        self._btnData1 = ft.ElevatedButton("Data inizio", icon=ft.Icons.CALENDAR_MONTH,
                                           on_click=lambda e: self._page.open(self._dp1))
```

**Nel controller**: `value` del DatePicker è un **`datetime.datetime`** (non una stringa!).

```python
    # (facoltativo) mostra la data scelta in un Text della view
    def handleDate1(self, e):
        self._view._txtData1.value = self._view._dp1.value.strftime("%Y-%m-%d")
        self._view.update_page()

    # in handleCreaGrafo:
        data1 = self._view._dp1.value
        data2 = self._view._dp2.value
        if data1 is None or data2 is None:
            self._view.txt_result.controls.append(ft.Text("Selezionare entrambe le date", color="red"))
            self._view.update_page()
            return
        if data1 > data2:                       # i datetime si confrontano direttamente
            self._view.txt_result.controls.append(ft.Text("Data inizio dopo data fine", color="red"))
            self._view.update_page()
            return

        data1 = data1.date()                    # tolgo l'ora -> datetime.date
        data2 = data2.date()
        self._model.buildGraph(genere, data1, data2)
```

**Nel DAO/SQL**: passo le date come parametri `%s` e confronto solo la parte data.

```sql
WHERE DATE(i.InvoiceDate) BETWEEN %s AND %s
```

> Se ti serve solo l'anno della data: `data1.year` (intero). Se il prof vuole la data stampata: `data1.strftime("%d/%m/%Y")`.

---

## 5. handleCreaGrafo (sempre uguale)

Leggo → controllo (errore + `return`) → `try`: buildGraph, stampo nodi e archi, riempio il dropdown dei nodi → `except`.

```python
    def handleCreaGrafo(self, e):
        genere = self._view._ddGenre.value
        anno1 = self._view._ddAnnoDa.value
        anno2 = self._view._ddAnnoA.value

        if genere is None or anno1 is None or anno2 is None:
            self._view.txt_result.controls.clear()
            self._view.txt_result.controls.append(ft.Text(f'Errore, inserire genere, anno di inizio e anno di fine', color="red"))
            self._view.update_page()
            return

        try:
            self._model.buildGraph(genere, anno1, anno2)
            self._view._btnDettagli.disabled = False
            self._view.txt_result.controls.clear()
            Nnodi, Narchi = self._model.getGraphDetails()
            self._view.txt_result.controls.append(ft.Text("Grafo correttamente creato."))
            self._view.txt_result.controls.append(ft.Text(f"Numero di nodi: {Nnodi}"))
            self._view.txt_result.controls.append(ft.Text(f"Numero di archi: {Narchi}"))

            self.fillDDNodi()                    # dropdown dei nodi per dettagli / ricorsione
            self._view.update_page()

        except Exception as ex:
            traceback.print_exc()
            self._view.create_alert(f"Errore nella creazione del grafo: {ex}")
            # self._view.txt_result.controls.append(ft.Text(f"Errore nella creazione del grafo: {ex}")) se non ho il metodo nella view
            # self._view.update_page()
```

---

## 6. handleDettagli: stampe senza for e con for

```python
    def handleDettagli(self, e):
        self._view.txt_result.controls.clear()

        if self._view._ddCliente.value is None:
            self._view.txt_result.controls.append(ft.Text("Selezionare cliente", color="red"))
            self._view.update_page()
            return
        cliente = self._model._idMapC[int(self._view._ddCliente.value)]

        try:
            # ===== CASO 1: il model restituisce UN VALORE (numero, stringa, un nodo) -> niente for =====
            num = self._model.getNumRaggiungibili(cliente)
            self._view.txt_result.controls.append(ft.Text(f"Nodi raggiungibili: {num}"))

            # ===== CASO 2: restituisce una TUPLA (nodo, valore) -> la spacchetto, niente for =====
            nodo, peso = self._model.getBestInfluenza()
            self._view.txt_result.controls.append(ft.Text(f"{nodo} con valore {peso}"))
            # attenzione: se il model restituisce None, 0 (grafo vuoto) -> if nodo is None: ...

            # ===== CASO 3: restituisce una LISTA DI NODI (o un set) -> for =====
            vicini = self._model.getVicini(cliente)
            self._view.txt_result.controls.append(ft.Text(f"Vicini ({len(vicini)}):"))
            for n in vicini:
                self._view.txt_result.controls.append(ft.Text(f"{n.FirstName} {n.LastName}"))
            # se è un SET (es. componente): for n in sorted(comp, key=lambda x: x.FirstName)

            # ===== CASO 4: LISTA DI TUPLE (nodo, peso) -> for con spacchettamento =====
            lista = self._model.getViciniPesati(cliente)
            for v, peso in lista:
                self._view.txt_result.controls.append(ft.Text(f"{v.FirstName} {v.LastName}, peso {peso}"))

            # ===== CASO 5: LISTA NUMERATA (top 5) -> enumerate, parte da 1 =====
            top = self._model.getNodiPiuProfittevoli()
            for i, (n, score) in enumerate(top, 1):
                self._view.txt_result.controls.append(ft.Text(f"{i}. {n} - score {score}"))

            # ===== CASO 6: lista + valore insieme (es. lista nodi a parimerito + max) =====
            best_nodes, max_peso = self._model.getMaxPesoUscente()
            for n in best_nodes:
                self._view.txt_result.controls.append(ft.Text(f"{n} - peso {max_peso}"))

            # ===== CASO 7: decimali -> :.2f =====
            self._view.txt_result.controls.append(ft.Text(f"spesa {cliente.spesa:.2f}"))

            self._view.update_page()
        except Exception as ex:
            traceback.print_exc()
            self._view.create_alert(f"Errore nella stampa delle info: {ex}")
```

> `str()` di una tupla o di un set stampa la "repr" brutta (`(Customer(...), 3)`): spacchetta sempre la tupla nel `for` e stampa gli elementi uno per uno.

---

## 7. Stampe tipiche del prof (componenti, archi con trattini, cammino)

**Componenti connesse – SENZA enumerate**

```python
            num, largest, largestOrdinata, listaEntranti = self._model.getComponentiConnesse()
            self._view.txt_result.controls.append(ft.Text(f"Numero di componenti connesse: {num}"))
            self._view.txt_result.controls.append(ft.Text(f"La componente più grande ha {len(largest)} nodi:"))
            for n in largest:
                self._view.txt_result.controls.append(ft.Text(f"{n}"))
```

**Componenti connesse – CON enumerate** (Componente 1, 2, 3... con dimensione)

```python
            components = self._model.getComponentiNonBanali()      # lista di set
            for i, c in enumerate(components, 1):
                self._view.txt_result.controls.append(ft.Text(f"Componente {i} - {len(c)} nodi:"))
                for n in sorted(c, key=lambda x: x.FirstName):
                    self._view.txt_result.controls.append(ft.Text(f"   {n}"))
```

**Componente su una riga sola con join**

```python
            nomi = [f"{n.FirstName} {n.LastName}" for n in sorted(c, key=lambda x: x.FirstName)]
            self._view.txt_result.controls.append(ft.Text(", ".join(nomi)))
```

**Archi stile prof: nodo1 -> nodo2 | peso** (lista di tuple (u, v, peso), es. getTop5Archi)

```python
            archi = self._model.getTop5Archi()
            for u, v, peso in archi:
                self._view.txt_result.controls.append(ft.Text(f"{u} -> {v} | peso: {peso}"))      # ORIENTATO
                # NON ORIENTATO: f"{u} -- {v} | peso: {peso}"   oppure   f"{u} <-> {v} ({peso})"

            # se il model restituisce edges(data=True) -> la terza cosa è un DIZIONARIO
            for u, v, d in self._model.getTop3Archi():
                self._view.txt_result.controls.append(ft.Text(f"{u} -> {v} | peso: {d['weight']}"))
```

**Cammino (lista di nodi) con il peso di ogni tratto**

```python
            path, peso_tot = self._model.getCamminoMinimo(source, target)
            if len(path) == 0:
                self._view.txt_result.controls.append(ft.Text("Nessun cammino"))
            else:
                for i in range(len(path) - 1):
                    w = self._model._graph[path[i]][path[i + 1]]["weight"]
                    self._view.txt_result.controls.append(ft.Text(f"{path[i]} --{w}--> {path[i + 1]}"))
                self._view.txt_result.controls.append(ft.Text(f"Archi: {len(path) - 1}, peso totale: {peso_tot}"))
```

---

## 8. handleRicorsione (punto 2)

Esempio completo (gruppo di N clienti): controllo grafo → nodo da dropdown con idMap → N intero → ricorsione → stampa.

```python
    def handleCammino(self, e):
        # Punto 2: legge cliente e N, lancia la ricorsione del Model, stampa il gruppo migliore
        self._view.txt_result.controls.clear()

        # 1. il grafo deve esistere
        nNodi, nArchi = self._model.getGraphDetails()
        if nNodi == 0:
            self._view.txt_result.controls.append(ft.Text("Creare prima il grafo", color="red"))
            self._view.update_page()
            return

        # 2. cliente selezionato (value è la key = CustomerId come stringa)
        if self._view._ddCliente.value is None:
            self._view.txt_result.controls.append(ft.Text("Selezionare un cliente", color="red"))
            self._view.update_page()
            return
        cliente = self._model._idMapC[int(self._view._ddCliente.value)]

        # 3. N intero valido
        try:
            n = int(self._view._txtN.value)
        except (ValueError, TypeError):
            self._view.txt_result.controls.append(ft.Text("N deve essere un intero", color="red"))
            self._view.update_page()
            return
        if n < 1:
            self._view.txt_result.controls.append(ft.Text("N deve essere >= 1", color="red"))
            self._view.update_page()
            return

        # 4. ricorsione
        sol, score = self._model.getBestSol(cliente, n)
        if len(sol) == 0:
            self._view.txt_result.controls.append(ft.Text("Nessun gruppo valido"))
            self._view.update_page()
            return

        # 5. stampa in ordine alfabetico
        for c in sorted(sol, key=lambda x: (x.FirstName, x.LastName)):
            self._view.txt_result.controls.append(
                ft.Text(f"{c.FirstName} {c.LastName} ({c.Country}) - {c.spesa:.2f}"))
        self._view.txt_result.controls.append(ft.Text(f"Spesa totale: {score:.2f}"))
        self._view.update_page()
```

> Se la soluzione è un **cammino** (lista ordinata) NON fare `sorted`: stampala nell'ordine in cui è, con il blocco "Cammino" della sezione 7.
