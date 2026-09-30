# Tower Vision: design and technical spec

Working title. Owner: Ben. Target: static web app on GitHub Pages. Sibling
project to Tunnel Vision (`../ant farm/docs/SPEC.md`) — same concept, inverted
direction: instead of a colony digging down into soil, a crew builds a tower
up into the sky.

## 1. Vision

A calm place to focus. The top of the screen is a soft storybook sky holding a
large, quiet timer. Below the horizon is a small grassy yard on a hilltop.
When the user starts focusing, a master builder walks onto the empty plot,
surveys it, and lays a foundation stone. Recruits arrive from the road below,
train up, and the crew raises a tower course by course — hauling stone and
timber up ladders and scaffolding, adding floors, a storeroom, a bunkroom, and
eventually a belfry at the top.

The tower only grows while the timer runs. At the end of a session, its
height and shape are a visible picture of the focus the user put in.

Every session is unique (seeded), but always sound (rules keep the tower
standing tall and true). The world around the tower lives on its own: days
turn to night, rain comes and goes, and small moments occasionally pass
through the sky.

### Design principles

1. **Calm over busy.** Low speeds, soft easing, few simultaneous events.
2. **Growth you can see.** The tower's height is the whole point — it should
   read clearly as "what I built this session," visible from across the yard.
3. **Unique but sound.** Randomness lives inside rules that guarantee a
   good-looking, structurally believable tower. It never looks like it's
   about to fall over.
4. **One memorable thing.** The tower is the hero. The UI is quiet and gets
   out of the way.
5. **Kind to the machine.** People will leave this open for hours.

## 2. Session model and pacing

### Focus-linked growth

- The **focus clock** accumulates time only while the timer is running.
- Crew growth systems (recruitment, training, hauling, building) advance on
  `focusTime`.
- When paused (or during a pomodoro break), builders finish their current
  small action, then rest: they set down their tools, sit on a beam or the
  grass, and idle. The master builder rests too. Recruits do not graduate.
- The environment (sky, day/night, weather, flags, surprises) keeps living on
  `realTime` regardless.
- Each page load is a fresh tower with a new random seed, unless `?seed=` is
  given. The seed is shown in settings with a copy button.

### Pacing curve

Growth is fast at first and slows logarithmically, so a 25-minute session is
satisfying and a 3-hour session doesn't tower absurdly out of frame.

Target crew size (a *target*, not an instant value; actual builders lag
because recruits take time to train):

```
t          = focused minutes
t_found    = 6            # founding period, master builder alone
tau        = 30           # curve softness
T_mature   = 120          # "full crew" at 2 focused hours
W_mature   = 50
W_cap      = 70

t' = max(0, t - t_found)
targetCrew(t) = min(W_cap, W_mature * ln(1 + t'/tau) / ln(1 + (T_mature - t_found)/tau))
```

Approximate targets (identical curve shape to Tunnel Vision's worker count —
same pacing feel, different label):

| Focused time | Target crew |
|---|---|
| 5 min | 0 (master builder alone) |
| 10 min | 4 |
| 15 min | 8 |
| 25 min | 16 |
| 45 min | 27 |
| 60 min | 33 |
| 90 min | 43 |
| 2 h | 50 |
| 3 h | 61 |
| 4 h+ | 70 (cap) |

Tower height tracks crew size and time invested (a real crew builds faster
with more hands, up to a point where the single stairwell/scaffold becomes the
bottleneck):

```
targetFloors = foundingFloors + FLOORS_PER_CREW_MINUTE * min(crew, EFFECTIVE_CREW_CAP) * focusedMinutes
```

All of these constants live in `src/config.ts`. Exact rates are tuned during
M4/M5 so a 25-minute session reaches a satisfying handful of floors and a
3-hour session reaches an impressive but still-framable tower — this is a
tuning pass, not a fixed target to hit on the first try.

### Founding timeline (focused time)

| Time | Event |
|---|---|
| 0:00 | Master builder walks in from the road, sets down a pack (about 10 s) |
| 0:15 | Paces off the footprint, drives corner stakes and string lines |
| 0:30 | Begins laying the foundation course |
| ~2:30 | Foundation complete; unrolls a blueprint and studies it |
| ~3:00 | First recruit arrives, walking up the road |
| ~5:00 | Recruit starts as an apprentice (half speed, smaller pack) |
| ~8:30 | First apprentice graduates to a full builder and takes over hauling |
| after | Master builder stays on site, checks work, and directs the foreman's plan |

Training duration and recruitment rate are config values. Recruitment rate is
driven by the gap between `targetCrew` and (builders + apprentices).

## 3. World

### Coordinates

- World units are "world pixels." y = 0 is the ground line; negative y is up
  (sky and tower), positive y is the shallow yard/foundation band.
- Tower footprint grid: `GRID_W = 160`, per-floor height `FLOOR_H = 28` world
  px, footprint cells `CELL = 4` world px (so a full-width floor is 40 cells).
  The tower's height is unbounded upward (grows one floor at a time); there is
  no fixed grid height the way Tunnel Vision's soil has a fixed depth.
- Sky extends as far above the tower's current top as `layout` requires to
  frame it, same idea as Tunnel Vision's fixed sky height, just recomputed as
  the tower grows.
- Builder height about 9 px (matches ant-scale suggestion-over-detail). Master
  builder about 12 px.
- Ladders/stairwells are one cell wide; floor interiors are wide enough for
  two builders to pass.

### Tower structure (`src/sim/tower/structure.ts`)

Rather than a 2D density grid (Tunnel Vision's soil), the tower is a **stack
of floors**, each stored as:

- `type` (foundation, workshop, storeroom, bunkroom, belfry — see section 4)
- `width` (cells; tapers with height per the structural rule below)
- `material` (fieldstone, brick, or timber-frame — chosen per seed, may shift
  partway up for visual interest, like a real building's additions over time)
- `buildProgress` (0..1 per floor, drives the soft "still being built" look —
  the direct equivalent of soil `density`)
- Optional `feature`: a side turret, a balcony, a scaffold arm, a window bay

Plus a **stockpile** at the yard (the mound's mirror image): stacked stone and
lumber that shrinks as builders haul it up and is replenished by quarriers.

### Structural soundness (seeded, but always sound)

- Each floor's width must be within a tapering ratio of the floor below it
  (no overhangs beyond a small allowance) — the tower's equivalent of the
  angle-of-repose rule that keeps Tunnel Vision's mound from spiking.
- A side turret or scaffold arm may only attach where the main tower is wide
  enough to support it, with a minimum vertical gap from any other feature.
- No feature or floor may be requested above what the current stockpile plus
  in-flight hauling can plausibly supply (mirrors Tunnel Vision's "every open
  cell connected to the entrance" — nothing appears out of nowhere).

### Invariants (tested)

- The tower is always a single connected stack, floor 0 (foundation) to the
  top — no floating floors, no gaps.
- Material is conserved: stockpile consumed ≈ material added to the tower
  (within a tolerance for load rounding).
- No floor is ever wider than the floor below it by more than the taper
  allowance (see above) — checked after long runs across seeds.

## 4. The foreman (`src/sim/tower/foreman.ts`)

The foreman decides *what* to build next. Builders decide *how*.

### Structure it aims for

- A single main tower rising straight up, course by course, with a slight
  seeded lean-and-correct wobble for hand-built character (never enough to
  look unstable).
- Workshops and storerooms mostly low, bunkrooms scattered through the
  middle, the belfry always last and always exactly once, at the current top.
- As the crew grows, a side turret or scaffold arm may attach partway up for
  silhouette variety (rare, like Tunnel Vision's secondary shafts).
- Windows are cosmetic, added opportunistically to floors that don't already
  have a feature, purely for visual rhythm.

### Floor types

| Type | Placement | Notes |
|---|---|---|
| Foundation | Ground level, first built, always widest | Anchors the tower |
| Workshop | Low floors | Workbenches and tools visible inside |
| Storeroom | Scattered | Stacked stone/lumber visible inside (like a granary) |
| Bunkroom | Scattered | Where idle/resting builders gather |
| Belfry | Always the topmost floor, exactly one | A bell or banner; the crew's lookout |

A new floor is requested once the current top floor's `buildProgress` reaches
1. A storeroom or bunkroom is requested instead of a plain workshop floor when
   a need crosses a threshold: stockpile exceeds storage capacity, idle
   builders exceed bunkroom capacity. The belfry is requested once the crew
   has been at (or near) its target size for a while, signaling "the tower is
   mature" — same spirit as a nest reaching its mature worker count.

### Placement rules

- Floor width starts at the foundation's width and tapers gradually with
  height (seeded taper rate within a band), with small per-floor jitter so
  towers don't look perfectly conical.
- Side turret / scaffold arm candidates are scored: minimum vertical spacing
  from other features, prefers a floor with a bit of spare width, small random
  jitter so ties break differently per seed.
- Material may shift once or twice as the tower rises (e.g., fieldstone base,
  timber-frame upper floors) — seeded, at most two transitions, always at a
  floor boundary, never mid-floor.

### Build jobs

The foreman emits a priority queue of **build jobs**:

- `FloorCourse { floorIndex, type, width, material }`
- `Feature { floorIndex, kind }` (turret, balcony, scaffold arm, window)
- `HaulRoute { fromStockpile, toFloorIndex }`

Jobs are claimed by builder crew. A floor is built from the bottom course up,
so it visibly rises rather than appearing all at once.

## 5. Crew

### Roles

- **Master builder**: founding sequence, then stays on site directing the
  foreman's plan and occasionally inspecting finished floors.
- **Builders**: claim build jobs, haul material up, place it, and grow the
  tower.
- **Trainers**: mentor apprentices so they graduate to full builders sooner.
- **Quarriers**: work the small quarry/woodpile at the yard's edge, gathering
  raw stone and lumber into the stockpile.
- **Idlers**: rest on the grass or a bunkroom bench, eat, chat, nap in the
  sun.

Role mix is recomputed every few seconds from the tower's needs. Following
Tunnel Vision's lead, a minority of the crew (about 30%) does most of the
hauling while many others idle — that loitering reads as peaceful, so keep it.

### Behavior (`src/sim/crew/`)

Each builder has a small state machine, for example for a builder:

```
idle -> goToStockpile -> loadMaterial -> goToTop -> climb -> placeCourse -> return
```

Other states: `goToTarget`, `pickUp`, `carry`, `drop`, `rest`, `chat`,
`shelter` (rain), `packTools` (weather closing in), `train` (apprentice with a
trainer).

### Building (the "how")

At the build site, a builder places material with a small amount of natural
variation — course lines aren't perfectly level, blocks have slight size
jitter, mortar lines wobble a touch — so the tower reads as hand-built, not
extruded. Each placement advances `buildProgress` on the current floor by an
amount scaled by the builder's carried load; a full load represents about the
same fraction of a course each time, but material type affects placement
speed slightly (stone is slower and sturdier-looking than timber).

### Stockpile

- Quarriers bring raw material to a stockpile at the tower's base.
- Builders draw from it to haul upward; the pile visibly shrinks and
  replenishes rather than sitting static.
- The pile settles with the same angle-of-repose relaxation Tunnel Vision uses
  for its mound, so it always looks like a natural pile, not a floating stack.

### Movement and pathing

- Builders move along the ground, ladders, and scaffolding — not through
  walls. A single central stairwell/ladder column is the main vertical route,
  mirroring the ant farm's single main shaft as the primary thoroughfare.
- A **height field** (distance up the stairwell from the ground) is
  recomputed when the tower grows. Builders heading up follow it.
- For other targets (stockpile, quarry, bunkroom), A* over the walkable
  graph, with results cached per target and invalidated when the tower
  changes shape.
- Paths are smoothed and followed with gentle steering (seek, arrive, and
  light separation so builders don't overlap on ladders).
- Speeds (world px/s): builder 18 to 26 (each gets its own), master builder
  10, carrying 80% of normal. Turning rate is capped. No instant turns.
- A simple walk-cycle animation whose rate matches speed, same approach as
  the ant gait.

## 6. Environment

### Day and night

Two modes, chosen in settings:

- **Compressed** (default): a full day cycle every 24 minutes (options 12, 24,
  48). Sessions start at mid-morning.
- **Real time**: follows the user's local clock.

Time of day is a pure function of mode, cycle length, session start, and
current real time. Reuses the same keyframe-blend approach as Tunnel Vision
(see palette section) — the sky system is close to a direct port.

Visuals:

- Sky gradient and scene light blend between four keyframes.
- The sun and moon travel on soft arcs behind the timer's area but never
  directly behind the digits, and never behind the tower's current top.
- Stars fade in at dusk with a very slow, subtle twinkle (no flicker).
- Lantern light glows in the tower's windows at night (a handful, slow glow
  pulses) — the direct equivalent of fireflies.
- The tower's interior is lit with storybook license: it dims with the scene
  but never goes fully dark, so the crew stays visible at night.
- Crew activity at night is reduced (movement speed × 0.8, fewer quarriers,
  more resting).

### Rain

Weather is a seeded schedule, a pure function of seed and real time:

- `clear -> clouding (60 to 90 s) -> rain (4 to 8 min) -> clearing (90 s) -> clear`
- First rain no earlier than 15 minutes into the session; mean gap about 45
  minutes. Light or medium rain only. No thunder, no lightning near the
  tower — nothing about weather should read as threatening to the structure.

Effects:

- Clouds drift in on two parallax layers and soften the light.
- Raindrops are short, soft streaks at a slight angle; small splash particles
  at the ground line; gentle ripples in puddles that form in the yard.
- Exposed floors darken slightly with damp (a moisture tint on stone/timber
  that fades over about 10 minutes after rain ends) — the direct equivalent
  of Tunnel Vision's soil moisture front, just applied to the tower's surface
  instead of diffusing through it.
- Quarriers head for shelter when clouds arrive. Once it rains, a builder or
  two may pull a tarp over the topmost, unfinished floor and remove it once
  the rain ends (the tower's equivalent of plugging the entrance).
- After rain, a faint rainbow may appear over the tower (seeded chance), and
  fade over a few minutes.

### Small surprises

A seeded schedule triggers one surprise every 6 to 12 minutes of real time, at
most one active at a time, weighted by time of day and weather:

| Surprise | When | What happens |
|---|---|---|
| Bird | Day | Lands on a windowsill or the roof, rests a while, flies off |
| Kite | Day, breezy weather | Drifts past, briefly catches on a high floor, flutters free |
| Balloon | Day, dusk | A hot-air balloon crosses the distant sky |
| Cat | Any | Prowls the yard, pauses to watch the crew, wanders off |
| Falling leaf | Any | Drifts down from a nearby tree, rests on the grass, fades later |
| Extra lantern light | Night | A slightly warmer glow in a few more windows |
| Rainbow | After rain | Arcs faintly over the tower, fades over a few minutes |
| Ivy growth | Continuous | Ivy on the lower stones lengthens almost imperceptibly |

The crew reacts lightly (for example, a quarrier pauses to watch a cat pass).

## 7. Art direction: warm storybook

### Look

Warm, earthy, gently textured, like an illustration in a picture book. Soft
shapes, no hard outlines, a subtle paper grain across everything. Colors are
muted and warm. Nothing is saturated or harsh. This is the same art direction
as Tunnel Vision — same illustrator, different subject.

### Palette (`src/theme/palette.ts`)

Time-of-day keyframes (blend smoothly between them) — start from Tunnel
Vision's values and retune during M1 sign-off; listed here as a working v1:

| Keyframe | Clock | Sky top | Sky horizon | Light tint | Exposure |
|---|---|---|---|---|---|
| Dawn | 06:00 | `#C9A9B8` | `#F4D3A8` | `#FFE4C4` | 0.90 |
| Day | 12:00 | `#A9CFD8` | `#EEE7CF` | `#FFF8EC` | 1.00 |
| Dusk | 18:30 | `#8C7BA6` | `#EDA774` | `#F6C79A` | 0.85 |
| Night | 00:00 | `#1E2640` | `#3C4868` | `#9AA8D0` | 0.55 |

Yard and ground:

| Name | Hex |
|---|---|
| Grass light | `#9BB36A` |
| Grass mid | `#7A9A52` |
| Grass dark | `#5C7A42` |
| Flower accents | `#E8C27A`, `#D98C7A`, `#F3EAD6` |
| Stockpile stone | `#9C8B7A` |
| Stockpile lumber | `#B8925C` |
| Dirt path | `#7A5A40` |

Tower materials, by era:

| Material | Base | Mortar/joint | Notes |
|---|---|---|---|
| Fieldstone | `#8A8078` | `#6B6058` | Rounded, irregular blocks; the humble starting material |
| Brick | `#A0664A` | `#6B4A38` | Tidier, used for mid-height additions |
| Timber-frame | `#7A5A40` | `#4A3428` | Upper floors on a mature tower; warmer, lighter-feeling |
| Fresh course highlight | `#B89270` | — | The most recently placed course, before it weathers in |

Tower interior (cross-section, mirrors Tunnel Vision's tunnel palette):

| Name | Hex |
|---|---|
| Interior shadow | `#2B1E17` |
| Interior rim (edge darkening) | `#22170F` |
| Floor highlight | `#6E4F38` |
| Damp material multiply | `#5A4A48` at up to 35% |

Crew and props:

| Name | Hex |
|---|---|
| Builder tunic | `#2E211C` |
| Builder highlight | `#5A4036` |
| Master builder coat | `#3A2620` |
| Master builder sash | `#7A4A2A` |
| Apprentice pack | `#EEF2F2` at 60% alpha |
| Recruit (new arrival) | `#F6EFDD` |
| Apprentice | `#F0E4C6` |
| Stone load | `#8A8078` |
| Lumber load | `#B8925C` |
| Lantern glow | `#F7E48A` |
| Stars | `#FDF6E3` |

UI:

| Name | Value |
|---|---|
| Text on sky | `#FFF8EC` |
| Text on panels (light theme panels) | `#3B2A20` |
| Honey accent (also the banner/flag color) | `#E3A857` |
| Panel background | `rgba(59, 42, 32, 0.55)` with backdrop blur |

Global: a paper grain texture over the whole scene at about 5% (multiply), and
a very soft vignette.

The tower's interior should also dim gradually with height above the ground
(about 25% darker toward the top before scene light is applied), independent
of time of day — the equivalent of Tunnel Vision's depth dimming, just
pointed upward instead of down. Retune or drop this during M1 if it fights
with the sky's own brightening toward the top; unlike soil, a tower's upper
floors are *closer* to open sky and daylight, so this may end up inverted or
removed rather than a straight port.

### Tower rendering (the key technique)

Never draw the tower as flat rectangular blocks stamped down. This is the
same problem Tunnel Vision solves for tunnels, applied to stacked floors:

1. Keep a **build mask** texture (one column × floor-height texel grid)
   holding each floor's `buildProgress`. Update only the current top floor's
   region as it fills in.
2. Sample it with bilinear filtering in a custom fragment shader over the
   tower layer.
3. In the shader:
   - `built = smoothstep(0.45, 0.55, mask)` gives a soft, hand-laid course
     edge instead of a hard rectangle snapping into place.
   - A second, slightly wider threshold band gives the darker **mortar rim**
     between courses.
   - The most recent course gets the **fresh-course highlight** before it
     "weathers in" over a minute, the same way Tunnel Vision's fresh pellets
     lighten and blend in.
   - Add a faint noise wobble to the threshold so block edges look
     hand-drawn, not ruled.
   - Blend material color with a hint of the floor type's own accent (a
     storeroom's stacked-goods color peeking through a window, for example).
4. Fallback if the shader approach fails: draw each course as a smoothed
   filled path from the build-progress field, same fallback Tunnel Vision
   uses for its tunnel mask.

### Tower texture

Generated once per seed into a render texture: material bands per the seed's
chosen material transitions, fine grain speckle, a bit of weathering/moss
near the base, ivy on the lower stones. Damp darkening and height dimming are
applied in the shader, not baked.

### Crew and cargo

- Builders: three soft ellipses (head, torso, legs-as-a-single-shape at this
  scale), thin arms and tool handles as lines with rounded caps. Suggestion
  beats detail, same as Tunnel Vision's ants.
- A gentle 1 px darker underside and a faint highlight on the torso.
- Carried loads sit on the shoulder or in both arms, ahead of the body.
- Recruits/apprentices are a smaller, paler version of the builder shape.

### Yard

- Grass: layered tufts with slow, per-tuft wind sway (phase offset by
  position, amplitude tied to weather) — direct reuse of Tunnel Vision's
  grass system.
- A dirt path from the road to the tower's base, worn in by foot traffic.
- The stockpile: a small, gently-sloped pile of stone and lumber beside the
  tower's base, settling the same way the mound does.
- A quarry/woodpile at the yard's edge where quarriers work.

### Camera

- Starts framed on the ground/yard and the tower's low, unfinished footprint,
  with room for the timer above.
- As the tower rises, the camera eases to keep the whole tower plus the yard
  in view, zooming out down to a minimum scale of 0.6. Below that, it pans up
  instead — the same rule as Tunnel Vision, just inverted (up instead of
  down).
- Camera changes use a critically damped spring with a time constant of about
  60 to 120 seconds. The user should never notice it moving.
- The user can drag to pan and scroll to zoom. After 20 seconds without
  input, the camera eases back to auto framing. Manual pan/zoom is bounded to
  the decorative backdrop's own extent, same lesson Tunnel Vision learned the
  hard way (see its `render/camera.ts` for the exact approach).

## 8. Timer and UI

### Layout

```
+--------------------------------------------------------------+
|                                                    (gear)    |
|                                                              |
|                         1:24:07                              |
|                What are you focusing on?                     |
|                  [ Pause ]   [ End session ]                 |
|        ~ sky, clouds, sun/moon ~                             |
|                     [tower rising]                            |
|~~~~~~~~~~~~~~~~ grass ~~~~ stockpile ~~~~~~~~~~~~~~~~~~~~~~~~|
|                                                              |
+--------------------------------------------------------------+
```

This is a near-exact port of Tunnel Vision's layout, just with the built
structure rising into the sky above the grass line instead of a cross-section
sitting below it.

- The timer is centered in the sky, large (clamp around 64 to 120 px), set in
  **Alegreya** with lining tabular figures (`font-variant-numeric: lining-nums
  tabular-nums`) so digits don't shift.
- Controls and labels use **Alegreya Sans**.
- Timer text sits on a very soft radial backdrop so it stays legible at every
  time of day, and clear of the tower's current top (the backdrop may need to
  nudge up slightly as the tower grows tall enough to reach it — check this
  during M4/M7 and adjust `layout.skyFraction` or the backdrop's position if
  it does).
- The task name is an inline editable field with the placeholder
  "What are you focusing on?"
- Timer format: `m:ss` under an hour, `h:mm:ss` after.

### Controls

- Start focusing / Pause / Resume (one button, label reflects the action)
- End session (opens the summary)
- Mode: Open-ended (counts up) or Pomodoro (focus and break lengths,
  default 25 and 5, with a long break of 15 every 4 rounds)
- Settings panel (gear): day/night mode and cycle length, sound (master volume
  plus toggles for nature sounds and music), frame rate (smooth 60 or battery
  saver 30), reduced motion (follow system, on, off), seed (view, copy, start
  a new tower with a new seed or a pasted seed)

### Behavior

- Timer is computed from timestamps: `focused = sum(finished segments) + (now -
  segmentStart)` while running.
- While the timer runs, the UI fades out after 4 seconds without mouse or
  keyboard activity and fades back in on any input. The timer digits stay
  visible at reduced opacity; everything else hides.
- Keyboard: Space start/pause, F fullscreen, M mute, S settings, Esc close
  panels.
- The browser tab title shows the timer so users working in other tabs can see
  it: `25:13 Writing report` while running, `Paused 25:13` when paused. Hidden
  tabs update the title at whatever rate the browser allows; show minutes only
  when the tab is hidden.
- Pomodoro transitions play a soft chime and change the button label; during
  breaks the crew rests.
- Settings persist in `localStorage` (wrapped in try/catch). The tower itself
  is not persisted in v1.

### Session summary

On End session, a soft panel shows:

- focused time,
- crew size, floors built, material used,
- the seed,
- **Start a new tower** and **Keep watching**.

### Returning to a hidden tab

People will usually work in other tabs, so the tab will often be hidden while
the timer runs. Browsers throttle hidden tabs heavily.

- The timer is always correct because it uses timestamps.
- Environment state is recomputed directly from time (no catch-up needed).
- The tower catches up when the tab becomes visible: compute missed focus
  time and run the simulation in **catch-up mode**, spread across frames
  (budget about 8 ms of sim per frame), shown as a brief time-lapse with a
  small caption, "While you were away, the crew kept building."
- Catch-up mode uses a coarser timestep and simplified movement (skip
  steering, advance along cached paths) so an hour of missed focus catches up
  in under about 10 seconds. Cap catch-up at 3 hours of focus time.
- Benchmark this in its milestone. If the main thread can't meet the budget,
  move the simulation into a Web Worker, passing state with transferable
  ArrayBuffers (not SharedArrayBuffer).

## 9. Sound

Everything is procedural first (no licensing issues, tiny download), using
Tone.js. All layers fade in and out over at least 3 seconds. A master limiter
keeps peaks soft. Audio starts only after the first Start click.

| Layer | When | How |
|---|---|---|
| Air bed | Always, very low | Brown/pink noise, low-pass filtered, slow LFO on cutoff |
| Rain | During rain, follows intensity | Filtered noise plus sparse soft droplet ticks, panned randomly |
| Crickets | Night | Sparse synthesized chirps (short high sine bursts with amplitude modulation), a few voices at different rates |
| Birdsong | Day, sparse | Optional CC0 samples in `public/audio/` (owner supplies them). If absent, skip this layer rather than using cheesy synthesized birds |
| Building | While builders are active | Very quiet hammer taps, saw rasps, and mallet knocks, density tied to build rate |
| Music (optional) | Off by default | Slow generative pad: pentatonic notes, long attack and release, lots of reverb, a new note every 6 to 12 seconds |
| Chime | Pomodoro transitions | Soft FM bell, low volume |

Settings: master volume, nature sounds on/off, music on/off, mute (M).

Any sample files go in `public/audio/` with sources and licenses listed in
`public/audio/LICENSES.md`.

## 10. Performance and accessibility

- Sim runs at a fixed 30 Hz; rendering interpolates between sim states.
- Frame rate options: 60 (default) or 30 (battery saver).
- Stop rendering when the tab is hidden (`visibilitychange`); the sim is
  caught up on return.
- Budgets: up to 70 crew, pooled particles (max 400 raindrops, 40 lantern
  glows). Target under 4 ms render and under 2 ms sim per frame on a
  mid-range laptop.
- Update the tower build-mask texture only in dirty regions.
- `prefers-reduced-motion` (or the setting): no rain streaks (show a soft
  overall rain tint and puddle ripples only), no star twinkle, fewer particles,
  UI transitions become fades only, camera snaps between framings with a slow
  cross-fade instead of zooming.
- All controls are real buttons with labels and visible keyboard focus. The
  timer is not an `aria-live` region (no per-second announcements); pomodoro
  transitions are announced once.
- UI text meets WCAG AA contrast against its backdrop at every time of day —
  check this separately at night the way Tunnel Vision eventually had to;
  don't assume a daytime-only check is enough.
- Layout works down to phone width (the sky area shrinks; the timer scales).

## 11. Dev mode

Enabled with `?dev=1`. A small collapsible panel:

- Time scale: 1×, 10×, 60×, 300×
- Jump focus time: +5 min, +30 min, +2 h (runs catch-up mode)
- Seed input and "regenerate"
- Force time of day (slider), force weather state, trigger any surprise
- Overlays: grid, build-progress field, foreman floors and jobs, crew states
  and paths, height field
- Stats: FPS, sim ms, render ms, crew counts by role, floors built vs target,
  material carried

A debug API on `window.__tower` (dev mode only) for Playwright:

```
setSeed(seed), setTimeScale(n), advanceFocus(minutes),
setTimeOfDay(hours), setWeather(state), triggerSurprise(name),
getStats(), getStateHash()
```

## 12. Testing

Vitest (`tests/`):

- Determinism: same seed and same focus timeline give the same `getStateHash()`
  after N ticks.
- Different seeds give different tower shapes.
- Invariants after long runs (several seeds, 3 h of focus): connectivity
  (no floating floors), material conservation, no floor wider than the one
  below it beyond the taper allowance.
- Pacing: crew size and floor count follow the target curves within
  tolerance at 25, 60, and 120 minutes.
- Focus clock: pause/resume math, pomodoro transitions, hidden-tab gaps
  (this module can likely be ported near-verbatim from Tunnel Vision).

Playwright (`e2e/`):

- Smoke test: loads, starts, pauses, ends session, no console errors.
- `npm run shots`: screenshot gallery for 3 seeds × focus times (0, 10 min,
  25 min, 2 h) × times of day (dawn, day, dusk, night), plus rain. Written to
  `screenshots/` (git-ignored) for visual review, not strict pixel diffs.
- When writing the gallery's dev-panel-driven tests, remember to actually
  click "Start focusing" before using the time-scale buttons — Tunnel
  Vision's gallery silently shipped several tests that forgot this once
  growth was gated behind focus running, and it went unnoticed for a while.

## 13. Project structure

```
tower-vision/
  CLAUDE.md
  docs/
    SPEC.md
    MILESTONES.md
    reference/            mood-board images
  public/
    audio/                optional CC0 samples + LICENSES.md
    textures/             paper grain, etc.
  src/
    main.ts
    config.ts             tunable sim, pacing, timing constants
    app/                  bootstrap, main loop, visibility, catch-up, settings store
    sim/                  pure simulation (no DOM, no Pixi)
      rng.ts  noise.ts  structure.ts  materials.ts  pacing.ts  sim.ts
      tower/               foreman.ts  lifecycle.ts  roles.ts
      crew/                agent.ts  behaviors.ts  steering.ts  pathing.ts  building.ts
      yard/                stockpile.ts  quarry.ts
      env/                 dayNight.ts  weather.ts  moisture.ts  surprises.ts
    render/               Pixi scene
      scene.ts  camera.ts  sky.ts  yard.ts  tower.ts
      tower.frag  crew.ts  cargo.ts  weather.ts  creatures.ts
      lighting.ts  particles.ts  snapshot.ts
    audio/                engine.ts and one file per layer
    timer/                focusClock.ts  pomodoro.ts
    ui/                   timer.ts  controls.ts  settings.ts  summary.ts  idleFade.ts  styles.css
    theme/                palette.ts  typography.ts
    dev/                  devPanel.ts  debugApi.ts  overlays.ts
  tests/
  e2e/
  .github/workflows/deploy.yml
```

## 14. Deployment

- GitHub Actions workflow on push to `main`: install, test, build with
  `BASE_PATH=/<repo-name>/`, upload `dist/`, deploy with the official Pages
  actions (`actions/upload-pages-artifact` and `actions/deploy-pages`).
- Repository setting: Pages source is "GitHub Actions."
- All asset URLs must respect Vite's `base`.

## 15. Later ideas (not in v1)

- Persist the tower between sessions (IndexedDB) and grow it over days.
- Gallery of saved tower pictures.
- Installable PWA with offline support.
- Seasons (autumn leaves, winter slowdown, snow settling on ledges).
- Share a seed link.
- Wind as a distinct weather state (flags snapping, a kite surprise becomes
  more likely) — deliberately left out of v1's rain-only weather so it isn't
  designed in a rush.
- A finished tower "graduates" into a small skyline that accumulates across
  sessions, echoing Tunnel Vision's later "grow a colony over days" idea.
