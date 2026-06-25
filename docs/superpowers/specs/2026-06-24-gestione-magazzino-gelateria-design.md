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
5. **Stagione** a **due livelli manuali**: estate / inverno (interruttore
   gestito dal titolare). Ogni prodotto ha due soglie minime corrispondenti
   (estate più alta, inverno più bassa).
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
- `min_estate` — soglia minima stagione estate (più alta)
- `min_inverno` — soglia minima stagione inverno (più bassa)
- `archiviato` — booleano, default `false`
- `archiviato_il` — data di archiviazione (opz.)

Tutte le quantità (giacenze, minime, righe inventario, righe consegna) sono
**numeri interi** — niente decimali. Il valore **0** è ammesso.

### Archiviazione prodotti
Un prodotto può essere **archiviato**: da quel momento `archiviato = true` e il
prodotto **non è più visibile in nessuna vista o menu attivo** — inventario,
registrazione consegne, consumi, dashboard/ordini e anagrafiche lo escludono di
default. I **record storici già salvati** (inventari/consegne passati che lo
contenevano) restano integri e continuano a mostrarlo: l'archiviazione vale "da
quel momento in poi", non riscrive il passato.

Le liste prodotti hanno un filtro **"Mostra archiviati"** (default `false`); quando
attivo, gli archiviati ricompaiono nell'elenco (marcati come archiviati) e possono
essere **ripristinati** (`archiviato = false`).

### `inventari`
- `id`
- `data` (date, tipicamente una domenica)
- `note` (opz.)

### `inventario_righe`
- `id`
- `inventario_id` → `inventari`
- `prodotto_id` → `prodotti`
- `quantita` (intero)

Ogni inventario è uno **snapshot completo**. La giacenza attuale di un prodotto =
la sua quantità nell'**ultimo** inventario.

### `consegne`
- `id`
- `data` (date)

### `consegna_righe`
- `id`
- `consegna_id` → `consegne`
- `prodotto_id` → `prodotti` (il fornitore è derivato dal prodotto)
- `quantita` (intero)

Una consegna ha una data e contiene più fornitori, ciascuno con i propri prodotti
e quantità; il raggruppamento per fornitore in UI deriva dal `fornitore_id` del
prodotto. Le consegne sono solo storico + input per il calcolo consumo.

### `impostazioni` (riga singola)
- `stagione_corrente` — enum `estate | inverno`
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
   Accanto a ogni prodotto è mostrato **sempre il fornitore** (oltre all'unità di
   misura), così prodotti con nome identico (es. "Pistacchio Sicilia" di G&P e di
   Mec3) restano distinguibili. È mostrato anche, come riferimento, il **valore
   dell'inventario precedente** per ciascun prodotto, così da accorgersi al volo di
   errori grossolani di conteggio. UI ottimizzata per il telefono.
2. **Consegne** — registrazione (data, per fornitore i prodotti con quantità) e
   storico **ordinato per fornitore → prodotto**.
3. **Consumi** — consumo stimato (settimanale) per prodotto e storico tra inventari,
   **ordinato per nome**. Pagina di sola consultazione/statistica.
4. **Dashboard / Ordini da fare** — prodotti sotto o vicini alla minima della
   stagione corrente, con previsione di urgenza, **raggruppati per fornitore**.
5. **Anagrafiche** — gestione fornitori (nome, telefono, email, note), prodotti
   (nome, unità di misura, fornitore, due minime stagionali estate/inverno,
   **archivia/ripristina**, con filtro **"Mostra archiviati"** default off) e
   lista unità di misura/confezioni.
6. **Impostazioni** — interruttore stagione (estate/inverno), cadenza inventario
   (settimanale/bisettimanale) e visualizzazione del prossimo inventario atteso.

## Ordinamenti richiesti
- **Inventario** → per **nome prodotto**.
- **Consegne** e **ordini da fare** → per **fornitore**, poi per **prodotto**.

## Dati iniziali (seed)

Sorgente canonica del seed prodotti: [`data/seed/prodotti.csv`](../../../data/seed/prodotti.csv)
(133 prodotti). Le tabelle sotto sono la copia leggibile.

**Nota modello:** un prodotto appartiene a **un solo fornitore**. Gli articoli
fungibili acquistabili da più fornitori (es. Destrosio da Docks e Reire) restano
come **righe distinte** — scelta voluta.

### Fornitori (19)

Benvenuto · Bonella · Coopbox · Di Sano · Docks · Elenka · Ferrero · Fructital ·
Fugar · G&P · Gaggino · Irca · Martini · Mec3 · Officine Grafiche · Ostificio P. ·
Pack Studio · Reire · Siculabrioche

### Prodotti (per fornitore → prodotto → unità di misura)

**Benvenuto** — Mandorla affettata (Kg.) · Nocciola pasta (Latta Kg. 5) · Nocciola granella (Kg.) · Nocciola intere (Kg.) · Pistacchi granella (Kg.) · Pistacchi interi (Kg.)

**Bonella** — Alluminio rotolo (Nr.) · Rotoloni carta (Nr.) · Shopper carta personalizzati (Ct. 300pz) · Tovagliolini 17x17 personalizzati (Ct. 20.000pz) · Scatola torta 33x33 (Scatola) · Scatola torta  31x44 (Scatola)

**Coopbox** — Vaschetta PCR0029 350gr (Ct. 50pz) · Vaschetta PCR0020 500gr. (Ct. 50pz) · Vaschetta PCR0023 750gr. (Ct. 50pz) · Vaschetta PCR0026 1000gr. (Ct. 50pz) · Vaschetta PCR0028 1500gr. (Ct. 24pz)

**Di Sano** — Cocco Rapè (Kg.)

**Docks** — Cucchiaini Granita (Sacch.) · Destrosio (Kg.) · Palette gelato cm.9,5 comp. (Sacch.) · Zucchero (Kg.) · Nescafè (Bar.)

**Elenka** — Agrolina Ciaculli (Bottiglia) · Base Nolat (Sacch.)

**Ferrero** — Nutella (Secchio)

**Fructital** — Arancia variegato (Latta Kg. 4) · Bianca Vega (Sacch.) · Bianca Vega Stevia (Sacch.) · Mandorla pasta (Latta Kg. 3,5) · Viola (Latta Kg. 4) · Yogurt (Sacch. Kg. 2) · Crumble cacao (Sacch. Kg. 2) · Crumble caramello (Sacch. Kg. 2) · Biscotto SL pasta (Latta Kg. 4) · Biscotto SL variegato (Latta Kg. 4)

**Fugar** — Base Panna (Sacch. Kg. 3) · CNC (Latta Kg. 6) · Crema emiliana (Latta Kg. 3) · Glassatura (Latta Kg. 5,5) · Menta (Latta Kg. 5,5) · Pistacchio Puro (Latta Kg. 5,5) · Pistacchio Salty (Latta Kg. 3)

**G&P** — Amaretti granella (Kg.) · Bignè mm. 45 (Ct.) · Dobus roll 12 fogli (Fogli) · Fragola variegato (Latta Kg. 4) · Frutti di bosco variegato (Latta Kg. 3) · Nonnakrem pasta (Latta Kg. 3) · Pan di spagna (Fogli) · Pistacchio Sicilia (Kg.) · Semifreddo Stracciatella (Latta Kg. 1,25)

**Gaggino** — Bignè mm. 45 cad. gr. ca.3/4 (Ct. 6Sxgr 250) · Cannucce (Nr.) · Cono Coppa piccola (Ct. 720pz) · Cono Super Granellato h180xd.50 (Ct. 150pz) · Cucchiaini Granita (Sacch.) · Dobus roll 12 fogli (Fogli) · Gocce bianco (Kg.) · Gocce fondente (Kg.) · Palette gelato cm.9,5 comp. (Sacch.) · Pan di spagna (Fogli) · Scaglie fondenti (Kg.) · Lamponi rotti (Kg.) · Fragole gelo (Kg.) · Mango (Kg.) · Misto bosco gelo (Kg.) · Cono senza glutine (Scatola)

**Irca** — Cacao 22/24 (Kg.) · CNC Nocciolata Ice (Latta Kg. 5) · Glassatura Joy Couv. extra dark (Latta Kg. 5) · Gocce bianco Reno Concerto (Kg.) · Gocce fondente Reno Concerto (Kg.) · Joycream raspberry milk (Latta Kg. 5) · Joycream raspberry white (Latta Kg. 5) · Joy couverture extra white (Latta Kg. 5) · Franui Milk (Bar. 12X0,150) · Joyquick lampone (Sacch. 6xKg. 1,25) · Purea Lampone essential (Bar. 5xKg. 1) · Joypaste caffè (Bar. 6xKg. 1,2)

**Martini** — Cacao Aymara 22/24 (Kg.) · CNC Brunella Gianduia top (Latta Kg. 5) · Glassatura Stracc Super. Fond. Più (Latta Kg. 5) · Gocce bianco (Kg.) · Gocce fondente (Kg.)

**Mec3** — Biscottino pasta (Latta Kg. 4,5) · Cherry Mania (Latta Kg. 5) · Cookies Variegato (Latta Kg. 6) · Frollini (Sacch. gr. 700) · Liquirizia (Latta Kg. 3) · Mini cookies (Scatola 300pz) · Mr Nico Pasta (Latta Kg. 5) · Mr Nico Variegato (Latta Kg. 4) · Neutral Jelly (Latta Kg. 3) · Pistacchio Sicilia (Latta Kg. 4) · Vanilla Gourmet SZ pasta (Latta Kg. 3) · Vov pasta (Latta Kg. 4,5) · Mirror Glaze Fragola (Latta Kg. 3) · Mirror Glaze Violet (Latta Kg. 3) · Mirror Glaze Mango (Latta Kg. 3) · Crumble neutro (Sacch. Kg. 2,5) · Crumble frutti di bosco (Sacch. Kg. 2,5) · Pasta mascarpone (Latta Kg. 4,5) · Base caramello salato (Sacch.) · Caramello salato variegatura (Latta Kg. 6) · Dubai pistacchio variegato (Latta Kg. 5,5) · Guinness Kit (Ct.) · Crumble bosco (Sacch. Kg. 2)

**Officine Grafiche** — Fascette adesive (Rotoli) · Scatola torta 14x14 (Nr.) · Scatola torta 19x19 (Nr.) · Scatola torta 23x23 (Nr.) · Scatola torta 27x27 (Nr.) · Scatola torta 31x31 (Nr.) · ShopperBio (Ct.)

**Ostificio P.** — Cono grande OP5 (Ct. 225pz) · Cono maxi Cialda F91 (Ct. 189pz) · Cono medio OP4 (Ct. 300pz) · Cono piccolo OP1 (Ct. 300pz) · Gusto Tondo personalizzato (Ct. 1000pz)

**Pack Studio** — Bicchiere grande 45W pers. (Nr.) · Bicchiere piccolo 25M pers. (Nr.) · Coperchio bicchiere grande 42C (Nr.) · Coperchio bicchiere piccolo 25C (Nr.) · Coppa G4 personalizzata (Nr.) · Coppa grande CX160 pers. (Nr.) · Coppa media 90 pers. (Nr.) · Coppa piccola 80 pers. (Nr.)

**Reire** — Destrosio (Kg.) · Glucosio polvere (Kg.) · Inulina (Kg.) · Latte in polvere intero istantaneo (Kg.)

**Siculabrioche** — Brioches (Nr.)

### Unità di misura

Lista precaricata in `unita_misura` (valori unici usati nei prodotti):

```
Bar.
Bar. 12X0,150
Bar. 5xKg. 1
Bar. 6xKg. 1,2
Bottiglia
Ct.
Ct. 1000pz
Ct. 150pz
Ct. 189pz
Ct. 20.000pz
Ct. 225pz
Ct. 24pz
Ct. 300pz
Ct. 50pz
Ct. 6Sxgr 250
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
Sacch.
Sacch. 6xKg. 1,25
Sacch. gr. 700
Sacch. Kg. 2
Sacch. Kg. 2,5
Sacch. Kg. 3
Scatola
Scatola 300pz
Secchio
```

## Linee guida per i mockup (Claude Design)

Indicazioni da dare in pasto allo strumento di design prima di disegnare le schermate.

### Priorità UX
- **Mobile-first sull'inventario.** La schermata **Inventario** è quella più usata,
  da telefono, dai dipendenti: è la priorità di UX numero uno. Deve essere
  velocissima e usabile con una mano/col pollice — lista prodotti **ordinata per
  nome**, inserimento quantità rapido (tastierino numerico, campi grandi, target
  tap ampi), nessun passaggio superfluo, salvataggio chiaro.
- **Dashboard / Ordini da fare**: leggibilità a colpo d'occhio — urgenza evidente
  (prodotti sotto/vicini alla minima) e raggruppamento per fornitore ben distinto.
- Le altre schermate (Anagrafiche, Impostazioni, Consegne, Consumi) sono
  CRUD/tabelle standard: design pulito ma senza sovra-progettare.

### Coerenza tecnica
- L'implementazione userà **shadcn/ui + Tailwind CSS**: nei mockup restare su
  quello stile, così la traduzione in codice è quasi 1:1.
- **Componenti puliti**, **spaziature standard** (scala di spacing di Tailwind),
  tipografia e bordi/raggi coerenti con i default di shadcn/ui.
- Privilegiare componenti già esistenti in shadcn/ui (Button, Input, Card, Table,
  Dialog, Select, Tabs, Badge) invece di elementi custom.
- Responsive: stesso codice per desktop (consultazione) e mobile (inventario, PWA).

## Fuori scope (v1)
- Gestione utenti / ruoli multipli.
- Funzionamento offline.
- App nativa.
- Registrazione manuale dei consumi (scarti/rotture).
- Media del consumo limitata alla stagione corrente (possibile estensione futura).
