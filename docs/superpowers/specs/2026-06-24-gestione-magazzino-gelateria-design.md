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
3. **Consumi** = **stimati**, mai inseriti a mano:
   `consumo = inventario_precedente + consegne_nel_periodo − inventario_attuale`.
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

### `prodotti`
- `id`
- `nome`
- `unita_misura` (es. kg, litri, pz, conf.)
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
Sull'intervallo tra l'inventario precedente `P` e l'attuale `A`:
```
consumo = qty_P + somma_consegne(prodotto, da data_P a data_A) − qty_A
settimane = (data_A − data_P) / 7
consumo_settimanale = consumo / settimane
```
v1: si usa l'**ultimo intervallo disponibile** (P = penultimo inventario,
A = ultimo). Estensione futura possibile: media su più intervalli.

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
5. **Anagrafiche** — gestione fornitori (nome, telefono, email, note) e prodotti
   (nome, unità di misura, fornitore, tre minime stagionali).
6. **Impostazioni** — interruttore stagione (bassa/media/alta), cadenza inventario
   (settimanale/bisettimanale) e visualizzazione del prossimo inventario atteso.

## Ordinamenti richiesti
- **Inventario** → per **nome prodotto**.
- **Consegne** e **ordini da fare** → per **fornitore**, poi per **prodotto**.

## Fuori scope (v1)
- Gestione utenti / ruoli multipli.
- Funzionamento offline.
- App nativa.
- Registrazione manuale dei consumi (scarti/rotture).
- Media del consumo su più intervalli (possibile estensione futura).
