# ZPMA

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
- **Rechte Maustaste**: Warsow Wall-Jump (in der Luft an Wand → Abprallen)
- **Q**: Wechsel zu Rocket (nur MIXED-Modus)
- **R**: Wechsel zu Rail (nur MIXED-Modus)
- **C**: Wechsel zu Lightning Gun (nur MIXED-Modus)
- **T**: Wechsel zu M4A1-S (nur MIXED-Modus)
- **E**: Wechsel zu Heart-Waffe (nur MIXED-Modus)
- **F**: Poster-Spot markieren (loggt Position + Normalen in Console)
- **Left Shift**: Schleichen (Geschwindigkeit 120 ups, keine Schrittgeräusche)
- Mausempfindlichkeit: sensitivity = 0.002
- **Maus Y-Achse**: Invertiert per Default (Toggle in Settings)

## Waffen
### Rockets
- Speed: 1000 ups, Feuerrate: 0.8s
- Splash-Radius: 120 units, Self-Knockback: 1170 (rk_selfKB=1.8)
- Kein Self-Damage (nur Bot-Rockets verletzen den Spieler)
- Damage: 100 base × Hitbox-Multiplikator × Splash-Falloff
- Rocket Jumps: runter schauen + springen + schießen → hoher Boost

### Railgun
- Hitscan, Cooldown: 1.5s
- **Instagib-Modus**: Instant Kill
- **Mixed-Modus**: 80 base × Hitbox-Multiplikator (Head 1.5x=120, Body 1.0x=80, Legs 0.75x=60)
- Synthesized Sound: Noise-Crack + High-Ping + Low-Thump
- Beam-Visual: Zylinder (r=2) + Glow-Hülle (r=6) + Impact-Flash, fade 0.5s
- Beam-Start: 30 units vor Kamera + 6 rechts + 4 runter (Waffenposition)
- Teal (Spieler), Accent/Rosa (Bot)

### Lightning Gun
- Continuous-fire, DPS-basiert, LG_TICK Intervall
- Beam: Zwei gekreuzte Planes (dicker Strahl)
- Farbe: Cyan (#4dc9f6)

### M4A1-S (CS-Style)
- Feuerrate: 0.09s (~666 RPM), Damage: 18 × Hitbox-Mul
- **Recoil**: Akkumulation-basierter Pitch-Kick (0.006 rad), 50% Recovery bei Loslassen
- **Spread**: Basis 0.004, Wachstum 0.001/Schuss, Max 0.04
- **Sound**: Q3 Machinegun (machgf1b.wav) durch Lowpass-Filter 3500Hz (Schalldämpfer-Effekt)
- **Ricochet**: 40% Chance auf Querschläger-Sound (ric1-3.wav) bei Wandtreffer
- **Bullet Holes**: Dunkle Kreise (r=1.5-2.5) an Wandtreffern, 12s Lebensdauer mit Fade
- **Tracers**: Dünne Linien zum Einschlagspunkt
- Spread-Berechnung mit Plain-Math (getCamForward() gibt {x,y,z}, NICHT THREE.Vector3)

### Heart-Waffe (♥)
- Taste E in MIXED-Modus, Feuerrate: 0.1s
- Speed: 1500 ups, Damage: 15 × Hitbox-Mul
- Herzen kleben an Wänden und Postern (stuckHearts[])
- Treffer auf Poster zählt loveCounter hoch
- Pink Crosshair, ♥ HUD-Icon

## Game-Modi (Settings-Menü via ESC)
- **ROCKET**: Nur Rockets
- **INSTAGIB**: Nur Railgun, ein Treffer = Tod
- **MIXED** (Default): Alle Waffen, Wechsel mit Q/R/C/T/E, Q3-like Balancing
- **Bot**: ARMED (schießt) / PASSIVE (Default: schießt nicht)
- **BOTS**: 1-4 (Multi-Bot System, dynamisch änderbar)
- **PK BHOP**: ON/OFF (Painkiller Bunny-Hop mit Timing-Fenster)
- **WALL JUMP**: ON/OFF (Warsow Wall-Jump)
- **TEXTURES**: ON/OFF (Boden- und Wandtexturen)
- **MOUSE Y**: INVERT (Default) / NORMAL

## Bewegungs-Mechaniken
### Painkiller Bhop
- Timing-Fenster: 80ms nach Landung
- Speed-Boost: 6% pro perfekten Hop (kumulativ)
- Max Chain: 20, Speed-Cap: 1200 ups

### Warsow Wall-Jump
- Rechte Maustaste in der Luft an einer Wand
- Push: 400 horizontal weg von Wand + 300 vertikal
- Cooldown: 0.4s
- `getNearestWallNormal()` für Wand-Erkennung

## Multi-Bot System
- `bots[]` Array, `createBotVisual()` Factory (eigene Materials pro Bot)
- **20% größer**: `group.scale.set(1.2, 1.2, 1.2)`
- Jeder Bot hat eigenes `hitFlash`, `vis.fillMat`, `vis.edgeMat`, `vis.hpMat`
- Anti-Overlap Spawn: weit voneinander und vom Spieler
- **Hit-Flash**: Roter Glow-Pulse (statt weiß), schnelles Pulsieren
- **No-Go Zone**: Bots meiden den tiefen RA-Pit (x:560-850, z:-200-400, y<-250)
- **Stuck-Detection**: XYZ-Bewegung prüfen, nur bei onGround, 2s Timeout → Respawn
- **isInsideSolid-Recovery**: Bot wird nach oben geschoben statt sofort respawnt (20 Ticks Toleranz)
- **Shaft-Bridging**: 2.5× playerRadius für Bots (40 units statt 16)

## Hitbox-System
- **Drei Kugeln** (Spieler + Bot, 20% größer für Bots):
  - Head: r=10, cy=feet+50, mul 1.5x
  - Body: r=16, cy=feet+28, mul 1.0x
  - Legs: r=12, cy=feet+10, mul 0.75x
- `getHitSpheres(x, y, z)` → Array von {cx, cy, cz, r, mul}
- `raySphereHit()` für Hitscan-Waffen (Rail, LG, M4)
- Rocket-Splash prüft nächste Kugel-Oberfläche

## Map — cpm3a
- **Quelle**: cpm3a.bsp aus Q3/CPMA (Quake 3 BSP v46)
- **Koordinaten-Konvertierung Q3→Three.js**: `x=q3_x, y=q3_z, z=-q3_y`
- **Dual-Geometry**: AABB-Boxen (757 Stück) für Kollision, BSP-Dreiecksmesh für Rendering
- **BSP-Mesh**: 8689 Vertices, 5115 Dreiecke → `assets/map.bin` (81KB, Int16+Uint16 binary)
- **Split-Geometry**: Boden (ny>0.5) und Wand/Decke als separate Meshes
- **polygonOffset**: factor=2, units=4 gegen z-fighting, degenerate Dreiecke gefiltert (area<0.5)
- **Skybox**: cpm3a-original (iceflow, 6 JPGs via CubeTextureLoader)
- **Fog**: Fog(0x1a3a5c, 2000, 5000) für Tiefenwirkung
- **Ground Plane**: y=-800 als visueller Boden bei Absturz

### Texturen
- **Boden**: MeshPhongMaterial + prozedurale Marmor-Textur (512×512 Canvas)
  - Turbulenz-modulierte Sinus-Adern, dezent (Stärke 0.12)
  - Vertex-Colors: pure white (1.0), Textur-Min: 220/255
  - Shininess: 60, Specular: 0x444444 → gebohnerter/polierter Look
- **Wände**: MeshLambertMaterial + subtile Steinkorn-Textur (256×256 Canvas)
  - 3 Noise-Layer, Range 215-255
- **Decken**: Dunkles Slate-Blau (0.12/0.16/0.25) in Vertex-Colors
- **Toggle**: `wallMatRef`, `floorMatRef`, `wallTexRef`, `marbleTexRef` für Settings-Toggle

### Teleporter (4 Stück, aus BSP-Entities)
- Trigger: AABB-Volumen mit ±20 Y-Toleranz
- Cooldown: 0.5s, Speed beibehalten + Richtung an Teleporter-Yaw
- **Visuals**: Portal-Textur (`assets/portal.jpg`) als Event-Horizon
  - Zwei gegenläufige Schichten mit UV-Scroll-Animation
  - Pulsierendes Teal-PointLight
  - Automatische Orientierung (schmale Achse = Durchgangsrichtung)

### Jump Pads (2 Stück, aus BSP-Entities)
- Trigger: AABB-Volumen mit ±10 Y-Toleranz, nur bei vy≤50
- Velocity: Q3-Trajektorie (time = sqrt(height/(0.5*gravity)))
- Cooldown: 0.3s
- Visuals: Orange Boxen + PointLight

### Spawn Points (8 Stück, aus BSP-Entities)
- Respawn bei y < -900 (zufälliger Spawn)

### Wand-Poster (Goldrahmen-System)
- `posterSpots[]`: Position, Rotation, Größe, Material-Key
- `loadPoster(name, url)` → TextureLoader + MeshBasicMaterial
- **9-Slice Goldrahmen**: `assets/frame.jpg` → Canvas-basiertes 9-Slice-Rendering pro Poster
  - 4 Ecken (fixe Proportionen) + 4 Kanten (gestreckt) + Flood-Fill Hintergrund entfernen
  - 30 Game-Units Rahmenbreite, 3px/Unit Auflösung
- **Poster-Spot-Tool**: F-Taste loggt Position + Normalen in Console für neue Poster-Platzierung
- Poster leicht von Wand versetzt (entlang Normal) gegen z-Fighting bei schrägen Wänden
- Aktuell 8 Poster (poster3-10.jpg)

## Audio-System
- **Web Audio API**: AudioContext + decodeAudioData + BufferSource + GainNode
- **Sounds** (aus Q3 pak0.pk3 extrahiert):
  - `rocketFire` (rocklf1a.wav), `rocketExplode` (rocklx1a.wav)
  - `jump` (jump1.wav), `land` (land1.wav)
  - `boot1-4` (boot1-4.wav) — Footsteps, distanzbasiert alle ~110 units
  - `telein` (telein.wav), `jumppad` (jumppad.wav)
  - `hit` (hit.wav) — Treffer-Feedback
  - `machgun` (machgf1b.wav) — M4 Feuer-Sound (durch Lowpass 3500Hz)
  - `ric1-3` (ric1-3.wav) — Ricochet/Querschläger
- **Rail-Sound**: Synthesized (Noise-Burst + Sine-Ping + Bass-Thump)
- AudioContext resume bei erstem Klick

## Kollisions-System
- `colBoxes[]`: 757 AABB-Boxen aus BSP-World-Model (*0)
- `getFloorHeight(x, z, entityY)`: Höchste begehbare Fläche, Shaft-Bridging (8 Samples)
  - Bots: 2.5× Radius (40 units) für breitere Überbrückung
- `getFloorAt(x, feetY, z)`: Direkter Boden-Check (Boxen + Rampen + Floor-Dreiecke)
- `isInsideSolid()`: Für Rocket-Kollision + Bot-Recovery
- `collideWalls()` / `collideBotWalls()`: Seitenkollision mit Kugel-vs-AABB Push-out
- Stair-Smoothing: `player._stairAccum` für sanftes Treppensteigen

## Beleuchtung & Schatten
- AmbientLight (0xaaa099, 0.7)
- DirectionalLight "Sonne" (0xfff0d0, 1.5) von (500, 800, 200) — **mit Schatten**
  - PCFSoftShadowMap, 2048×2048
  - Shadow Camera: ±1500, near 1, far 3000, bias -0.002
- DirectionalLight "Fill" (0x6688aa, 0.3) von (-400, 300, -500)
- HemisphereLight (0xaabbdd / 0x665544, 0.8)

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
- `zpma.html` — Das komplette Spiel (single-file)
- `assets/map.bin` — BSP-Dreiecksmesh binary (81KB)
- `assets/frame.jpg` — Goldrahmen-Bild für 9-Slice Poster-Rahmen
- `assets/portal.jpg` — Teleporter Portal-Textur
- `assets/sounds/` — 15 WAV-Dateien (12 aus Q3 + machgun + ric1-3)
- `assets/skybox/` — 6 iceflow JPGs aus cpm3a
- `assets/poster3-10.jpg` — 8 Wand-Poster Bilder
- `Q3 with CPMA/` — Original-Spieledateien (pak0.pk3, map_cpm3a.pk3, CPMA-Paks)
- `prompt-cpma-3d-prototyp.md` — Ursprünglicher Design-Prompt
- `CLAUDE.md` — Diese Datei (Projekt-Konventionen)

## Bekannte Einschränkungen
- **Kollisionslücken**: AABB-Boxen decken nicht alle BSP-Geometrie ab → Spieler kann an manchen Stellen ins Nichts fallen
- **Teleporter-Trigger**: Volumen manuell angepasst, da Teleporter-Pads in Brush-Models (*1-*4) liegen, nicht im World-Model
- **RA-Pit**: Tiefer Bereich der Map als Bot-No-Go-Zone, Bots vermeiden die Area komplett
- **getCamForward()**: Gibt plain {x,y,z} zurück, NICHT THREE.Vector3 — bei neuen Waffen Plain-Math verwenden

## Wichtige Code-Patterns
- `getCamForward()` → `{x, y, z}` (plain object, keine THREE.Vector3-Methoden!)
- `settings.invertY` → Pitch-Richtung: `* (settings.invertY ? 1 : -1)`
- Recoil (M4): `player.pitch += kick` (invertiert-kompatibel)
- Continuous-fire: `firePressed` nicht zurücksetzen für LG, M4, Heart
- Textur-Toggle: `wallMatRef.map = null/wallTexRef`, `floorMatRef.map = null/marbleTexRef`
