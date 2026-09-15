Fixes #221.

Speech engines apply inverse text normalisation to spoken digit runs, so "five eight eight" reaches VoiceAttack as `588` rather than `5 8 8`. The AIRIO and RadioControl handlers read one digit per `{CMDSEGMENT:n}`, which assumes the opposite. The command either fails to match at all, or — once a phrase is added that does match — the whole number lands in segment 1 and the remaining segments silently parse as zero, so `laser code 688` was actioned as `68800`.

These handlers now read their digits with `{TXTNUM:"{CMD}"}`, which returns them in order however the engine segmented them. That is the approach the F-4E WSO handlers already use, in `WSOCommandHandler.GetNumberFromCommand()` — this mostly brings the older handlers up to the same pattern, via a small shared helper so the two do not drift apart again.

## Commits

1. **Digit-run reading** for laser code, datalink tune, TACAN tune and `dev.radio.setfrq`. Also accepts the value spoken in full, stripping the prefix that is fixed in hardware (`1` for laser codes, `3` for datalink frequencies), and replaces the silent zero-fill with a length check.
2. **Per-digit validation of the laser code wheels.** The only guard was `combinedvalue > 788`, which `699` passes even though the second and third wheels stop at 8 — those digits matched no case, queued no action, and the aircraft ended up on a different code than the one reported.
3. **Rejections reported on screen**, not only in the VoiceAttack log. In VR a refused command was indistinguishable from one that silently did nothing.
4. **Same treatment for AN/ARC-182 manual tuning**, and frees its fraction from a fixed segment index. That segment is matched as a string and accepts spelled-out and non-English forms carrying no digits, so it is kept: where it exists its digits are stripped from the tail of the run, and where it does not the fraction is taken from the run instead.

## Behaviour

Additive. A profile that emits one digit per segment produces identical results to today. The changes only add paths that previously failed or failed silently.

## Testing

Built locally and flown in the F-14.

```
Recognized : 'laser code 577' (66)  → Laser Code 577  → AIRIO : Laser code set to 1577
Recognized : 'laser code 699' (89)  → AIRIO : 1699 is not a valid laser code. Range is 1511 to 1788.

'link tune 9 9 decimal 9'  → Datalink Tune 399.90 MHz   (existing phrase, unchanged)
'link tune 3199'    (67)   → Datalink Tune 319.90 MHz   (frequency spoken in full)

'radio frequency 32600' (98) → AN/ARC-182 Tune 326.00 MHz
'radio frequency 03000' (79) → AN/ARC-182 Tune 030.00 MHz
'radio frequency 12325' (66) → AN/ARC-182 Tune 123.250 MHz
```

The ARC-182 lines are worth singling out: that command could not be invoked at all before this change, in any spoken form, paced or collapsed. The engine renders a spoken frequency as a decimal number (`radio frequency 225.0`), which no range-based phrase can generate, and no phrase that avoided the decimal could be written while the fraction was tied to segment 6.

## Profile changes are also required

The code alone does nothing for a user whose phrase cannot match a collapsed digit run — matching is decided by the profile, parsing by the handler, and both halves are needed. Suggested phrases for all four commands are in #221, along with the reasoning for enumerating some of them rather than using numeric ranges.

## Notes

- `{TXTNUM:"{CMD}"}` takes digits from the whole spoken phrase, so a phrase with a digit in its literal text would pick that up. None of the current phrases for these commands do, and the WSO handlers already carry the same assumption, but it is the one behavioural difference from per-segment reads.
- `.github/workflows/dotnet.yml` pins `actions/checkout` to `ref: 'Open-Beta'`, so a PR build compiles the base branch rather than the PR head. Verification here was a local build plus the in-sim testing above rather than CI.
- Based on `master`. Happy to rebase onto `Open-Beta` if that is the preferred target.
