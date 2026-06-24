# Gestione magazzino gelateria — Design

**Data:** 2026-06-24
**Stato:** approvato in brainstorming, in attesa di revisione spec

## Obiettivo

Software per gestire il magazzino di una gelateria con **unico punto vendita**:
giacenza, consegne, consumi, fornitori e prodotti, con supporto all'inventario
periodico e suggerimento degli ordini.

### Obiettivi di business
- Evitare di rimanere sotto scorta su un prodotto.
- Facilitare l'inventario ai dipendenti (interfaccia mobile semplice e veloce).
- Consultare tutti i dati da web (desktop).

### Utilizzo
- **Web (desktop):** consultazione di tutti i dati e gestione anagrafiche/impostazioni.
- **Mobile (PWA):** esecuzione dell'inventario.

## Decisioni chiave (dal brainstorming)

1. **Modello giacenza = snapshot puro.** La giacenza mostrata di un prodotto è
   **esclusivamente la quantità dell'ultimo inventario**. Le consegne **non si
   sommano mai** alla giacenza.
2. **Consegne ricevute** = registrate solo come storico ed entrano nel calcolo
   del consumo. Non modificano la giacenza.
3. **Consumi** = **stimati**, mai inseriti a mano: per ogni intervallo tra due
   inventari `consumo = inventario_precedente + consegne_nel_periodo − inventario_attuale`,
   e il consumo settimanale di riferimento è la **media degli ultimi N intervalli**
   (default N = 4, con fallback a meno intervalli se non disponibili).
4. **Suggerimento ordini** = previsione di quando ogni prodotto scenderà sotto la
   minima della stagione corrente, ordinato per urgenza.
5. **Stagione** a **tre livelli manuali**: bassa / media / alta (interruttore
   gestito dal titolare). Ogni prodotto ha tre soglie minime corrispondenti.
6. **Cadenza inventario** ragiona in **domeniche**: l'inventario si fa di domenica.
   - Settimanale → prossimo inventario = prossima domenica.
   - Bisettimanale → prossimo inventario = tra due domeniche.
   - Modificabile in qualsiasi momento; ricalcola da lì in avanti senza
     riscrivere lo storico.
7. **Accesso:** login unico condiviso (una password per la gelateria).
8. **Piattaforma:** un'unica web app responsive / PWA (no app nativa, no offline).

## Stack tecnico

- **Next.js (App Router) + TypeScript** — frontend + backend nello stesso progetto.
- **Postgres (Neon, via Vercel Marketplace)** + **Drizzle ORM**.
- **Tailwind CSS + shadcn/ui** per UI pulita e mobile-friendly.
- **PWA** installabile (manifest + service worker minimale, no logica offline).
- **Deploy su Vercel.**
- **Auth:** password unica condivisa, validata via middleware + cookie di sessione
  HTTP-only. Nessuna gestione utenti.

## Modello dati

### `fornitori`
- `id`
- `nome`
- `telefono` (opz.)
- `email` (opz.)
- `note` (opz.)

### `unita_misura`
- `id`
- `nome` (unico) — etichetta **opaca** della confezione/formato (es. `Latta Kg. 3`,
  `Ct. 50pz`, `Bottiglia`). Nessun parsing di pesi/quantità: si conta in pezzi di
  quella confezione. Precaricata con la lista sotto, deduplicata; estendibile da
  anagrafica.

### `prodotti`
- `id`
- `nome`
- `unita_misura_id` → `unita_misura` (scelta via menu a tendina)
- `fornitore_id` → `fornitori`
- `min_bassa` — soglia minima stagione bassa
- `min_media` — soglia minima stagione media
- `min_alta` — soglia minima stagione alta

### `inventari`
- `id`
- `data` (date, tipicamente una domenica)
- `note` (opz.)

### `inventario_righe`
- `id`
- `inventario_id` → `inventari`
- `prodotto_id` → `prodotti`
- `quantita`

Ogni inventario è uno **snapshot completo**. La giacenza attuale di un prodotto =
la sua quantità nell'**ultimo** inventario.

### `consegne`
- `id`
- `data` (date)

### `consegna_righe`
- `id`
- `consegna_id` → `consegne`
- `prodotto_id` → `prodotti` (il fornitore è derivato dal prodotto)
- `quantita`

Una consegna ha una data e contiene più fornitori, ciascuno con i propri prodotti
e quantità; il raggruppamento per fornitore in UI deriva dal `fornitore_id` del
prodotto. Le consegne sono solo storico + input per il calcolo consumo.

### `impostazioni` (riga singola)
- `stagione_corrente` — enum `bassa | media | alta`
- `cadenza` — enum `settimanale | bisettimanale`

Il "prossimo inventario atteso" è **calcolato** da `data` dell'ultimo inventario +
cadenza, agganciato alla domenica corrispondente (non serve memorizzarlo).

## Calcoli

### Giacenza attuale
`giacenza(prodotto) = quantità del prodotto nell'ultimo inventario` (0/ignota se
il prodotto non compare nell'ultimo inventario).

### Consumo stimato (per prodotto)
Per ogni intervallo tra l'inventario precedente `P` e il successivo `A`:
```
consumo_intervallo = qty_P + somma_consegne(prodotto, da data_P a data_A) − qty_A
settimane = (data_A − data_P) / 7
consumo_settimanale_intervallo = consumo_intervallo / settimane
```
v1: il consumo settimanale di riferimento è la **media degli ultimi N intervalli**
(default `N = 4`), con fallback a meno intervalli se non ce ne sono abbastanza.
Smussa le settimane anomale restando reattivo. Estensione futura possibile:
limitare la media agli intervalli della stagione corrente.

### Previsione ordini
```
min_attiva = soglia del prodotto per la stagione_corrente
settimane_residue = (giacenza_attuale − min_attiva) / consumo_settimanale   (se consumo_settimanale > 0)
```
- Lista ordinata per `settimane_residue` crescente (più urgenti in cima).
- Se `giacenza_attuale ≤ min_attiva` → urgenza massima (residue ≤ 0).
- Se `consumo_settimanale ≤ 0` → nessuna previsione (non in lista urgenti).
- Raggruppabile per fornitore per facilitare gli ordini.

## Schermate (6)

1. **Inventario (mobile)** — elenco prodotti **ordinato per nome**; per ciascuno si
   inserisce la quantità; salva → crea un nuovo inventario (snapshot completo).
   UI ottimizzata per il telefono.
2. **Consegne** — registrazione (data, per fornitore i prodotti con quantità) e
   storico **ordinato per fornitore → prodotto**.
3. **Consumi** — consumo stimato (settimanale) per prodotto e storico tra inventari,
   **ordinato per nome**. Pagina di sola consultazione/statistica.
4. **Dashboard / Ordini da fare** — prodotti sotto o vicini alla minima della
   stagione corrente, con previsione di urgenza, **raggruppati per fornitore**.
5. **Anagrafiche** — gestione fornitori (nome, telefono, email, note), prodotti
   (nome, unità di misura, fornitore, tre minime stagionali) e lista unità di
   misura/confezioni.
6. **Impostazioni** — interruttore stagione (bassa/media/alta), cadenza inventario
   (settimanale/bisettimanale) e visualizzazione del prossimo inventario atteso.

## Ordinamenti richiesti
- **Inventario** → per **nome prodotto**.
- **Consegne** e **ordini da fare** → per **fornitore**, poi per **prodotto**.

## Dati iniziali — unità di misura (seed)

Lista precaricata in `unita_misura` (deduplicata; "Ct. 50pz" era ripetuto):

```
Bottiglia
Cartone
Ct 6Sx250gr.
Ct. 1000pz
Ct. 150pz
Ct. 189pz
Ct. 20.000pz
Ct. 225pz
Ct. 24pz
Ct. 300pz
Ct. 30pz
Ct. 50pz
Ct. 720pz
Fogli
Kg.
Latta Kg. 1,25
Latta Kg. 3
Latta Kg. 3,5
Latta Kg. 4
Latta Kg. 4,5
Latta Kg. 5
Latta Kg. 5,5
Latta Kg. 6
Nr.
Rotoli
Sacchetto
Sacchetto gr. 700
Sacchetto Kg. 1,25
Sacchetto Kg. 2
Sacchetto Kg. 2,5
Sacchetto Kg. 3
Sacch. 6xKg. 1,25
Scatola 300pz
Secchio
Cartone 10SxKg.1,2
Bar. 12X0,150
Bar. 6x1.2Kg
Bar. 5x1Kg
```

## Fuori scope (v1)
- Gestione utenti / ruoli multipli.
- Funzionamento offline.
- App nativa.
- Registrazione manuale dei consumi (scarti/rotture).
- Media del consumo limitata alla stagione corrente (possibile estensione futura).
