<!--
  ═══════════════════════════════════════════════════════════════════════════════
  App.svelte  Hauptkomponente: Sekante 
  ═══════════════════════════════════════════════════════════════════════════════

  Eine Svelte-Komponente besteht aus drei Blöcken, die in einer einzigen Datei
  zusammenleben:

    1. <script>   →  Logik & Zustand (JavaScript)
    2. <template> →  HTML-Struktur (kein eigenes Tag, einfach der freie HTML-Teil)
    3. <style>    →  CSS (automatisch auf diese Komponente beschränkt / "scoped")

  Svelte kompiliert diese Datei zur Build-Zeit in reines, optimiertes JavaScript.
  Es gibt keine Virtual DOM – Svelte aktualisiert das DOM chirurgisch direkt.
  ═══════════════════════════════════════════════════════════════════════════════
-->

<script>
  // ── Importe ───────────────────────────────────────────────────────────────
  // onMount ist ein Lifecycle-Hook: Die übergebene Funktion wird einmalig
  // aufgerufen, nachdem die Komponente ins DOM eingehängt wurde.
  // Wir brauchen ihn, weil wir die SVG-Größe erst dann auslesen können.
  import { onMount } from 'svelte';

  // ── Mathematik: Funktion & Ableitung ──────────────────────────────────────
  // f(x) = x² – 2x + 3  →  Parabel, Scheitelpunkt bei x=1
  const f  = x => x * x - 2 * x + 3;

  // Analytische Ableitung f'(x) = 2x – 2  (wird nur für die Tangente genutzt)
  const df = x => 2 * x - 2;

  // ── Reaktive Zustandsvariablen ─────────────────────────────────────────────
  // In Svelte wird Reaktivität durch einfache let-Deklarationen ausgedrückt.
  // Wenn sich eine dieser Variablen ändert, aktualisiert Svelte automatisch
  // alle Template-Stellen, die von ihr abhängen.

  let deltaX   = 1.5;   // aktueller Δx-Wert des Sliders
  let x0       = 2.0;   // Entwicklungspunkt (fest, könnte man auch reaktiv machen)

  // SVG-Viewbox-Koordinaten - das "Koordinatensystem" unseres Diagramms
  const VW = 600;  // Viewbox-Breite
  const VH = 400;  // Viewbox-Höhe

  // Plotbereich (Abstand vom SVG-Rand für Achsenbeschriftungen)
  const PAD = { top: 20, right: 20, bottom: 50, left: 55 };

  // Sichtbarer x- und y-Bereich der Mathematik (nicht der Pixel!)
  const xMin = 0, xMax = 4;
  const yMin = 0, yMax = 9;

  // ── Koordinatentransformation ──────────────────────────────────────────────
  // SVG hat den Ursprung oben-links, y wächst nach unten.
  // Wir müssen mathematische (x, y) ↔ SVG-Pixel (px, py) umrechnen.

  // Plotbereich in Pixeln
  const plotW = VW - PAD.left - PAD.right;
  const plotH = VH - PAD.top  - PAD.bottom;

  // Mathematisches x → SVG-Pixel-x
  const toSvgX = x  => PAD.left + (x  - xMin) / (xMax - xMin) * plotW;
  // Mathematisches y → SVG-Pixel-y  (y-Achse gespiegelt!)
  const toSvgY = y  => PAD.top  + (1 - (y  - yMin) / (yMax - yMin)) * plotH;

  // ── Hilfsfunktion: Polyline-Punkte erzeugen ───────────────────────────────
  // Erzeugt einen SVG "points"-String aus einem Array von [x, y]-Paaren
  // (mathematische Koordinaten), z. B. "120,80 130,75 140,72 ..."
  function makeLine(xArr, fn) {
    return xArr.map(x => `${toSvgX(x)},${toSvgY(fn(x))}`).join(' ');
  }

  // Dichte x-Werte für den Funktionsgraph (300 Punkte → glatte Kurve)
  const xs = Array.from({ length: 300 }, (_, i) => xMin + i * (xMax - xMin) / 299);

  // ── Reaktive Berechnungen ($:) ─────────────────────────────────────────────
  // Der $:-Operator kennzeichnet eine "reaktive Deklaration":
  // Svelte erkennt automatisch, von welchen Variablen ein Ausdruck abhängt,
  // und führt ihn neu aus, sobald sich eine dieser Variablen ändert.
  // Hier: Alles, was von deltaX oder x0 abhängt, wird neu berechnet.

  // Sekanten-Steigung: klassischer Differenzenquotient
  $: slope_sek = (f(x0 + deltaX) - f(x0)) / deltaX;

  // Tangenten-Steigung: analytische Ableitung am Punkt x0
  $: slope_tan = df(x0);

  // Sekantensteigung als gerundeter Anzeigewert
  $: slopeDisplay = slope_sek.toFixed(4);

  // Differenz zur Tangentensteigung (zeigt den Approximationsfehler)
  $: errorDisplay = Math.abs(slope_sek - slope_tan).toFixed(4);

  // SVG-Punkte für den Funktionsgraph (ändert sich nie, aber schadet nicht)
  $: funcPoints = makeLine(xs, f);

  // SVG-Punkte für die Sekante:
  // Wir zeichnen sie über den gesamten x-Bereich, nicht nur zwischen x0 und x0+Δx
  $: sekantePoints = makeLine(xs, x => f(x0) + slope_sek * (x - x0));

  // SVG-Punkte für die Tangente
  $: tangentePoints = makeLine(xs, x => f(x0) + slope_tan * (x - x0));

  // SVG-Koordinaten der beiden Sekantenendpunkte (für die Markierungspunkte)
  $: px0 = { x: toSvgX(x0),          y: toSvgY(f(x0)) };
  $: px1 = { x: toSvgX(x0 + deltaX), y: toSvgY(f(x0 + deltaX)) };

  // ── Achsen-Hilfsfunktionen ─────────────────────────────────────────────────
  // Erzeugt ein Array gleichmäßiger Tick-Werte für eine Achse
  function ticks(min, max, step) {
    const result = [];
    for (let v = min; v <= max + 1e-9; v += step) result.push(+v.toFixed(6));
    return result;
  }
  const xTicks = ticks(xMin, xMax, 0.5);
  const yTicks = ticks(yMin, yMax, 1);

  // ── Farben ────────────────────────────────────────────────────────────────
  const COLOR_FUNC  = '#005A94';  // Dunkelblau  – Funktion f(x)
  const COLOR_SEK   = '#E60000';  // Rot         – Sekante
  const COLOR_TAN   = '#27ae60';  // Grün        – Tangente
  const COLOR_POINT = '#e87846';  // Violett     – Endpunkte der Sekante

  // ── Farbinterpolation für den Fehler-Indikator ────────────────────────────
  // Wir interpolieren von Rot (großer Fehler) zu Grün (kleiner Fehler),
  // um den Approximationsfehler visuell zu kodieren.
  $: errorRatio = Math.min(1, Math.abs(slope_sek - slope_tan) / 3);
  $: errorColor = interpolateColor(errorRatio);

  function interpolateColor(t) {
    // t = 0 → grün (#27ae60), t = 1 → rot (#c0392b)
    const r = Math.round(0x27 + t * (0xc0 - 0x27));
    const g = Math.round(0xae + t * (0x39 - 0xae));
    const b = Math.round(0x60 + t * (0x2b - 0x60));
    return `rgb(${r},${g},${b})`;
  }

  // ── Slider-Einstellungen ──────────────────────────────────────────────────
  const DX_MIN  = 0.01;   // Kleinster Δx-Wert  (fast = Tangente)
  const DX_MAX  = 2.0;    // Größter  Δx-Wert
  const DX_STEP = 0.01;   // Schrittweite des Sliders

  // ── onMount: Größenanpassung ───────────────────────────────────────────────
  // Wir nutzen viewBox statt fester Pixel-Größen → das SVG skaliert automatisch.
  // onMount wird hier nur für zukünftige Erweiterungen (ResizeObserver) reserviert.
  onMount(() => {
    // Hier könnte man z. B. einen ResizeObserver einbauen,
    // falls die Container-Breite zur Laufzeit bekannt sein muss.
  });
</script>

<!-- ══════════════════════════════════════════════════════════════════════════
     TEMPLATE  -  HTML-Struktur der Komponente
     ══════════════════════════════════════════════════════════════════════════

     Svelte-Template-Syntax:
       {ausdruck}           →  Wert einbetten (reaktiv)
       bind:value={var}     →  Zwei-Wege-Bindung: Änderungen im DOM schreiben
                               direkt in die Variable zurück (und umgekehrt)
       on:event={handler}   →  Event-Handler
       class:name={bool}    →  CSS-Klasse bedingt hinzufügen
     ══════════════════════════════════════════════════════════════════════════ -->

<div class="applet">

  <!-- ── Kopfzeile ─────────────────────────────────────────────────────────── -->
  <header>
    <h1>Applet Sekante → Tangente</h1>
    <p>
      Verschiebe Sie den Schieberegler und beobachten Sie, wie sich die Sekante
      der Tangente annähert, wenn&nbsp;Δx&nbsp;→&nbsp;0.
    </p>
  </header>

  <!-- ── SVG-Diagramm ──────────────────────────────────────────────────────── -->
  <!--
    viewBox="0 0 600 400" definiert das interne Koordinatensystem des SVG.
    Das SVG skaliert automatisch auf die CSS-Breite des Containers.
    preserveAspectRatio hält die Proportionen.
  -->
  <div class="chart-wrap">
    <svg viewBox="0 0 {VW} {VH}" preserveAspectRatio="xMidYMid meet" role="img"
         aria-label="Diagramm: Sekante nähert sich der Tangente">

      <!-- ── Hintergrund-Gitter ──────────────────────────────────────────── -->
      <!-- Svelte: {#each array as item} iteriert über ein Array -->
      {#each xTicks as tx}
        <line
          x1={toSvgX(tx)} y1={PAD.top}
          x2={toSvgX(tx)} y2={PAD.top + plotH}
          class="grid-line"
        />
      {/each}
      {#each yTicks as ty}
        <line
          x1={PAD.left}          y1={toSvgY(ty)}
          x2={PAD.left + plotW}  y2={toSvgY(ty)}
          class="grid-line"
        />
      {/each}

      <!-- ── Achsen ──────────────────────────────────────────────────────── -->
      <!-- x-Achse -->
      <line x1={PAD.left} y1={toSvgY(0)} x2={PAD.left + plotW} y2={toSvgY(0)} class="axis" />
      <!-- y-Achse -->
      <line x1={toSvgX(0)} y1={PAD.top} x2={toSvgX(0)} y2={PAD.top + plotH} class="axis" />

      <!-- ── Achsenbeschriftungen ────────────────────────────────────────── -->
      {#each xTicks as tx}
        {#if tx > 0}
          <text x={toSvgX(tx)} y={PAD.top + plotH + 18} class="tick-label" text-anchor="middle">
            {tx}
          </text>
        {/if}
      {/each}
      {#each yTicks as ty}
        {#if ty > 0}
          <text x={PAD.left - 8} y={toSvgY(ty)} class="tick-label" text-anchor="end" dominant-baseline="middle">
            {ty}
          </text>
        {/if}
      {/each}

      <!-- Achsentitel -->
      <text x={PAD.left + plotW / 2} y={VH - 4} class="axis-title" text-anchor="middle">x</text>
      <text x={14} y={PAD.top + plotH / 2} class="axis-title" text-anchor="middle"
            transform="rotate(-90, 14, {PAD.top + plotH / 2})">f(x)</text>

      <!-- ── Schattiertes Δx-Intervall ──────────────────────────────────── -->
      <!--
        Zeigt den Abstand Δx visuell als gefülltes Rechteck auf der x-Achse.
        Die Breite des Rechtecks ändert sich reaktiv mit dem Slider.
      -->
      <rect
        x={toSvgX(x0)}
        y={toSvgY(0) - 6}
        width={toSvgX(x0 + deltaX) - toSvgX(x0)}
        height={12}
        fill={COLOR_SEK}
        opacity="0.25"
      />
      <!-- Δx-Beschriftung über dem Intervall -->
      <text
        x={(toSvgX(x0) + toSvgX(x0 + deltaX)) / 2}
        y={toSvgY(0) - 10}
        class="delta-label"
        text-anchor="middle"
      >Δx = {deltaX.toFixed(2)}</text>

      <!-- ── Tangente (immer sichtbar, grün, gestrichelt) ───────────────── -->
      <polyline points={tangentePoints} class="line-tangente" />

      <!-- ── Sekante (rot, verändert sich mit Δx) ───────────────────────── -->
      <polyline points={sekantePoints} class="line-sekante" />

      <!-- ── Funktionsgraph (immer obenauf, dunkelblau) ─────────────────── -->
      <polyline points={funcPoints} class="line-func" />

      <!-- ── Markierungspunkte der Sekante ──────────────────────────────── -->
      <!-- Punkt P₁ = (x0, f(x0)) - Ausgangspunkt -->
      <circle cx={px0.x} cy={px0.y} r="5" fill={COLOR_POINT} />
      <text x={px0.x - 8} y={px0.y - 10} class="point-label" text-anchor="middle">P₁</text>

      <!-- Punkt P₂ = (x0+Δx, f(x0+Δx)) - wandert mit dem Slider -->
      <circle cx={px1.x} cy={px1.y} r="5" fill={COLOR_SEK} />
      <text x={px1.x - 8} y={px1.y - 10} class="point-label" text-anchor="middle">P₂</text>

      <!-- ── Legende ─────────────────────────────────────────────────────── -->
      <g transform="translate({PAD.left + 8}, {PAD.top + 10})">
        <!-- Funktion -->
        <line x1="0" y1="8"  x2="22" y2="8"  stroke={COLOR_FUNC} stroke-width="2.5" />
        <text x="27" y="9" class="legend-text">f(x) = x² − 2x + 3</text>

        <!-- Sekante -->
        <line x1="0" y1="26" x2="22" y2="26" stroke={COLOR_SEK} stroke-width="2.5" />
        <text x="27" y="27" class="legend-text">
          Sekante (Steigung = {slopeDisplay})
        </text>

        <!-- Tangente -->
        <line x1="0" y1="44" x2="22" y2="44" stroke={COLOR_TAN} stroke-width="2.5"
              stroke-dasharray="5,3" />
        <text x="27" y="45" class="legend-text">
          Tangente f′({x0}) = {slope_tan.toFixed(2)}
        </text>
      </g>

    </svg>
  </div>

  <!-- ── Steuerung: Slider ──────────────────────────────────────────────────── -->
  <div class="controls">
    <div class="slider-row">
      <!-- Svelte bind:value verknüpft den Slider bidirektional mit deltaX.
           Jede Bewegung des Sliders schreibt sofort in deltaX,
           und alle $:-Blöcke, die von deltaX abhängen, werden neu berechnet. -->
      <label for="dx-slider">Δx</label>
      <input
        id="dx-slider"
        type="range"
        min={DX_MIN}
        max={DX_MAX}
        step={DX_STEP}
        bind:value={deltaX}
      />
      <span class="dx-value">{deltaX.toFixed(2)}</span>
    </div>

    <!-- Schaltflächen für Preset-Werte – nützlich für Demonstrationen -->
    <div class="presets">
      <span class="preset-label">Voreinstellungen:</span>
      <!-- on:click setzt deltaX direkt; Svelte re-rendert reaktiv alles. -->
      {#each [2.0, 1.0, 0.5, 0.1, 0.01] as preset}
        <button
          class:active={Math.abs(deltaX - preset) < 0.005}
          on:click={() => deltaX = preset}
        >
          {preset}
        </button>
      {/each}
    </div>
  </div>

  <!-- ── Info-Kacheln: Steigungen und Fehler ───────────────────────────────── -->
  <div class="info-bar">

    <div class="info-tile">
      <span class="info-label">Sekantensteigung</span>
      <!--
        Die Steigung der Sekante = Differenzenquotient Δy/Δx.
        Der Wert ändert sich reaktiv mit deltaX.
      -->
      <span class="info-value" style="color: {COLOR_SEK}">{slopeDisplay}</span>
    </div>

    <div class="info-tile">
      <span class="info-label">Approximationsfehler</span>
      <!--
        Der Fehler zeigt, wie weit der Differenzenquotient von der echten
        Ableitung entfernt ist. Er wird mit Δx kleiner - das ist der
        Kern der numerischen Differentiation.
        Die Farbe interpoliert von Rot (großer Fehler) zu Grün (kleiner Fehler).
      -->
      <span class="info-value" style="color: {errorColor}">{errorDisplay}</span>
    </div>

  </div>

</div>

<!-- ══════════════════════════════════════════════════════════════════════════
     STYLES  -  Scoped CSS
     ══════════════════════════════════════════════════════════════════════════

     Svelte fügt automatisch ein eindeutiges Attribut (z. B. svelte-xyz123)
     zu jedem Element dieser Komponente hinzu. Dadurch gelten diese CSS-Regeln
     NUR für dieses Komponente - keine unbeabsichtigten globalen Seiteneffekte.

     Ausnahme: :global(...) um Stile auf Kind-Elemente anzuwenden, die von
     Svelte nicht direkt kontrolliert werden (hier nicht nötig).
     ══════════════════════════════════════════════════════════════════════════ -->

<style>
  /* ── Design-Tokens ──────────────────────────────────────────────────────── */
  :root {
    /* Alle wiederverwendbaren Werte als CSS-Variablen definieren -
       ändert man sie hier, ändert sich alles konsistent. */
    --bg:        #fafaf7;
    --surface:   #ffffff;
    --border:    #ddddd0;
    --text:      #1c1c1a;
    --muted:     #666660;
    --radius:    10px;
    --shadow:    0 3px 16px rgba(0,0,0,.09);
    --font-body: 'Georgia', 'Times New Roman', serif;
    --font-ui:   'Helvetica Neue', Helvetica, Arial, sans-serif;
  }

  /* ── Reset & Basis ────────────────────────────────────────────────────── */
  *, *::before, *::after { box-sizing: border-box; margin: 0; padding: 0; }

  /* ── Applet-Container ─────────────────────────────────────────────────── */
  .applet {
    font-family: var(--font-body);
    color: var(--text);
    background: var(--surface);
    border: 1px solid var(--border);
    border-radius: var(--radius);
    box-shadow: var(--shadow);
    overflow: hidden;
    /* max-width ermöglicht das Einbetten in schmalere iframes */
    max-width: 680px;
    margin: 0 auto;
  }

  /* ── Kopfzeile ─────────────────────────────────────────────────────────── */
  header {
    padding: 20px 24px 16px;
    border-bottom: 1px solid var(--border);
    background: #f3f4f6;
  }
  header h1 {
    font-size: 1.1rem;
    font-weight: normal;
    letter-spacing: .02em;
  }
  header p {
    margin-top: 5px;
    font-size: .82rem;
    font-family: var(--font-ui);
    color: var(--muted);
    line-height: 1.5;
  }

  /* ── SVG-Wrapper ───────────────────────────────────────────────────────── */
  .chart-wrap {
    padding: 16px 20px 0;
    background: var(--surface);
  }
  .chart-wrap svg {
    width: 100%;
    height: auto;
    display: block;
  }

  /* ── SVG-Stile (werden auf Elemente innerhalb des SVG angewendet) ─────── */
  .grid-line {
    stroke: #e8e8e0;
    stroke-width: 1;
  }
  .axis {
    stroke: #aaa;
    stroke-width: 1.5;
  }
  .tick-label {
    font-family: var(--font-ui);
    font-size: 11px;
    fill: var(--muted);
  }
  .axis-title {
    font-family: var(--font-ui);
    font-size: 12px;
    fill: var(--muted);
    font-style: italic;
  }
  .delta-label {
    font-family: var(--font-ui);
    font-size: 11px;
    fill: #e60000;
    font-weight: 600;
  }
  .point-label {
    font-family: var(--font-ui);
    font-size: 11px;
    fill: #8e44ad;
  }
  .legend-text {
    font-family: var(--font-ui);
    font-size: 11px;
    fill: var(--text);
    dominant-baseline: middle;
  }

  /* Kurven-Stile - fill:none ist bei polyline wichtig, sonst wird die
     Fläche unter der Kurve als Polygon gefüllt */
  .line-func {
    fill: none;
    stroke: #1a5276;
    stroke-width: 2.5;
    stroke-linejoin: round;
  }
  .line-sekante {
    fill: none;
    stroke: #c0392b;
    stroke-width: 2;
    stroke-linejoin: round;
    /* Sanfte Überblendung beim Wechsel der Steigung */
    transition: stroke 0.2s;
  }
  .line-tangente {
    fill: none;
    stroke: #27ae60;
    stroke-width: 1.8;
    stroke-dasharray: 7 4;
    stroke-linejoin: round;
  }

  /* ── Steuerleiste ─────────────────────────────────────────────────────── */
  .controls {
    padding: 16px 24px 14px;
    display: flex;
    flex-direction: column;
    gap: 10px;
    border-top: 1px solid var(--border);
  }

  .slider-row {
    display: flex;
    align-items: center;
    gap: 12px;
    font-family: var(--font-ui);
    font-size: .85rem;
  }
  .slider-row label {
    min-width: 24px;
    color: var(--muted);
    font-style: italic;
  }
  .slider-row input[type=range] {
    flex: 1;
    accent-color: #c0392b;
    cursor: pointer;
    height: 6px;
  }
  .dx-value {
    min-width: 40px;
    font-weight: 700;
    font-size: .95rem;
    text-align: right;
    font-variant-numeric: tabular-nums;
    color: #c0392b;
  }

  /* ── Preset-Schaltflächen ─────────────────────────────────────────────── */
  .presets {
    display: flex;
    align-items: center;
    gap: 8px;
    flex-wrap: wrap;
  }
  .preset-label {
    font-family: var(--font-ui);
    font-size: .78rem;
    color: var(--muted);
    margin-right: 2px;
  }
  button {
    font-family: var(--font-ui);
    font-size: .8rem;
    padding: 3px 10px;
    border: 1px solid var(--border);
    border-radius: 4px;
    background: var(--bg);
    cursor: pointer;
    color: var(--text);
    transition: background .15s, border-color .15s;
  }
  button:hover {
    background: #f0ede0;
    border-color: #bbb;
  }
  /* class:active wird von Svelte gesetzt, wenn deltaX ≈ preset */
  button.active {
    background: #c0392b;
    color: white;
    border-color: #c0392b;
  }

  /* ── Info-Kacheln ─────────────────────────────────────────────────────── */
  .info-bar {
    display: flex;
    border-top: 1px solid var(--border);
    background: #f3f4f6;
  }
  .info-tile {
    flex: 1;
    padding: 12px 16px;
    display: flex;
    flex-direction: column;
    gap: 2px;
    border-right: 1px solid var(--border);
  }
  .info-tile:last-child { border-right: none; }

  .info-label {
    font-family: var(--font-ui);
    font-size: .7rem;
    text-transform: uppercase;
    letter-spacing: .05em;
    color: var(--muted);
  }
  .info-value {
    font-family: var(--font-ui);
    font-size: 1.15rem;
    font-weight: 700;
    font-variant-numeric: tabular-nums;
    /* Farbe wird per inline-style reaktiv gesetzt */
  }
</style>
