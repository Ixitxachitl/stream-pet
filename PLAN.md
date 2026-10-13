# Stream Pet — plan

An OBS Browser Source overlay with a small voxel creature that lives on your stream. It works like a
Tamagotchi with nobody pressing the buttons. It gets hungry, sleepy, bored and messy, and it deals with all
of that by itself. Chat activity and stream events change its mood and how it grows up. It ages
across streams and evolves according to how its life went.

It follows the sibling overlays (cat-rave, snow-overlay, countdown-timer). It is one `index.html` with no
build step, gets a transparent background automatically, has a settings panel behind a gear (via OBS
Interact), saves to localStorage, accepts every setting as a URL parameter, runs `?demo=1`, has a
`window.pet` API for Streamer.bot, is GPL-3.0, and has a `CNAME` of `pet.ixitxachitl.com`.

**Decided:** the pet uses the whole bottom of the screen, v1 is one generic critter, and evolution is a
surprise.

---

## 1. Tech choices

| Concern   | Choice | Why |
| --------- | ------ | --- |
| Rendering | **three.js** (ES module, pinned version from jsDelivr via `<script type="importmap">`) | WebGL2 with a transparent canvas, shadows and instancing. Works in OBS's CEF. |
| Physics   | **Rapier** (`@dimforge/rapier3d-compat`, WASM inlined in JS, pinned) | Fast and stable, with joints, sleeping bodies and a character controller. No separate .wasm fetch. `cannon-es` is the fallback if WASM is a problem. |
| Voxels    | Models defined in code (palette + layer strings), greedy-meshed at load | No asset pipeline. Easy to recolour per pet. An optional MagicaVoxel `.vox` loader can come later. |
| Camera    | Orthographic, nearly front-on (6° tilt) | **Almost 2D**: it reads like a sprite, but the voxels keep real depth and lighting. Same size anywhere on screen. |
| Shadows   | 2D-style blob shadow under each object | Readable from the front, and much cheaper than shadow maps. |
| Loop      | Fixed 60 Hz physics step with an accumulator, render on rAF, dt clamped | Keeps it stable when OBS throttles the source. |

Render at pixelRatio 1 and cap the frame rate. No outlines: the voxels are drawn plain, with lighting
and AO doing the shaping.
Pause on `visibilitychange` / `obsSourceVisibleChanged`, as in `cat-rave/index.html`. On wake, don't
replay the hidden time as physics. Hand it to the life simulation as "time passed" (see §5).

## 2. Stage / layout

The pet roams the **whole bottom of the screen**. The ground is an invisible floor along the bottom edge
of the canvas, with invisible walls at the left and right edges. There's no visible terrain, so it sits
on top of whatever scene is below it.

- **Ground height**: `ground` (px from the bottom, default 0) raises the floor, for example to sit on
  top of a chat box or a lower third.
- **2D plane**: everything lives on the z = 0 plane. Physics bodies can only move left/right and up/down,
  and only tip over within the screen. Turning is a quick sprite-like flip to a slight 3/4 pose
  (about 24°), not a physical rotation.
- **Edges**: `xmin`/`xmax` (% of width, default 0/100) can pull the walls in to stay clear of a webcam
  or alerts. They are optional.
- **Fixed spots**: the bed, food dispenser and toy box are small voxel props on the floor. The pet
  places them itself, and moves them so they stay inside the walls. The dispenser drops from the top of
  the screen above the food spot.

Recommended source size is the full canvas (1920×1080). Everything above the floor stays transparent
apart from falling props and thought bubbles.

Motion is designed to read as 2D: ears swing in the screen plane, tails wag sideways and peek out when
it turns, and squash & stretch does most of the acting.

## 3. Voxel pipeline

1. **Parts, not whole models**: an adult is put together from parts (body, head, ears/horns, back, tail,
   feet, face) at load time, so hundreds of forms come from a few dozen pieces (see §5a). Each part is
   either:
   - **hand-drawn**: `{ palette, size:[x,y,z], layers:['....', …] }`, one string grid per Y layer. Used
     for small detailed bits such as faces, horns and crowns.
   - **procedural**: a small generator function `(params, rng) → voxel grid`. Used for anything that has
     to scale or bend, such as bodies (ellipsoid, bean, tall, long), tails (curve with taper), wings and
     spikes (rows along a curve).

   Each body defines **sockets** (head, ears, back, tail, feet L/R) with a position, a normal and a
   size budget. Parts attach to sockets, are scaled to the budget and mirrored for symmetry. Each part
   keeps its own pivot so it can animate separately.
2. **Mesher**: greedy meshing per part into one `BufferGeometry` with vertex colours. Per-vertex AO
   uses the classic 3-neighbour voxel AO. Flat shading.
3. **Faces**: eyes and mouth are small swappable voxel sprites (open, closed, happy ^ ^, sad, sleepy —,
   sick @ @). This is the main way it shows emotion.
4. **Variation**: the form sets the parts and the colour family. On top of that, each pet's seed
   shifts the shades and picks the pattern (spots, stripes, socks), so two streamers with the same form
   still have different-looking pets.
5. **Bits**: an `InstancedMesh` pool of single voxels (400 max) for crumbs, sparkles, Zzz and eggshell
   bits. They skip Rapier: in the screen plane they only need to bounce off the floor and walls, so a
   few lines of their own physics is enough and much cheaper.

## 4. Physics

- **World**: gravity, a floor and two side walls. All bodies are locked to the screen plane (no z
  movement, rotation only about z), so it plays like a 2D physics game.
- **Pet body**: one dynamic rigid body (rounded cuboid) for the torso. A PD torque toward upright (about z only) keeps it
  standing, but a strong enough hit (raid confetti, a falling item) can still knock it over. It then
  wobbles back up, which is half the charm. Movement is **hops**: impulses timed to a squash-and-stretch
  animation, not a sliding walk.
- **Secondary motion**: ears, tail and head bob are damped springs driven by the torso's acceleration.
  That's cheap jiggle without joint chains. Each part type has its own motion profile: floppy ears are
  loose, horns are stiff, long tails are a chain of 3–4 springs, wings flap on each hop.
- **The form changes how it moves**: each body type sets the collider shape, mass and hop (a heavy
  form does short, heavy hops and knocks props further; a tiny one does quick bounces). Some parts also
  change the physics: wings mean less gravity while it's in the air (it glides), floaty forms with no
  feet hover above the floor on a spring, and long legs mean higher jumps.
- **Props**: all fully physical, and all made of voxels:
  - **Food** drops from the top of the screen out of an automatic dispenser, bounces, and gets eaten
    chunk by chunk (voxels break off as debris).
  - **Ball / toy**: the pet chases it and kicks it with impulses.
  - **Poop**: spawns behind the pet and gets cleaned up automatically (the "automatic" part).
  - **Bed / nest**: a small floor prop. If something is in the way, the pet curls up where it is.
- Clean up any body that leaves the bounds. Cap the number of live props.

## 5. Life simulation (the Tamagotchi part)

### Needs (0–100, decaying over time)
`hunger`, `energy`, `fun`, `hygiene`, `social`. `health` is derived: it drops while any need sits near
zero and recovers slowly when they don't.

### Brain: utility AI
Every ~0.5 s, score the available actions from the needs, the time of day and recent events, then pick the
best with a little randomness and hysteresis so it doesn't flip-flop:

| Action | Triggered by | What happens on screen |
| ------ | ------------ | ---------------------- |
| Eat | hunger | Dispenser drops food, pet hops over, chomps, crumbs fly |
| Sleep | energy, or the real-world night hours | Goes to bed or curls up, Zzz voxels drift up, eyes closed |
| Play | fun | Spawns or chases the ball, does zoomies or a flip |
| Clean | hygiene | Shakes off (droplet voxels); poop gets swept by a tiny auto-broom |
| Greet | social + a new chatter | Turns to the camera, waves, and a name tag pops up briefly |
| Wander / idle | default | Hops around, sniffs, looks at things, sits |
| Sulk / sick | low health | Slow, sad face, a little thermometer prop; medicine auto-drops |

Everything resolves on its own. Neglect can't happen because the overlay is the caretaker. Instead,
**how well** it was cared for comes from things the streamer doesn't control directly: how active chat was,
how often it got to sleep through the night, how often it played. That's what drives evolution.

### Life stages
Egg → Baby → Child → Teen → Adult → Elder. Growth is measured in **stream hours** (only while the
source is visible) by default, so it grows with the channel. `agerate` scales this.
- Each stage swaps the model and scale, and gets a hatch/evolve cutscene (a burst of voxel shards).
- The **teen** form comes from the pet's strongest trait so far (8 teen forms). The **adult** form is
  one of hundreds (see §5a). The **elder** keeps the adult form, a bit greyer and slower.
- **Evolution is a surprise.** Nothing on stream, in the stats card or in the settings panel shows the
  traits, the form it's heading for, or how many forms exist. The first time anyone sees a form is
  the evolve cutscene, which also shows the form's name. The save stores the form as an opaque encoded
  recipe. `debug=1` is the only way to see the logic, and it's for development only.
- **No death.** An elder eventually "retires": it lays an egg, waves goodbye and leaves. The new egg
  inherits some palette and traits (a lineage), and a small "family tree" is kept in the save.

## 5a. Evolution: hundreds of adult forms

### Traits (how its life went)
Eight running traits build up over the pet's life, from things that happen on their own:

| Trait | Grows from |
| ----- | ---------- |
| Well-fed | needs met quickly, treats from subs |
| Rested | sleeping through its night hours |
| Playful | time spent playing, zoomies |
| Social | chat messages, unique chatters |
| Hyped | emote spam, raids, big events |
| Calm | quiet stretches with little chat |
| Night owl | streams that run late in real time |
| Generous | subs, gifts and bits |

### From traits to a form
1. Take the pet's **top trait** (8), its **second trait** (7) and how strongly the top one leads
   (3 bands: slight, clear, extreme). That's 8 × 7 × 3 = **168 trait combos**.
2. Add the **element**, picked from when and how it lived: what time of day it mostly streamed, the season
   it hatched in, and the kind of events it saw. There are 4 elements per combo from a pool of 8 (fire,
   water, leaf, stone, crystal, shadow, candy, storm), which gives **672 regular adult forms**.
3. Each combo + element maps, through a **fixed global table**, to a recipe:
   - a body type (8: round, chonky, tall, tiny, long, bean, blob, floaty)
   - parts for ears/head (≈12), back (≈10), tail (≈10), feet (≈5) and face (≈8)
   - a colour family from the element

   Because the table is the same for everyone, the same kind of life gives the same form on any
   channel. Viewers who watch several streamers can compare and discover forms together, and they can't
   look any of it up in the overlay.
4. **Rare forms**: about 20–30 hand-designed recipes with unique parts. Each needs a specific unusual
   condition, such as hatching during a raid, a perfect sleep record, or hitting a chatter milestone.
   These are checked before the regular table.

The table is built deterministically from a fixed seed plus rules, so it doesn't need to be typed out:
- **Teen-to-adult link**: the teen form's main part always stays in the adult, so the teen is an
  honest hint.
- **Compatibility rules**, for example: floaty bodies have no feet; long bodies can't take big wings;
  a back part and a tail part may not both be "big".
- **Element-flavoured parts**: fire leans toward spikes and flame tails, water toward fins, and so on.
- **Silhouette check**: each recipe should differ from its neighbours in at least two big parts, so
  forms don't look like near-duplicates.
- **Overrides**: a small hand-written table replaces any recipe that turns out ugly.

**Names** are made from the element and the parts ("Embernib", "Mossgloop", "Fizzhorn"), with a
hand-written list of overrides. Each name is checked to be unique.

**Saves are future-proof**: the save stores the full recipe, not only an index. Changing the generator
later never changes an existing pet. A `FORMS_VERSION` field marks which table produced it.

### Curation tools
- `?gallery=1` renders every adult form in a grid, with names and the trait combo under each, so they
  can be reviewed and fixed in bulk. Hover a form to spin it, click it to see it hop with physics.
- `?form=<id>` spawns a specific form, for testing and screenshots.

### Persistence and time away
- The save lives in localStorage under a key that `?slot=` can change, so several pets can run. It holds
  the seed, stage, age, needs, averages, lineage and timestamps. Autosave every 30 s and on `pagehide`.
- When the overlay loads again (or comes back from hidden), simulate the time it was away in large steps,
  capped. The pet will have slept, eaten from the auto-feeder and be a bit bored. It never comes back in
  a bad state, so a week off stream doesn't punish anyone.
- Export/import the save as JSON in the settings panel. Moving PCs or resetting OBS shouldn't lose a pet.

## 6. Chat and stream events

Copy the connectors from cat-rave unchanged: Twitch anonymous IRC, Kick Pusher, TikFinity bridge and
YouTube. All of them are passive by default:

- **Message**: social +, the pet glances toward a spot on the screen. A first-time chatter gets a greet.
- **Emote-only / hype spam**: excitement, so the pet does zoomies.
- **Sub / resub / gift**: a treat rains down (physical voxel cake or fish).
- **Bits / cheer**: coins drop, which raise fun.
- **Raid**: confetti voxels everywhere and the pet gets knocked over happily. Raid size scales it.
- Optional `cmds=1` for `!pet` (heart burst) and `!pet stats` (the status card for a few seconds). These
  only ever add flavour. The pet never *needs* them.
- `window.pet.feed() / .play() / .treat(kind) / .event(name, data) / .stats()` for Streamer.bot and similar
  tools.

## 7. UI on stream

- A small optional **status card** with the name, stage, age and five need icons, and nothing about
  evolution. It can be always on, shown on events, or off. It sits just above the pet and follows it.
- A thought bubble with a voxel icon when an action starts (🍖 💤 ⚽), in the same comic bubble style as
  the Rave overlays.
- The name comes from the settings, or from a generated cute name.

## 8. Settings panel (same pattern as cat-rave `FIELDS`)

Chat sources · Demo · Ground height · Left/right edges · Pet size · Pet name ·
Age rate · Sleep hours (follow real clock / never) · Status card mode · Commands · Quality ·
SFX volume · Save slot · Export / Import / **Reset pet** (with confirm) · Copy setup link.

## 9. Dev and test helpers

- `?demo=1`: fake chat and events, as in the other overlays.
- `?speed=N`: time warp for needs and ageing, to see a full lifetime in minutes.
- `?debug=1`: physics collider wireframes, a needs/utility HUD, and buttons to force any action
  or stage.
- Headless Chrome screenshots for the stream-tools site, as described in its README.

## 10. Milestones

1. **Scaffold**: `index.html`, README, LICENSE (GPL-3.0), CNAME, transparent three.js scene, ortho
   camera, shadow catcher, fixed-step loop, visibility pause.
2. **Voxels**: part format, procedural part generators, sockets, greedy mesher with AO, egg + baby +
   one hand-built adult recipe, face sprites, palette seeding.
3. **Physics**: Rapier world, upright-controller pet that hops, spring secondary motion, food, ball,
   debris pool.
4. **Brain**: needs, utility AI, all actions in §5 with thought bubbles. *Here it first feels alive, so
   tune it in demo mode with `speed`.*
5. **Life cycle**: stages, traits, persistence, time-away catch-up, retire/egg/lineage.
6. **Form generator**: the full parts library (bodies, ears, backs, tails, feet, faces), compatibility
   rules, the 672-form table, names, the gallery page, then a curation pass over all forms. Rare forms
   come last. *This is the biggest step.*
7. **Chat and events**: port the connectors, event reactions, `window.pet` API, optional commands.
8. **Settings**: full panel, export/import, reset.
9. **Polish**: SFX, the 8 teen forms, performance pass in OBS
   (low-end GPU, browser-source hardware acceleration off), status card.
10. **Site**: `stream-tools/tools/stream-pet.html` guide, icon, home card, nav on every page, screenshots.

## 11. Risks

- **OBS CEF**: importmaps, WebGL2 and WASM are fine on OBS 30+. Test on the oldest OBS you want to support.
  If "Browser source hardware acceleration" is off, WebGL falls back to software. Without shadow maps
  or outlines the scene is light, so it should still keep up; check it.
- **Physics stability**: tipping, tunnelling of small debris, stacking jitter. Mitigate with the upright
  controller, CCD on small fast props, sleeping and caps.
- **Readability at stream size**: voxel pets get small fast. Default to a fairly large size (~160 px
  tall adult) and strong colour contrast between body, belly and accents.
- **Ugly or samey forms**: with hundreds of forms put together by rules, some will look bad or too
  alike. That's what the compatibility rules, the silhouette check, the gallery and the override table
  are for. Plan real time for reviewing the gallery.
- **Single-file size**: the parts library and generator are code, not data, so 672 forms add little
  size. If `index.html` gets unwieldy, split it into `parts.js` / `forms.js` modules, still with no build
  step.
- **Build cost**: building a form takes a few ms at evolve time. Never build all forms at runtime
  outside the gallery.

## The critter

A generic, chunky, round critter about 10×9×8 voxels as an adult. It has a bean-shaped body, two
stubby feet, nubby ears, a short tail and big 2×2 eyes. It isn't any particular animal, so the seed's
palette and pattern give it its personality. Adult forms can change its silhouette a lot (spikes,
wings, fluff, a long tail), but they all keep the big eyes and stubby proportions, so they still read
as the same creature grown up differently.

- **Egg**: 5×6×5, with a crack animation that builds up before hatching.
- **Baby**: 6×5×5. Mostly head, no tail, and it falls over a lot.
- **Child**: one form, with the seed's colours.
- **Teen**: 8 forms, one per top trait. It's the first hint of what's coming.
- **Adult**: 672 regular forms plus 20–30 rare ones (§5a).
- **Elder**: the adult form, a bit greyer, with slower hops.
