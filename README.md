# Rebuilding RuneScape in a browser engine

*A devlog about building an Old-School-RuneScape-style world inside a browser
three.js game engine: cache → assets → movement → render → interaction → HUD. The
journey, the decisions, and the traps — written from the OSRS research behind the AE1
("Angel's Egg") engine.*

This is a **process** write-up. It contains **no Jagex data** — no cache bytes, no
extracted models, maps, textures or names, no proprietary format dumps. Every tool
described reads **your own legally-owned cache**, exactly the way the archival and
runtime-client communities (OpenRS2, RuneLite) operate. Read their stance before you
point anything at a cache.

---

## Why do this at all

RuneScape's look is deceptively specific: low-poly models lit with flat per-vertex
colour, a fog that eats the horizon, a camera that orbits a tile grid, and an
interaction model — left-click the default action, right-click for "Choose Option" —
that an entire genre quietly copied. Rebuilding it in a modern browser engine is a
great forcing function: you touch asset decoding, a renderer, a movement system, an
interaction layer and a HUD, and each one has exactly one "right" answer that looks
obviously wrong until you find it.

The target was an authentic patch of world — real terrain heights, real object
placement — walked by a full-body character, rendered in the browser, driven by
click-to-move. Here's the path.

## 1. Cache → a headless decode pipeline

The game ships its content in a **cache**: a packed archive of models, map/terrain
regions, object definitions, item definitions, NPC definitions, textures and more,
indexed by id. The official tooling to read it is a desktop client; what you want is a
**headless, scriptable** decoder you can run from Python, in CI, pointed at the cache
**you own**.

The shape that worked:

- `rscache` — open the cache container, enumerate indices/archives, pull a record by id.
- `rsmodel` — decode a model record to vertices/faces/face-colours.
- `rsmap` / `rsregion_export` — decode a map region to terrain heightmap + tile
  attributes + the object instances placed on it.
- `rsobject`, `rsitems`, `rsnpc` — decode the definition tables (what an id *is*).
- `rstextures` — decode the texture atlas entries.
- `rsassemble` / `assemble_npc` — stitch a model + its parts + its definition into one
  engine-ready glTF.

Each stage is a small dependency-light Python module that does one decode and writes an
intermediate (JSON or glTF), so you can inspect every step. **Nothing ships the bytes** —
the tools are the deliverable; the cache stays on your disk, gitignored.

**Trap — the cache format is versioned.** Record layouts differ between revisions. The
NPC definition format in one build we targeted stored its strings `\n`-terminated and
used a specific opcode (`op4`) where an older reference used another; decode it with the
wrong revision's reader and you get plausible-looking garbage. Pin the revision, and
find the maintained reference for *that* revision rather than assuming one layout.

## 2. Assets → glTF, with the axis baked in

Decoded geometry has to become something the engine's loader accepts. glTF is the
natural bridge, and here the first real trap appears.

Our engine's loader grabs a **named mesh** out of the glTF and drops the node-graph
transform on it (a common optimisation — you want the mesh, not the scene). So any
orientation you leave sitting on the node is silently lost, and the model loads lying on
its side. The fix is to **bake the axis conversion into the POSITION accessor** before
export — rotate the geometry itself, apply the transform, then write Y-up — and verify
by reading the accessor's min/max bounds, not by eyeballing the viewport.

> This exact problem got farmed into two standalone tools:
> [`gltf-axis-bake`](https://github.com/iniGlowee/gltf-axis-bake) (Blender recipe +
> bounds inspector) and [`gltf-flatten`](https://github.com/iniGlowee/gltf-flatten)
> (bake node transforms into the vertices with no Blender at all).

**Trap — terrain normals.** The map mesh came in with **inverted normals**: lit from
below, dark on top. Terrain is a single giant mesh you generate yourself from the
heightmap, so the winding/normal convention is on you — get it wrong and the whole world
looks like an overcast basement. Flip at generation time; don't "fix" it with a
double-sided material (that's a band-aid that costs you backface culling forever).

**Trap — textures mip to black.** Cut-out textures (anything with transparent texels)
keyed their transparent pixels to black RGB. Under mipmapping/KTX2 downsampling the
black bleeds into the opaque edge and foliage darkens to mush. The fix is in the *data*:
flood the opaque colour outward into the transparent region before compression
([`alpha-bleed`](https://github.com/iniGlowee/alpha-bleed) is that fix, standalone).

**The "diet cache."** You don't want to re-decode the whole cache on every run, and you
don't want to commit any of it. Keying each decoded intermediate by a **content hash**
gave a local, rebuildable, gitignored cache of just the assets a scene actually
references — fast iteration, zero bytes in git.

## 3. Movement → BFS click-to-move on a tile grid

RuneScape movement is **tile-based**: the world is a grid, each tile is walkable or
blocked (a packed clip/collision map), and clicking a tile walks you there along the
shortest unobstructed path. That's a **breadth-first search** on the grid from the
player's tile to the clicked tile, honouring the clip flags, producing a queue of tiles
the character steps through one per tick.

Getting it to *feel* right is less about the search and more about the presentation: a
click marker where you clicked, the character turning to face each step, and the camera
following smoothly. The movement itself is textbook BFS — the authenticity is in the
constraints (you move on tile centres, you can't cut corners through a blocked diagonal)
and the tick cadence.

> The interaction half of this — the tile mover, the click marker, the "Choose Option"
> menu and the custom cursor — is being packaged as a genre-neutral three.js addon,
> `three-pointclick`.

## 4. Render → the OSRS look is a shader, not a model

What makes it *read* as RuneScape isn't polygon count — it's the shading. Two
ingredients:

- **Flat per-vertex (Gourand) colour in HSL.** Models carry a face colour, not a PBR
  material; lighting is baked per-vertex and interpolated flat across the face. Working
  in **HSL** (not RGB) for the shading maths is what gives that characteristic banded,
  slightly-saturated look when light falls off.
- **Distance fog.** A fog that fades geometry into the sky colour at a fairly short
  distance. It's half the aesthetic and, conveniently, half the performance strategy —
  you never draw what the fog would hide.

Chasing a photoreal material here is the wrong instinct; the "cheap" shading *is* the
art direction. (This HSL-Gouraud + fog "OSRS-look" is itself a candidate engine-agnostic
write-up.)

## 5. Interaction + HUD → left default, right "Choose Option"

The interaction model is a **tag graph** over the scene: every interactable node carries
its actions ("Walk here", "Examine", object/NPC-specific verbs). Hovering shows the
default action as **top-left text**; left-click performs it; right-click opens the
**"Choose Option"** context menu listing every action on whatever is under the cursor,
sorted by priority. A custom **canvas-drawn cursor** and a **click marker** finish the
feel.

The HUD sits in a DOM overlay *above* the 3D canvas rather than being drawn into it —
keeping UI in the DOM means you get text rendering, fonts and layout for free and never
fight the GL context for a menu. (The dialogue/overlay approach is its own engine
pattern.)

## 6. Verify like you mean it

The milestone was **"a full-body character standing on authentic terrain, click-moving
around real object placement, in the browser."** You don't get to claim that from a
screenshot of the character in a void. The checks that mattered:

- **Bounds, not vibes** — read accessor min/max to confirm the axis bake, every export.
- **Flat-colour pass** — strip textures to see pure geometry/orientation (caught the
  inverted terrain normals).
- **Walk the grid** — click every edge case (blocked tile, diagonal squeeze, long path)
  and watch the BFS honour the clip map.
- **Honest status** — "terrain + objects + one NPC on authentic tiles, click-move
  working; items/textures partial" is a status; "RuneScape, done" is not.

## IP discipline — the whole reason this can be shared

Everything in a game cache is Jagex's copyrighted content. The rule that keeps a project
like this shareable:

- **Ship code and knowledge, distribute no assets.** The decoders are yours to release;
  the cache, the decoded models/maps/textures and any names stay out of version control
  and out of releases.
- **The user brings their own cache.** Tools point at files the user legally owns — the
  same line OpenRS2 and RuneLite draw. Read how the established community tools handle
  this and align with it; review the rights-holder's stance before you publish anything
  that touches the cache.
- **No trademarks implied.** This is an independent research project; it is not
  affiliated with or endorsed by the rights-holders of the game it studies.

---

## The pipeline in one breath

Headless Python cache decoders (pinned to the cache revision) → glTF with the axis baked
into the vertices and alpha-bled cut-outs → a content-hashed local diet-cache (gitignored)
→ an HSL-Gouraud + distance-fog renderer → BFS click-to-move on the clip grid → a tag-graph
interaction layer with a DOM HUD → verified by bounds, flat-colour and walking the grid.
No desktop client, no committed bytes, every step scriptable.

---

*From the AE1 ("Angel's Egg") browser three.js engine project's OSRS research. Method
and journey only — no game assets, cache data, or proprietary formats included. Released
under [MIT](LICENSE) (covers the prose and any snippets). RuneScape and Old School
RuneScape are trademarks of their respective owners; this project is independent,
unaffiliated and not endorsed. OpenRS2 and RuneLite are credited as references for how
the community draws the IP line.*
