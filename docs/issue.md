# [BUG] Numeric AIRIO / radio commands fail when the speech engine returns a digit run as a single number

**Affects:** Laser Code, Link Tune (manual datalink), TACAN Tune, Radio Frequency
**Version:** VAICOM PRO Community Edition 3.1.6.1
**VoiceAttack:** 2.2.0
**Source referenced:** `master` @ `3042c825` (2026-08-21)

## Summary

Commands that take a multi-digit parameter work only intermittently. The cause is not speech recognition accuracy — it is a mismatch between how VoiceAttack generates command phrases and how speech engines transcribe spoken digit sequences.

The AIRIO and RadioControl handlers read their parameters as one `{CMDSEGMENT:n}` per digit. VoiceAttack always generates bracketed sections as **space-separated** tokens, so `Laser Code [5..7] [1..8] [1..8]` matches only the literal text `Laser Code 5 8 8`.

But speech engines apply inverse text normalisation to digit runs: "five eight eight" is transcribed as `588`, not `5 8 8`. When that happens the phrase does not match, and when a concatenated form *is* matched the whole number lands in the first segment and the remaining ones parse as zero.

This is engine-independent. It reproduces on the Windows engine (with and without VSPX) and on WhisperAttack, which does not use VoiceAttack's grammar at all.

**The F-4E WSO handlers added later already solve this** — see `WSOCommandHandler.GetNumberFromCommand()` — so the fix below is mostly a matter of bringing the older handlers up to the same pattern.

## Reproduction

VoiceAttack profile command:

```
When I say:  Laser Code [5..7] [1..8] [1..8]
Action:      Execute external plugin 'VAICOM ...' using context 'airio.dev.laser.code'
```

Say "laser code six eight eight". Typical log across several attempts:

```
Unrecognized : 'Laser Code 577'
Unrecognized : 'Laser Code 588'
Unrecognized : 'Laser Code 688'
Recognized   : 'laser code 5 7 7' (confidence 36)
```

The engine is hearing correctly. It emits `688` as one token, which matches nothing. The occasional success is the engine happening to emit separate tokens, at low confidence because the 192 near-identical permutations are hard to discriminate.

Adding a concatenated literal alternative makes it recognise reliably (confidence 91–93) but produces a wrong result:

```
Recognized : 'laser code 688' (confidence 51)
TX5 | ICS: [ RIO ],[  ],[  ] Laser Code 68800 [  ] [  ]
```

`688` is read into `majval1`; segments 2 and 3 are empty so `Int32.TryParse` leaves them 0; the message becomes `"688" + "0" + "0"`. It then fails the `combinedvalue > 788` range check.

## Root cause

`RIO_SetDeviceSequenceLaserCode.cs` (lines 43, 80, 117):

```csharp
Int32.TryParse(State.Proxy.Utility.ParseTokens("{CMDSEGMENT:1}"), out majval1);
Int32.TryParse(State.Proxy.Utility.ParseTokens("{CMDSEGMENT:2}"), out majval2);
Int32.TryParse(State.Proxy.Utility.ParseTokens("{CMDSEGMENT:3}"), out minval);
```

Each segment is assumed to hold exactly one digit. Nothing validates that assumption, and a partial read silently zero-fills rather than failing.

### Affected handlers and their segment contracts

| Command | File | Digit segments | Notes |
|---|---|---|---|
| Laser Code | `Extensions/AIRIO/RIO_SetDeviceSequenceLaserCode.cs` | 1, 2, 3 | leading `1` of the real code is implicit (line 157) |
| Link Tune | `Extensions/AIRIO/RIO_SetDeviceSequenceManualDatalink.cs` | 1, 2, 4 | segment 3 is the literal "decimal" |
| TACAN Tune | `Extensions/AIRIO/RIO_SetDeviceSequenceManualTACAN.cs` | 2, 3, 4 | segment 1 is the band (`x-ray` / `yankee`); 2+3 form the two-digit major |
| Radio Frequency | `Extensions/RadioControl/RadioControl_TuneFreq.cs` | 2, 3, 4, 6, 7 | context `dev.radio.setfrq`; segment 1 is the band (`am` / `fm`); segment 5 is the decimal word |

The F-4E WSO handlers (`wso.*` contexts) are **not** affected — they already use `GetNumberFromCommand()`.

## Why this cannot be fixed in the VoiceAttack profile

1. **Bracket sections are always space-separated.** VoiceAttack normalises `[5..7][1..8]` identically to `[5..7] [1..8]` — per the help documentation, `[Hello;Greetings]computer` generates `Hello computer`. No phrase syntax maps one spoken token to three segments.
2. **A concatenated literal phrase misparses.** `Laser Code [511..788]` recognises well but puts all three digits in segment 1, producing the `68800` above.
3. **`{CMDSEGMENT}` cannot be injected.** Segments belong to the command VoiceAttack matched. A wrapper command that parses the number and calls a second command by name hands the handler the *second* command's segments, not the parsed values.

## Proposed fix

Use the pattern the WSO handlers already use — `{TXTNUM:"{CMD}"}`, which returns just the digits from the whole spoken phrase and is therefore indifferent to how the engine split them.

`Extensions/AIWSO/WSOCommandHandler.cs:475`:

```csharp
private static string GetNumberFromCommand()
{
    return State.Proxy.Utility.ParseTokens("{TXTNUM:\"{CMD}\"}");
}
```

`"laser code 6 8 8"`, `"laser code 68 8"` and `"laser code 688"` all yield `688`. Band words (`am`, `fm`, `x-ray`, `yankee`) and the literal `decimal` / `point` contribute no digits, so they need no special handling. Where a handler needs the digits positionally, index into the resulting string.

Suggested shared helper alongside the existing one, so the AIRIO and RadioControl handlers can use it too:

```csharp
/// <summary>
/// Digits of the spoken command, independent of how the speech engine
/// segmented them. Speech engines apply inverse text normalisation to
/// spoken digit runs, so "five eight eight" may arrive as three segments
/// ("5","8","8"), two ("58","8") or one ("588").
/// </summary>
public static string CommandDigits()
{
    return State.Proxy.Utility.ParseTokens("{TXTNUM:\"{CMD}\"}") ?? string.Empty;
}

/// <summary>Digit at position i as an int, or -1 if absent.</summary>
public static int DigitAt(string digits, int i)
{
    if (string.IsNullOrEmpty(digits) || i < 0 || i >= digits.Length) return -1;
    return digits[i] - '0';
}
```

### Laser Code

Replace the three `TryParse` calls (lines 42–43, 79–80, 116–117) with one read before the first `switch`:

```csharp
string code = CommandDigits();

// The Tomcat's thumbwheels are fixed at 1xxx, so accept the full real-world
// code ("laser code one six eight eight") as well as the short form.
if (code.Length == 4 && code[0] == '1') code = code.Substring(1);

if (code.Length != 3)
{
    State.currentmessage.dspmsg = "AIRIO : could not read laser code.\n";
    State.currentmessage.msgdur = 5;
    Log.Write("Laser code: expected 3 digits, got '" + code + "'", Colors.Warning);
    UI.Playsound.Recipientna();
    return;
}

int majval1 = DigitAt(code, 0);
int majval2 = DigitAt(code, 1);
int minval  = DigitAt(code, 2);
```

The three existing `switch` blocks and the `combinedvalue > 788` range check are unchanged. This also removes the silent zero-fill: a short or unreadable run now reports instead of sending a wrong code.

### Link Tune

```csharp
string chan = CommandDigits();          // expect 3 digits
if (chan.Length != 3) { /* report and return, as above */ }

int majval1 = DigitAt(chan, 0);
int majval2 = DigitAt(chan, 1);
int minval  = DigitAt(chan, 2);
```

### TACAN Tune

Band stays on segment 1 (it carries no digits, so it is unaffected):

```csharp
string chan = CommandDigits();          // expect 3 digits
if (chan.Length != 3) { /* report and return */ }

int majval = (10 * DigitAt(chan, 0)) + DigitAt(chan, 1);
int minval = DigitAt(chan, 2);
```

### Radio Frequency

Band stays on segment 1:

```csharp
string freq = CommandDigits();          // expect 6 digits: 3 MHz + 3 fractional
if (freq.Length != 6)
{
    Log.Write("Radio frequency: expected 6 digits, got '" + freq + "'", Colors.Warning);
    UI.Playsound.Recipientna();
    return;
}

string combinedfreq = (freq + "000000000").Substring(0, 9);
```

This replaces the five separate segment reads and the existing `combinedfreq` concatenation. Keep the interleaved `SendRadioControlMessage(SendMessage)` calls where they are. It also handles a frequency the engine renders with a real decimal point — `251.750` as one token yields `251750`, because `{TXTNUM}` keeps only the digits.

## Test matrix

All of these should produce laser code 588. Only the first works today.

| Spoken | Likely transcript | Segments 1/2/3 | Before | After |
|---|---|---|---|---|
| "laser code five eight eight" (paced) | `laser code 5 8 8` | `5` / `8` / `8` | 588 | 588 |
| "laser code five eight eight" (natural) | `laser code 588` | `588` / — / — | 68800-style error | 588 |
| "laser code fifty-eight eight" | `laser code 58 8` | `58` / `8` / — | wrong | 588 |
| "laser code one five eight eight" | `laser code 1588` | `1588` / — / — | no match | 588 |

Equivalent cases apply to Link Tune (`12 decimal 5` / `125` / `12.5`), TACAN (`1 2 9` / `12 9` / `129`) and Radio Frequency (`2 5 1 point 7 5 0` / `251.750`).

## Secondary point: two radio-tuning handlers, two contracts

There are two frequency-tuning paths, and they behave differently:

- `wso.radio.tunefreq` → `WSOCommandHandler.RadioTuneFrequency()` — uses `GetNumberFromCommand()`, so it is already immune to how the digits were segmented. No band section in its phrase.
- `dev.radio.setfrq` → `RadioControl_TuneFreq()` — reads segments, and takes the modulation band from segment 1:

```csharp
string band = State.Proxy.Utility.ParseTokens("{CMDSEGMENT:1}").Replace(" ", "");
switch (band) { case "am": ... case "fm": ... default: tunemod = null; }
```

Its contract is therefore:

```
Radio Frequency [AM;FM;] [0..3] [0..9] [0..9] [Point;Decimal] [0..9] [0; 2 5; 5 0; 7 5]
   seg 0            1        2       3       4        5           6           7
```

A hand-written phrase that omits the band section shifts every digit down one position — the first MHz digit is read as the band, and `majval3` ends up holding the word "point". The frequency sent is garbage and nothing reports it. The length validation proposed above turns that from a silent wrong-frequency into a visible error.

Worth documenting the segment contract for `dev.radio.setfrq` and for the TACAN handler (`x-ray` / `yankee` on segment 1) either way, since neither is discoverable outside the source.

## Note on recognition vs parsing

The `{TXTNUM}` fix addresses parsing. Matching is separate: VoiceAttack still has to match the phrase before any handler runs, and a phrase built from `[0..9] [0..9]` sections only matches space-separated tokens.

So for the handlers that already use `GetNumberFromCommand()`, a profile can simply offer a concatenated alternative — e.g. `... Radio [Frequency] [200..399] decimal [0..9] [0; 2 5; 5 0; 7 5]` — and it works end to end, because the handler does not care how the digits arrived. That option is not available for the segment-reading handlers, which is the substance of this report.

## Notes

- The fix is additive. Existing profiles that emit one digit per segment behave identically.
- These segment contracts are not documented in the repo or the manual; they are only discoverable from source. The band segments in Radio Frequency (`am` / `fm`) and TACAN (`x-ray` / `yankee`) are easy to omit when writing a phrase, with the silent misalignment described above.
- Happy to raise this as a PR if the approach looks right.
