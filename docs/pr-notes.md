# VAICOM PR — how to apply and what to say

## Applying the patch

The patch is based on **`master`** at `3042c825`. The four handlers differ between
`master` and `Open-Beta`, so it will not apply cleanly to `Open-Beta` — ask which
branch they want before you push, and I can rebase it.

```bash
# fork Penecruz/VAICOM-Community on GitHub first, then:
git clone https://github.com/<you>/VAICOM-Community.git
cd VAICOM-Community
git checkout -b fix/collapsed-digit-runs master
git am /path/to/0001-Accept-collapsed-digit-runs-in-AIRIO-and-radio-numer.patch
```

If `git am` complains, `git apply --3way` on the same file is the fallback.

## Building

Visual Studio on Windows, .NET Framework 4.7.2 developer pack, `VAICOM28.sln`.
NuGet restores Fody, SharpDX, NAudio and friends. Build the `VAICOM` project.

Note the pre-build event kills `VoiceAttack.exe`, so close it first or expect it
to be closed for you.

## What to test in the Tomcat

Each command in both delivery styles — paced ("five eight eight") and natural
("fivehundredeightyeight"), with the profile phrase unchanged:

| Command | Say | Expect |
|---|---|---|
| Laser code | "laser code five eight eight" | code 1588 |
| Laser code | "laser code one five eight eight" | code 1588 (full form now accepted) |
| Laser code | "laser code nine nine nine" | logs "expected 3 digits" / out of range, no action |
| Link tune | "link tune one two decimal five" | channel 12.5 |
| TACAN | "tacan one two nine x-ray" | channel 129X |
| Radio frequency | non-WSO aircraft, "radio frequency two five one point seven five zero" | 251.750 MHz |

The last one uses `dev.radio.setfrq`; the F-4E WSO command (`wso.radio.tunefreq`)
goes through a different handler and is untouched by this change.

## Suggested PR title

```
Accept collapsed digit runs in AIRIO and radio numeric commands
```

## Suggested PR description

> Fixes #<issue number>.
>
> Speech engines apply inverse text normalisation to spoken digit runs, so
> "five eight eight" reaches VoiceAttack as `588` rather than `5 8 8`. The laser
> code, datalink tune, TACAN tune and radio frequency handlers read one digit per
> `{CMDSEGMENT:n}`, which assumes the opposite. When the run collapses the command
> either fails to match, or — if a concatenated phrase is used to make it match —
> the whole number lands in segment 1 and the remaining segments silently parse as
> zero, so `laser code 688` is actioned as `68800`.
>
> This reads the digits with `{TXTNUM:"{CMD}"}` instead, which returns them in
> order regardless of how the engine segmented them. It is the approach the F-4E
> WSO handlers already use, in `WSOCommandHandler.GetNumberFromCommand()` — this
> change just brings the older handlers up to the same pattern, via a small shared
> helper so the two do not drift again.
>
> **Behaviour:**
> - Profiles that emit one digit per segment produce identical results — the change
>   is additive.
> - The silent zero-fill is replaced by a length check, so an unreadable command
>   reports instead of actioning a wrong value.
> - The full four-digit laser code is now accepted as well as the short form, since
>   the leading 1 is fixed in hardware.
>
> **Not built by CI:** `.github/workflows/dotnet.yml` pins `actions/checkout` to
> `ref: 'Open-Beta'`, so a PR build compiles the base branch rather than the PR
> head. I've built and tested this locally against the Tomcat — test matrix in the
> issue.
>
> Based on `master`; happy to rebase onto `Open-Beta` if that's the preferred
> target.

## One judgement call worth flagging to them

`{TXTNUM:"{CMD}"}` takes digits from the *whole* spoken phrase, so a phrase with a
digit in its static text (e.g. "Radio 2 Frequency ...") would pick that up. None of
the current phrases for these four commands do, and the WSO handlers already carry
the same assumption, but it is the one behavioural difference from per-segment
reads and worth stating rather than letting a reviewer find it.
