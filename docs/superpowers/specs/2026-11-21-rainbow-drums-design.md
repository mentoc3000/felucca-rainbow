# DRUMS in Rainbow mode

A child can make a beat on the FM-1 in a few taps. Pick a drum, light up a few keys, and press PLAY. The lights show when each hit is coming, and the child can change the beat as it plays.

## Three buttons, three places

| Tap | Where it goes |
| --- | --- |
| HOME | Normal Rainbow play, with the friend or game the child was using. |
| SAVE | WRITE, with the last song open. |
| ARP | DRUMS, with the last drum pattern open. |

The button for the current mode stays lit. Tapping it again leaves the mode and playback as they are. Switching modes stops playback and saves any edits before opening the next mode. At power-on, start in normal Rainbow play.

DRUMS takes the place of Sparkle throughout Rainbow, including in ACCORDION. HOME takes the child straight back to playing. Surprise Friend is retired. Hold HOME for the grown-up controls, or hold HOME and SAVE together for three seconds to open the full synth. A hold or the two-button shortcut takes priority over the taps.

In WRITE, tap REC to erase the selected column while stopped, or the column that is playing. Hold REC for two seconds to start a new song. A hold performs the whole-song action once.

## Making a beat

Start with an empty pattern. The first three black keys pick and audition the kick, clap, or hi-hat. The screen marks the drum that's selected, and the 16 white keys light up where that drum has a hit. A white key toggles its hit, even during playback. The same step can play all three drums.

```text
        1    2    3    4
             ▼
Kick    ●··· ···· ●··· ····
Clap    ···· ●··· ···· ●···
Hat     ●·●· ●·●· ●·●· ●·●·
```

Use drum pictures for the rows and mark the playing column separately from the hits. Each group of four is one beat. PLAY starts or stops the one-bar loop, and SELECT changes the speed.

Keep one drum pattern and its tempo in Rainbow's settings, separately from WRITE's song and the full synth's projects. Save after a short idle period while stopped and on leaving DRUMS. Old installs open an empty drum pattern and read their existing WRITE song as before.

## Why these buttons

Holding SEQ would put DRUMS alongside the beat and lesson picker and ask the child to learn a hold. A chooser on SAVE would put both editors together and add a step to opening WRITE. Direct buttons give the child one tap between any two modes. ARP makes that possible by giving up Sparkle.

## Build and check

Reuse Felucca's drum playback and the [16-step key mapping](../../../firmware/src/ui_input.c#L87). Draw the three rows with Rainbow's existing screen renderer. Verify space for saved settings and measure the FM-1 build against its flash and RAM limits. The FM-1 is the target for the screen and key layout. Check that the shared firmware still builds for the SMK-37.

In the emulator, check all six mode changes, a repeat tap during playback, live editing on all three rows, REC tap and hold in WRITE, and the grown-up gestures from each mode. Reopen a saved WRITE song from the previous firmware and power-cycle with both a song and a drum pattern. On a real FM-1, check that the playhead and audible hits line up and that the black keys make it easy to tell which drum the white keys control.
