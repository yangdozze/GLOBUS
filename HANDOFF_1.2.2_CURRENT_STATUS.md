# GLOBUS 1.2.2 — WIP HANDOFF (play-mode selector), 2026-07-25

Emergency snapshot of in-progress 1.2.2 work. **Nothing here is released.**
`main` still holds the verified GLOBUS 1.2.1 (`ci-latest` release is green).

## Commits

- **Base commit (main, verified 1.2.1):** `c1fd6b57b2bcd8df899b07aa2ab74745fcd2f3c4`
- **WIP commit:** on branch `wip/globus-1.2.2-play-mode-handoff` (this commit — see branch head)

## Changed files (all intentional, nothing else)

| File | Change |
|---|---|
| `Source/UI/TopBar.h` | New `PlayModeSelector` component (juce::Button + juce::ParameterAttachment on the existing `playMode` parameter) replacing the passive `modeLabel`. Left-click / Space / Return cycles POLY→MONO→LEGATO, right-click menu for direct selection, Up/Right & Down/Left arrow stepping, mouse-wheel stepping with 250 ms cooldown, per-mode colours (POLY=theme::accent blue, MONO=theme::warn amber, LEGATO=theme::accent2 violet), hover/pressed/focused/disabled states, tooltip. Timer no longer polls the mode text (attachment is push-based). |
| `Source/Engine/SynthEngine.cpp` | Documentation comment only, above `modeChangeFlush()`: the deterministic mode-change-while-held policy (gentle release of all voices, cleared stacks, no stuck state; held keys re-press in the new mode). **No behavioural change.** |
| `Tests/HeadlessTests.cpp` | New `testPlayModeSelectorAndSync()` (+ helpers in `namespace playmodetest`), wired into `main()` after `testMidiNoteReliability()`. Covers: preset→param→engine→UI sync for Poly/Mono/Legato presets; click cycling; direct selection; keyboard; host automation; state save/restore; preset-after-manual-change; Init/Randomize policy; rapid-preset-switch sync; mode change while held / with sustain (44.1/48/96 kHz × blocks 32/512/4096); mode change with arp latched + CC123 clear. |

## Completed work

1. **Pre-flight verified:** main == 1.2.1, CI run 30113746615 green, `ci-latest` = "GLOBUS 1.2.1" with 4 hash-verified assets. Baseline suite at base commit: **1265 checks, 0 failures**.
2. **Preset-mode diagnosis (Section 3) — COMPLETE, root cause established:**
   - Empirical scan of **171 presets** (71 factory + 100 NIGHT ORBIT) with the real processor AND a live editor under Xvfb: **saved JSON mode == APVTS parameter == engine behaviour (voice-count probe) == top-bar text in 171/171 cases. Zero loading/sync bugs.**
   - Factory data: 57 presets omit `playMode` (→ Poly via the frozen defaults-then-overrides contract, intentional generator design), 10 store Mono, 4 store Legato. Explicit non-poly presets all carry matching glide/priority values (intentional).
   - `randomizePatch` provably never touches `playMode` (protected list); `initPatch` resets to default Poly.
   - **Conclusion:** the "Lead presets appear as Poly" report is not a defect in loading — leads such as Chrome Cutter / Digital Cry / Neon Bite are stored Poly by design, and the old top-bar indicator was a passive always-blue label (mode text only, tooltip pointed at the GLOBAL page). The UX fix is the new functional selector with per-mode colours. **No presets were changed (none justified by evidence).**
   - Scan CSV: delivered in-thread as `modescan_121.csv` (also `/tmp/modescan_121.csv` in the work sandbox).
3. **Selector implemented and integrated** (see file table).
4. **Mode-change policy documented** at `modeChangeFlush()`; engine already implements it (existing `allNotesOff(false)` on mode change — click-free release, stacks cleared).
5. NIGHT ORBIT Vol. 1 untouched throughout (no file in the bank opened for writing).

## Unfinished work (exact next steps, in order)

1. **Re-run the full suite** after the final two keyboard fixes (see Known errors):
   `cmake --build build --target YDCoreTests && xvfb-run -a build/YDCoreTests_artefacts/Release/YDCoreTests`
   Expected: ~1321 checks, 0 failures.
2. Run `--baseline check` (153 legacy renders must stay bit-exact — no DSP was changed, only a comment, so this must pass).
3. Build Linux VST3/Standalone; run pluginval strictness 10.
4. Generate screenshots at 900x600, 1200x760, 1600x1000, 2400x1520 (`YDCoreTests --screenshots <dir> <size>`); verify the selector never clips (its bounds: x=722 w=60 in a 990-min-width top bar).
5. Verify NIGHT ORBIT byte-identical (`sha256sum -c` package + release sums).
6. Version bump to 1.2.2: `CMakeLists.txt` project VERSION, `installer/GLOBUS-Installer.iss` (two places), `Source/UI/Pages.h` footer string, `CHANGELOG.md`, README if versioned.
7. Author the **self-maintaining Windows workflow** (Section 9 of the 1.2.2 prompt): version derived from CMake, no hard-coded version/test count, dynamic previous-release lookup for the upgrade test (fallback: pinned 1.2.1 installer SHA-256 `f0e91bba0a55577f9977b41e1e1245639150bbdb5205232e22f4a38273bc5415`), `gh release upload --clobber` publish (no delete-first), retries + asset verification, versioned `v{VERSION}` releases kept forever, optional signing (action ref `azure/trusted-signing-action@v0.5.1` — beware email-style paste mangling, it broke run 30113344669). Integration token CANNOT push `.github/workflows/**` (403, both trees and contents APIs) — one manual commit will be needed.
8. Push code to main via `github__push_files` (works for non-workflow paths), monitor CI via the public runs/releases API (no Actions API in the MCP server), verify upgrade test 1.2.1→1.2.2, verify `ci-latest` + `v1.2.2` releases and SHA-256 sums.

## Known errors / open items

- **Last verified suite run (with the new tests): 1321 checks, 3 failures** — all three in keyboard activation: JUCE `Button` handles Space via `keyStateChanged`, not `keyPressed`, so the Space case fell through (Up/Down then landed one step off).
- **Two fixes were applied after that run and are INCLUDED in this WIP commit but NOT yet verified by a completed suite run** (the verification rerun was interrupted by this handoff):
  1. `keyPressed` now handles Space/Return explicitly (single deterministic step);
  2. `keyStateChanged` override returns false so a real key press can never double-step (keyPressed + Button's built-in space handling).
- No other known errors. The engine, presets, installer, workflows are untouched by this WIP.

## Compatibility identifiers that must never change

- VST3 manufacturer code `Ydzz`, plugin code `YdC1`
- Bundle ID `com.ninthparallel.globus`
- APVTS state tag `YDCORE`
- Every existing parameter ID (incl. `playMode`) and every existing choice-list order (`Poly, Mono, Legato`)
- Preset format `YDCore-1`, defaults-then-overrides loading
- 51 original factory preset names; legacy `YDCore` preset-directory fallback
- Installer AppId; VST3 install path (`Common Files\VST3`)
- CMake target names (`YDCore`, `YDCoreTests`)
- Calibrated legacy envelope behaviour (153 baseline renders bit-exact)
- The 1.2.1 note-reliability fix semantics (releasing voice = finished phrase; re-gate on fresh press; arp latch cleared by CC120/123)
- NIGHT ORBIT Vol. 1: byte-for-byte frozen

## Test status — honest separation

**Actually run (this WIP session):**
- Baseline at base commit `c1fd6b5`: full suite **1265/0** (Linux, 48 kHz paths as coded).
- Preset-mode scan (external diag tool, real processor + live editor, Xvfb): **171/171 clean**, report CSV produced.
- Full suite WITH the new play-mode tests, BEFORE the two keyboard fixes: **1321 checks, 3 failures** (the keyboard cases described above); all 1.2.1 note-reliability, all preset/state/engine sync assertions green.

**NOT run (must be done next):**
- Full suite after the two keyboard fixes (rerun was interrupted mid-flight — treat as unverified).
- `--baseline check` at this WIP state.
- pluginval, screenshots, Linux plugin builds at this WIP state.
- Any Windows build/test/installer/upgrade verification for 1.2.2.
- Any FL Studio testing (never available in this environment).

## Environment notes for the next agent

- Local clone: `/agent/workspace/globus` (build dir `build/` with fetched JUCE at `build/_deps/juce-src` — excluded from this snapshot; a fresh checkout re-fetches JUCE via CMake and regenerates the 71 loose factory preset JSONs from `Presets/factory_bundle.json` at configure time).
- External diagnostic harness (not part of the repo): `/agent/workspace/hotfix` (`HotfixDiag` tool: `repro`, `matrix`, `scan`, `modescan` commands).
- GitHub MCP: can push any path EXCEPT `.github/workflows/**` (403). No Actions/release-create API; monitor via public REST (`/actions/runs`, `/releases/tags/...`) and let the workflow create releases.
