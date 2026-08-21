# C1 — Fleet-MIDI Byte-Walk (Verification Kata)

Serves: **Phase 0/Phase 3** (per `06-EDUCATIONAL-PLATFORM.md`) — needs no hardware, works for a Phase 0 trainee anywhere; also usable as a Phase 3 supervised-install companion kata.
Source repo: [`SuperInstance/fleet-midi`](https://github.com/SuperInstance/fleet-midi), branch `production-round3-2026-07-10`.

✅ Everything in this kata — the baseline, the deliberate bug, the failure output — was actually run for this document. Nothing is simulated.

## 1. Clone & Baseline

```
$ git clone https://github.com/SuperInstance/fleet-midi.git
$ cd fleet-midi
$ git checkout production-round3-2026-07-10
$ cargo test
```

Real baseline:

```
test result: ok. 12 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

## 2. MIDI 1.0 Reference: Running Status

MIDI 1.0 status bytes always have bit 7 set (`byte & 0x80 != 0`) — this is how a receiver tells a status byte apart from a data byte (data bytes always have bit 7 clear). **Running status** is a real-world MIDI bandwidth optimization: once a channel-voice status byte (Note On, Note Off, Control Change, etc.) has been sent, a transmitter may omit it on subsequent messages of the same type and just send the data bytes — the receiver "remembers" (runs with) the last status byte until a new one arrives. `fleet-midi`'s `ParserState.last_status` field implements exactly this: the receiver's memory of the most recent status byte.

In `src/lib.rs`, `parse_midi_byte`'s very first check does the bit-7 test that makes this whole scheme work:

```rust
pub fn parse_midi_byte(state: &mut ParserState, byte: u8) -> Option<MidiMessage> {
    if byte & 0x80 != 0 {
        // Status byte.
        ...
```

Every byte that arrives is routed here first: bit 7 set means "this is a new status byte, update `last_status`"; bit 7 clear means "this is a data byte, use whatever status byte we're already running with." Flip that condition and the parser's entire sense of "what kind of byte is this" inverts.

## 3. The Deliberate Bug

**Prediction, before running anything**: flipping `byte & 0x80 != 0` to `byte & 0x80 == 0` on line 56 should make the parser treat every real status byte as if it were a data byte, and every real data byte as if it were a new status byte. Since almost every test constructs messages using real status bytes (`0x80`-`0xEF` range) followed by real data bytes (`0x00`-`0x7F` range), this should break nearly everything — anything that depends on a status byte actually being recognized as one. The `stub_hello_still_works` test doesn't touch the parser at all, so it should be the one survivor (plus a small number of tests whose assertions happen to tolerate the corrupted output).

**Do this yourself:**

```
$ sed -n '56p' src/lib.rs
    if byte & 0x80 != 0 {
$ sed -i '56s/byte \& 0x80 != 0/byte \& 0x80 == 0/' src/lib.rs
$ cargo test
```

**Real result** (4 passed, 8 failed — close to the prediction; check for yourself which 4 survived and think about why):

```
failures:
    tests::control_change
    tests::note_on_and_off
    tests::note_on_velocity_zero_is_note_off
    tests::pitch_bend_14bit_lsb_first
    tests::program_change_single_data_byte
    tests::running_status_note_off_after_status_change
    tests::running_status_note_on
    tests::state_accumulates_across_calls

test result: FAILED. 4 passed; 8 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

One representative failure, `state_accumulates_across_calls`:

```
assertion `left == right` failed
  left: None
 right: Some(NoteOn { channel: 0, note: 60, velocity: 64 })
```

With the condition inverted, a real Note On status byte (`0x90 | channel`) now fails the `byte & 0x80 == 0` check (since a real status byte does *not* have bit 7 clear), so the parser routes it down the data-byte path instead — it gets buffered as "pending data" against whatever `last_status` happens to be (usually `None` at the start of a test), and no message is ever emitted. The parser isn't crashing; it's just silently producing nothing, which is the most dangerous kind of bug — no panic, no error, just missing output. This is the same shape of bug as the real off-by-2 payload-decode bug found in `nexus-edge-runtime`'s wire protocol this round: a parser that "runs" cleanly while quietly discarding real data.

**Trainee exercise**: before reading the explanation above, predict which specific tests will fail and why, then run the experiment yourself and compare.

## 4. Revert

```
$ git diff --stat
```

Confirmed clean — `git diff --stat` on this repo produced no output after reverting, meaning the working tree exactly matches `production-round3-2026-07-10` with no leftover changes. (This is a local sandbox check only; nothing was ever pushed to the real GitHub repo during this kata's preparation.)
