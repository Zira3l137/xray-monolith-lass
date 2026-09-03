# X-Ray Monolith LASS

Character-rendering fixes and secondary-motion bone physics for S.T.A.L.K.E.R. Anomaly,
built on top of [X-Ray Monolith True Picture-in-Picture](https://github.com/TheLostInPlace/xray-monolith-pip),
which is itself built on the multithreaded branch of
[X-Ray Monolith](https://github.com/themrdemonized/xray-monolith).

> [!IMPORTANT]
> This is a fork of a fork. Every engine change in `xray-monolith-pip` — true
> picture-in-picture scopes included — is present here unmodified; nothing from
> upstream was removed or reverted. This repository adds a separate, independent set
> of changes on top: a first-person character rendering fix, a secondary-motion bone
> physics system, and a corrected alpha-test pipeline. The two feature sets do not
> interact and can be used independently.
>
> As with upstream, the engine, bundled `gamedata`, and any external compatibility
> patches must come from the same revision.

## What this fork adds

Three changes, layered on the full PiP feature set described below. All three are
engine-side; no `gamedata` scripts were added or changed.

### 1. Correct first-person body pose

Previously, the first-person body model (the "legs" visible when looking down) collapsed
its entire torso onto a single point above the pelvis. This was invisible for stock
character meshes because the collapsed geometry sat inside the camera, but any bone
added under `bip01_spine` — for a custom outfit mesh, a physics attachment, jewelry,
anything — would render glued to its parent bone instead of at its proper bind-pose
position.

The fix replaces a recursive bone-tree rebuild that assumed an animated skeleton (this
model has none — it's a bone copy, not an independent animation) with a direct
translation of the already-correct copied pose. The whole first-person torso now poses
correctly, and any additional bones under the spine follow automatically.

Tuning is unchanged: `g_legs_spine_offset_y` and `g_legs_fwd_offset` still work as
before, though the useful range of the former has shifted now that it moves a real torso
instead of a collapsed one.

### 2. Secondary-motion ("jiggle") bone physics

A lightweight damped-spring controller drives per-bone secondary motion — hair,
clothing accents, or, as shipped, a pair of bust bones (`bip01_l_boob` /
`bip01_r_boob`) — entirely in engine code, with no ODE physics shell involved. It runs
as a per-bone callback on the actor's skeleton and, because of the first-person fix
above, the same simulation drives first person, third person, and the shadow pass from
one implementation.

Key properties:

- **Config-driven per outfit.** Bone names, stiffness, damping, and displacement limits
  are read from the equipped outfit's LTX section (with an `[actor]` fallback), so
  different outfits can have different jiggle behavior without a recompile.
- **Live-tunable.** Every parameter is also a console variable
  (`g_jiggle_stiffness`, `g_jiggle_damping`, `g_jiggle_max_offset`,
  `g_jiggle_translate_gain`, `g_jiggle_rotate_gain`, `g_jiggle_bone_length`,
  `g_jiggle_world_gain`, `g_jiggle_axis`, `g_jiggle_enabled`) for live tuning before
  writing final values into an outfit's config.
- **Whole-body motion aware.** A separate world-space acceleration term gives jumps and
  landings a distinct response from steady locomotion.
- **Safe by construction.** Frame-hitch and teleport guards prevent the spring from
  diverging; a bone that already has an engine callback (e.g. the head or spine sway
  callbacks) is left untouched rather than overridden.

### 3. Corrected alpha testing

Two independent bugs meant that, in the deferred renderer, an alpha-tested material's
authored threshold (`Alpha ref`) never reached the GPU, and even a correctly-set value
would have been ignored, because the shipped shaders test against a compile-time
constant rather than the runtime one. In practice this meant every alpha-tested surface
in the game clipped near zero: any texel above near-transparent rendered fully opaque,
turning soft alpha gradients (hair, foliage, fine mesh) into hard, jagged cutouts
regardless of what the source texture or shader config specified.

The fix:

- routes each material's authored `Alpha ref` value all the way to the GPU instead of
  discarding it after shader-variant selection;
- aliases the shaders' compile-time alpha-test constant onto the live per-pass value, so
  every existing alpha-tested shader honors its authored threshold without being
  rewritten or redistributed;
- adds an optional hashed/dithered alpha-test mode (`r__alpha_dither`) that, combined
  with TAA, resolves into a close approximation of true soft transparency while
  remaining in the fully-lit deferred path — no forward-rendering fallback needed;
- preserves the original appearance of existing content by keeping the default
  threshold for world geometry, trees, and detail foliage at the same value the engine
  always effectively used.

This fix builds directly on diagnostic and prototype code contributed by
[TheLostInPlace](https://github.com/TheLostInPlace) — the author of
`xray-monolith-pip` — who identified where the authored alpha ref was being discarded
and provided the working approach used to route it through to the shader and alias it
onto the shipped shaders' compile-time constant. See [Credits](#credits).

## Compatibility

Everything in the [Compatibility](#compatibility-1) section below (inherited from
`xray-monolith-pip`) applies unchanged. Additionally:

- The first-person pose fix and jiggle system only affect the actor's own model; they
  do not touch NPC rendering, weapon HUD models, or the PiP scope camera.
- The alpha-test fix affects every alpha-tested material in the game, not just
  character assets — this includes world geometry, foliage, and any third-party content
  using the deferred alpha-test shader path. Visual changes should generally read as
  corrections toward originally-authored intent rather than regressions, but mods that
  were tuned around the old (effectively-zero) clipping behavior may need their alpha
  ref revisited. See the transparent-materials guide for details.
- None of the three changes touch `gamedata` scripts, LTX schemas, or the typed PiP Lua
  API. A mod built against `xray-monolith-pip`'s modder-integration surface needs no
  changes to run on this fork.

## Installing a matched build

Same process as upstream:

1. Install the required 3DSS and weapon-mod dependencies through MO2.
2. Build either `DX11-AVX | x64` or `DX11 | x64`.
3. Copy the matching executable and PDB from `_build/_game/bin_dbg` into the Anomaly
   `bin` directory.
4. Install this repository's `gamedata` as a separate MO2 mod, or package it while
   preserving its directory layout.
5. Install the matching 3DSS PiP compatibility patch at higher MO2 priority than 3DSS,
   exactly as with upstream — this fork changes no scope-related code.
6. Clear `appdata/shaders_cache` before the first launch. This is required after any
   build of this fork, because the alpha-test fix changes served shader source at
   compile time.

Do not install only the executable. The scripts, configs, and shaders in `gamedata` are
part of the runtime contract.

## Building

Requirements are unchanged from upstream:

- Windows 10 or newer;
- Visual Studio 2022;
- Desktop development with C++;
- current MSVC, Windows SDK, MFC, and ATL components;
- Git submodules initialized recursively.

```powershell
git clone --recursive https://github.com/<your-org>/xray-monolith-lass.git
```

Open `src/engine-vs2022.sln`, select an x64 DX11 configuration, and build the solution.

## Configuration

The Modded Exes options page and in-game ImGui menu are unchanged from upstream; see
[Configuration](#configuration-1) below for the inherited PiP controls.

New console commands from this fork:

| Command | Purpose |
| --- | --- |
| `g_jiggle_enabled 0/1` | Enable or disable secondary-motion bone physics globally |
| `g_jiggle_stiffness <float>` | Spring constant for jiggle bones; higher is snappier |
| `g_jiggle_damping <float>` | Velocity damping for jiggle bones; lower is more wobble |
| `g_jiggle_max_offset <float>` | Hard displacement clamp, in metres |
| `g_jiggle_translate_gain <float>` | Fraction of simulated lag applied as translation |
| `g_jiggle_rotate_gain <float>` | Fraction of simulated lag applied as a swing rotation |
| `g_jiggle_bone_length <float>` | Virtual bone length used to convert lag into an angle |
| `g_jiggle_world_gain <float>` | Response strength to whole-body acceleration (jumps, landings) |
| `g_jiggle_axis 0/1/2` | Which bone-local axis is treated as the bone's length axis |
| `g_legs_spine_offset_y <float>` | Vertical offset of the first-person torso; unchanged key, new behavior |
| `r__alpha_ref_live 0/1` | Route each material's authored alpha ref to the shader; needs shader cache clear |
| `r__alpha_dither 0/1` | Hash the alpha-test threshold per pixel for a soft edge under TAA |

`r__alpha_ref_live` and `r__alpha_dither` change the shader source served to the
compiler and require a shader cache clear plus restart to take effect, same as any
other shader-affecting console variable.

## Modder integration

Everything in [Modder integration](#modder-integration-1) below — DLTX roots, physical
optic profiles, hybrid magnifiers, the typed Lua API — is inherited unchanged from
`xray-monolith-pip` and applies exactly as documented there.

For the two systems this fork adds:

- **Jiggle bones** are configured per outfit via LTX keys on the outfit's section:
  `jiggle_bones` (comma-separated bone names, defaults to `bip01_l_boob, bip01_r_boob`),
  `jiggle_stiffness`, `jiggle_damping`, `jiggle_max_offset`, `jiggle_translate_gain`,
  `jiggle_rotate_gain`, `jiggle_bone_length`, `jiggle_world_gain`, and `jiggle_axis`.
  Every key has a matching `g_jiggle_*` console variable for live tuning before writing
  final values into the config; an `[actor]` section acts as the fallback for any key an
  outfit doesn't specify.
- **Transparent materials** need `Use alpha-channel` on and an authored `Alpha ref` set
  on the material's shader; textures must be DXT5 (not DXT1, which has only one-bit
  alpha and cannot carry a gradient). `r__alpha_dither` further softens the cut edge
  under TAA. Leave `Strict sorting` off unless the material genuinely needs true
  blending — it moves the material out of the deferred, fully-lit path.

---

## Everything below is inherited from `xray-monolith-pip`

The following sections are the original `xray-monolith-pip` documentation, reproduced
here because this fork ships that engine and `gamedata` in full. None of it was written
for or by this fork; consult the [upstream repository](https://github.com/TheLostInPlace/xray-monolith-pip)
as the source of truth for the PiP system itself.

### What true PiP means

Traditional shader scopes magnify or distort the main camera image. This project renders
a second view of the world through the physical objective of the optic.

The current implementation provides:

- one entrance-pupil camera for the scoped world and eligible weapon geometry;
- a synchronized weapon pose across the main view and scope view;
- continuous barrel and handguard geometry through the objective;
- authored or measured objective diameter, eye relief, exit pupil, and magnification;
- physical field-stop, pupil, twilight, tunneling, and near-field behavior;
- off-axis deferred reconstruction from the active scope projection;
- separate scope exposure, light capture, bloom, emissive, grass, and LOD controls;
- a hard TAA exclusion stamp for the lens image and reticle;
- objective-only NVG processing without copying the wearer's mask into the scope;
- hybrid magnifier support with reflex-element capture;
- typed Lua-to-engine optic profiles with legacy LTX and script fallbacks;
- diagnostic overlays and structured runtime logging.

`r__svpscope 0` keeps the original non-PiP path. Any positive value selects objective
true PiP; the retired classic mode is not a separate rendering path.

### Compatibility {#compatibility-1}

The engine is designed to preserve the existing Anomaly and Modded Exes ecosystem.
Legacy weapon sections, upgrades, DLTX patches, 3DSS parameters, Mark Switch data, and
script-driven zoom controllers remain supported.

Tested integration targets include:

- 3D Shader Scopes for GAMMA;
- ARC 3DSS content;
- Modular Attachment System;
- Modular Attachment System for GAMMA;
- Pizza's Ultimate Sight Selection;
- hybrid reflex and magnifier combinations.

MAS, MASG, and Pizza are optional. A regular 3DSS setup does not require them.

Third-party mods and their assets are not included in this repository. Shader
compatibility files derived from 3DSS are distributed separately and should be
installed as their own MO2 patch above the scope mods they extend.

### Configuration {#configuration-1}

The normal Modded Exes options page exposes:

- Picture in Picture;
- smooth variable zoom;
- analog zoom input.

The in-game ImGui menu contains the supported optical, rendering, and performance
controls. Experimental invariants are intentionally fixed internally rather than
exposed as player settings.

Important console commands:

| Command | Purpose |
| --- | --- |
| `r__svpscope 0` | Disable PiP and retain the original rendering path |
| `r__svpscope 2` | Enable objective true PiP |
| `r__svp_diag 1` | Enable throttled pass and activation logging |
| `r__svp_cop_diag 1` | Log optic camera and HUD-pose state |
| `r__svp_cop_diag 2` | Add detailed per-frame alignment data |
| `r__svp_report 1` | Emit a one-shot configuration and file report |
| `r__scope_debug 1` to `4` | Show camera, buffer, mask, and geometry diagnostics |
| `svp_dump_optic` | Dump the resolved typed optic profile and provenance |

### Modder integration {#modder-integration-1}

Existing weapon and scope definitions remain valid. New physical data can be authored
without replacing the legacy fields used by older engines.

#### Choose the smallest integration surface

| Need | Recommended route |
| --- | --- |
| Give a normal weapon-pack optic physical data | Add a DLTX profile and runtime alias |
| Tune eye tracking or zeroing without claiming physical measurements | Add a controller-tuning DLTX section |
| Mark an engaged or flipped hybrid magnifier | Add `svp_hybrid_reflex` to the existing runtime section |
| Inspect the profile selected by the engine | Use `svp_dump_optic` or the read-only Lua inspector |
| Publish an optic assembled dynamically at runtime | Use the typed Lua API |

Most mods should use the data route. The bundled
[`zzz_extra_scope_features.script`](gamedata/scripts/zzz_extra_scope_features.script)
resolves the active weapon and scope, publishes one complete profile, and falls back to
the legacy console route on an older executable. A second publisher should not compete
with it unless the mod deliberately owns the entire optic lifecycle.

#### Keep each value in the correct DLTX root

DLTX selects the root file from the mod-file name. Give every file a unique author or
mod suffix:

| Addon file | Extends | Put these values here |
| --- | --- | --- |
| `mod_system_<author-or-mod>_pip.ltx` | `system.ltx` | Runtime scope, weapon, or upgrade fields such as objective geometry, PiP magnification, and hybrid state |
| `mod_pip_optic_physical_specs_<author-or-mod>.ltx` | `pip_optic_physical_specs.ltx` | Canonical real-world measurements and aliases from runtime sections to those records |
| `mod_pip_optic_profiles_<author-or-mod>.ltx` | `pip_optic_profiles.ltx` | Optional controller response and zeroing values |
| `mod_pip_hybrid_optics_<author-or-mod>.ltx` | `pip_hybrid_optics.ltx` | Fallback hybrid-state registry entries for a standalone compatibility patch |

The section operator is relative to that root:

- use `![existing_section]` to add keys to a section that already exists in that root;
- use `[new_section]` to create a section that does not exist in that root;
- never repeat an existing section as `[existing_section]`; DLTX treats that as an
  unmarked duplicate rather than an override.

For example, an installed scope already exists in `system.ltx`, so its runtime patch
uses `![my_pack_scope]`. Its alias usually does not yet exist in
`pip_optic_physical_specs.ltx`, so the alias uses `[my_pack_scope]` in that separate
root.

Do not ship replacement copies of the root files. A replacement can hide every record
supplied by the engine and other addons. `pip_objective_mm.ltx` is a built-in legacy
lookup table rather than an addon merge target; new physical data belongs in
`pip_optic_physical_specs.ltx`. The `pip_optic_external_aliases.ltx` file is a
schema-checked, single-owner contract for the official MAS/Pizza compatibility package,
not a general extension point.

> The remaining PiP modder-integration topics — patching an existing scope, structuring
> a new scope, adding a physical profile, controller tuning, hybrid magnifiers, and the
> typed Lua API — are unchanged from upstream. See the
> [upstream README](https://github.com/TheLostInPlace/xray-monolith-pip#modder-integration)
> for the complete reference; nothing in this fork alters that surface.

### Reporting problems

For PiP-related issues, follow upstream's process and include:

- the exact weapon and optic section names;
- whether the issue occurs with `r__svpscope 0` or `2`;
- `appdata/logs/pip.log`;
- the current `appdata/logs/xray_*.log`;
- the relevant portion of the MO2 mod list;
- a short video when the problem is motion-dependent.

For issues specific to this fork's changes, additionally include:

- for jiggle-bone issues: the outfit section name, and output of
  `g_jiggle_enabled`/`g_jiggle_stiffness`/etc. if you changed them from defaults;
- for alpha/transparency issues: the shader name assigned to the material, its
  `Alpha ref` and `Strict sorting` values, and whether `r__alpha_ref_live` and
  `r__alpha_dither` are at their defaults;
- for first-person pose issues: whether the outfit adds bones under `bip01_spine`, and
  the current value of `g_legs_spine_offset_y`.

## Credits

- [X-Ray Monolith](https://github.com/themrdemonized/xray-monolith) and its
  contributors;
- [TheLostInPlace](https://github.com/TheLostInPlace), author of
  [`xray-monolith-pip`](https://github.com/TheLostInPlace/xray-monolith-pip), the base
  this fork is built on — and, specifically, for the diagnostic and fix code behind this
  fork's corrected alpha-testing pipeline. The approach used here to route the authored
  alpha ref to the shader and alias it onto the shipped shaders' `def_aref` constant
  originates from code TheLostInPlace provided;
- the original PiP work in the
  [gc64 fork](https://github.com/CnRJay/xray-monolith-gc64) that `xray-monolith-pip` in
  turn builds on;
- the authors and maintainers of 3DSS, ARC, MAS, MASG, and the supported weapon packs;
- the OpenXRay, IX-Ray, Call of Chernobyl, and wider Anomaly modding communities.

## License

This repository inherits the upstream X-Ray engine licensing terms. See
[License.txt](License.txt). The original S.T.A.L.K.E.R. engine code is available for
non-commercial use under its applicable terms.