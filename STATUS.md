STATUS: DONE - #4806 fix on contrib/svg-text-transform (1766173), re-authored as ahmed0magdy and force-pushed; no PR opened
UPDATED: 2026-10-13T13:49:11Z
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
   \`ahmed0magdy <110637215+ahmed0magdy@users.noreply.github.com>\`, and the commit was re-authored
   (author and committer) as 1766173. The \`Co-Authored-By: Claude Opus 5.5\` trailer is kept, and there is no
   \`Claude-Session:\` trailer. The file tree is byte-identical to 58a82b1. Re-checked: fmt clean,
   clippy clean, svg tests 17/17 pass. Pushed with \`--force-with-lease\` to contrib/svg-text-transform only.
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
