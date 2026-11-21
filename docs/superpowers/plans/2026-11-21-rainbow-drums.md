# Rainbow DRUMS Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal** — Give kids a one-bar drum editor on ARP and direct buttons for normal Rainbow play, WRITE, and DRUMS.

**Architecture** — Keep Rainbow input and mode changes in `kid.c`. Project the drum pattern onto the existing drum track for playback, use the screen's existing strip renderer, and store the pattern alongside Rainbow preferences.

**Tech stack** — Felucca C firmware, its Node/WebAssembly emulator tests, and both board builds.

**Spec** — [DRUMS in Rainbow mode](../specs/2026-11-21-rainbow-drums-design.md).

## Global Constraints

- FM-1 is the acceptance target. Verify the SMK-37 build with the shared input and button aliases.
- HOME, SAVE, and ARP select normal play, WRITE, and DRUMS. An active button is lit. Tapping it again leaves mode and transport as they are.
- A mode change stops the old transport and saves edits before replacing the tracks. Power-on opens normal Rainbow play.
- HOME held one second opens grown-up controls. HOME + SAVE held three seconds opens the full synth. REC in WRITE clears one column on tap and the song on a two-second hold.
- DRUMS is one bar of 16 sixteenth-note steps with kick, clap, and closed hi-hat. SAVE and the full synth keep their existing storage formats.

## Files

- `firmware/src/kid.c` owns mode transitions, the drum pattern's input and save state, and its screen.
- `firmware/src/kid_smk.c` mirrors shared mode lights and maps the existing sequencer trigger lights onto the pads.
- `web/emu/felucca_web.c` exposes the mode, drum row, and tempo for tests. `web/emu/kid_test.mjs` drives physical buttons, notes, playback, and power cycles.
- `tools/lang/*.py`, `RAINBOW.md`, `SMK37.md`, `landing/index.html`, and `landing/readme/index.html` carry the translated title and current controls.

## Review focus

- HOME + SAVE works with either press order and never selects WRITE as part of the hold. Task 1.
- A held REC never also clears a single column. Task 1.
- A key driving the old mode cannot leak a note or edit into the new mode. Task 2.
- Leaving a loop started after BREAK or CHORDS restores the original play mode and beat. Task 2.
- An older settings record or a save requested during playback does not erase either editor's work. Task 3.

---

### Task 1. Direct buttons and WRITE erase

**Files** — `firmware/src/kid.c`, `firmware/src/kid_smk.c`, `web/emu/felucca_web.c`, `web/emu/kid_test.mjs`.

**Interfaces** — Add `kid_view_t` with `KV_PLAY = 0`, `KV_WRITE = 1`, `KV_DRUMS = 2`. Use `static void kid_switch(kid_view_t next)` as the mode transition entry point. Expose `uint32_t web_kid_view(void)` to the emulator.

- [ ] Add the test export with the existing `kw.on` value until the view is wired in. Add `checkWriteNavigation()` using actual HOME, SAVE, REC, and PLAY presses. Assert the boot view is 0 and HOME is lit. Assert SAVE opens view 1 and a second tap during playback keeps view 1 and `web_playing() === 1`. Assert HOME returns to view 0 with playback stopped and leaves the WRITE notes intact on reopening.
- [ ] Update existing WRITE erase checks from HOME to REC. On a known populated column, assert tap clears only that column while stopped and while playing. On a fresh populated song, assert the note count is unchanged before 2,000 ms of a REC hold, then zero after the threshold and release.
- [ ] Extend the existing HOME hold and exit tests. Assert HOME held one second leaves the WRITE view selected while the volume control works. Test HOME + SAVE from play and WRITE, including a SAVE-first gesture and releasing both after exit. Assert no interim WRITE opening, no extra action on release, and the full synth opens.
- [ ] Build and run the Rainbow emulator test. The new mode and REC checks should fail on the prior behavior.
- [ ] Route SAVE and HOME taps through `kid_switch()`. Resolve HOME and SAVE on release and give their chord precedence, consuming the chord's releases. Finish a transition only after transport stops or a pending start is canceled. Keep the play friend's ID, game, backing beat, sky, and tempo separate from the WRITE editor's changes. Use the existing WRITE save path and restore the play tracks from that state. Derive WRITE's active state from the view. Remove HOME's surprise action. In `kid_smk.c`, retire the redundant surprise shortcut and use the shared view for mode lights.
- [ ] Move WRITE's clear-column operation into the REC short-release path. Reuse `kw_at()` for the playing column and `kw_clear()` for the two-second hold. Suppress a short-release action once the hold fires or if the view changes before release.
- [ ] Rebuild and rerun the Rainbow emulator test. Expect the new cases and existing WRITE, game, volume, and full-synth exit checks to pass. Commit the input and tests as one change.

### Task 2. Play and edit a drum loop

**Files** — `firmware/src/kid.c`, `firmware/src/kid_smk.c`, `web/emu/felucca_web.c`, `web/emu/kid_test.mjs`, `tools/lang/*.py`.

**Interfaces** — Add a drum editor state in `kid.c` with `uint16_t hit[3]`, a tempo, and a selected row. Implement `static void kd_enter(void)`, `static void kd_leave(void)`, `static void kd_keys(uint32_t notes)`, and `static int32_t kd_step(void)` (0–15, or -1 when stopped). Tests use `web_kid_view()`, existing `web_step()`, and new `uint32_t web_kid_drum_row(void)` and `uint32_t web_kid_drum_bpm(void)`.

- [ ] Add `checkDrumNavigation()` for every directed pair among play, WRITE, and DRUMS. Assert one lit mode button and stopped playback after a real transition. Assert an ARP tap during DRUMS playback leaves it running. From DRUMS, exercise HOME hold for volume and HOME + SAVE for full synth. Cover a key held across each direction of the WRITE/DRUMS change.
- [ ] Add `checkDrumGrid()`. From fresh boot, enter DRUMS and assert all 16 steps are empty. Tap F3 (key 0) and G5 (key 26) and assert steps 0 and 15 are set. Press F#3, G#3, and A#3 (keys 1, 3, and 5) and assert the selected rows 0, 1, and 2 and an audible audition for each. Put all three hits on step 0 and assert their actual drum mask is `0x0d`. Toggle one row during PLAY and verify the other two hit bits survive. Assert an added hit is audible when the playhead next crosses that step. Check the selected row and moving playhead are visually distinct and verify the corresponding white key lights. Check SELECT changes tempo by five BPM and clamps to 60–180.
- [ ] Add `checkDrumRestore()`. Enter DRUMS from BREAK and from an active CHORDS lesson. Verify normal play returns to the same beat, lesson selection, friend sound, and tempo. Assert taps in DRUMS do not write into WRITE or sound a pitched keyboard note.
- [ ] Build and run the Rainbow emulator test. Expect these cases to fail while ARP still controls Sparkle.
- [ ] Connect ARP to `kid_switch(KV_DRUMS)` and remove the Rainbow-specific Sparkle flags, hints, and ACCORDION split-track branch. Leave the full synth's arpeggiator reachable through its own UI. Reuse `key_black()` and `key_place()`. In DRUMS, set the keyboard to the existing silent mode (`song.grid = 2`), and audition the three drum notes with `trk_note_on()` under the audio IRQ guard used by `kid_sing()`.
- [ ] In `kd_enter()`, load a fixed drum kit onto track 4 and reset its sequencer to length 16, division 1/16, and straight timing. Use Felucca's `DK_80` kit. Route pattern rows to `DV_KICK`, `DV_CLAP`, and `DV_HATC` (GM 36, 39, 42). Write initial and live changes using `grid_hit()` with the audio IRQ excluded. Start with a 120 BPM default. Route PLAY and SELECT in DRUMS through the existing transport and tempo controls. Stop or release old track voices during transitions, and prevent normal backing tracks, game notes, and WRITE input from running in DRUMS. Restore the next mode's tracks via the existing loading paths.
- [ ] Add `static uint16_t kd_px(int32_t x, int32_t y)` and `static void kd_draw(void)` to the existing display path. Give each row a drum picture and selected outline, show four groups of four, and mark the playhead separately from hits. Redraw the previous and current columns as it moves. Show the selected row's white-key triggers on the FM-1 and use the same mask for the SMK-37 pads. Add DRUMS in the supported language tables and remove the Sparkle entries for literals removed from the source.
- [ ] Rebuild and run the Rainbow emulator test. Expect all modes, drum tests, and the other ACCORDION cases to pass. Inspect stopped and playing screen captures at 240 × 240. Commit the drum editor and tests.

### Task 3. Save the pattern and verify both builds

**Files** — `firmware/src/kid.c`, `web/emu/kid_test.mjs`, `RAINBOW.md`, `SMK37.md`, `landing/index.html`, `landing/readme/index.html`.

**Interfaces** — Implement `static void kd_load(void)` and `static void kd_save(void)` in `kid.c`. Use the existing `settings_save()` and `settings_poll()`.

- [ ] Add `checkDrumPowerCycle()`. Build a baseline WASM from `e010124` in a separate checkout. Pass its path in `RAINBOW_BASELINE_WASM` for the upgrade check. Using its existing button and NOR APIs, write a three-note WRITE song with a late backing beat, then carry its NOR image into the new WASM. Assert the WRITE notes and beat survive and the first drum loop is empty at 120 BPM. In a second case, edit all three rows and change tempo during playback, switch to WRITE and make a different edit, then reboot with that NOR image. Assert both editors reopen with the exact data and start stopped. Exercise the stopped idle-save path separately by editing, waiting four seconds, and rebooting without leaving DRUMS. Also edit and exit through HOME + SAVE, then reboot and assert the pattern survives.
- [ ] Run the new emulator checks against Task 2. Expect the drum pattern round-trip to fail.
- [ ] Reserve `favorites.factory[13][6..13]`: byte 6 format tag `0xd1`, byte 7 BPM, bytes 8–13 the three little-endian hit masks in screen order. Row 13 is SLICE's and only its first byte is currently used by its two presets. Guard the reservation with a compile-time limit of 48 presets where SLICE is enabled. Treat an unknown tag or tempo outside 60–180 as an empty 120 BPM pattern. Keep the existing PER4 settings layout. Mark drum edits dirty, save after four idle seconds with the transport stopped, and queue any remaining write on leaving DRUMS, including the full-synth shortcut. Preserve the other settings bytes on export.
- [ ] Update the control tables and WRITE instructions to match the three mode buttons, the REC gestures, and the drum controls. Use the SMK-37's printed button aliases in its guide.
- [ ] Run `./build.sh` for FM-1 and record the app image, RAM, and pool measurements from the build. Run `sh web/emu/build.sh`, then `node web/emu/kid_test.mjs build/emu/felucca.wasm` and `node web/emu/emu_test.mjs build/emu/felucca.wasm`. Repeat the Rainbow check for the localized WASM targets using the workflow's generation loop. Run `BOARD=smk37 ./build.sh`, rebuild FM-1, and run `sh tests/run_tests.sh`. Expect all commands to exit 0 and both board images to fit their build limits.
- [ ] On a real FM-1, listen to a bar with hits on beats 1 and 3 and add one while it plays. Check the moving marker against the sound, with the white LEDs following the selected drum. Switch between all three modes, then switch off and reopen both editors to check their saved work. Record this separately from the emulator checks.
- [ ] Review the diff against the spec and commit the save path, tests, and control docs.
