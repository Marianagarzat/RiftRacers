# RIFT RACERS — Phase 1 (Playable Foundation)

Version `0.1.1-phase1`. This build has the Nexus lobby, the original RIFT VX-1 car, a full **Cyberpulse City** circuit, arcade driving on PC, mobile and gamepad, server-checked checkpoints and laps, a countdown with start lights, a race HUD and speedometer, and finish and results screens.

> **Read this first: what was and wasn't tested**
> I built this without access to Roblox Studio, so I couldn't playtest it there. Here's what I did check:
> - All 24 Luau files (≈5,300 lines) pass **`--!strict`** type-checking (`luau-lsp analyze`) against the current Roblox API definitions, with no errors and no deprecated-API warnings.
> - The track spline and formatting modules ran in the Luau CLI. The lap is 4,346 studs long. The tightest corner has a 176-stud radius. The steepest grade is 11.6 %. The track doesn't cross itself (the closest separate sections are 284 studs apart). Sample spacing is a constant 10 studs.
> - The place file was built with Rojo 7.6.1.
>
> **Everything below that needs Studio is UNVERIFIED** until you run the test checklist in §4. Vehicle feel in particular will probably need tuning (see §5).

---

## 1. Install (2 options)

### Option A — open the place file (simplest)
1. Open **`RiftRacers_Phase1_v3.rbxl`** in Roblox Studio.
2. Press **Play**. The world builds itself when the server starts, which takes about 1–3 s. The Output window prints `World built in …s · N parts` followed by `Server ready`.
3. Publish when you're ready (*File → Publish to Roblox*).

The place file already sets: `Lighting.Technology = Future`, `LightingStyle = Realistic`, `PrioritizeLightingQuality = true`, `StreamingEnabled = false` (Phase 1), and `CharacterUseJumpPower = false`.

### Option B — Rojo workflow (recommended for ongoing development)
```
rojo serve default.project.json      # then connect from the Rojo Studio plugin
rojo build default.project.json -o RiftRacers.rbxl
```

---

## 2. Explorer hierarchy (exact locations)

```
ReplicatedStorage
└── Shared                         (Folder)
    ├── Config                     ModuleScript  — every tunable: race rules, handling, camera, graphics presets, audio IDs, UI palette
    ├── MapDefinitions             ModuleScript  — Cyberpulse City data (control points, sections, palette, lighting) + lobby lighting
    ├── Net                        ModuleScript  — the one list of remotes (server creates them at boot)
    ├── Types                      ModuleScript  — payload types crossing client/server
    └── Util
        ├── Spline                 ModuleScript  — closed centripetal Catmull-Rom, resampled by arc length
        ├── Signal                 ModuleScript
        └── Format                 ModuleScript
    (RiftRemotes Folder is created here at runtime)

ServerScriptService
└── Server                         Script  (the ONLY server entry point)
    ├── Builders
    │   ├── Style                  ModuleScript  — part/neon/light/sign/ring helpers
    │   ├── CarFactory             ModuleScript  — RIFT VX-1 car, modular named slots
    │   ├── LobbyBuilder           ModuleScript  — THE NEXUS
    │   └── TrackBuilder           ModuleScript  — road, barriers, tunnels, skybridge, gates, grid, city
    └── Systems
        ├── Vehicles               ModuleScript  — collision groups, spawn/seat/release cars
        ├── World                  ModuleScript  — builds lobby + tracks at boot
        └── Race                   ModuleScript  — queue, race state machine, checkpoint/lap validation, standings, anti-cheat, results

StarterPlayer
└── StarterPlayerScripts
    └── Client                     LocalScript  (the ONLY client entry point)
        └── Controllers
            ├── Settings           — graphics preset / reduced motion / camera mode
            ├── RaceClient         — client mirror of race state + signals
            ├── Input              — keyboard, gamepad (analog), touch controls
            ├── Vehicle            — raycast-suspension arcade physics (your car)
            ├── VehicleVisuals     — wheels, suspension, steering, brake lights, smoke, skid trails, boost flames (ALL cars)
            ├── RaceCamera         — intro flyover, chase/close/hood, speed FOV, drift lean, shake, results orbit
            ├── Environment        — lighting presets, post FX, rain, quality scaling, ambient animation, start lights, gate highlight
            └── HUD                — lobby bar, queue, race HUD, speedometer, results, settings

Workspace (generated at runtime)
├── NexusLobby      — plaza, showroom + turntable car, holo logo, Race Portal, pavilions, winners board, parked cars, skyline, spawn
├── Tracks/CyberpulseCity — Road, Barriers, Decor, City, Checkpoints (Gate0 = start/finish … Gate11)
└── ActiveRace/Cars — live race cars (Car_<UserId>)
```

**Design rules followed:** there is exactly one Script and one LocalScript, and every system boots in two phases (`init` then `start`). The server is authoritative for queueing, race start, gate order, lap timing, finish order, recovery and the speed check. Clients only render and send *requests*, and every request is rate-limited and checked on the server.

---

## 3. How a race works (Phase 1)

1. Walk onto the **Race Portal** pad (press **E** or the gamepad **X**) or click **PLAY**. This puts you in the queue.
2. The queue starts **10 s** after the first player joins (or **3 s** once 8 players are in). `MinPlayers = 1`, so you can test solo.
3. Cars spawn on the grid. A **4 s camera flyover** plays, then a **3-2-1-GO** countdown while the five start lights fill. Cars are anchored until GO, so a false start is impossible.
4. At GO the server unanchors each car and hands physics ownership to its driver.
5. The race is **3 laps** through **12 gates**, which must be crossed in order. A crossing is detected as a segment-plane intersection, so it can't be missed at any speed.
6. The race ends when everyone has finished, or **25 s** after the winner finishes, or after 7 minutes. The results screen shows position, total time and best lap (DNF for anyone who didn't finish). You go back to the Nexus after **14 s**, or straight away with **BACK TO NEXUS**.
7. The winner is added to the **Recent Winners** board in the lobby.

**Anti-cheat (lenient to cope with lag):** gates only count in order and in the forward direction. Laps under 14 s are rejected. Average speed is checked over 0.5 s windows and must stay under 380 studs/s; two strikes resets you to your last checkpoint. Falling below Y = −60 also resets you. Quitting mid-race always counts as a DNF.

### Controls
| | PC | Gamepad | Mobile |
|---|---|---|---|
| Accelerate | W / ↑ | RT (analog) | GAS |
| Brake / reverse | S / ↓ | LT (analog) | BRK |
| Steer | A D / ← → | Left stick | drag on the left side |
| Drift (handbrake) | Space | X / B | DRIFT |
| Boost | Shift | A | BOOST |
| Reset to checkpoint | R | Y | RESET |
| Camera (Chase → Close → Hood) | C | D-pad up | CAM |

Boost charges slowly on its own and quickly while you drift. Every car has identical handling (`Config.Vehicle`), so nothing is pay-to-win.

---

## 4. Studio test checklist (run before calling Phase 1 done)

Use **Test → Start** with **2–4 players** (Clients and Servers) as well as solo Play. Use the **Device Emulator** for phone and tablet.

**Core gameplay**
- [ ] The player spawns in the Nexus. The showroom car turns, the holo logo spins and bobs, and the floor rings pulse.
- [ ] Portal pad prompt / PLAY → queue card shows count + countdown; LEAVE QUEUE works.
- [ ] The flyover plays, then 3-2-1-GO with the start lights filling red and then turning green. The car can't move before GO.
- [ ] The car accelerates, steers, brakes, reverses and drifts (you see smoke and skid trails), and boost shows flames, raises the FOV and fills the speedometer gold.
- [ ] The wheels spin, steer and move with the suspension. Brake lights come on when braking.
- [ ] Each gate crossing moves the pink highlight and the ▼ NEXT marker to the next gate.
- [ ] Laps go up only after all 12 gates. The FINAL LAP banner appears. Driving backwards shows WRONG WAY.
- [ ] Finishing is registered **once**. The FINISH banner appears and the results screen shows correct times. You go back to the lobby after 14 s.
- [ ] R or flipping the car over resets you to the last checkpoint.

**Multiplayer (2–4 clients)**
- [ ] Everyone gets the same countdown.
- [ ] Standings on the left update about 4 times per second.
- [ ] Opponents' wheels, brake lights and smoke animate.
- [ ] If a client quits mid-race, the race continues, that player shows as DNF and the race still ends.

**Exploit spot-checks** (with a *gated* test tool, never shipped):
- [ ] Teleporting a car forward resets it and doesn't award the skipped gates.
- [ ] Skipping a gate stops lap progress.

**Visual & device**
- [ ] The Ultra, Balanced and Performance presets visibly change the number of lights, the rain density, the steam vents and the headlight beams.
- [ ] Mobile: the touch controls show only during races, the buttons are reachable, and the HUD doesn't overlap them.
- [ ] Check the frame rate with the MicroProfiler (Ctrl+F6) on Ultra (PC) and on Performance (mobile emulator).

---

## 5. Tuning guide (most likely edits)

All values are in `ReplicatedStorage.Shared.Config`:
- **Car bounces or floats** → raise `RestCompression` (makes the springs softer) or `SpringDamping`.
- **Understeers** → raise `MaxYawRateHigh`. **Too twitchy** → lower `MaxYawRateLow` or `SteerResponse`.
- **Drifts too easily or not enough** → `DriftGrip` (lower = more slide) and `DriftYawMultiplier`.
- **Too slow or too fast** → `TopSpeed`, `Acceleration`, `BoostTopSpeed`.
- **Race length** → `Race.Laps`. **Allow solo play or not** → `Race.MinPlayers`.
- **Cars bump each other** → `Vehicles.init()` collision groups (set `RiftCars`↔`RiftCars` to false for ghost races).

---

## 6. Assets: what's a prototype and what's final

Everything is currently **procedural Roblox geometry** (Parts, Neon, Glass, SurfaceGuis, ParticleEmitters with built-in `rbxasset://` textures). It's deliberately art-directed, but it is **not** final production art, and **no asset IDs have been invented**.

### Hero car "RIFT VX-1" — replacement spec
The gameplay code only uses the invisible `Chassis` box (7 × 1.6 × 14 studs, front = −Z). Each visual slot can be swapped for imported MeshParts **without touching code**, as long as the names and positions below are kept:

| Slot (Model under car) | Parts | Notes |
|---|---|---|
| `Body` | LowerBody, Waist, Fender×4, Hood, RearDeck | Paintable, so use one MeshPart + SurfaceAppearance (Color tint enabled) |
| `Windows` | Cabin, Windshield, RearWindow | Glass material; tint becomes a Phase 3 option |
| `Aero` | FrontSplitter, Grille, RearDiffuser(+fins), SideSkirt×2 | Phase 3 body-kit variants swap here |
| `Spoiler` | Wing, WingEdge, Stand×2, EndPlate×2 | Variants: none / street / race wing / futuristic |
| `Lights` | Headlight×2 (with SpotLight "Beam"), DRL×2, Taillights (with PointLight "BrakeGlow"), SideLine×2 | VehicleVisuals finds them **by name** |
| `Underglow` | Strip (+PointLight "UnderLight") | |
| `Exhaust` | Tip×2 | Particles live on `Chassis.ExhaustL/R` attachments |
| `Wheels` | FL/FR/RL/RR (tire + Rim + Spokes + RimAccent) | Pivot at the wheel centre, axle along local X, radius 1.6. Driven by `Chassis.<name>Motor` (Motor6D) |
| `DriverSeat` | Seat | |

Mesh guidance: keep it to **≤ 8k triangles** for the body, 1–2k per wheel, **one** 1024² SurfaceAppearance set (Color/Normal/Metalness/Roughness) for the body and a shared 512² set for the wheels. Set `Massless = true`, `CanCollide = false`, `CanQuery = false` and weld everything to `Chassis`. Import with *Avatar → 3D Importer*, then put the model in `ServerStorage/VehicleTemplates` (the Phase 3 garage will clone from there).

### Audio
`Config.Audio` contains **empty** slots: engine loop, screech, boost, countdown beep/GO, checkpoint, lap, finish, UI click, lobby/map music and city ambience. Paste in IDs for audio **you own or have licensed**; empty slots are skipped silently.

### Other art worth commissioning before Phase 7
A Cyberpulse skybox (6 faces, night skyline), hologram ad textures, a rain streak texture, a speed-line texture, a RIFT RACERS logo mark, and UI icons.

---

## 7. Known limitations (honest list)

- **Not playtested in Studio.** Physics, camera feel, UI layout on small screens and frame rate all need the §4 run.
- **One race at a time per server.** Phase 4 adds instanced races, team modes, map voting and MemoryStore/Teleport matchmaking.
- **No saving yet.** Coins, XP, perks and saved settings arrive in Phase 2. The results screen deliberately shows **no coins** because nothing is saved yet.
- **Garage and Mystery Box pavilions** are closed buildings labelled with the phase they open in. They have no fake buttons.
- `StreamingEnabled` is off for Phase 1 because the world is generated on the server. Phase 7 should turn it on with `ReplicationFocus` set to the car and LOD/streaming models for the city.
- Each driver's client simulates its own car, which is standard for responsive Roblox driving. The server validates the results. A determined exploiter could still drive "perfectly" within the physical limits, which is the normal Roblox trust boundary for vehicles.
- The driver avatar is hidden inside the tinted cabin.

## 8. Next: Phase 2 (perks & economy)
DataStore profiles with session locking, migrations and retry/backoff; coins earned only through races; the three free first-race perks (with a one-time server-side grant); the 40-perk data catalog; the 5-slot equip system with an effect budget; Street, Turbo and Cosmic boxes using server-side weighted random selection with the real probabilities shown; duplicates turned into shards; and the box-opening animation.
