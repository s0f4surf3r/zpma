# Space CPMA

3D CPMA-Defrag Bewegungs-Prototyp im Browser (Three.js).

## Arbeitsweise
- Sprache: Deutsch bevorzugt
- Alle rahmengebenden Entscheidungen (Architektur, Konventionen, Physik-Regeln, Design-Grundsätze) werden in dieser CLAUDE.md dokumentiert
- Diese Datei ist die Single Source of Truth für das Projekt
- Neue Beschlüsse aus Sessions werden hier ergänzt

## Projekt-Grundsätze
- **Single-File**: Alles in einer HTML-Datei (HTML + JS + CSS inline)
- **Three.js per CDN** (r128), kein Build-System, kein npm, kein Framework
- **Fester Timestep**: Physik läuft mit dt=0.008 (125Hz intern), entkoppelt vom Rendering
- **FOV**: 90 (Quake-Standard)

## Physik-Konstanten
```
sv_gravity = 800
sv_maxspeed = 320
sv_friction = 6
sv_stopspeed = 100
sv_accelerate = 10
sv_airaccelerate = 1
cpm_aircontrol = 150        // CPMA PM_Aircontrol (dot²-moduliert, k=32*ac*dot²*dt)
cpm_strafeaccelerate = 70   // Hohe Strafe-Beschleunigung in Luft (reines A/D)
cpm_airstopaccelerate = 2.5
cpm_wishspeed = 30          // Wishspeed-Cap bei Air-Strafe
sv_jumpvelocity = 280
stepUp = 22                 // Auto-Step-Up Höhe (Quake: 18)
```

## Air Control (CPMA-Style)
- **PM_Aircontrol**: Echte CPMA-Formel, nur bei Forward-Input (W)
- `k = 32 * cpm_aircontrol * dot² * dt` → dot²-moduliert (stark bei Alignment, null bei 90°)
- Speed bleibt erhalten (normalize + rescale), nur Richtung ändert sich
- cpm_aircontrol=150 (CPMA Promode Default)
- Danach CPMA-Acceleration je nach Input:
  - Reiner Strafe (A/D ohne W): `cpm_strafeaccelerate=70`, wishspeed gedeckelt auf 30
  - Gegen Bewegungsrichtung: `cpm_airstopaccelerate=2.5`
  - Sonst: `sv_airaccelerate=1` (VQ3-Standard)

## Steuerung
- WASD + Maus (Pointer Lock) + Leertaste
- **Linke Maustaste**: Aktive Waffe feuern
- **Q**: Wechsel zu Rocket (nur MIXED-Modus)
- **R**: Wechsel zu Rail (nur MIXED-Modus)
- **Left Shift**: Schleichen (Geschwindigkeit 120 ups, keine Schrittgeräusche)
- Mausempfindlichkeit: sensitivity = 0.002
- **Maus Y-Achse invertiert** (Maus hoch = Blick runter)

## Waffen
### Rockets
- Speed: 1000 ups, Feuerrate: 0.8s
- Splash-Radius: 120 units, Self-Knockback: 650
- Damage: 100 base * Hitbox-Multiplikator * Splash-Falloff
- Rocket Jumps funktionieren: runter schauen + springen + schießen
- Rockets kollidieren mit allen Oberflächen (isInsideSolid)

### Railgun
- Hitscan, Cooldown: 1.5s
- **Instagib-Modus**: Instant Kill
- **Mixed-Modus**: 80 base * Hitbox-Multiplikator (Head 1.5x=120, Body 1.0x=80, Legs 0.75x=60)
- Synthesized Sound: Noise-Crack + High-Ping + Low-Thump
- Beam-Visual: Linie + Impact-Flash, fade 0.5s
- Teal (Spieler), Accent/Rosa (Bot)

## Game-Modi (Settings-Menü via ESC)
- **ROCKET**: Nur Rockets
- **INSTAGIB**: Nur Railgun, ein Treffer = Tod
- **MIXED**: Beide Waffen, Wechsel mit Q/R, Q3-like Balancing
- **Bot**: ARMED (schießt) / PASSIVE (schießt nicht)
- **BOTS**: 1-4 (Multi-Bot System, dynamisch änderbar)

## Multi-Bot System
- `bots[]` Array statt einzelnem `bot` Objekt
- `createBotVisual()` Factory für Bot-Meshes (eigene Materials pro Bot)
- `createBot()` erstellt State + Visual
- `initBots()` spawnt alle Bots neu (bei Settings-Änderung)
- Jeder Bot hat eigenes `hitFlash`, `vis.fillMat`, `vis.edgeMat`, `vis.hpMat`
- Anti-Overlap Spawn: Bots werden weit voneinander und vom Spieler gespawnt

## Hitbox-System
- **Drei Kugeln** (Spieler + Bot): Head (r=8, mul 1.5x), Body (r=13, mul 1.0x), Legs (r=10, mul 0.75x)
- `getHitSpheres(x, y, z)` → Array von {cx, cy, cz, r, mul}
- `raySphereHit()` für Railgun-Hitscan
- Rocket-Splash prüft nächste Kugel-Oberfläche

## Map — cpm3a
- **Quelle**: cpm3a.bsp aus Q3/CPMA (Quake 3 BSP v46)
- **Koordinaten-Konvertierung Q3→Three.js**: `x=q3_x, y=q3_z, z=-q3_y`
- **Dual-Geometry**: AABB-Boxen (757 Stück) für Kollision, BSP-Dreiecksmesh für Rendering
- **BSP-Mesh**: 8689 Vertices, 5115 Dreiecke → `assets/map.bin` (81KB, Int16+Uint16 binary)
- **Vertex-Colors**: Höhe + Normal-basiert (Sandstein-Gradient, Böden hell, Wände dunkler, Decken dunkel)
- **polygonOffset**: factor=2, units=4 gegen z-fighting, degenerate Dreiecke gefiltert (area<0.5)
- **Skybox**: cpm3a-original (iceflow, 6 JPGs via CubeTextureLoader)
- **Fog**: Fog(0x1a3a5c, 2000, 5000) für Tiefenwirkung
- **Ground Plane**: y=-800 als visueller Boden bei Absturz

### Teleporter (4 Stück, aus BSP-Entities)
- Trigger: AABB-Volumen mit ±20 Y-Toleranz
- Cooldown: 0.5s, Speed beibehalten + Richtung an Teleporter-Yaw
- Visuals: Teal Boxen + PointLight

### Jump Pads (2 Stück, aus BSP-Entities)
- Trigger: AABB-Volumen mit ±10 Y-Toleranz, nur bei vy≤50
- Velocity: Q3-Trajektorie (time = sqrt(height/(0.5*gravity)))
- Cooldown: 0.3s
- Visuals: Orange Boxen + PointLight

### Spawn Points (8 Stück, aus BSP-Entities)
- Respawn bei y < -900 (zufälliger Spawn)

## Audio-System
- **Web Audio API**: AudioContext + decodeAudioData + BufferSource + GainNode
- **Sounds** (aus Q3 pak0.pk3 extrahiert):
  - `rocketFire` (rocklf1a.wav), `rocketExplode` (rocklx1a.wav)
  - `jump` (jump1.wav), `land` (land1.wav)
  - `boot1-4` (boot1-4.wav) — Footsteps, distanzbasiert alle ~110 units
  - `telein` (telein.wav), `jumppad` (jumppad.wav)
  - `hit` (hit.wav) — Treffer-Feedback
- **Rail-Sound**: Synthesized (Noise-Burst + Sine-Ping + Bass-Thump)
- AudioContext resume bei erstem Klick

## Kollisions-System
- `colBoxes[]`: 757 AABB-Boxen aus BSP-World-Model (*0)
- `getFloorHeight()`: Höchste begehbare Fläche unter Spieler (best=-9999 Start)
- `isInsideSolid()`: Für Rocket-Kollision mit Welt
- `collideWalls()`: Seitenkollision mit Kugel-vs-AABB Push-out (Step-Up Toleranz)

## Beleuchtung
- AmbientLight (0x887766, 0.5)
- DirectionalLight "Sonne" (0xfff0d0, 1.5) von (500, 800, 200)
- DirectionalLight "Fill" (0x6688aa, 0.3) von (-400, 300, -500)
- HemisphereLight (0x88aacc / 0x554433, 0.6)

## Unit-Scale
- 1 Unit = 1 Quake Unit
- Spielerhöhe: 56, Augenhöhe: 52
- Spieler-Kollision: Radius 16

## Farbpalette (aus Zoe-Webseite)
- `#0a1628` (bg) → Body, Overlay, Ground Plane
- `#1a3a5c` (primary) → Fog, Active Buttons
- `#3a6080` (secondary) → Labels, Debug-Text
- `#e86a7a` (accent) → Bot, Bot-Beam, Damage-Flash, Start-Button
- `#2abfbf` (teal) → HUD, Spieler-Beam, Teleporter, Crosshair Rail
- `#f4a842` (warm) → Rockets, Explosionen, Jump Pads, Gib-Edges
- `#f0ece6` (light) → Menu-Titel, Crosshair default

## Dateien
- `space-cpma.html` — Das komplette Spiel (single-file)
- `assets/map.bin` — BSP-Dreiecksmesh binary (81KB)
- `assets/sounds/` — 12 WAV-Dateien aus Q3
- `assets/skybox/` — 6 iceflow JPGs aus cpm3a
- `assets/poster.jpg` — Wand-Poster Bild 1
- `assets/poster2.jpg` — Wand-Poster Bild 2
- `Q3 with CPMA/` — Original-Spieledateien (pak0.pk3, map_cpm3a.pk3, CPMA-Paks)
- `prompt-cpma-3d-prototyp.md` — Ursprünglicher Design-Prompt
- `CLAUDE.md` — Diese Datei (Projekt-Konventionen)

## Bekannte Einschränkungen
- **Kollisionslücken**: AABB-Boxen decken nicht alle BSP-Geometrie ab → Spieler kann an manchen Stellen ins Nichts fallen
- **Teleporter-Trigger**: Volumen manuell angepasst, da Teleporter-Pads in Brush-Models (*1-*4) liegen, nicht im World-Model
- **Keine Texturen**: Vertex-Colors statt echter Map-Texturen
