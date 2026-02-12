# Prompt für Claude Code: 3D CPMA-Defrag Prototyp

## Was gebaut werden soll
Ein 3D-Bewegungs-Prototyp im Browser. Keine Waffen, keine Gegner, keine Objectives – NUR Movement. Der Spieler bewegt sich durch eine 3D-Röhre/Map mit exakter CPMA-Physik (Quake 3 CPMA/Promode). Das Ziel: Strafe-Jumping und Air Control müssen sich richtig anfühlen. Wer CPMA Defrag kennt, soll sofort sagen "ja, das ist es".

## Tech-Stack
- **Three.js** (r128 oder aktuell) für 3D-Rendering
- **Pointer Lock API** für Maussteuerung
- **Einzelne HTML-Datei** – alles in einem File (HTML + JS + CSS)
- Three.js per CDN einbinden: `https://cdnjs.cloudflare.com/ajax/libs/three.js/r128/three.min.js`
- Kein Build-System, kein npm, kein Framework

## Steuerung
- **Maus**: Blickrichtung (Yaw + Pitch). Pointer Lock beim Klick auf Canvas aktivieren.
- **W**: Vorwärts bewegen
- **S**: Rückwärts bewegen
- **A**: Strafe links
- **D**: Strafe rechts
- **Leertaste**: Springen
- **Mausempfindlichkeit**: Variable `sensitivity = 0.002` (anpassbar)

## KERN: Die CPMA-Physik

Das ist das Herzstück. Die Physik muss dem Quake 3 CPMA-Movement-Code folgen. Hier ist die Logik:

### Grundprinzip
Der Spieler hat einen Velocity-Vektor (vx, vy, vz). Y ist oben. Jeder Frame berechnet:
1. Input → wish direction + wish speed
2. Beschleunigung (unterschiedlich am Boden vs. in der Luft)
3. Reibung (nur am Boden)
4. Gravität
5. Kollision

### Konstanten (Quake-nah, zum Tunen)
```
sv_gravity = 800           // Schwerkraft (units/s²)
sv_maxspeed = 320          // Maximale Ground-Speed
sv_friction = 6            // Bodenreibung
sv_stopspeed = 100         // Ab dieser Speed greift Friction voll
sv_accelerate = 10         // Bodenbeschleunigung
sv_airaccelerate = 1       // Luftbeschleunigung (VQ3-Wert)
sv_aircontrol = 150        // CPM Air Control Stärke
sv_jumpvelocity = 270      // Sprungkraft (Y-Velocity)
```
**WICHTIG:** Die Simulation läuft mit festem Timestep: `dt = 0.008` (125fps intern, wie Quake). Die Physik wird pro Render-Frame ggf. mehrfach berechnet um Frame-unabhängig zu sein.

### Bodenbewegung (Spieler steht auf dem Boden)
```
Pseudocode pro Physik-Tick:

1. wishdir = normalisierter Richtungsvektor aus Input + Blickrichtung (nur XZ-Ebene)
   - W: vorwärts (Blickrichtung projiziert auf XZ)
   - S: rückwärts
   - A: 90° links von Blickrichtung
   - D: 90° rechts von Blickrichtung
   - Kombinationen addieren und normalisieren
   
2. wishspeed = sv_maxspeed (wenn Input vorhanden, sonst 0)

3. Friction anwenden (VOR Beschleunigung):
   speed = length(velocity.xz)
   if speed > 0:
     drop = speed * sv_friction * dt
     if speed < sv_stopspeed: drop = sv_stopspeed * sv_friction * dt
     newspeed = max(0, speed - drop)
     velocity.xz *= newspeed / speed

4. Accelerate:
   currentspeed = dot(velocity.xz, wishdir)
   addspeed = wishspeed - currentspeed
   if addspeed > 0:
     accelspeed = min(addspeed, sv_accelerate * wishspeed * dt)
     velocity.x += accelspeed * wishdir.x
     velocity.z += accelspeed * wishdir.z
```

### Luftbewegung (Spieler ist in der Luft)
HIER passiert die Magie. Das ist der Code der Strafe-Jumping ermöglicht:

```
Pseudocode pro Physik-Tick:

1. wishdir und wishspeed wie am Boden berechnen

2. KEINE Friction

3. Air Accelerate (VQ3-Stil):
   currentspeed = dot(velocity.xz, wishdir)
   addspeed = wishspeed - currentspeed
   if addspeed > 0:
     accelspeed = min(addspeed, sv_airaccelerate * wishspeed * dt)
     velocity.x += accelspeed * wishdir.x
     velocity.z += accelspeed * wishdir.z

4. CPM Air Control (DAS macht CPMA besonders):
   Wenn der Spieler W NICHT drückt, aber A oder D:
     speed = length(velocity.xz)
     if speed > 0.1:
       // wishdir ist die Strafe-Richtung
       // dot = wie sehr velocity schon in wishdir zeigt
       dot = dot(normalize(velocity.xz), wishdir)
       // k bestimmt wie stark die Kontrolle ist
       k = sv_aircontrol * dot * dot * dt
       // Velocity in Richtung wishdir biegen
       velocity.x = velocity.x * speed + wishdir.x * k
       velocity.z = velocity.z * speed + wishdir.z * k
       // Auf alte Speed normalisieren (KEIN Speed-Verlust!)
       normalize velocity.xz und multipliziere mit speed

5. Gravität:
   velocity.y -= sv_gravity * dt
```

### Springen
```
Wenn Leertaste gedrückt UND Spieler am Boden:
  velocity.y = sv_jumpvelocity
  → Spieler ist jetzt in der Luft
  
Bunny-Hopping: Wenn Leertaste gehalten wird, springt der Spieler sofort 
wenn er den Boden berührt. KEINE Verzögerung. Das ist essentiell.
```

### Warum Strafe-Jumping funktioniert (für dein Verständnis beim Implementieren)
Der Trick liegt in `sv_airaccelerate = 1` (sehr niedrig) kombiniert mit der Tatsache dass `currentspeed = dot(velocity, wishdir)`. Wenn der Spieler sich vorwärts bewegt aber zur Seite straft, ist der Dot-Produkt klein (die Richtungen sind fast 90° auseinander). Also ist `addspeed` groß. Und wenn der Spieler gleichzeitig die Maus dreht, bleibt der Dot-Produkt klein weil sich die wishdir mitdreht. So addiert sich jedes Frame ein kleiner Velocity-Boost → Speed baut sich auf.

## Level / Map

### Stufe 1: Flacher Boden
Erstmal nur ein großer flacher Boden (Plane) mit Wänden drumrum. 200x200 Units. Damit kann man Strafe-Jumping testen ohne an irgendetwas zu crashen.

### Stufe 2: Einfache Rampen und Plattformen
Wenn die Physik stimmt, ein paar Elemente hinzufügen:
- Eine lange Rampe (zum Hochspringen)
- Plattformen in verschiedenen Höhen
- Eine breite Röhre / Tunnel

Aber: ERST wenn die Physik auf dem flachen Boden perfekt sitzt.

## Kollision
- Spieler ist eine Kugel (Radius 16 Units) oder ein Zylinder
- Boden-Check: Einfacher Raycast nach unten, wenn Distanz < Spielerhöhe → am Boden
- Wand-Check: Sphere-vs-Mesh Kollision
- Für den Prototyp reicht simple AABB-Kollision mit dem Boden und den Wänden
- Kein Clipping durch Wände!

## Kamera
- First-Person: Kamera sitzt auf Augenhöhe des Spielers (Y + 28 Units über Boden)
- Mausbewegung → Rotation (Yaw = Links/Rechts, Pitch = Hoch/Runter)
- Pitch begrenzen auf -89° bis +89°
- Kein Head-Bob, kein Tilt, keine Effekte – erstmal clean

## HUD (minimal)
- **Speed**: Aktuelle horizontale Geschwindigkeit (XZ-Ebene), groß und lesbar unten mittig
- **Position**: X, Y, Z Koordinaten klein oben links (Debug-Info)
- Farbe: Grün (#0f0) auf transparent, Monospace-Font
- Speed-Farbe wechselt: Grün (< 320) → Gelb (320-600) → Rot (> 600)

## Visuals (minimalistisch aber lesbar)
- Boden: Grid-Textur (generiert, nicht geladen) – muss man sehen dass man sich bewegt
- Wände: Einfache Farbe, leicht unterschiedlich zum Boden
- Himmel: Dunkelblau oder Schwarz
- Beleuchtung: Ein Directional Light + Ambient. Simpel.
- **Kein Fog, keine Schatten, keine Post-Processing** – Performance first

## Dateistruktur
EINE einzelne HTML-Datei. Alles inline. Aufbau:
```
<!DOCTYPE html>
<html>
<head>
  <style> ... </style>
</head>
<body>
  <canvas id="game"></canvas>
  <div id="hud"> ... </div>
  <div id="crosshair"> ... </div>
  <div id="overlay">Klick zum Starten</div>
  <script src="three.js CDN"></script>
  <script>
    // === CONFIG ===
    // === PHYSICS ===
    // === INPUT ===
    // === LEVEL ===
    // === RENDERER ===
    // === GAME LOOP ===
  </script>
</body>
</html>
```

## Kritische Details

### Frame-Timing
Die Physik MUSS mit festem Timestep laufen. Nicht an requestAnimationFrame koppeln:
```
let accumulator = 0;
const TICK = 0.008; // 125Hz

function gameLoop(timestamp) {
  const delta = Math.min(0.05, (timestamp - lastTime) / 1000);
  lastTime = timestamp;
  accumulator += delta;
  while (accumulator >= TICK) {
    physicsTick(TICK);
    accumulator -= TICK;
  }
  render();
  requestAnimationFrame(gameLoop);
}
```

### Input-Handling
- KeyDown/KeyUp Events für WASD und Space → Boolean-Flags
- MouseMove im Pointer Lock → direkt auf Kamera-Rotation anwenden
- Pointer Lock bei Klick auf Canvas aktivieren
- ESC verlässt Pointer Lock (Browser-Standard) → Overlay "Klick zum Fortsetzen" zeigen

### Unit-Scale
- 1 Unit ≈ 1 Quake Unit
- Spielerhöhe: 56 Units (Augenhöhe 52)
- Sprunghöhe: ca. 44 Units
- Standard-Laufspeed: 320 Units/Sekunde
- Die Boden-Plane: 2000 x 2000 Units (groß genug zum Üben)

## Was NICHT gebaut werden soll
- Keine Waffen, kein Schießen
- Kein Multiplayer
- Keine Texturen laden (alles generiert)
- Keine komplexe Map
- Kein Sound (kommt später)
- Kein Menü (kommt später)
- Keine Settings-UI (kommt später)

## Testing-Prioritäten
1. Maus-Steuerung flüssig? Kein Lag, kein Drift?
2. WASD am Boden: Fühlt sich responsiv an? Friction stoppt den Spieler wenn man loslässt?
3. Springen: Sofortiger Absprung? Bunny-Hop funktioniert (Leertaste halten = sofort wieder springen bei Landung)?
4. **Strafe-Jump Test**: W+A halten, Maus langsam nach links drehen → Speed MUSS über 320 steigen und weiter wachsen. Das ist DER Test.
5. **Air Control Test**: In der Luft nur A oder D (ohne W) + Maus drehen → enge Kurven ohne Speed-Verlust
6. Speed-Anzeige stimmt mit dem Gefühl überein?
7. Kein Clipping durch den Boden oder Wände?

## Zusammenfassung
Bau einen CPMA-Defrag-Prototyp. Eine große Fläche, First-Person, Maus+Keyboard, und Physik die sich anfühlt wie Quake 3 CPMA. Wenn ich W+A halte und die Maus nach links ziehe, will ich Speed aufbauen. Wenn ich in der Luft A oder D halte und die Maus drehe, will ich enge Kurven fliegen. Das ist alles was zählt.
