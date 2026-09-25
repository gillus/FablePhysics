# FablePhysics

Laboratorio di fisica 2D in un unico file HTML, senza librerie: motore fisico scritto da zero su canvas.

**Apri `fisica.html` nel browser e basta.** Oppure la demo su GitHub Pages: https://gillus.github.io/FablePhysics/fisica.html

**Per la scuola:** [`docs/guida-studenti.pdf`](docs/guida-studenti.pdf) è una guida di 8 pagine da distribuire agli studenti (scuola secondaria di primo grado), con i comandi, le idee di fisica spiegate in modo semplice ed esperimenti guidati. Il sorgente è `docs/guida-studenti.html`, da cui il PDF si rigenera con Chrome (il comando è in testa al file).

## Cosa fa

- Gravità (preset Luna / Terra / Giove / zero-g), urti con elasticità e attrito regolabili, palline di massa diversa
- Le palline ruotano (disco pieno, I = ½mr²): l'attrito le fa rotolare; resistenza al rotolamento regolabile
- Clic per creare palline, trascina per lanciarle, tasto destro per disegnare muri e rampe
- Molle e aste rigide tra palline o verso perni fissi: pendoli, catene, pendoli a molla
- Pausa, avanzamento a passo singolo, rallentatore
- Vettori velocità e grafico in tempo reale di energia cinetica, potenziale, elastica e totale
- Scene pronte: pendolo di Newton, piano inclinato, valanga, scontro tra palline che rotolano, catena

## Motore in breve

- Integrazione di Eulero semi-implicita a sotto-passi fissi
- Urti a impulsi con restituzione, attrito di Coulomb con coppia, correzione posizionale
- Broadphase a griglia per le scene con molti corpi
- Vincoli di distanza (aste) risolti a livello di velocità e posizione, molle smorzate a forza
- Scala: 100 px = 1 m; energie in Joule

Il motore (`<script id="engine">`) non dipende dal DOM ed è testabile in Node.

## Modelli fisici

Questa sezione descrive i modelli usati dal motore, nell'ordine in cui entrano in gioco. I nomi tra parentesi sono quelli del codice in `fisica.html`.

### Unità di misura

Internamente posizioni e velocità sono in **pixel** e **secondi**; la scala è `PPM = 100` pixel per metro. I parametri fisici invece sono in unità SI:

| Parametro | Unità | Default | Cursore |
|---|---|---|---|
| Gravità `g` | m/s² | 9.81 | 0 – 30 |
| Densità `ρ` | kg/m² (densità superficiale, siamo in 2D) | 10 | 1 – 50 |
| Rigidità molla `k` | N/m | 60 | 5 – 400 |
| Smorzamento molla `c` | N·s/m | 0.3 | 0 – 5 |
| Elasticità `e` | adimensionale | 0.8 | 0 – 1 |
| Attrito `μ` | adimensionale | 0.2 | 0 – 1 |
| Resistenza al rotolamento | adimensionale | 0.3 | 0 – 3 |

La conversione con `PPM` avviene dentro il solutore, così energia (`energy()`) e quantità di moto (`momentum()`) escono già in Joule e kg·m/s.

### Corpi: dischi rigidi (`Body`)

Ogni pallina è un **disco pieno** omogeneo di raggio `r`:

- massa `m = π·r²·ρ`, con `r` in metri (una pallina da 18 px con densità 10 pesa circa 1 kg);
- momento d'inerzia `I = ½·m·r²`;
- stato: posizione `(x, y)`, velocità `(vx, vy)`, angolo `θ` e velocità angolare `ω`.

Un corpo con `fixed: true` è un **perno**: massa e inerzia infinite (`invMass = 0`), non si muove, non collide ed è escluso dalla broadphase. Serve come punto di aggancio per molle e aste.

### Muri (`Wall`)

Un muro è un **segmento spesso**: il segmento da `(x1, y1)` a `(x2, y2)` più un semispessore `half`, cioè una "capsula". Per l'urto si cerca il punto del segmento più vicino al centro della pallina; c'è contatto se la distanza è minore di `r + half`. I muri sono statici (massa infinita). Anche i bordi del canvas si comportano come muri.

### Integrazione nel tempo

L'interfaccia avanza la simulazione a **passo fisso** `Δt = 1/120 s` (moltiplicato per il fattore del rallentatore), con un accumulatore che esegue al massimo 40 passi per fotogramma. Ogni passo (`World.step`) è diviso in **8 sotto-passi** da `h = Δt/8` (circa 1 ms). In ogni sotto-passo:

1. **Forze**: si azzerano e si sommano le forze delle molle.
2. **Integrazione di Eulero semi-implicita** (o simplettica): prima la velocità, poi la posizione con la velocità *nuova*:

   ```
   v ← v + (F/m + g)·h
   x ← x + v·h
   θ ← θ + ω·h
   ```

   Rispetto a Eulero esplicito conserva molto meglio l'energia nei sistemi oscillanti (pendoli, molle). Qui si applicano anche la resistenza dell'aria lineare opzionale (`v ← v·(1 − airDrag·h)`, disattivata di default) e un limite di velocità di 60 m/s che evita instabilità numeriche.
3. **Broadphase**: si trovano le coppie di palline che potrebbero toccarsi.
4. **Solutore dei vincoli**: 6 iterazioni di tipo Gauss-Seidel. In ognuna si risolvono in sequenza le aste rigide, gli urti tra palline e gli urti con muri e bordi. Ripetere il giro fa convergere i contatti che si influenzano a vicenda (pile, catene, il pendolo di Newton).

### Molle (`Link` con `rigid: false`)

Molla lineare di **Hooke con smorzamento viscoso**, applicata lungo l'asse che unisce i centri dei due corpi:

```
F = k·(d − L₀) + c·v_rel
```

dove `d` è la distanza attuale, `L₀` la lunghezza a riposo e `v_rel` la velocità relativa lungo l'asse. La forza è uguale e opposta sui due estremi. Se la molla non ha `k` e `c` propri usa i valori globali dei cursori. L'energia elastica è `½·k·(d − L₀)²`.

### Aste rigide (`Link` con `rigid: true`)

Un'asta è un **vincolo di distanza** `|x_b − x_a| = L₀`, risolto direttamente invece che con una forza:

1. **Velocità**: si toglie la componente della velocità relativa lungo l'asta, con un impulso distribuito in proporzione all'inverso delle masse.
2. **Posizione**: si riportano i due corpi alla lunghezza `L₀`, spostandoli anch'essi in proporzione all'inverso delle masse (un perno non si muove, quindi si sposta solo la pallina).

È l'approccio dei motori "position based": stabile anche con catene lunghe, al prezzo di una piccola dissipazione di energia.

### Rilevamento delle collisioni

- **Broadphase**: sotto i 40 corpi si provano tutte le coppie (O(n²)). Da 40 in su si usa una **griglia spaziale hash** con celle di lato pari al diametro della pallina più grande: ogni pallina viene confrontata solo con quelle delle 9 celle vicine.
- **Narrowphase**: due palline si toccano se la distanza tra i centri è minore di `r_a + r_b`. La normale `n` è la direzione tra i centri e la compenetrazione è `r_a + r_b − d`.

### Risposta all'urto: impulsi

Gli urti sono istantanei e risolti con **impulsi** (variazioni istantanee di quantità di moto), calcolati solo se i corpi si stanno avvicinando (`v_n < 0`).

**Impulso normale con restituzione**, dove `v_n` è la velocità relativa lungo la normale:

```
j = −(1 + e)·v_n / (1/m_a + 1/m_b)
```

`e = 1` è un urto perfettamente elastico, `e = 0` perfettamente anelastico. Se la velocità d'urto è sotto 0.09 m/s la restituzione viene azzerata: così una pallina appoggiata non "vibra" rimbalzando all'infinito su scala microscopica.

**Attrito di Coulomb con rotazione.** Si prende la velocità relativa tangenziale *nel punto di contatto*, che tiene conto della rotazione (`v_t = (v_b − v_a)·t − ω_a·r_a − ω_b·r_b`), e si calcola l'impulso che la annullerebbe:

```
j_t = −v_t / (1/m_a + 1/m_b + r_a²/I_a + r_b²/I_b)
```

L'impulso è poi limitato dal cono di Coulomb, `|j_t| ≤ μ·j`. Se il limite non interviene le superfici aderiscono (la pallina rotola senza strisciare); se interviene, strisciano. L'impulso tangenziale produce anche una **coppia** che cambia `ω`: è questo che fa girare le palline sulle rampe e trasferisce rotazione negli urti tra palline.

Per i muri si usano le stesse formule con massa e inerzia infinite dal lato del muro.

**Resistenza al rotolamento.** Un disco ideale che rotola su un piano non perderebbe mai energia. Per simulare la deformazione reale delle superfici, a ogni contatto con un muro la velocità angolare viene ridotta di un piccolo fattore (`ω ← ω·(1 − 10⁻⁴·c_r)`); l'attrito poi riaccoppia la rotazione alla traslazione, e la pallina rallenta gradualmente. Vale solo per il contatto con muri e bordi, non tra palline.

**Correzione posizionale.** Gli impulsi correggono le velocità ma non eliminano la compenetrazione già avvenuta. Dopo ogni urto i corpi vengono separati del 60% della compenetrazione che supera una tolleranza di 0.05 px, in proporzione all'inverso delle masse. La tolleranza evita tremolii nei contatti a riposo; la correzione parziale evita di "sparare" via i corpi.

### Energia e quantità di moto

Il grafico mostra, in Joule:

- **cinetica**: traslazionale più rotazionale, `½·m·v² + ½·I·ω²`;
- **potenziale gravitazionale**: `m·g·h`, dove `h` è l'altezza del punto più basso della pallina sopra il bordo inferiore del canvas;
- **elastica**: `½·k·(d − L₀)²` sommata su tutte le molle;
- **totale**: la somma delle tre.

La quantità di moto totale `Σ m·v` è disponibile tramite `world.momentum()`.

### Approssimazioni e limiti noti

- L'energia totale **non è conservata esattamente**. Restituzione minore di 1, attrito, resistenza al rotolamento e smorzamento la dissipano per costruzione; anche con questi a zero, la proiezione delle aste e la correzione posizionale introducono piccole perdite. Il grafico lo rende visibile.
- Molle e aste agiscono sui **centri** dei corpi: non generano coppia e non collidono con nulla.
- I perni non collidono con le palline.
- Solo dischi e segmenti: niente poligoni o corpi di forma arbitraria.
- Il solutore è iterativo con un numero fisso di passate (6 iterazioni × 8 sotto-passi): i contatti multipli sono risolti in modo approssimato, non esatto.

## Provare il motore in Node

```bash
node -e '
const src = require("fs").readFileSync("fisica.html","utf8").match(/<script id="engine">([\s\S]*?)<\/script>/)[1];
const m = { exports: {} }; new Function("module", src)(m);
const { World, Scenes } = m.exports;
const w = new World(800, 600); Object.assign(w, Scenes.cradle(w));
const e0 = w.energy().tot; for (let i = 0; i < 240; i++) w.step(1/120);
console.log(e0, w.energy().tot);   // energia iniziale e dopo 2 secondi
'
```
