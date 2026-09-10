# MOGSsoccer — Full Roadmap

**Engine:** Unreal Engine 5.7 · **Team:** 4 people, Blueprint-based, ~5–10 hrs/week combined
**Last worked on:** paused mid-way through the Game Over screen

---

## The Concept

An arcade football game. Gameplay feel modeled on **Pro Soccer Online** — responsive, skill-based, simple controls. Match structure modeled on **EA FC Clubs mode**: each player controls one outfield player, chosen before queuing. Up to 10 real players per side, AI fills empty slots, 11v11 on a full-size pitch. Goalkeeper gameplay is out of scope for now.

---

## DONE

### Player control
- Custom character model (hand-drawn → Meshy → FBX), retargeted mannequin AnimBP
- Camera-relative rotation: `Use Controller Rotation Yaw` on, `Orient Rotation to Movement` off
- Jog 700 / Sprint 1200 via `Set Max Walk Speed`

### Ball mechanics
All three use hold-to-charge, check distance ≤ 100 units to the ball, and are gated on `BallInPlay`.

| Input | Action | Direction source | Notes |
|---|---|---|---|
| Left click | Kick | Camera (`Get Control Rotation` → forward vector) | Power = `Elapsed Seconds / 1.0`, hold threshold 1.0s |
| Right click | Dribble | WASD (`Get Last Movement Input Vector`) | Weaker force, hold threshold 0.5s |
| Middle mouse | Flick | WASD + fixed `(0,0,5000)` vertical | Horizontal power × charge, constant lift |

Ball setup: 5kg, linear/angular damping 0.5, CCD on, Pawn collision set to Overlap so the player body doesn't shove it.

### Sprint & stamina
Rebuilt from scratch after Enhanced Input events proved unreliable. Runs entirely on Event Tick.

- Three conditions ANDed: `Is Input Key Down (Left Shift)` + forward check + `Stamina > 0`
- Forward check = **dot product of normalized velocity vs actor forward > 0.5**
- AND result drives a `Select Float` (1200/700) into a **single** `Set Max Walk Speed`
- Drain 20/s while sprinting; `RegenDelay` resets to 3.0 every sprint frame
- On release: delay counts down, then regen at 15/s, `Min` node clamps at MaxStamina

### Status rings (2K-style, under the player, local player only)
- **`M_PowerCircle`** — inner ring. Radial fill: TexCoord → Subtract (0.5,0.5) → Arctangent2Fast → ÷6.283 → +0.5 → Step against `FillAmount`. Colour via `ColorR/G/B` scalar params → AppendMany. **Red** = kick, **Blue** = dribble, **Yellow** = flick. Resets to 0 on every release, even if the kick doesn't land.
- **`M_StaminaCircle`** — separate material on a second, larger plane. Fixed orange, `StaminaAmount` driven from Tick as `Stamina / 100`. Split from the power ring because colours bled when combined.
- Both use Dynamic Material Instances created in BeginPlay.

### Animation
- Mixamo animations imported **directly** to the custom skeleton (avoids double retargeting)
- **Layered Blend Per Bone** at bone `Spine01` — gestures play upper-body only while legs keep running
- Graph: `Control Rig` → `Save Cached Pose (BasePose)` → two `Use Cached Pose` → `Slot 'DefaultSlot'` (Base Pose) + `Slot 'UpperBody'` (Blend Pose 0) → `Layered Blend Per Bone` → Output
- `BS_Movement` Blend Space 1D driven by `Ground Speed / 12`; separate `Ground Speed / 600` into Play Rate so sprint doesn't look slow-motion
- `Force Root Lock` on locomotion animations
- Montages: AM_Kick, AM_LightKick, AM_Flick, AM_DribbleForward/Back, AM_CallBall, AM_GroundPass, AM_ThroughBall, AM_LoftedBall
- **Design decision:** montages fire simultaneously with the impulse, not on an AnimNotify — the animation delay hurt competitive responsiveness

### Gestures
Q/E/R/F set a `CurrentGesture` string, play an upper-body montage, clear after 3s. Built for AI to read later:

- **Q** — call for ball
- **E** — ground pass to feet
- **R** — through ball
- **F** — lofted through ball

### Match systems
- `BP_GameState`: `ScoreA`, `ScoreB` (RepNotify), `MatchTime`, `BallInPlay`, `GameEnded`, `TeamA`, `TeamB`
- Timer counts up on Tick, displayed ×6 so 15 real minutes reads as 90:00
- Physical stadium scoreboard — invisible cube + Widget Component rendering `WBP_Scoreboard`
- `BP_Goal`: Ball tag check → `GoalScored` gate → `IsTeamA` branch (goal A conceded ⇒ Team B scores) → freeze ball → 3s celebration → re-fetch ball → teleport to centre → zero velocity → release gate

### Out of bounds
The foundation for every restart type.

- `BP_Boundary` actor with `BoundaryType` string (Instance Editable), dispatched via `Switch on String`
- Goal-line boundaries use 3 box components each, leaving a gap for the goal mouth
- **Sideline** → throw-in: keeps ball's X, snaps Y to ±3800
- **GoalLineLeft** → corner at X `-5000` · **GoalLineRight** → corner at X `6700` · Y ±3800 by ball side
- Timing: `BallInPlay=false` → 3s pause → teleport → freeze → 2s settle → `BallInPlay=true`

### Pitch geometry
Scaled ×2 for 11v11. Goal lines at X `-5330` / `7010` (asymmetric — centre isn't at world origin, left alone deliberately). Sidelines at Y ±4200. Ball rests at Z 300.

### Team architecture — built, intentionally not wired up
- `BP_Team` actor: `TeamID` (Integer, leaves room for >2 teams), `TeamColor`, `AttackingGoal`, `TeamPlayers` array, `Score`
- `BP_PlayerState`: `TeamID` (default `-1` = unassigned), `MyTeam` reference
- GameState spawns both teams in BeginPlay and holds references
- `AttackingGoal` and player registration left empty on purpose — untestable until players exist
- ⚠️ Needs a `Has Authority` check on the spawn before multiplayer

---

## IN PROGRESS

### `WBP_GameOver`
Triggers correctly from GameState at `MatchTime >= 900`, behind a `GameEnded != true` gate so it spawns once. Score binding works.

**Open bugs:**
- Border renders white instead of black at 0.7 alpha
- Doesn't cover the right ~20% of screen — needs full-screen anchor preset with offsets zeroed
- Rematch and Main Menu buttons placed but not wired

### Version control
Not set up. Months of work with no backup. Highest-priority non-gameplay task — needs a Unreal-appropriate `.gitignore` (exclude `Binaries/`, `Intermediate/`, `Saved/`, `DerivedDataCache/`) plus Git LFS for `.uasset` files.

---

## TO DO

### Near term — testable with one player
1. **Finish Game Over screen** — anchors, colour, button wiring
2. **Kickoff mechanic** — spawn positions, countdown, conceding team restarts
3. **Half time** at 45:00 game-time, teams switch sides
4. **Tackle mechanics** — slide and standing (needs a simple dummy to test against)
5. **Card system:**
   - Slide tackle: always yellow; **red** if ball >400 units away
   - Standing tackle: ~50% yellow when close; **100% yellow** if ball >400 units away

### The big phase
6. **AI players** — Blackboard + Behavior Tree + AI Controller
   - Blackboard keys: ball location, target goal, assigned position, nearest-to-ball flag, attacking/defending state
   - Build in layers: chase ball → hold formation → shoot/pass decisions → team coordination
   - Expect this to be long and heavily iterative
7. **Online multiplayer via EOS** — dedicated server, replication, client-side prediction, lag compensation, lobbies, matchmaking
   - Hardest part of the project by a wide margin
   - Strongly worth hiring a freelance Unreal networking specialist for this phase
8. **Character/jersey selection screen**

### After players exist
9. **Offside** — active only, no passive
10. **Jersey colours** — separate jersey faces into their own material slot in Blender post-Meshy, then tint via dynamic material instance
11. **Player-painted 512×512 pixel badges**
12. **Skill moves** (scissor, spin), injury mechanic, free kicks

---

## Hard-Won Lessons

Don't relearn these — each one cost a debugging session.

- **Actor Tags ≠ Component Tags.** `Get All Actors With Tag` only reads Actor Tags.
- **Actor references go stale across a `Delay`.** Re-fetch the ball after any wait.
- **Physics velocity functions need the Static Mesh Component**, not the actor — use `Get Component By Class`.
- **Restart positions must sit clearly inside the boundary volume**, or you get infinite retrigger loops.
- **Enhanced Input `Started`/`Completed` drop inputs** on rapid press. For held states, poll on Tick with `Is Input Key Down`.
- **`Get Last Movement Input Vector` goes stale.** Use velocity·forward dot product instead.
- **Mixamo animations carry root motion** — apply `Force Root Lock` or the character physically drifts.
- **Bone names are case-sensitive** — this skeleton uses `Spine01`, not `spine_01`.
- **One `Set Max Walk Speed`** driven by a Select beats multiple scattered Sets across branches.
- **Split materials** rather than fighting colour bleed between regions in one material.
- **Level-editor component instances are not the same as Blueprint components.** Configure components in the Blueprint editor.

---

## Working with Claude Code on this project

Blueprints are binary `.uasset` files — Claude Code **cannot read or edit them**. Its useful scope here is Git setup, `.ini` configs, build scripts, and C++ once you reach networking. Blueprint node-wiring still needs a conversational back-and-forth with screenshots.

Save this file as `CLAUDE.md` in the project root (next to the `.uproject`) so it loads automatically each session.

---

## Timeline reality check

Roughly 8–18 months to a playable online build at a casual pace, assuming momentum holds. Everything before multiplayer is achievable with the current workflow. Networking is the wall — it's where most hobby projects stall, and where outside help is worth budgeting for.
