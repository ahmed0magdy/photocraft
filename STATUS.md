STATUS: DONE - #4806 fix on contrib/svg-text-transform (1766173), re-authored as ahmed0magdy and force-pushed; no PR opened
UPDATED: 2026-10-13T13:52:49Z
BRANCH: contrib/svg-text-transform
MODE: Auto

## Result

Fixed **#4806** (SVG text not rendered when opening or dropping an SVG) on
`contrib/svg-text-transform`, one commit (1766173, previously 58a82b1) on top of main f57db2b. Pushed to origin. No PR opened.
Nothing added to contributors/people.toml.

**Cause:** any SVG `<text>` positioned by a `transform` (its own, or an ancestor group's) lost it on
import. usvg 0.45 leaves the glyph outlines in `Text::flattened()` with identity absolute
transforms, and `crates/io/src/svg.rs` read `abs_transform()`. Illustrator writes every text
element as `transform="matrix(1 0 0 1 x y)"`, so that text landed off the canvas (invisible) or
at its untransformed place and size. On main, `<text transform="translate(10 40)">` imports with
no visible pixels, while resvg draws it at (12,23)-(39,39).

**Fix:** the converter composes each node's transform from its parent chain (the same value as
`abs_transform()` outside text), and sizes raster fallbacks from that transform.

## Gates (exact results)

| Gate | Result |
|---|---|
| `cargo fmt -p photocraft-io -- --check` | clean |
| `cargo clippy -p photocraft-io --all-targets -- -D warnings` | clean |
| `cargo test -p photocraft-io` | 553 passed, 0 failed, 5 ignored (pre-existing: need external Camera Raw / Krita / Paint.NET files or ~12 GB) |
| New test, without the fix | **fails** (imported text has no pixels; resvg draws it at 12,23-39,39) |
| New test, with the fix | passes (svg.rs: 17 passed) |
| `cargo xtask layers` | OK, 29 crates, no violations |
| `cargo xtask wasm` | all ok |
| `cargo xtask test-corpus` | **not run**: this VM's egress policy blocks www.schaik.com (PngSuite) and returns 403 for codeload.github.com (every pinned corpus). Upstream CI runs the corpus job on every PR. |
| parity, panic_hunt, scorecard, perf | not applicable (no commands or numbers changed, no hot path) |

Not done: the `log/devlog.md` entry. `log/` is gitignored, so it can't travel with the branch.

## Things for you to check

1. **Commit author: done.** Per your standing instruction, the repo's git identity is now
   `ahmed0magdy <110637215+ahmed0magdy@users.noreply.github.com>`, and the commit was re-authored
   (author and committer) as 1766173. The `Co-Authored-By: Claude Opus 5.5` trailer is kept, and there is no
   `Claude-Session:` trailer. The file tree is byte-identical to 58a82b1. Re-checked: fmt clean,
   clippy clean, svg tests 17/17 pass. Pushed with `--force-with-lease` to contrib/svg-text-transform only.
2. **Credit:** nothing was added to contributors/people.toml, as you asked.
3. **Scope:** I couldn't read the #4806 issue body. If the reporter's file fails for a different
   reason (for example, no usable font), this fix won't cover it. Worth asking them for a sample.

## PR notes (raw material, rewrite as you like)

The project has no PR template and no rule about who writes the PR text.

Title: `fix(io): SVG text placed by a transform keeps it on import (#4806)`

- **What:** SVG `<text>` positioned by a `transform` (on the text or on an enclosing group) now
  imports where it is drawn. Before, the transform was dropped, so the text was off the canvas
  (nothing visible) or at the wrong place and size.
- **Why:** usvg's `Text::flattened()` glyph outlines carry identity absolute transforms. The
  importer now composes transforms from the parent chain instead of reading `abs_transform()`,
  which gives the same values outside text, and sizes raster fallbacks the same way.
- **Real-world trigger:** Illustrator exports every text element as `transform="matrix(1 0 0 1 x y)"`.
- **Tests:** `transformed_text_lands_where_resvg_draws_it` (crates/io/tests/svg.rs) covers translate,
  matrix, translate + x/y, scale, a translated group and a nested rotate. It compares the imported
  layers with resvg's render of the same file (bounding box within 1 px, at least 70% pixel overlap),
  fails on main and passes here. The full `cargo test -p photocraft-io` passes; clippy and fmt are
  clean; `xtask layers` and `xtask wasm` pass. The corpus tests could not be fetched in my
  environment (network policy); CI runs them.
- **Fixes #4806**
- **AI disclosure:** AI-assisted; the commit carries `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>`.

## Explain it to the owner

### 1. The bug

When PhotoCraft opens an SVG, it turns each text element into vector shapes, one outline per letter. If the text is positioned by a `transform` attribute instead of `x`/`y`, that transform was silently thrown away. The `transform` can sit on the `<text>` itself or on a `<g>` group around it.

Tiny example:

- **Input:** `<svg width="120" height="60"><text transform="translate(10 40)" font-size="24">IHI</text></svg>`
- **Wrong (main):** an empty canvas. With the transform lost, the text sits at y = 0, and since text grows *upward* from its baseline, all of it is above the canvas.
- **Right (this branch):** "IHI" with its baseline at (10, 40), where every browser draws it.

Similar cases on main: `transform="translate(10 0)" x="20"` drew at x = 20 instead of 30, and `transform="scale(2)"` drew the text at half size. Adobe Illustrator exports *every* text element as `transform="matrix(1 0 0 1 x y)"`, so "text from Illustrator SVGs is missing" is the likely real-world symptom behind #4806.

### 2. How the fix works

Background: PhotoCraft does not parse SVG itself. The `usvg` library parses it into a tree of groups, paths, images and text. Every node has a `transform()` (its own, relative to its parent) and an `abs_transform()` (all transforms from the root down, already multiplied together). For text, usvg also gives `Text::flattened()`, a mini-tree of glyph-outline paths.

The usvg quirk: inside `flattened()`, every `abs_transform()` is the identity, meaning "no transform". The outlines are in the text element's *own* coordinate space, and usvg's own source has a `TODO` about this. Our importer read `abs_transform()` on every path, so for glyph outlines it got "no transform" and drew them as if the text had none.

The fix is all in `crates/io/src/svg.rs`, `impl Conv`. The converter already walks the tree top-down and passes each level a `parent` argument (the parent's absolute transform). The fix makes the converter build transforms from that argument instead of trusting `abs_transform()`:

1. `convert_node`, `usvg::Node::Group` branch: a group's absolute transform is now `parent.pre_concat(g.transform())`. Before, it was `g.abs_transform()`. This value is passed down to the children and to `clip_mask`.
2. `convert_node`, `usvg::Node::Text` branch: the glyph outlines are converted with `t.abs_transform()` (which *is* correct on the text node itself) times `flattened().transform()` as their parent. Before, it was `flattened().abs_transform()`, the identity.
3. `shape_layer`: a path has no transform of its own in usvg, so its absolute transform is just `parent`. Before, it was `p.abs_transform()`, the identity inside text.
4. `clip_mask`: now takes the group's computed absolute transform instead of the group, for the same reason.
5. `raster_layer` (used for things that can't stay vectors, such as pattern fills or colour-emoji glyphs) does two things:
   - it computes where the node lands from its local bounding box mapped through `parent`;
   - it still undoes the shift resvg applies using `abs_layer_bounding_box()`, the same value resvg itself uses, so the two cancel exactly.

Why this approach and not the alternatives:

- **Only special-casing the Text branch (pass `t.abs_transform()` down) doesn't work.** Paths ignored `parent` and read `p.abs_transform()`. Making paths use `parent` is the real fix, and then groups have to provide a correct `parent` too.
- **Multiplying a correction onto the stale values** ("text transform × stale abs") breaks for colour-font glyph groups. usvg doesn't fill in their abs transforms at all.
- **Rasterising all text** (letting resvg draw it to pixels) would work, but the text would stop being editable vector shapes, a behaviour change maintainers would likely reject.
- **Patching usvg** means forking a third-party crate; the workspace pins crates.io versions.

Outside text, the new code produces bit-identical numbers to the old code. usvg computes `abs_transform` with exactly the same operation (`parent.abs_transform.pre_concat(transform)` in usvg's `parser/converter.rs`, and a path takes its parent's).

### 3. Why it's correct, and what's not covered

What the tests prove:

- The new test `transformed_text_lands_where_resvg_draws_it` in `crates/io/tests/svg.rs` imports six SVGs: translate, matrix (the Illustrator form), translate plus x/y, scale, a translated parent group, and two nested groups with a 90° rotation.
- For each one, it compares the imported document against **resvg's own rendering of the same file**. resvg is the reference renderer from the same project as usvg, and what PhotoCraft uses to render SVG elsewhere.
- It checks that the text bounding boxes match within 1 px and that at least 70% of the dark pixels overlap. The overlap isn't 100% because two different rasterisers anti-alias thin letter stems differently by about half a pixel.
- The test **fails on main** (the imported text is entirely missing) and passes with the fix.
- All 16 existing SVG tests still pass, including clip-path masks, nested transforms, rasterised fallbacks, gradients and the `.pcraft` round trip. The full io suite passes: 553 passed, 0 failed.

What is NOT covered:

- **Colour-emoji / COLR / bitmap-glyph fonts inside text.** That path goes through `raster_layer`. It should be right by the same reasoning, but no test font of that kind is available, so it's untested.
- **Skew transforms and `<textPath>`.** Not tested.
- **A real Illustrator-exported file.** Not tested; the test uses hand-written equivalents.
- **The real-file corpus tests (`cargo xtask test-corpus`).** Could not run here, because the network policy blocks the downloads. CI runs them.
- **Font availability.** On a machine with no fonts at all, the test skips (the import warns instead). If the reporter's real problem is "no matching font", this fix doesn't address it; I couldn't read the issue body.

### 4. Likely maintainer questions

1. **"Why not just use `abs_transform()` like before?"** usvg leaves it as the identity for the glyph outlines inside `Text::flattened()`, with a TODO in usvg's source. Building it from the parent is the same arithmetic usvg uses everywhere else, so nothing changes outside text.
2. **"Does this change anything for SVGs without text?"** No. A group's absolute transform is computed exactly as usvg computes it, and a path's equals its parent's, so the numbers are bit-identical. All existing SVG tests pass unchanged.
3. **"Why the bbox-plus-70% check instead of exact pixels?"** Imported text is filled by PhotoCraft's shape rasteriser, while the reference is resvg's. Their anti-aliasing differs by about half a pixel at stem edges; a rotated case measured 77% overlap even though placement was exact. The 1 px bounding-box check is what catches a wrong position or scale. On main, the bounding box is empty or far off.
4. **"What about emoji or colour fonts in text?"** They go through `raster_layer`. It now places them from the parent transform and keeps resvg's own shift, so they should land correctly, but there's no test font for it. A test can be added if they have one.
5. **"Should this be fixed in usvg instead?"** Possibly worth reporting upstream too, but PhotoCraft pins usvg 0.45 from crates.io. The importer fix is local, small, and remains correct even if usvg starts filling those values in.

### 5. Risks, and how they were checked

- **Every SVG import now takes transforms from a different source.** Checked by confirming in usvg's source that its computation is the same operation, and by running all existing SVG tests (shapes, groups, nested transforms, clip masks, raster fallbacks, gradients, pcraft round trip). All pass.
- **Raster fallback size:** the pixel layer's box is computed differently. Under rotation or skew it can be slightly larger than before. It is still clipped to the canvas, and empty tiles are pruned, so there's no visual change. Existing tests for rasterised clip/mask/filter/pattern/image layers pass.
- **Crashes:** no `unwrap`, `expect` or indexing was added. New fallible steps use `?` (the function already returns `Option`), matching the never-crash rule.
- **wasm build:** `cargo xtask wasm` passes. Layering (`cargo xtask layers`) passes.
- **Gates:** clippy with `-D warnings` and `cargo fmt --check` are clean.

### 6. Read the diff in this order (~10 minutes)

1. **`crates/io/tests/svg.rs`, the new `transformed_text_lands_where_resvg_draws_it`.** Read the six SVG cases first. They show exactly what was broken.
2. **`crates/io/src/svg.rs`, the doc comment on `convert_children`.** Explains the usvg quirk and the rule: "compose from `parent`, don't read `abs_transform()`".
3. **`svg.rs`, `convert_node` → `usvg::Node::Text` branch.** The core of the bug: the outlines' parent is now the text's real transform.
4. **`svg.rs`, `shape_layer`.** One line: `parent.post_concat(self.global)` replaces `p.abs_transform()...`. This is what makes glyph outlines move.
5. **`svg.rs`, `convert_node` → `usvg::Node::Group` branch and `clip_mask`.** Groups compute `parent.pre_concat(g.transform())`; `clip_mask` takes that value instead of the group.
6. **`svg.rs`, `raster_layer`.** `bb` (where the layer goes) versus `shift` (what resvg subtracts). Check that the `pre_translate(shift…)` line still cancels resvg's internal shift.
