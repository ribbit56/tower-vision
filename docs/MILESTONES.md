# Milestones

Build in this order. Finish each milestone, verify it, show screenshots, and
stop for Ben's review before starting the next. Look before behavior: the art
style is locked in before any real simulation exists.

Each milestone lists its goal, scope, and "done when" criteria. Where useful,
a milestone notes the equivalent piece in Tunnel Vision (`../ant farm`) that
solved the same problem in the other direction.

---

## M0. Scaffold and deploy pipeline

**Goal:** an empty but real project that deploys to GitHub Pages.

Scope:
- Vite + TypeScript (strict), PixiJS, ESLint, Prettier
- Vitest and Playwright configured (including the headless WebGL flag that works)
- `src/config.ts`, `src/theme/palette.ts` (all palette values from SPEC),
  `src/sim/rng.ts` (seeded PRNG with named streams)
- Seed from `?seed=` or random; `?dev=1` shows an empty dev panel
- A full-window Pixi canvas showing the day sky gradient and a flat grass color
- Fonts installed via `@fontsource`
- GitHub Actions workflow deploying to Pages with the correct `base`
- `.gitignore` includes `screenshots/`

Done when:
- `npm run dev`, `build`, `test`, `e2e` all work
- The deployed Pages URL shows the sky and grass with no console errors
- RNG test: same seed and stream name give the same sequence

---

## M1. Art direction lock (sign-off gate)

**Goal:** a static, beautiful scene that defines the look. No simulation yet.

Scope:
- Sky gradient with the four time-of-day keyframes (dev slider to blend)
- Sun, moon, stars, a couple of soft clouds
- Grass tufts with gentle sway, a few flowers
- A **hand-authored** tower: foundation, three plain floors, one storeroom,
  one bunkroom, a belfry on top — enough floor-type variety to judge the look
- Tower shader (SPEC section 7) driven by the hand-authored floor data
- A stockpile of stone and lumber beside the tower's base
- Static crew (a master builder and a few builders, including one carrying a
  load) and a couple of idlers resting near the stockpile
- Paper grain and vignette
- Static timer text and controls in place (not functional), in Alegreya and
  Alegreya Sans

Done when:
- Screenshots at dawn, day, dusk, night, at desktop and phone widths
- The tower has soft, hand-laid course edges, a darker mortar rim, and a
  lighter fresh-course highlight. No visible flat rectangular blocks anywhere
- Palette matches SPEC; nothing looks saturated or harsh
- The tower reads as structurally sound, not precarious
- Timer is legible at every time of day, and clear of the tower's silhouette
- **Ben approves the look.** Iterate here as long as needed; everything later
  depends on it

---

## M2. Living structure and a single builder

**Goal:** real tower data and one builder working believably.

Scope:
- Tower structure data (floors, widths, materials, build progress) generated
  from the seed
- Build-mask texture updates from the structure with dirty-region uploads
- One builder: goes to the stockpile, loads material, climbs to the top
  floor, places it, returns — no steering complexity yet
- Stockpile deposit and consumption with angle-of-repose settling on the pile
- Fixed 30 Hz sim with render interpolation
- Dev mode: time scale, build-progress overlay, stats

Done when:
- Course-by-course growth looks organic at 1× and 60×, with no flat blocks
  snapping into place
- Tests: connectivity (no floating floors) and material conservation hold
  after a long run
- Determinism test passes
- (Tunnel Vision equivalent: M2, "Living terrain and a single digger.")

---

## M3. Movement, pathing, and a full crew

**Goal:** calm, natural motion for a small crowd on ladders and scaffolding.

Scope:
- Height field up the main stairwell (throttled recompute as the tower grows)
- A* with caching and invalidation over the walkable graph; path smoothing
- Steering (seek, arrive, separation), speed variation, turn-rate cap
- A simple walk-cycle animation tied to speed
- 20 crew wandering, hauling, and idling together

Done when:
- A 60-second recording or screenshot sequence shows no jitter, snapping, or
  overlapping crew
- Builders pass each other on ladders and floors gracefully
- Sim stays under budget with 70 crew (benchmark in dev stats)
- (Tunnel Vision equivalent: M3.)

---

## M4. The foreman, seeded towers, and the camera

**Goal:** towers that are unique per seed and always sound.

Scope:
- Foreman: floor types, needs thresholds, candidate scoring, build job queue
- Foundation, tapering floors, storerooms/bunkrooms placed by need, the
  belfry capstone, rare side turret/scaffold arm
- Floors built from the bottom course up
- `advanceFocus(minutes)` in the debug API (temporarily driving growth by a
  crew-size stub until M5)
- Camera: auto framing with a slow spring, drag and zoom, return to auto
- Dev overlays for floors and jobs
- Screenshot gallery script (`npm run shots`)

Done when:
- Gallery of at least 6 seeds at a 2-hour-equivalent size: every tower looks
  different and every tower looks good (and sound — nothing looks like it's
  about to topple)
- Tests: no floor wider than the one below it beyond the taper allowance, no
  feature placed without minimum spacing
- Camera movement is not noticeable at 1×
- (Tunnel Vision equivalent: M4.)

---

## M5. Lifecycle, roles, and focus-linked growth

**Goal:** the tower's story, driven by focus time.

Scope:
- Master builder founding sequence (arrive, survey, stake, lay foundation,
  settle in)
- Recruit arrival and training into full builders
- Roles: builders, trainers, quarriers, idlers; role rebalancing; the lazy
  majority
- Storeroom stacking, bunkroom resting, quarry material gathering
- Pacing curve and floor-count target from SPEC section 2
- Two clocks (`realTime`, `focusTime`), with a stub focus clock toggled from
  dev mode
- Resting behavior when focus is paused

Done when:
- Minute 2 of a fresh session has something pleasant to watch
- Pacing test passes at 25, 60, and 120 minutes
- Pausing visibly calms the crew within a few seconds; resuming restarts it
- Screenshots at 0, 10, 25 minutes and 2 hours for 3 seeds
- (Tunnel Vision equivalent: M5.)

---

## M6. Timer, UI, and returning to the tab

**Goal:** the real focus tool.

Scope:
- Focus clock from timestamps; open-ended and pomodoro modes
- Controls, task name field, settings panel with persistence
- Idle fade, keyboard shortcuts, fullscreen
- Tab title updates
- Session summary with stats (crew size, floors built, material used)
- Catch-up mode with the "While you were away" time-lapse; benchmark it (move
  sim to a Web Worker only if the budget can't be met)
- Rendering stops while hidden

Done when:
- Playwright smoke test covers start, pause, resume, pomodoro transition, and
  end session
- Hiding the tab for a simulated hour, then returning, catches up in under
  about 10 seconds with a smooth time-lapse
- Keyboard-only use works; focus is visible
- UI copy follows the style rules in CLAUDE.md
- (Tunnel Vision equivalent: M6 — its `timer/focusClock.ts` and pomodoro
  logic can likely be ported near-verbatim.)

---

## M7. Day and night

**Goal:** a living sky.

Scope:
- Compressed (12, 24, 48 min) and real-time modes
- Keyframe blending for sky and scene light; tower interior dimming rules
  (check whether height-based dimming reads well at all — see SPEC section 7's
  note that this may need to invert or drop compared to Tunnel Vision's depth
  dimming)
- Sun and moon arcs, star fade and slow twinkle, lantern-lit windows at night
- Night activity reduction

Done when:
- A 24-step screenshot strip across one compressed day shows smooth, pleasing
  transitions with no abrupt jumps
- The tower stays visible and the timer stays legible at night
- (Tunnel Vision equivalent: M7 — the sky/day-night system is close to a
  direct port.)

---

## M8. Rain

**Goal:** gentle weather.

Scope:
- Seeded weather schedule (pure function of seed and time)
- Clouds, rain streaks, splashes, puddles and ripples in the yard
- Damp darkening on exposed floors, fading after rain
- Quarriers sheltering, a tarp pulled over the unfinished top floor and
  removed after
- Post-rain rainbow

Done when:
- Screenshots of clouding, rain, and clearing at day and night
- Rain feels soft, not stormy; reduced-motion variant looks good
- Particle counts stay within budget
- (Tunnel Vision equivalent: M8.)

---

## M9. Small surprises

**Goal:** occasional moments of delight.

Scope:
- Surprise scheduler (seeded, weighted by time and weather, one at a time)
- Bird, kite, balloon, cat, falling leaf, extra lantern light, rainbow, ivy
  growth
- Light crew reactions
- Dev triggers for each

Done when:
- Each surprise can be triggered from dev mode and looks at home in the art
  style
- Over a 2-hour accelerated run, surprises feel occasional, never busy
- (Tunnel Vision equivalent: M9.)

---

## M10. Sound

**Goal:** a gentle, ambient soundscape.

Scope:
- Tone.js audio engine that starts on the first Start click
- Layers from SPEC section 9, driven by weather, time of day, and build
  activity
- Optional birdsong samples loaded only if present in `public/audio/`
- Settings: master volume, nature sounds, music, mute
- Fades of at least 3 seconds; master limiter

Done when:
- No clicks, pops, or sudden changes when toggling layers or when weather and
  time of day change
- Listening test at 60× shows smooth transitions (Ben reviews by ear)
- (Tunnel Vision equivalent: M10 — deferred there too; build this one for
  real or defer it the same way, owner's call when the time comes.)

---

## M11. Polish, performance, accessibility

**Goal:** ready to leave open all day.

Scope:
- Performance pass against the budgets in SPEC section 10, including the 30
  fps battery saver
- Reduced-motion audit
- Contrast audit at all times of day — check night separately from day; don't
  assume a daytime-only pass is enough (Tunnel Vision's M11 found a real
  night-only regression this way)
- Phone and tablet layout pass
- Final visual critique pass using the screenshot gallery
- README with a short description, the live link, and how to run locally

Done when:
- A 3-hour accelerated run shows no memory growth and stays within budgets
- Ben signs off on the full experience
- (Tunnel Vision equivalent: M11.)
