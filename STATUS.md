STATUS: WORKING - fixing #4806 (SVG text placed by a transform is lost on import); fix + test done, running gates
UPDATED: 2026-10-13T07:29:39Z
BRANCH: contrib/svg-text-transform
MODE: Auto

## Pick: #4806, SVG text elements are not rendered when opening/dropping an SVG

Reproduced on main (f57db2b). Any `<text>` positioned by a `transform` (its own or an ancestor
group's) loses that transform on import. Illustrator writes every text element as
`transform="matrix(1 0 0 1 x y)"`, so such text lands off the canvas (invisible) or at the wrong
place/size. Example on main: `<text transform="translate(10 40)">` imports with no visible pixels;
resvg draws it at (12,23)-(39,39).

Cause: usvg 0.45 builds `Text::flattened()` glyph outlines with identity `abs_transform`s, and
`crates/io/src/svg.rs` converted paths with `abs_transform()`. Fix: compose each node's transform
from its parent chain (identical result outside text), and size raster fallbacks from that.

Test: new `transformed_text_lands_where_resvg_draws_it` in `crates/io/tests/svg.rs` (translate,
matrix, translate + x/y, scale, translated group, nested rotate) compares against resvg's render.
It fails on main and passes with the fix.

Gates so far:
- cargo fmt --check: clean
- cargo clippy -p photocraft-io --all-targets -D warnings: clean
- cargo test -p photocraft-io: 553 passed, 0 failed, 5 ignored (pre-existing, need external files or ~12 GB)
- cargo xtask layers: OK

Still to run: cargo xtask wasm, cargo xtask test-corpus. Then commit, push the branch, and post PR notes here.
No PR will be opened, and nothing was added to contributors/people.toml.
