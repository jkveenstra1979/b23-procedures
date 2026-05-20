# B23 Flight Procedures PWA

Interactieve checklist-app voor Eindhovense Aeroclub Motorvliegen (EACM), gebaseerd op de B23 Flight Training Procedures (rev 4, 5-12-2025).

## Projectstructuur

```
b23-procedures.html   — volledige single-file PWA (HTML + CSS + JS)
CLAUDE.md             — dit bestand
```

De app is één zelfstandig HTML-bestand zonder build-stap, dependencies, of server. Alles — data, UI, Service Worker — zit erin.

## Architectuur

### Data

Alle vliegprocedures staan als een `const PROCEDURES` array bovenin het `<script>` blok. Elke categorie heeft deze structuur:

```js
{
  id: "takeoff",          // unieke slug, gebruikt als URL-state
  cat: "Takeoff",         // weergavenaam
  icon: `<svg .../>`,     // inline SVG, 22×22px, stroke-width 1.5, outline
  color: "#1a6fd4",       // accentkleur voor icoon-tekst
  bg: "rgba(...,0.15)",   // achtergrond van het icoon-blokje
  procs: [
    {
      id: "normal-to",           // unieke slug
      name: "Normal Take-off",   // weergavenaam
      warning: "...",            // optioneel — rood blok bovenaan detail
      note: "...",               // optioneel — blauw blok bovenaan detail
      steps: [
        { t: "Staptekst, mag <em>highlight</em> en <span class='warn'>rood</span> bevatten" }
      ]
    }
  ]
}
```

Stapteksten mogen beperkt HTML bevatten:
- `<em>waarde</em>` — amber highlight voor getallen/call-outs
- `<span class='warn'>tekst</span>` — rood voor kritieke waarschuwingen
- `<strong>LABEL</strong>` — wordt een sectiekop (niet aanklikbaar, geen stap-nummer)

### State

```js
let state = {
  view: 'home' | 'section' | 'detail',
  catId: null | string,
  procId: null | string,
  checkedSteps: { [procId]: { [stepIndex]: boolean } },
  favorites: string[],     // array van proc-id's, opgeslagen in localStorage
  darkMode: boolean,       // opgeslagen in localStorage
  favView: boolean,
  searchQuery: string
}
```

State wordt **niet** gepersisteerd tussen sessies, behalve `favorites` (`b23_favs`) en `darkMode` (`b23_dark`) via `localStorage`.

### Render-cyclus

Er is één centrale `render()` functie die bepaalt welke view actief is en de juiste render-functie aanroept:

```
render()
  → renderHome()     — categorie-kaarten
  → renderSection()  — procedure-lijst voor een categorie
  → renderFavs()     — bladwijzer-lijst
  → renderDetail()   — stappenlijst voor een procedure
  → renderSearch()   — zoekresultaten
```

Elke render-functie schrijft HTML rechtstreeks naar `el.innerHTML`. Er is geen virtual DOM of framework.

### Service Worker

De SW wordt inline geregistreerd via een `Blob` URL. Cache-naam: `b23-v1`. Bij het aanpassen van procedures: verhoog de cache-versie naar `b23-v2` (etc.) zodat gebruikers de nieuwe data krijgen.

```js
const CACHE = 'b23-v1';  // ← verhoog bij inhoudelijke updates
```

## Design tokens

```css
--bg           /* pagina-achtergrond */
--surface      /* kaart-achtergrond */
--surface2     /* ingedrukte staat / invoervelden */
--border       /* subtiele rand */
--accent       /* primaire blauw */
--accent-dim   /* accent met lage opacity */
--text         /* primaire tekst */
--text-2       /* secundaire tekst */
--text-3       /* placeholder / uitgeschakeld */
--warning      /* amber — call-outs en highlights */
--danger       /* rood — kritieke waarschuwingen */
--success      /* groen — afgevinkte stappen */
```

Donkere modus is de standaard. Lichte modus wordt ingeschakeld door `.light-mode` op `<body>`. Alle kleuren zijn CSS-variabelen — gebruik nooit hardcoded hex in nieuwe CSS.

## Veelvoorkomende taken

### Procedure toevoegen

Voeg een nieuw object toe aan het `procs`-array van de betreffende categorie in `PROCEDURES`. Gebruik een unieke kebab-case `id`. Verhoog daarna de Service Worker cache-versie.

### Categorie toevoegen

Voeg een nieuw object toe aan `PROCEDURES`. Kies een SVG-icoon van 22×22px, stroke-width 1.5, geen fill (outline). De kleur/bg volgen het bestaande patroon (zie andere categorieën).

### Icoon aanpassen

Alle categorie-iconen zijn inline SVG met deze attributen:
```
viewBox="0 0 24 24" fill="none" stroke="currentColor"
stroke-width="1.5" stroke-linecap="round" stroke-linejoin="round"
width="22" height="22"
```
Gebruik Tabler Icons (tabler.io) als bron — outline-variant, zelfde stroke-gewicht.

### Staptekst opmaken

```js
// Getal of call-out highlighten (amber):
{ t: "Lift-off at <em>50 kt</em>, initial climb <em>64 kt</em>" }

// Kritieke waarschuwing (rood, vet):
{ t: "When airspeed approaches <span class='warn'>RED ARC — DO NOT EXCEED Vne!</span>" }

// Sectiekop binnen een procedure (geen stap-nummer):
{ t: "<strong>RECOVERY</strong>" }
```

### Zoekfunctie aanpassen

`renderSearch()` doorzoekt `proc.name` en alle `step.t` strings (stripped van HTML-tags via `.toLowerCase()`). Uitbreiden naar `proc.note` of `proc.warning`: voeg die velden toe aan de match-logica in `renderSearch()`.

### Favorieten

Opgeslagen als array van `proc.id` strings in `localStorage` onder sleutel `b23_favs`. `toggleFav(pid)` voegt toe of verwijdert. `isFav(pid)` controleert.

## Beperkingen & aandachtspunten

- **Geen nested backtick template literals** — de render-functies gebruiken stringconcatenatie (`+`) voor dynamische HTML binnen `.map()`. Gebruik dit patroon consequent; nested backticks breken de JS-parser.
- **Geen externe dependencies** — de app werkt volledig offline en heeft geen CDN nodig.
- **localStorage** — checklist-vinkjes worden niet opgeslagen (bewust: per vlucht opnieuw beginnen). Alleen favorieten en thema-voorkeur worden bewaard.
- **iOS PWA** — installeer via Safari → Deel → "Voeg toe aan beginscherm". Push notifications werken alleen als geïnstalleerd.
- **Service Worker cache** — na aanpassen van procedures altijd `CACHE` verhogen, anders krijgen gebruikers verouderde data.
- **Single-file** — voeg geen externe bestanden toe. CSS en JS horen in het HTML-bestand.

## Bronbestand

De vliegprocedures zijn ontleend aan:
`B23 Flight Training Procedures 5-12-2025 (rev 4)` — Eindhovense Aeroclub Motorvliegen (EACM), Vliegbasis Eindhoven.

Bij revisies van het brondocument: controleer alle snelheden, hoogtes en RPM-waarden op wijzigingen en pas de `PROCEDURES` data aan. Verhoog vervolgens de SW cache-versie.
