STATUS: DONE - test-only follow-up to #5334 pushed on contrib/svg-text-test-font (781a842d); the proposed "name Inter first" idea does not work, so a different test-only fix was used; no PR opened
UPDATED: 2026-10-18T05:24:39Z
BRANCH: contrib/svg-text-test-font
MODE: Auto

## Result

One commit, `781a842d`, on `contrib/svg-text-test-font`, branched from `origin/main` at `2d0f3b61`, pushed to origin. It's authored by `ahmed0magdy <110637215+ahmed0magdy@users.noreply.github.com>`, with a `Co-Authored-By: Claude Opus 5.5` trailer and no `Claude-Session` trailer. No PR, issue or comment was opened, and nothing was scheduled.

**The proposed idea is wrong as stated, but the problem is real.** Naming Inter first in `font-family` changes nothing. In a normal build `photocraft_text::CRAFT_FONTS` is **empty**, and Inter is not in it anyway: Inter is in a different list, `photocraft_text::fonts::BUNDLED`, which SVG import does not load. I proved it: with the system fonts hidden, a test SVG naming `"Inter, DejaVu Sans, …"` still imports 0 layers, with the warning "the text needs a font and none is installed".

**What I did instead** (still test-only): I moved the test into the existing unit-test module of `crates/io/src/svg.rs`. There it can use the private `parse_with` hook, which the existing `no_fonts()` test already uses, with a font database holding **only the bundled Inter Regular**. That font is always compiled in. The early `return` is gone, and an empty import now fails the test.

**Scope note:** you asked for changes only in `crates/io/tests/svg.rs`. The fix needs the private hook, so the test moved into `src/svg.rs` (test code only, inside `#[cfg(test)]`) and the integration-test copy was deleted. The alternative, a public API for tests, would be a product change, so I rejected it.

## Answers to questions 1-4

1. **How do `CRAFT_FONTS` get into a build, and can they be empty?** `crates/text/build.rs` embeds the craft fonts only when `CRAFT_FONTS_DIR` is set to a non-empty path **and** the target is not wasm32. Otherwise `CRAFT_FONTS` is an empty list. So it **is empty** in:
   - every default build (locally, and in this VM: I printed `CRAFT_FONTS.len()`, and it is 0);
   - the main CI `test` jobs (`ubuntu-latest` and the Windows legs). In `ci.yml`, only the separate "with craft-fonts" variant sets `CRAFT_FONTS_DIR`, and it runs only the heif, codecs and io crates;
   - **every wasm build**, by design (the 24 MiB size gate).

   The FreeBSD job passes `craft-fonts-ref` to a reusable workflow in `storytold/craft-common-actions`, which probably sets `CRAFT_FONTS_DIR`. That repository is outside this session's access, so I could not read it. **Unverified.**

   The fonts that are always present, wasm included, are `fonts::BUNDLED`: Inter Regular, Medium and SemiBold, and JetBrains Mono, from `assets/fonts`. SVG import's `fonts()` loads `CRAFT_FONTS` plus system fonts, never `BUNDLED`. That is why the new test builds its own database from `INTER_REGULAR`.

2. **Does the reference renderer use the same font database?** In the old test, yes: `svg::rasterize` calls `parse()`, which uses the same `fonts()` database as the import, so both sides picked the same face. In the new test this is guaranteed by construction: the reference is drawn by `resvg::render` from the **same parsed tree** the import converts, from a single `parse_with(…, inter)` call.

3. **Is the current test really vacuous anywhere in CI?** It is vacuous on **any machine with no fonts**. I reproduced that: with system fonts hidden (0 fonts), the merged test reports "1 passed … finished in 0.00s" and checks nothing.
   - **Main CI legs:** most likely **not** vacuous. Windows runners ship Arial. On Ubuntu, DejaVu is normally present (fontconfig, pulled in by `libgtk-3-dev`, depends on a font package). I could not inspect the runner images, so this is likely but not verified.
   - **FreeBSD:** `freebsd.yml` installs no font packages itself (there is no `pkg install` line). All of its setup lives in the reusable workflow I couldn't read. So **I cannot confirm** whether the FreeBSD VM has DejaVu, Arial or Helvetica, or craft fonts with a usable face.

   Honest summary: I could not prove a CI job where the test is vacuous today. The PR's case is that the test depends on whatever fonts the machine has. It silently checks nothing on a fontless machine, as the reviewer said, and after this change it can't.

   A trap worth knowing: **this VM has Inter installed system-wide** (`/usr/share/fonts/opentype/inter`). "Name Inter first" would have *looked* like it worked here and failed on a clean machine.

4. **Has anyone already fixed this?** No. `git log origin/main -- crates/io/tests/svg.rs`: the last commit is `4e5c6a01`, the #5334 merge. `origin/main` is at `2d0f3b61`.

## The older font test: left as it is, on purpose

`text_becomes_outline_shapes_or_warns_without_a_font` (in `crates/io/tests/svg.rs`) goes through the public `import()` with the **real** font lookup, system fonts included. Its either/or *is* the product behaviour: with a font, the text becomes shapes; with none, there's a warning. Both outcomes are real checks, and the "no font" branch is also pinned by the `no_fonts()` unit test.

The transform test was different. Its purpose (transform math) has nothing to do with which fonts are installed, so skipping when there's no font made it check nothing. Moving the older test to the bundled font would stop it testing the real font lookup.

## Gates (exact results)

| Check | Result |
|---|---|
| Probe on unmodified main, system fonts hidden | merged test "passes" in 0.00 s (vacuous); SVG naming Inter first imports 0 layers |
| New test with the #5334 fix **reverted** (fonts visible) | **FAILED**: imported text empty, resvg draws it at (12,23)-(38,39) |
| Same, with system fonts hidden | **FAILED** the same way |
| Fix restored; new test, fonts visible | passed |
| New test with system fonts hidden (0 fonts) | passed; all 6 svg unit tests and all 16 svg integration tests pass |
| `cargo fmt --all -- --check` | clean |
| `cargo clippy --workspace --all-targets --locked --features heif -- -D warnings` (CI's exact invocation) | clean |
| `cargo test -p photocraft-io` | 604 passed, 0 failed, 5 ignored (the same pre-existing ones that need external files or ~12 GB) |
| `cargo xtask layers` | OK, 31 crates, no violations |
| `cargo xtask wasm` | **not run**: only `#[cfg(test)]` code changed, and wasm checks don't compile tests |
| `cargo xtask test-corpus` | **not run**: test-only change, and the corpus downloads are blocked by this VM's network policy (as before) |
| full `cargo test --workspace`, panic_hunt, parity, scorecard, perf | not run / not applicable: one crate's tests changed; no commands, numbers or hot paths |

**How the system fonts were hidden** (as root, nothing on disk changed):
`unshare -m sh -c 'mount -t tmpfs none /usr/share/fonts && mount -t tmpfs none /usr/local/share/fonts && cargo test …'`

That is a private mount namespace in which both font directories are empty in-memory mounts. Inside it, `fc-list` reported 0 fonts; this VM has no `~/.fonts` or `~/.local/share/fonts`. Outside it, all 103 fonts were still there afterwards.

**How the fix was reverted:**
`git show 4e5c6a01 -- crates/io/src/svg.rs > fix5334.patch`, then `git apply -R fix5334.patch`, then re-applied with `git apply fix5334.patch`. Afterwards the diff contained only the test change.

**Housekeeping:** I installed CI's Linux dev packages with apt to build the workspace for clippy. I also deleted `target/` once mid-run because the disk filled up; the test run that failed then was a "No space left on device" build error, not a test failure, and was re-run cleanly.

## Explain it to the owner

**What was wrong.** The test from your merged PR checks that SVG text moved by a `transform` lands in the right place. To draw text, the importer needs a font, and it only looks at fonts installed on the computer plus an optional font pack that normal builds don't include. On a computer with no fonts the import correctly drops the text with a warning, and the test then just said "no font, nothing to check" and passed. So on such a machine it could never catch the bug it was written for. tanish2k09 pointed this out in review.

**Why the suggested fix didn't work.** The idea was "PhotoCraft always bundles Inter, so ask for Inter first". PhotoCraft does bundle Inter, but for its own interface. The SVG importer never loads it, and the optional font pack is empty in normal builds. I checked with fonts hidden: asking for Inter still gave an empty import.

**What changed.**
- The test moved next to the importer's code (`crates/io/src/svg.rs`, test section).
- There it can hand the importer a font list containing only the bundled Inter, which is always in the binary. The existing "no fonts" test already uses the same hook.
- The picture it compares against is drawn from that same parse, so both sides use the same font.
- The old copy (in `crates/io/tests/svg.rs`) is deleted. The "if no font, return" line is gone, so an empty import now fails the test.
- No product code changed.

**How I proved it.**
1. With the old fix removed, the new test **fails**, both with fonts and with all fonts hidden.
2. With the fix back, it **passes**, also with all fonts hidden.
3. On unmodified main with fonts hidden, the old test "passed" in 0.00 seconds, which shows it really was checking nothing there.

**If a reviewer asks:**
- *Why move the test?* The hook that lets a test choose the font database is private to that file; making it public would be a product change for a test's sake.
- *Is CI affected today?* Probably not on Linux or Windows, which have fonts. FreeBSD runs through a shared workflow I couldn't inspect. The point is that the test no longer depends on the machine's fonts.
- *Why keep the other font test as either/or?* That one tests real font lookup, where "no font → warning" is correct behaviour; it also has its own unit test.
- *Possible follow-up (product change, not done here):* SVG import could fall back to the bundled Inter when no font exists, so text is never dropped. It needs maintainer agreement.

## PR title and body (ready to paste)

**Title:** `test(io): SVG text-transform test can't pass without a font (#4806)`

**Body:**

Follow-up to #5334, from tanish2k09's review note there: `transformed_text_lands_where_resvg_draws_it` returned early, and passed without checking anything, when the import found no font.

`CRAFT_FONTS` is empty unless the build sets `CRAFT_FONTS_DIR`, and SVG import doesn't load the bundled UI fonts (`fonts::BUNDLED`). So naming a bundled family in the test doesn't help. With the system fonts hidden, the test passed in 0.00 s.

Changes (test-only):
- Move the test into the `src/svg.rs` unit tests, so it can use `parse_with` with a font database holding only the bundled Inter Regular (always compiled in). The resvg reference is rendered from the same parse, so both sides use the same face.
- Drop the early return: an empty import now fails.
- Remove the old copy from `tests/svg.rs`. `text_becomes_outline_shapes_or_warns_without_a_font` stays either/or on purpose, because it exercises the real font lookup.

Tested:
- Fails with the #5334 fix reverted, with system fonts present and with them hidden (private mount namespace with tmpfs over the font directories, `fc-list` = 0).
- Passes with the fix, including with system fonts hidden.
- `cargo fmt --all -- --check`, `cargo clippy --workspace --all-targets --locked --features heif -- -D warnings`, `cargo test -p photocraft-io` (604 passed, 0 failed, 5 ignored) and `cargo xtask layers` all pass.

Not run: `cargo xtask wasm` (only `#[cfg(test)]` code changed), the corpus tests, and the FreeBSD job. I couldn't check which fonts the FreeBSD VM has, because its setup lives in the shared workflow.

AI-assisted; the commit carries `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>`.
