# FablePhysics

Laboratorio di fisica 2D in un unico file HTML, senza librerie: motore fisico scritto da zero su canvas.

**Apri `fisica.html` nel browser e basta.** Oppure la demo su GitHub Pages: https://gillus.github.io/FablePhysics/fisica.html

## Cosa fa

- Gravità (preset Luna / Terra / Giove / zero-g), urti con elasticità e attrito regolabili, palline di massa diversa
- Le palline ruotano (disco pieno, I = ½mr²): l'attrito le fa rotolare; resistenza al rotolamento regolabile
- Clic per creare palline, trascina per lanciarle, tasto destro per disegnare muri e rampe
- Molle e aste rigide tra palline o verso perni fissi: pendoli, catene, pendoli a molla
- Pausa, avanzamento a passo singolo, rallentatore
- Vettori velocità e grafico in tempo reale di energia cinetica, potenziale, elastica e totale
- Scene pronte: pendolo di Newton, piano inclinato, valanga, scontro tra palline che rotolano, catena

## Motore

- Integrazione di Eulero semi-implicita a sotto-passi fissi
- Urti a impulsi con restituzione, attrito di Coulomb con coppia, correzione posizionale
- Broadphase a griglia per le scene con molti corpi
- Vincoli di distanza (aste) risolti a livello di velocità e posizione, molle smorzate a forza
- Scala: 100 px = 1 m; energie in Joule

Il motore (`<script id="engine">`) non dipende dal DOM ed è testabile in Node.
