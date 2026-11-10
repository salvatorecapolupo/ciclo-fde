# Il ciclo macchina (sistemi e reti)
## 1. Il modello a due fasi

Il microprocessore è il nucleo di una **macchina di Von Neumann**: programma e dati stanno nella stessa memoria, ogni istruzione ha un indirizzo e un contatore di istruzioni (**PC**) percorre gli indirizzi in sequenza, a meno che l'istruzione stessa non imponga un salto. È un'architettura **SISD** (una istruzione alla volta, un flusso di dati).

Ogni istruzione viene eseguita in due fasi:

- **Fetch**: lettura dell'istruzione dalla memoria.
- **Execute**: interpretazione ed esecuzione effettiva.

Il simulatore separa l'interpretazione in una fase a sé, **Decode**, solo per chiarezza didattica: le fasi mostrate sono tre, ma Decode appartiene a "Execute" del modello a due fasi.

---

## 2. Gli ingredienti

Per capire il perché dei singoli microstep servono sei elementi:

1. **Tre bus**: indirizzi, dati, controllo. Su un bus passa **un solo valore per volta**.
2. **Registri interni (REG)** e **ALU**: la ALU opera solo su operandi che stanno già nei registri.
3. **Istruzione = codice operativo + operando**.
4. **Macchina a 1 indirizzo**: l'istruzione indica un solo operando (`ADD B` significa `A = A + B`); l'altro operando e il risultato sono nell'**accumulatore**, implicito nel codice operativo.
5. **Indirizzamento diretto**: l'operando non è il dato ma l'**indirizzo** in cui il dato si trova (`A = B`: il valore è all'indirizzo B).
6. **Organizzazione lineare**: un solo registro (PC) basta a indicare la prossima istruzione.

---

## 3. La regola che spiega tutto

> **Un microstep = un solo trasferimento tra registri, oppure un solo accesso alla memoria.**

La CPU non può "andare" in memoria con un comando solo. Per accedere a una cella deve fare tre cose in ordine:

| Cosa | Chi lo fa | Bus |
|---|---|---|
| 1. Dire **dove** | indirizzo nel registro **MAR** | bus indirizzi |
| 2. Dire **cosa fare** (leggere/scrivere) | unità di controllo | bus controllo |
| 3. Trasportare il **dato** | registro **MDR** | bus dati |

Quindi ogni accesso a memoria si spacchetta **sempre** nella stessa coppia:

```
MAR ← indirizzo
MDR ← M[MAR]        (lettura)     oppure     M[MAR] ← MDR   (scrittura)
```

Questo schema si ripete identico in fetch e in execute: è il motivo per cui i microstep sembrano "gli stessi comandi" in fasi diverse.

---

## 4. Fetch: 4 microstep

| Microstep | Perché |
|---|---|
| `MAR ← PC` | Il PC contiene l'indirizzo dell'istruzione; per leggere la memoria quell'indirizzo deve stare nel MAR. |
| `MDR ← M[MAR]` | Lettura: il contenuto della cella arriva sul bus dati e si ferma nel MDR. |
| `IR ← MDR` | L'istruzione va "parcheggiata" nell'**IR**, perché il MDR serve ancora (in execute conterrà l'operando) e verrebbe sovrascritto. |
| `PC ← PC + 1` | Il PC punta già alla prossima istruzione (organizzazione lineare). |

**Perché PC+1 nel fetch e non alla fine?** Perché a fine fetch la CPU deve già sapere dove andare dopo. Se l'istruzione è un salto, sovrascrive questo valore (vedi JUMP).

---

## 5. Decode: 2 microstep

Nel simulatore `IR = codice·1000 + operando`:

| Microstep | Significato |
|---|---|
| `OP = IR ÷ 1000` | Estrae il **codice operativo** (cosa fare). |
| `OPR = IR mod 1000` | Estrae l'**operando** (su cosa farlo). |

In un processore reale queste sono due porzioni di bit dell'IR (campi), non una divisione decimale: il simulatore usa il decimale per renderlo leggibile.

Dopo il decode OP e OPR sono validi; prima, quelli della istruzione precedente non hanno senso, per questo il simulatore li svuota all'inizio del fetch.

---

## 6. Execute: dipende dall'istruzione

### LOAD / ADD / SUB: 3 microstep

| Microstep | Perché |
|---|---|
| `MAR ← OPR` | L'operando è un **indirizzo** (indirizzamento diretto): va messo nel MAR. |
| `MDR ← M[MAR]` | Lettura del dato: stessa coppia MAR/MDR del fetch. |
| `ACC ← MDR` (LOAD) | Il dato entra nell'accumulatore. |
| `ACC ← ACC + MDR` (ADD) | La ALU somma accumulatore (implicito) e dato. |
| `ACC ← ACC − MDR` (SUB) | Stesso schema con la sottrazione. |

**Perché 3 e non 1?** Con l'indirizzamento **diretto** il dato non è nell'istruzione: va prima cercato in memoria. Con l'indirizzamento **immediato** (il dato è dentro l'istruzione) basterebbe `ACC ← OPR`: un solo microstep.

### STORE: 3 microstep (la scrittura è l'inverso)

```
MAR ← OPR
MDR ← ACC
M[MAR] ← MDR
```

Il dato parte dall'accumulatore, passa per il MDR e viene scritto in memoria. Nota: qui il MDR **non si legge dalla memoria**, si carica dall'ACC.

### JUMP: 1 microstep

```
PC ← OPR
```

Nessun accesso a memoria: l'operando è già l'indirizzo di destinazione. Il PC era stato incrementato nel fetch e ora viene **sovrascritto**: è così che l'istruzione "impone di modificare la sequenza".

### HALT

Nessun registro cambia: la CPU si ferma.

---

## 7. Esempio: `ADD 901` (istruzione `3901` all'indirizzo 101)

Stato di partenza: PC = 101, ACC = 5, M[901] = 7.

| Fase | Microstep | MAR | MDR | IR | PC | OP | OPR | ACC |
|---|---|---|---|---|---|---|---|---|
| FETCH | MAR ← PC | 101 | | | 101 | | | 5 |
| FETCH | MDR ← M[MAR] | 101 | 3901 | | 101 | | | 5 |
| FETCH | IR ← MDR | 101 | 3901 | 3901 | 101 | | | 5 |
| FETCH | PC ← PC+1 | 101 | 3901 | 3901 | **102** | | | 5 |
| DECODE | OP = IR÷1000 | 101 | 3901 | 3901 | 102 | 3 | | 5 |
| DECODE | OPR = IR mod 1000 | 101 | 3901 | 3901 | 102 | 3 | 901 | 5 |
| EXECUTE | MAR ← OPR | **901** | 3901 | 3901 | 102 | 3 | 901 | 5 |
| EXECUTE | MDR ← M[MAR] | 901 | **7** | 3901 | 102 | 3 | 901 | 5 |
| EXECUTE | ACC ← ACC+MDR | 901 | 7 | 3901 | 102 | 3 | 901 | **12** |

Osserva: il MDR contiene prima l'**istruzione** (fetch), poi il **dato** (execute). Per questo serve l'IR.

---

## 8. Domande ricorrenti

- **Perché serve l'IR se c'è già il MDR?** Perché il MDR è il "varco" verso il bus dati e viene riusato; l'istruzione deve restare ferma mentre viene decodificata ed eseguita.
- **Perché il PC si incrementa prima del decode?** Così il fetch successivo è già pronto, e un JUMP può semplicemente sovrascrivere il PC.
- **Perché l'accumulatore non compare nell'istruzione?** Macchina a 1 indirizzo: `ADD 901` ha un solo operando esplicito, l'altro operando e il risultato sono nell'ACC.
- **E nell'8086?** Il principio è lo stesso, ma la memoria è segmentata: l'indirizzo del fetch si calcola come `EA = CS × 16 + IP` (CS:IP sostituisce il PC). Esempio: segmento FFF0, offset EEE0 → EA = FFF00 + EEE0 = 10EDE0.
- **I processori reali fanno davvero un passo per volta?** No: con **prefetch** e **pipeline** il fetch dell'istruzione successiva si sovrappone all'execute di quella corrente. Il modello a microstep sequenziali è una semplificazione didattica.

---microstep: quello spacchettamento è il modello didattico standard della macchina di Von Neumann, costruito sugli ingredienti della sezione 2.
