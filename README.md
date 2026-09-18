# ami-bios-settings

Decode the settings out of an **AMI Aptio** UEFI firmware image, and diff two of them.

Two units of the same model should have the same BIOS configuration. Proving that is
harder than it sounds: there is no supported way to ask a BIOS image "what are your
settings and what are they called". This reads the answer out of the firmware itself.

```
bios-settings-decode  FIRMWARE.bin --model TB-7393 --serial A1B2C3   # .bin  -> named settings
bios-settings-compare A.txt B.txt                                    # two captures -> JSON diff
bios-settings-diff    capture.txt                                    # capture vs a golden baseline
```

## What it actually does

`bios-settings-decode` takes a raw flash dump and:

1. runs `uefiextract` to explode the image into its modules,
2. finds the BIOS's own compiled Setup form driver (AMI names the PE32 module literally
   `Setup`),
3. decompiles its IFR forms with `ifrextractor` to recover, for every question, the
   prompt text, the named NVRAM store that holds it, the byte offset within that store,
   and the option value to label map,
4. reads that byte straight out of the correctly split store body,
5. and falls back to an opaque per-store CRC32 only for questions it genuinely cannot
   resolve.

On a real image that produced **2,887 of 4,808 questions resolved to named values** on the
first pass, and after the `CheckBox`/`Numeric` handling described below, only 103 of 4,660
entries had neither a name nor help text.

## Try it

The repo ships a synthetic pair of captures from a fictional board. Build B is build A
with one setting inserted early, which is exactly the situation that breaks naive matching:

```bash
cd samples
python3 ../bios-settings-compare DEMO-BOARD__buildA*.txt DEMO-BOARD__buildB*.txt --identity
#   mode identity  matched 13  differing 12     <- noise
python3 ../bios-settings-compare DEMO-BOARD__buildA*.txt DEMO-BOARD__buildB*.txt
#   mode sequence  matched 13  differing 3      <- the two real changes, plus the new setting
```

## The interesting problem

A setting's identity looks obvious: AMI gives each question a `Token` of
`QuestionId:store:offset`. It is not obvious at all, because **recompiling the BIOS
reassigns both the QuestionId and the offset**. Insert one setting anywhere and everything
after it moves. On a real pair of builds of the same board, token overlap was **9%**, and a
token-keyed diff reported 1,468 "differences" that were almost entirely the same settings
failing to line up.

Worse, the identifiers are actively misleading. On one real pair, QuestionId `0x89` meant
`SAR 2400 MHz Set1 Chain A`, a WiFi power calibration number, in one build and the
completely unrelated `11Ax Mode for Russia`, a regulatory toggle, in the other. The
compiler had simply reused the number. Matching on ID alone silently spliced two unrelated
settings into one nonsense row.

So the tool does three things:

**Build-order alignment (the default for two files).** The same technique `diff` uses on
text: `difflib.SequenceMatcher` over each capture's settings **in their original compiled
order**, keyed on the prompt text rather than the token. An insertion reads as an
insertion, and everything after it realigns. On the real pair this moved matching from
3,969/5,257 (75%) to 4,607/4,619 (99.7%), and the differing count from 1,468 to **30**.

**Reconciliation passes, most precise signal first**, for genuine reordering that
sequence alignment cannot see by construction (a setting moved to another menu page
breaks the relative-order assumption LCS depends on). Same QuestionId + store, then same
store + offset, then name only. Every pass requires the prompt text to agree as well,
because of the `0x89` case above, and requires the match to be **unambiguous**: each file
may contribute at most one candidate. Plenty of prompts legitimately repeat (one real
capture has `I2C Address` 168 times, one per device) and blind merging would combine
unrelated settings. The shipped sample keeps its three `I2C Address` rows correctly
unmerged.

**Within-file consolidation**, which is safe where the cross-file version is not. A single
build can expose the identical physical byte through two internal QuestionIds. Two entries
sharing a store and offset **within one capture** are deterministically the same byte, so
they collapse unconditionally. Left alone, that compounds with a cross-build offset shift
into a two-candidates-per-file situation the ambiguity guard correctly refuses to touch,
which is what produced four disconnected rows for one setting.

## Things that were wrong and are worth knowing

- **Matching an NVRAM store by name alone can hit the driver module's own PE32 folder** if
  it happens to share that name. The `Setup` driver's 1 MB of machine code was read as if
  it were the 3,352-byte `Setup` NVRAM store, producing confident garbage. A match now
  requires `NVAR store` as an ancestor path component.
- **`ifrextractor` output embeds literal newlines inside some help strings.** Collapsing
  those with a whole-document "match every quoted span" regex is unsafe: one unescaped
  quote anywhere desynchronizes quote pairing for everything after it and silently
  corrupts far later, unrelated lines. It undercounted questions 676 vs ~4,800 and looked
  fine until the count was checked against the raw line total.
- **`CheckBox` questions have no option child lines at all**, so they were falling back to
  raw hex. They are always boolean per the IFR spec. `Numeric` questions have no discrete
  options by design, but the BIOS's own min/max/step and help text were parsed and then
  discarded. Fixing both dropped unlabeled entries from 982 to 42.
- **A blank prompt is not always a decode failure.** Some questions have a genuinely empty
  compiled prompt (`StringId 0x0`). They render as `(no label in BIOS form)` so they read
  as "the BIOS did not name this" rather than "the tool broke".
- **A value-coincidence dedup pass was tried and reverted.** Three genuinely distinct
  `Stop Bits` settings (COM1, COM2, and an unrelated debug entry) all happened to read `1`
  on both builds, and collapsing on "same prompt + same value" merged all three.
- **A factory BIOS update file carries no live state**, only the manufacturer defaults
  template. Only a dump from a booted unit has real per-unit values, and those rows are
  tagged accordingly. Memory timings and turbo ratio tables reading 0 on one side are
  MRC-populated fields, not configuration: read a 0-vs-value row as "not populated in this
  dump", never as a setting.
- **There is no BIOS version string to find.** A vendor's release code is not embedded as
  text anywhere in the image, checked with `strings` in both 8-bit and UTF-16LE against the
  raw file and the fully decompressed tree. Label your captures yourself.

## Requirements

- Python 3.8+, standard library only
- [`uefiextract`](https://github.com/LongSoft/UEFITool) and
  [`ifrextractor`](https://github.com/LongSoft/IFRExtractor-RS) on `PATH`
- A firmware image. Getting one off a running machine is out of scope here;
  `flashrom -p internal -r dump.bin` is the usual route on Linux.

Set `BIOS_SETTINGS_HOME` to choose where the `captures/` and `golden/` trees live
(default `~/.local/share/bios-settings`).

## Scope

This is the decode and diff engine only. The capture front end it was built against uses
AMI's own SCEWIN utility, which is not redistributable and is not included; the decoder
path here deliberately depends only on open-source tooling. A web UI and a Word exporter
also exist but are tied to one organization's branding and are not part of this repo.

## License

MIT. See `LICENSE`.
