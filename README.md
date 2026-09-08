# VAICOM collapsed digit runs

Investigation and proposed fix for a class of bug in [VAICOM PRO Community
Edition](https://github.com/Penecruz/VAICOM-Community) affecting every voice
command that takes a multi-digit parameter.

**Environment:** VoiceAttack 2.2.0, VAICOM CE 3.1.6.1, Windows Speech (with and
without VSPX) and WhisperAttack.

## The symptom

Commands like `Laser Code [5..7] [1..8] [1..8]` work perhaps one time in five.
The VoiceAttack log shows the engine hearing correctly but the command not
matching:

```
Unrecognized : 'Laser Code 577'
Unrecognized : 'Laser Code 588'
Unrecognized : 'Laser Code 688'
Recognized   : 'laser code 5 7 7' (confidence 36)
```

Link Tune, TACAN Tune and Radio Frequency behave the same way.

## What it is not

Ruled out in order:

- **Not recognition accuracy.** Settings already matched VAICOM's documented
  spec (Command Weight 85, Recognized Speech Delay 0, Unrecognized 700 for VSPX).
- **Not the speech engine.** WhisperAttack fails identically, and it does not use
  VoiceAttack's grammar, SAPI, or any of those settings.
- **Not phrase formatting.** Varying whitespace between bracket sections changes
  nothing: VoiceAttack normalises `[5..7][1..8]` and `[5..7] [1..8]` to the same
  space-separated tokens.

## What it is

Inverse text normalisation. Every speech engine renders a spoken digit run as a
single number — "five eight eight" becomes `588`, not `5 8 8`. VoiceAttack
generates the phrase as `Laser Code 5 8 8`, so the two never meet.

Proved with a control: a literal command `test code 577` recognises at 81–95
confidence while the bracket-range form logs `Unrecognized` for the same spoken
words.

## Why it can't be fixed in the profile

Adding a concatenated alternative (`Laser Code [511..788]`) makes it recognise
reliably at 91–93 — and then produces a wrong result:

```
Recognized : 'laser code 688' (confidence 51)
TX5 | ICS: [ RIO ],[  ],[  ] Laser Code 68800 [  ] [  ]
```

`RIO_SetDeviceSequenceLaserCode.cs` reads one digit per segment:

```csharp
Int32.TryParse(State.Proxy.Utility.ParseTokens("{CMDSEGMENT:1}"), out majval1);
Int32.TryParse(State.Proxy.Utility.ParseTokens("{CMDSEGMENT:2}"), out majval2);
Int32.TryParse(State.Proxy.Utility.ParseTokens("{CMDSEGMENT:3}"), out minval);
```

`688` lands in segment 1; segments 2 and 3 are empty and parse as 0; the code
becomes `"688" + "0" + "0"`. Nothing reports the partial read.

Segments belong to the command VoiceAttack matched, so they cannot be injected —
a wrapper command that parses the number and calls another command by name hands
the handler the *second* command's segments.

## The fix

VAICOM already solves this elsewhere. `WSOCommandHandler.GetNumberFromCommand()`:

```csharp
return State.Proxy.Utility.ParseTokens("{TXTNUM:\"{CMD}\"}");
```

That returns the digits of the whole spoken command, in order, however the engine
segmented them. The F-4E WSO handlers use it; the AIRIO and RadioControl handlers
predate it.

The patch applies the same pattern to the four affected handlers via a shared
helper, adds a length check in place of the silent zero-fill, and accepts the
full four-digit laser code since the leading `1` is fixed in hardware.

## Affected handlers

| Command | File | Digit segments |
|---|---|---|
| Laser Code | `Extensions/AIRIO/RIO_SetDeviceSequenceLaserCode.cs` | 1, 2, 3 |
| Link Tune | `Extensions/AIRIO/RIO_SetDeviceSequenceManualDatalink.cs` | 1, 2, 4 |
| TACAN Tune | `Extensions/AIRIO/RIO_SetDeviceSequenceManualTACAN.cs` | 2, 3, 4 |
| Radio Frequency | `Extensions/RadioControl/RadioControl_TuneFreq.cs` | 2, 3, 4, 6, 7 |

The F-4E WSO handlers (`wso.*` contexts) are unaffected.

## Contents

- `docs/issue.md` — the bug report as filed upstream
- `docs/pr-notes.md` — apply, build and test instructions, plus PR text
- `patches/` — the change, as a `git am`-ready patch against `master` @ `3042c825`

## Status

- [ ] Issue filed
- [ ] Patch built locally
- [ ] Tested in DCS
- [ ] PR opened
