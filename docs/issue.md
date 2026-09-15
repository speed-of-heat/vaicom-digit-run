# [BUG] Numeric AIRIO commands fail when the speech engine returns a digit run as a single number

**Affects:** Laser Code, Link Tune (manual datalink), TACAN Tune, AN/ARC-182 manual tuning, Radio Frequency (`dev.radio.setfrq`)
**Version:** VAICOM PRO Community Edition 3.1.6.1
**VoiceAttack:** 2.2.0
**Source referenced:** `master` @ `3042c825`

## Summary

Commands that take a multi-digit parameter work only intermittently. The cause is not recognition accuracy — it is a mismatch between how VoiceAttack generates command phrases and how speech engines transcribe spoken digit sequences.

The AIRIO and RadioControl handlers read one digit per `{CMDSEGMENT:n}`. VoiceAttack generates bracketed sections as **space-separated** tokens, so `Laser Code [5..7] [1..8] [1..8]` matches only the literal text `Laser Code 5 8 8`.

But speech engines apply inverse text normalisation to digit runs: "five eight eight" is transcribed as `588`, not `5 8 8`. When that happens the phrase does not match; and when a concatenated phrase is added so that it does match, the whole number lands in segment 1 and the remaining segments parse as zero.

Engine-independent — reproduced on the Windows engine with and without VSPX, and on WhisperAttack, which does not use VoiceAttack's grammar at all.

**The F-4E WSO handlers added later already solve this**, in `WSOCommandHandler.GetNumberFromCommand()`. This brings the older handlers up to the same pattern.

## Reproduction

```
When I say:  Laser Code [5..7] [1..8] [1..8]
Action:      Execute external plugin ... context 'airio.dev.laser.code'
```

Saying "laser code six eight eight":

```
Unrecognized : 'Laser Code 577'
Unrecognized : 'Laser Code 588'
Unrecognized : 'Laser Code 688'
Recognized   : 'laser code 5 7 7' (confidence 36)
```

The engine hears correctly; it emits `688` as one token, which matches nothing. The occasional success is the engine happening to emit separate tokens, at low confidence because the 192 near-identical permutations are hard to discriminate.

Adding a concatenated alternative makes it recognise reliably (confidence 91–93) and then produces a wrong result:

```
Recognized : 'laser code 688' (confidence 51)
TX5 | ICS: [ RIO ],[  ],[  ] Laser Code 68800 [  ] [  ]
```

`688` is read into `majval1`; segments 2 and 3 are empty so `Int32.TryParse` leaves them 0; the message becomes `"688" + "0" + "0"`.

## Root cause

`RIO_SetDeviceSequenceLaserCode.cs` (lines 43, 80, 117):

```csharp
Int32.TryParse(State.Proxy.Utility.ParseTokens("{CMDSEGMENT:1}"), out majval1);
Int32.TryParse(State.Proxy.Utility.ParseTokens("{CMDSEGMENT:2}"), out majval2);
Int32.TryParse(State.Proxy.Utility.ParseTokens("{CMDSEGMENT:3}"), out minval);
```

Each segment is assumed to hold exactly one digit. Nothing validates that assumption, and a partial read silently zero-fills rather than failing.

| Command | File | Digit segments |
|---|---|---|
| Laser Code | `Extensions/AIRIO/RIO_SetDeviceSequenceLaserCode.cs` | 1, 2, 3 |
| Link Tune | `Extensions/AIRIO/RIO_SetDeviceSequenceManualDatalink.cs` | 1, 2, 4 |
| TACAN Tune | `Extensions/AIRIO/RIO_SetDeviceSequenceManualTACAN.cs` | 2, 3, 4 |
| AN/ARC-182 tuning | `Extensions/AIRIO/RIO_SetDeviceSequenceManualRadio.cs` | 1, 2, 3, 5 (6 is a string) |
| Radio Frequency | `Extensions/RadioControl/RadioControl_TuneFreq.cs` | 2, 3, 4, 6, 7 |

The F-4E WSO handlers (`wso.*` contexts) are unaffected.

## Two separate problems: matching and parsing

Worth stating plainly, because the fix only addresses one of them and the other has to be handled in the profile:

- **Matching** is VoiceAttack deciding whether what was heard corresponds to a command phrase. Only the profile can change this.
- **Parsing** is VAICOM extracting values from the matched command. Only the code can change this.

A collapsed digit run breaks matching first. Fixing parsing alone changes nothing until the profile offers a phrase the collapsed form can match. **Both halves are required** — see "Profile changes" below.

## The fix

Four commits.

### 1. Read digits with `{TXTNUM:"{CMD}"}`

The pattern the WSO handlers already use — returns the digits of the whole spoken command, in order, however the engine segmented them:

```csharp
public static string CommandDigits()
{
    return State.Proxy.Utility.ParseTokens("{TXTNUM:\"{CMD}\"}") ?? string.Empty;
}
```

`5`,`8`,`8` / `58`,`8` / `588` all yield `588`. Band words (`am`, `fm`, `x-ray`, `yankee`) and the literal `decimal` / `point` contribute no digits and need no special handling — which also means a frequency the engine renders as `319.9` parses identically to `3 1 9 decimal 9`.

Both handlers additionally accept the value spoken in full, stripping the prefix that is fixed in hardware: `1` for laser codes, `3` for datalink frequencies.

The silent zero-fill is replaced by a length check.

### 2. Validate each digit against its own wheel

The laser code's only guard was `combinedvalue > 788`, which `699` passes even though the second and third thumbwheels stop at 8. Those digits match no case in the switches, so no action is queued for them and the aircraft ends up on a different code while the message reports the one requested.

This became reachable once a profile could use a collapsed range such as `[511..788]`, which generates every number in the span rather than only valid codes.

### 3. Report rejections on screen

Input errors previously reached the VoiceAttack log only, which the player cannot see from the cockpit — in VR a refused command was indistinguishable from one that silently did nothing. Rejections are now sent as a message with an empty action sequence, matching how the existing out-of-range branch already reports:

```
AIRIO : 1699 is not a valid laser code.
Range is 1511 to 1788.
```

### 4. Free the ARC-182 fraction from a fixed segment index

`SetRioDeviceSequence_Radio_Tuning` reads segment 6 — the `00`/`25`/`50`/`75` part — as a **string**, matched against many spellings including non-English ones (`"2 5"`, `"twenty five"`, `"dos cinco"`, `"zwo funf"`). Those carry no digits, so a plain digit-run rewrite would have silently broken every one of them.

That switch is kept. The handler now works either way round:

- the phrase has that section, so the digits it contributed are the tail of the run: strip them and let the switch do its work;
- the phrase does not, which is what a profile looks like once the megahertz digits are collapsed into one section: take the fraction from the tail of the run instead.

The second case is the point. While the fraction was tied to segment 6, no collapsed phrase could be written at all — any alternative with fewer bracket sections moved that index and the fraction read nothing. This is the one handler where the profile could not be worked around, which is why it needs the code change to be usable at all.

It also now rejects a fractional value that is not a 25 kHz step. Previously anything else matched no case and queued no keypress for the last two digits, tuning a different frequency than the one reported.

## Profile changes

The code change alone does nothing for a user whose phrase cannot match a collapsed digit run. Suggested shipped phrases:

**Laser code** — three forms: paced, collapsed, and the full real-world code.

```
Laser Code [5..7] [1..8] [1..8];Laser Code 511;Laser Code 512; ... ;Laser Code 1788
```

The 192 codes are enumerated twice (as `xxx` and `1xxx`) rather than expressed as `[511..788]` / `[1511..1788]`, because a range also generates 590, 699, 1780 and other combinations the thumbwheels cannot produce.

**Datalink**

```
Link Tune [0..9] [0..9] decimal [0..9];Link Tune [3000..3999]
```

The first form is the existing phrase, unchanged, so existing users keep working exactly as they do today. The second lets the frequency be spoken with its fixed leading 3. Between them these cover all 1000 wheel combinations, including values below 100 such as 055, which a numeric range cannot express because leading zeros are dropped.

**AN/ARC-182 manual tuning**

One section, digits only, no decimal word. Five digits are megahertz plus
hundredths — `32600` is 326.00, `12325` is 123.25 — and the handler pads a
shorter run, so `123` / `1230` / `12300` all tune 123.000.

```
Radio Frequency [432..695,25];Radio Frequency [900..1599,25];Radio Frequency [108..173];Radio Frequency [225..399];Radio Frequency 03000;Radio Frequency 03025; ... ;Radio Frequency 08775
```

The two multiplier ranges cover 108.00–173.75 and 225.00–399.75 in 25 kHz steps
in 964 phrases rather than 30,000. The VHF-low band is spelled out as 232
literals because a numeric range drops the leading zero and `3000` cannot be
distinguished from 300.0 — the engine does emit `03000` when the digits are
spoken individually, confirmed at confidence 96.

Bands are the radio's own; the gaps between them are not offered, so a frequency
the set cannot tune fails to match rather than being sent to it.

This phrase shape is the general answer to the whole class of problem: a single
section with no decimal word is the only form a normalised transcript reliably
matches. Multi-section numeric phrases lose either to the engine collapsing
digits across a section boundary, or to a spoken decimal arriving as a period,
which no phrase can contain.

A collapsed `99.9` still cannot be matched by any phrase — VoiceAttack cannot generate a string containing a period. Either form above avoids the problem.

## Test evidence

Built against `master` and flown in the F-14.

Working, from the VoiceAttack log:

```
Recognized : 'laser code 577' (confidence 66)
TX5 | ICS: [ RIO ],[  ],[  ] Laser Code 577 [  ] [  ]
AIRIO : Laser code set to 1577

Recognized : 'Laser Code 677' (confidence 98)
Recognized : 'laser code 699' (confidence 89)
Laser code: 1699 is not a valid code (wheels are 5-7, 1-8, 1-8)
```

Datalink, old phrase — unchanged behaviour for existing users:

```
'link tune 9 9 decimal 9' → Datalink Tune 399.90 Mhz
'link tune 2 9 decimal 9' → Datalink Tune 329.90 Mhz
'link tune 1 9 decimal 9' → Datalink Tune 319.90 Mhz
```

Datalink, frequency spoken in full:

```
'Link Tune 3199' (67) → Datalink Tune 319.90 Mhz
'link tune 3209' (70) → Datalink Tune 320.90 Mhz
'link tune 3099' (59) → Datalink Tune 309.90 Mhz
'link tune 3261' (69) → Datalink Tune 326.10 Mhz
```

AN/ARC-182 manual tuning — a command that could not be invoked at all before,
in any spoken form, paced or collapsed:

```
'radio frequency 32600' (98) → AN/ARC-182 Tune 326.00 MHz
'radio frequency 03000' (79) → AN/ARC-182 Tune 030.00 MHz
'radio frequency 12325' (66) → AN/ARC-182 Tune 123.250 MHz
'radio frequency 12300' (96) → AN/ARC-182 Tune 123.00 MHz
```

## Notes

- The change is additive. Profiles that emit one digit per segment produce identical results.
- `{TXTNUM:"{CMD}"}` takes digits from the whole spoken phrase, so a phrase with a digit in its literal text would pick that up. None of the current phrases for these commands do, and the WSO handlers already carry the same assumption, but it is the one behavioural difference from per-segment reads.
- `RadioControl_TuneFreq` reads the modulation band from segment 1 (`am` / `fm`), and the TACAN handler reads its band from segment 1 (`x-ray` / `yankee`). Neither contract is documented outside the source, and a hand-written phrase that omits the band section silently shifts every digit by one position. The length checks added here surface that instead of tuning a wrong frequency.
- CI note: `.github/workflows/dotnet.yml` pins `actions/checkout` to `ref: 'Open-Beta'`, so a PR build compiles the base branch rather than the PR head. Verification here was a local build and in-sim testing.
- Happy to raise this as a PR.
