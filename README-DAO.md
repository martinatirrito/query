# DAO – scheletri pronti (TdP)

Scheletri da copiare: la query la scrivo io ogni volta, qui c'è solo la struttura (connessione, cursor, riempimento, chiusura).

## Indice

1. [Import e classe DAO](#1-import-e-classe-dao)
2. [getAllNodes (oggetti con **row)](#2-getallnodes-oggetti-con-row)
3. [getAllYears / valori semplici (row["colonna"])](#3-getallyears--valori-semplici-rowcolonna)
4. [getAllEdges – TUPLA senza peso](#4-getalledges--tupla-senza-peso)
5. [getAllEdges – TUPLA con peso](#5-getalledges--tupla-con-peso)
6. [getAllEdges – DTO Arco (con idMap dentro l'arco)](#6-getalledges--dto-arco-con-idmap-dentro-larco)
7. [Dataclass del nodo e dell'arco](#7-dataclass-del-nodo-e-dellarco)
8. [Cose da sapere per l'orale](#8-cose-da-sapere-per-lorale)

---

## 1. Import e classe DAO

```python
from database.DB_connect import DBConnect
from model.artist import Artist
from model.arco import Arco


class DAO():
    def __init__(self):
        pass
```

---

## 2. getAllNodes (oggetti con **row)

```python
    @staticmethod
    def getAllNodes(genere, anno):
        conn = DBConnect.get_connection()
        result = []

        if conn is None:                      # connessione fallita: get_connection() restituisce None
            print("Connessione fallita")
            return result                     # lista vuota, così il model non si rompe

        cursor = conn.cursor(dictionary=True)  # ogni row è un dizionario {nome_colonna: valore}
        query = """ ... """

        cursor.execute(query, (genere, anno))  # un valore per ogni %s, NELLO STESSO ORDINE della query
        for row in cursor:
            result.append(Artist(**row))      # **row: ogni colonna diventa il parametro omonimo del dataclass

        cursor.close()
        conn.close()
        return result
```

---

## 3. getAllYears / valori semplici (row["colonna"])

Per i dropdown (anni, generi, paesi): non creo oggetti, prendo solo il valore della colonna.

```python
    @staticmethod
    def getAllYears():
        conn = DBConnect.get_connection()
        result = []

        if conn is None:
            print("Connessione fallita")
            return result

        cursor = conn.cursor(dictionary=True)
        query = """ SELECT DISTINCT YEAR(i.InvoiceDate) AS anno ... ORDER BY anno """

        cursor.execute(query)                 # nessun parametro -> niente tupla
        for row in cursor:
            result.append(row["anno"])        # la chiave è l'ALIAS della colonna nella SELECT

        cursor.close()
        conn.close()
        return result
```

---

## 4. getAllEdges – TUPLA senza peso

Il model passa l'`idMap`: qui converto gli id in **oggetti nodo**, così nel model faccio direttamente `add_edge(e1, e2)`.

```python
    @staticmethod
    def getAllEdges(genere, anno, idMap):
        conn = DBConnect.get_connection()
        result = []

        if conn is None:
            print("Connessione fallita")
            return result

        cursor = conn.cursor(dictionary=True)
        query = """ SELECT ... AS id1, ... AS id2 ... """

        cursor.execute(query, (genere, anno))
        for row in cursor:
            result.append((idMap[row["id1"]], idMap[row["id2"]]))          # tupla (nodo1, nodo2)

        cursor.close()
        conn.close()
        return result
```

Nel model:

```python
        edges = DAO.getAllEdges(genere, anno, self._idMap)
        for e1, e2 in edges:
            self._graph.add_edge(e1, e2)
        # oppure in una riga: self._graph.add_edges_from(edges)
```

---

## 5. getAllEdges – TUPLA con peso

```python
    @staticmethod
    def getAllEdges(genere, anno, idMap):
        conn = DBConnect.get_connection()
        result = []

        if conn is None:
            print("Connessione fallita")
            return result

        cursor = conn.cursor(dictionary=True)
        query = """ SELECT ... AS id1, ... AS id2, COUNT(DISTINCT ...) AS peso ... """

        cursor.execute(query, (genere, anno))
        for row in cursor:
            result.append((idMap[row["id1"]], idMap[row["id2"]], row["peso"]))   # tupla (nodo1, nodo2, peso)

        cursor.close()
        conn.close()
        return result
```

Nel model:

```python
        edges = DAO.getAllEdges(genere, anno, self._idMap)
        for e1, e2, peso in edges:
            self._graph.add_edge(e1, e2, weight=peso)
```

---

## 6. getAllEdges – DTO Arco (con idMap dentro l'arco)

Creo un oggetto `Arco` (sezione 7) e ci metto dentro direttamente i nodi presi dall'idMap.

```python
    @staticmethod
    def getAllEdges(genere, anno, idMap):
        conn = DBConnect.get_connection()
        result = []

        if conn is None:
            print("Connessione fallita")
            return result

        cursor = conn.cursor(dictionary=True)
        query = """ SELECT ... AS id1, ... AS id2, COUNT(DISTINCT ...) AS peso ... """

        cursor.execute(query, (genere, anno))
        for row in cursor:
            result.append(Arco(idMap[row["id1"]], idMap[row["id2"]], row["peso"]))
            # SENZA PESO: Arco con solo c1, c2 -> Arco(idMap[row["id1"]], idMap[row["id2"]])

        cursor.close()
        conn.close()
        return result
```

Nel model:

```python
        edges = DAO.getAllEdges(genere, anno, self._idMap)
        for e in edges:
            self._graph.add_edge(e.c1, e.c2, weight=e.peso)
```

---

## 7. Dataclass del nodo e dell'arco

**Nodo** (`model/artist.py`): i campi devono avere **gli stessi nomi delle colonne** della SELECT, altrimenti `**row` dà errore.

```python
from dataclasses import dataclass

@dataclass
class Artist:
    ArtistId: int
    Name: str

    def __hash__(self):
        return hash(self.ArtistId)      # serve per usarlo come NODO del grafo (e chiave di dizionari/set)

    def __eq__(self, other):
        return self.ArtistId == other.ArtistId

    def __str__(self):
        return self.Name                # quello che si vede nelle stampe e nei dropdown
```

**Arco** (`model/arco.py`):

```python
from dataclasses import dataclass
from model.artist import Artist

@dataclass
class Arco:
    c1: Artist
    c2: Artist
    peso: int
```

---

## 8. Cose da sapere per l'orale

- **`if conn is None`**: `DBConnect.get_connection()` prende una connessione dal pool; se il DB non risponde restituisce `None`, e senza il controllo `conn.cursor()` darebbe `AttributeError`. Restituendo la lista vuota il programma continua (grafo vuoto).
- **`cursor(dictionary=True)`**: ogni riga arriva come dizionario `{colonna: valore}` invece che come tupla, quindi accedo per nome (`row["peso"]`) e posso fare `Classe(**row)`.
- **`**row`**: "spacchetta" il dizionario in parametri con nome: `Artist(**{"ArtistId": 1, "Name": "AC/DC"})` = `Artist(ArtistId=1, Name="AC/DC")`.
- **`%s` + tupla in `execute`**: i parametri li passa il driver (niente concatenazione di stringhe → evita SQL injection). Ordine dei valori = ordine dei `%s` nel testo. Con **un solo** parametro serve la virgola: `(anno,)`.
- **idMap passata al DAO**: il DB conosce solo gli id, il grafo vuole gli oggetti nodo; con `idMap[id]` uso **lo stesso oggetto** creato in `getAllNodes`, non un duplicato.
- **`__hash__`**: networkx salva i nodi in un dizionario, quindi un oggetto per essere nodo deve essere hashabile; un dataclass normale (con `eq`) non lo è da solo.
- **`close()`**: chiudo cursor e connessione per restituire la connessione al pool.
- **KeyError su `idMap[...]`**: succede se la query degli archi non ha gli **stessi filtri** della query dei nodi (esce un id che non è un nodo). Soluzione: stessi filtri, oppure `if row["id1"] in idMap and row["id2"] in idMap:` prima dell'append.
