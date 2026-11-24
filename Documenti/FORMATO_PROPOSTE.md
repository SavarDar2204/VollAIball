# Formato dei file delle proposte dei gesti (modello di Dario → strato adattivo e dashboard)

**Versione del 21/11 sera.** Niente più validazione incrociata per i gesti: il modello è addestrato a parte (immagini di
pallavolo generiche), POL e SRB non si usano per addestrarlo. Il modello si **applica così com'è**, senza riaddestrarlo.

## Cosa serve

| Partita | File | Uso | Priorità |
|---|---|---|---|
| DOM (set 1–3) | `proposte_DOM.csv` + `embedding_DOM.csv` | proposte della demo e curve di apprendimento | **necessaria** |
| SRB e POL (set 1–3) | `proposte_SRB.csv` + `embedding_SRB.csv`, `proposte_POL.csv` + `embedding_POL.csv` | taratura dei parametri dello strato adattivo senza toccare DOM | consigliata |

Senza SRB e POL i parametri dello strato restano quelli tarati sulle proposte finte (lo diremo nelle slide).
Proposte ed embedding vanno salvati **nella stessa esecuzione**: rifarla dopo costa ore di GPU.

## 1. `proposte_<PARTITA>.csv` — una riga per ogni tocco **rilevato**

Tutti i tocchi che il modello rileva, anche quelli falsi (fuori dal gioco, doppioni): lo scout li scarta e lo strato
impara a riconoscerli. I tocchi che il modello non vede li inserisce lo scout.

| Colonna | Tipo | Note |
|---|---|---|
| `id` | intero | univoco nella partita, **lo stesso nel file degli embedding** |
| `fotogramma`, `tempo_s` | intero / decimale | istante del tocco (video a 30 fps) |
| `squadra` | testo | ITA / DOM (POL, SRB), se il modello la ricava; altrimenti vuota |
| `p_battuta`, `p_ricezione`, `p_alzata`, `p_attacco`, `p_muro`, `p_difesa` | decimale 0–1 | probabilità dei 6 gesti (somma 1), **senza regole di sequenza**: la grammatica dello scambio la applica lo strato |
| `conf_tocco` | decimale 0–1 | confidenza che il tocco esista: molto utile, lo strato la usa per imparare cosa scartare |
| `palla_x`, `palla_y` | decimale, metri | facoltative: posizione a terra sotto la palla al tocco |
| `giocatore_x`, `giocatore_y` | decimale, metri | facoltative: piedi del giocatore che tocca (servono ai consigli sulle traiettorie) |

Coordinate del campo: x da 0 a 18 m lungo il campo (rete a 9 m), y da 0 a 9 m.

Non servono: la decisione del gesto (si ricava dalle probabilità), `set`, `scambio`, `ordine` (l'abbinamento con le
annotazioni lo fa lo strato per tempo, entro ±0,5 s), le valutazioni (le calcola Francesco col modello del giro 1 di Marta).

## 2. `embedding_<PARTITA>.csv` — un vettore per tocco

`id, e0, e1, …, eN`

- Le caratteristiche dell'**ultimo strato prima della decisione della classe**, così come escono, per **quella
  rilevazione** (il tocco), non per l'immagine intera: devono descrivere il tocco, perché lo strato cerca i tocchi
  simili a quelli già confermati o corretti dallo scout.
- **Nessuna riduzione**: la dimensione è quella del modello; la porta a 64 Francesco.
- Quattro decimali bastano. Se per un tocco il vettore non c'è, riga con i valori vuoti.

## Esempio

```
proposte_DOM.csv
id,fotogramma,tempo_s,squadra,p_battuta,p_ricezione,p_alzata,p_attacco,p_muro,p_difesa,conf_tocco,palla_x,palla_y,giocatore_x,giocatore_y
1,21417,713.900,DOM,0.81,0.02,0.03,0.02,0.01,0.11,0.92,,,17.6,4.3

embedding_DOM.csv
id,e0,e1,e2,...
1,0.1234,-0.5678,0.0012,...
```
