# Adding Custom Text / Captions to VHS Demo GIFs

**Date:** 2026-07-08
**Status:** Research + recommendation
**Scope:** How to overlay custom text (captions / lower-thirds / title cards) onto the VHS-generated demo
GIFs and MP4s under `demos/out/` at chosen points in the timeline.

---

## 1. Goal

The `demos/` pipeline renders scripted terminal walkthroughs of the ACE TUI into GIF + MP4 via
[Charmbracelet VHS](https://github.com/charmbracelet/vhs). The current biggest gap is the **inability to add
custom explanatory text to the output at specific moments** — e.g. a lower-third caption reading
"Recall prior prompts instantly" while the prompt-history feature is being demonstrated. This document surveys the
options, proves out the most promising one against a real demo artifact, and ends with a recommended solution.

---

## 2. Current state of the sase demo pipeline

Grounded in the repo as of this research:

- Tapes live in `demos/tapes/*.tape` (5 tapes today: `sase_ace_prompt_input`, `sase_ace_agents_observability`,
  `sase_ace_prompt_history_stash`, `sase_ace_prs_pipeline`, `sase_ace_multi_model_fanout`).
- `just demos` runs `vhs demos/tapes/<name>.tape` for each, writing **both** `demos/out/<name>.gif` **and**
  `demos/out/<name>.mp4`, then stamps `demos/out/last_generated_date.txt` and offers to commit.
- Each tape sets a fixed geometry (`Set Width 1920`, `Set Height 1080`), theme (`GitHub Dark`), font
  (`Fira Code`, size 20), and `Set Framerate 30`, and is heavily **event-driven**: it uses `Wait+Screen /regex/`
  to synchronize on rendered content rather than fixed sleeps. Setup/teardown is wrapped in `Hide`/`Show`.
- `demos/README.md` already anticipates post-processing: *"Future compression work can add a small
  `ffmpeg`/`gifsicle` post-processing recipe."* — so a post-processing stage is an accepted extension point.
- Toolchain present on this host (verified): **VHS v0.11.0**, **ffmpeg 7.1.5** (built with `libfreetype`,
  `libass`, `libfontconfig`, `librsvg`), **ImageMagick 7** (`magick`/`convert`). `gifsicle` is **not** installed.
- ffmpeg exposes the relevant filters: `drawtext`, `subtitles`, `ass`, `overlay`, `palettegen`, `paletteuse`.

**Incidental finding worth flagging:** although the tapes declare `Set Framerate 30`, the rendered
`sase_ace_prompt_input.mp4` reports `r_frame_rate=25/1` (duration 28.84 s). The output frame rate is therefore not
simply the tape's `Framerate` setting. This is a concrete reason why **caption timing must be measured against the
rendered artifact, not computed from tape settings.**

---

## 3. Key finding: VHS has no released native caption/subtitle command

I verified this three ways rather than trusting a single source:

1. **Documentation.** The VHS command set is: `Output`, `Require`, `Set`, `Type`, arrow keys,
   `Backspace`/`Enter`/`Tab`/`Space`, `ScrollUp`/`ScrollDown`, `Ctrl`, `Sleep`, `Wait`, `Hide`, `Show`,
   `Screenshot`, `Copy`, `Paste`, `Source`, `Env`. `Set` options include `WindowBar`, `WindowBarSize`,
   `MarginFill`, `Margin`, `BorderRadius`, `Theme`, `PlaybackSpeed`, `LoopOffset`, etc. **None** add text overlays.
   Output formats: `.gif`, `.mp4`, `.webm`, and `frames/` (PNG sequence).
2. **The installed binary.** `vhs validate` on a tape containing `Subtitle "hello world"` returns
   `Invalid command: Subtitle` (exit 1), while an otherwise-identical tape validates cleanly. So v0.11.0 — the
   latest release (2026-03-10) — does not understand any caption keyword. (The strings `Subtitle` /
   `json:"subtitle,omitempty"` do appear inside the binary, but they come from vendored dependencies — the chroma
   syntax highlighter and bluemonday CSS sanitizer — not the tape grammar.)
3. **Upstream tracking.** [Issue #735 "Allow specifying subtitles in tape files"](https://github.com/charmbracelet/vhs/issues/735)
   requests exactly this (`SetSubtitle`-style text pinned to the bottom of the screen). As of this research it is an
   **open enhancement with no merged implementation.** No released version (`v0.8`–`v0.11`) mentions subtitle /
   caption / overlay / annotation in its changelog.

**Conclusion:** captions must be added by a layer we own, either *before* VHS (baked into the terminal content —
undesirable) or *after* VHS (post-processing the rendered video — the viable path). Waiting on upstream is not a
near-term option.

---

## 4. Option survey

| # | Approach | How | Pros | Cons |
|---|----------|-----|------|------|
| 1 | **Native VHS `Subtitle`** | Wait for / contribute upstream #735, or fork | Zero external tooling; lives in the tape | Not available in any release; unknown API/styling; fork = maintenance burden |
| 2 | **Type text into the terminal** | `echo`/banner or an app caption line inside the tape | No new tooling | Pollutes the demo content; no styling/placement control; can't overlay outside the terminal grid; interferes with `Wait+Screen` |
| 3 | **PNG frames + ImageMagick** | `Output frames/`, annotate frame ranges with `magick`, reassemble | Pixel-perfect per-frame control; can do arbitrary graphics | Slowest; most code; must map frames↔time and re-derive an optimized palette; loses VHS's gif encoding |
| 4a | **ffmpeg `drawtext`** on the MP4 | `drawtext=...:enable='between(t,a,b)'` per cue, then regenerate gif | Simple for 1–2 labels; no sidecar file | Escaping gets painful; styling/positioning limited; unwieldy for many cues; timing hard-coded in the filter string |
| 4b | **ffmpeg + `.ass` (libass)** on the MP4 | Author a styled `.ass` subtitle track, burn via `subtitles=` filter, regenerate gif | Rich styling (box, fade, position, multiple simultaneous captions, word-wrap); timing/content in a clean sidecar; deterministic | Need to author/generate the `.ass`; still need a time base (see §5) |

Approaches 1 and 2 are rejected (unavailable / degrades the demo). Approach 3 is overkill unless we later need
non-text graphics. **The recommended core is 4b (ffmpeg + ASS)**, with 4a as a one-off shortcut. All of 3/4 share
the same enabling fact: **VHS already emits an `.mp4` per tape, which is a high-quality source to post-process** —
we never have to re-run the TUI to add or tweak captions.

---

## 5. The real hard part: syncing captions to "certain points"

Placing text is easy; deciding *when* it appears is the actual design problem, because the sase tapes are
**event-driven** (`Wait+Screen /history/`). The wall-clock time of a moment is not known from the tape and can
drift between renders (and, per §2, isn't even a clean function of `Framerate`). Three anchoring strategies:

- **(A) Absolute timestamps** — cue = `{at: 12.0s, until: 15.0s}`. Simplest; works today. Fragile: any tape edit
  or renderer drift shifts every downstream cue. Fine for *finished, stable* tapes; annoying while authoring.
- **(B) Content markers (most robust)** — drop hidden `Screenshot .markers/<tape>/<cue>.png` calls at cue points
  in the tape, then, after render, locate the frame in the output that matches each marker and convert marker →
  timestamp. Cues then track *content*, surviving `Wait` nondeterminism and edits. Cost: a frame-matching step.
  Honest caveat: a raw VHS `Screenshot` PNG will **not** be byte-identical to an h264/`yuv420p` mp4 frame (chroma
  subsampling + compression + any `WindowBar`/`MarginFill` compositing), so matching must be **nearest-frame by
  lowest MSE within a search window** (or matched against the lossless-ish gif frames), not exact equality. Because
  renders are deterministic, argmin-MSE is reliable in practice.
- **(C) Terminal sentinels** — not viable: the only natural place to emit a unique marker is a `Hide` block, whose
  frames are excluded from the video.

Recommendation: start with (A) and add (B) as an opt-in when timing drift becomes painful. Author (A) timestamps
by scrubbing the rendered mp4 once (commands in §7).

---

## 6. Proof of concept (verified on a real demo artifact)

I burned a styled caption into the **actual** `demos/out/sase_ace_prompt_input.mp4` and regenerated a captioned gif,
using only the installed toolchain, without touching `demos/out/`.

ASS track (`cap.ass`) — a faded lower-third in Fira Code:

```ass
[Script Info]
ScriptType: v4.00+
PlayResX: 1920
PlayResY: 1080
WrapStyle: 2
ScaledBorderAndShadow: yes

[V4+ Styles]
Format: Name, Fontname, Fontsize, PrimaryColour, OutlineColour, BackColour, Bold, BorderStyle, Outline, Shadow, Alignment, MarginL, MarginR, MarginV, Encoding
Style: Lower,Fira Code,40,&H00FFFFFF,&H00000000,&HB0000000,-1,3,8,0,2,80,80,90,1

[Events]
Format: Layer, Start, End, Style, Name, MarginL, MarginR, MarginV, Effect, Text
Dialogue: 0,0:00:01.00,0:00:03.00,Lower,,0,0,0,,{\fad(250,250)}Recall prior prompts instantly
```

Burn + regenerate gif in a single ffmpeg pass (palette computed and applied inline via `split`):

```bash
ffmpeg -y -i in.mp4 \
  -filter_complex "subtitles=cap.ass,split[s0][s1];[s0]palettegen=stats_mode=diff[p];[s1][p]paletteuse=dither=bayer:bayer_scale=3" \
  out.captioned.gif

# and a captioned mp4:
ffmpeg -y -i in.mp4 -vf "subtitles=cap.ass" -c:v libx264 -pix_fmt yuv420p -movflags +faststart out.captioned.mp4
```

**Result:** both files produced cleanly. The captioned gif was 721 frames / 3.8 MB; the captioned mp4 preserved the
28.84 s duration. A frame extracted at t=2.0 s (inside the cue) showed the white Fira Code caption on a
semi-transparent black box as a lower-third over the live TUI frame; a frame at t=5.0 s (after the cue) had no
caption. The `BorderStyle 3` + translucent `BackColour` gives a readable box; `{\fad(250,250)}` fades it in/out.

This confirms the recommended pipeline works end-to-end with what's already installed.

---

## 7. Recommended solution

**Add an ffmpeg + libass caption post-processing stage to the demo pipeline, driven by a per-tape sidecar file,
producing separate captioned outputs. Anchor cues by absolute timestamps first (Phase 1); add optional
content-marker anchoring later (Phase 2).** Keep it entirely in this repo's `demos/` tooling.

### Why this over the alternatives
- **Available today** — proven above with the installed ffmpeg/VHS; no upstream dependency (unlike native
  `Subtitle`), no forking.
- **Decoupled & re-runnable** — operates on VHS's `.mp4`; captions can be edited/retimed without re-recording the
  TUI. Falls exactly into the `ffmpeg` post-processing slot `demos/README.md` already anticipates.
- **Rich, on-brand styling** — libass gives boxes, fades, positioning, multiple simultaneous captions, and
  word-wrap. Reusing `Fira Code` (already pinned for the deterministic visual-snapshot suite) makes captions match
  the terminal typography and stay hermetic/deterministic.
- **Sidecar-driven** — caption content and timing live in a reviewable text file next to the tape, not buried in a
  shell filter string; diffs are readable and captions are optional per tape.

### Boundary & determinism notes
- **Stays in this repo, not `sase-core`.** Per the Rust-core boundary rule, the litmus test is "would another
  frontend need this behavior to match the TUI?" Demo captioning is a build-time presentation artifact, not shared
  runtime domain logic, so it correctly stays as Python/Just tooling under `demos/`.
- **Determinism.** The burn is deterministic given the mp4; pin the caption font via the same fontconfig setup the
  visual suite uses so CI and local renders agree. VHS's own gif encoder uses `palettegen`/`paletteuse`, so our
  re-encode matches its quality characteristics.
- **Don't clobber the clean outputs.** Write `demos/out/<name>.captioned.gif` / `.captioned.mp4` (or gate on the
  presence of a sidecar) so both narrated and clean versions coexist and tapes without captions are untouched.
- `gifsicle` is not required (ffmpeg handles the palette); the README's future `gifsicle` note is orthogonal
  (compression), and can compose after this stage.

### Phase 1 — MVP (ship now)
1. **Sidecar schema** — `demos/tapes/<name>.captions.yml`:
   ```yaml
   defaults:
     font: "Fira Code"
     size: 40
     position: lower-third   # -> ASS Alignment/MarginV
     fade_ms: 250
   cues:
     - at: 2.0s
       until: 4.5s
       text: "Recall prior prompts instantly"
     - at: 12.0s
       until: 15.0s
       text: "One prompt → three models"
   ```
2. **Generator** — a small Python module in `demos/` (e.g. `demos/scripts/captions.py`) that reads the sidecar and
   the rendered mp4's duration (`ffprobe`), emits an `.ass` (styles from `defaults`, one `Dialogue` per cue with
   `{\fad(...)}`), and shells out to the two ffmpeg commands from §6. Timestamps map to ASS `H:MM:SS.cc`.
3. **Authoring aid** — pick timestamps by scrubbing the rendered mp4:
   ```bash
   ffprobe -v error -show_entries format=duration -of csv=p=0 demos/out/<name>.mp4
   # extract a frame at t to eyeball the moment:
   ffmpeg -y -ss <t> -i demos/out/<name>.mp4 -frames:v 1 /tmp/frame.png
   ```
4. **Justfile integration** — after the `vhs ...` render of each tape, if `demos/tapes/<name>.captions.yml` exists,
   run the generator to write `<name>.captioned.{gif,mp4}`. Leaves the existing commit/stamp flow intact.

### Phase 2 — content-marker anchoring (add if/when timing drift bites)
- Allow `at: marker:history` in the sidecar. In the tape, add (inside `Hide`/`Show`)
  `Screenshot demos/out/.markers/<name>/history.png` at the cue moment.
- The generator extracts the mp4's frames (or reuses the gif's frames) and resolves each marker to the
  **nearest frame by MSE** within a search window, converting frame index → timestamp via the measured fps. Cues
  then track content and survive re-renders and `Wait+Screen` nondeterminism.
- If any marker fails to match above a confidence threshold, fail loudly (don't silently mis-place a caption).

### Fallback / shortcut
For a quick one-off label on a single tape without a sidecar, a single ffmpeg `drawtext` pass (Approach 4a) is fine:
```bash
ffmpeg -y -i in.mp4 -vf \
  "drawtext=fontfile=/path/FiraCode.ttf:text='Recall prior prompts':fontcolor=white:fontsize=40:box=1:boxcolor=black@0.7:boxborderw=16:x=(w-tw)/2:y=h-140:enable='between(t,2,4.5)'" \
  out.mp4
```

---

## 8. Risks & open questions

- **Timestamp drift (Phase 1).** Absolute cues break if a tape is edited. Mitigation: keep captions sparse, regen
  after any tape change, or adopt Phase 2 marker anchoring.
- **Marker match reliability (Phase 2).** Nearest-frame-by-MSE needs a sensible window and threshold; validate on
  each tape and fail loudly on low-confidence matches.
- **Font hermeticity.** Ensure the caption font resolves through the same fontconfig setup as the visual suite so
  CI and local renders match; otherwise captioned goldens could drift.
- **Snapshot/golden policy.** Decide whether `*.captioned.*` artifacts are committed like the current `demos/out/`
  files (and whether they participate in any visual-diff gating) or are treated as regenerated-on-demand.
- **Output size.** Burned captions add frames' worth of high-contrast pixels; monitor gif size and tune
  `paletteuse` dithering if needed.
- **Upstream convergence.** If VHS ships native subtitles (#735), revisit — but our sidecar approach would still
  offer richer styling and per-cue control, and can coexist.

---

## 9. Sources

- [charmbracelet/vhs — repository & README](https://github.com/charmbracelet/vhs)
- [VHS README (tape command & Set reference)](https://github.com/charmbracelet/vhs/blob/main/README.md)
- [VHS releases / changelog](https://github.com/charmbracelet/vhs/releases)
- [VHS issue #735 — "Allow specifying subtitles in tape files" (open)](https://github.com/charmbracelet/vhs/issues/735)
- [FFmpeg `drawtext` dynamic overlays / timed text (OTTVerse)](https://ottverse.com/ffmpeg-drawtext-filter-dynamic-overlays-timecode-scrolling-text-credits/)
- [Adding text & subtitles with FFmpeg — `drawtext`, `subtitles`, `force_style` (FFmpegLab)](https://www.ffmpeglab.com/articles/ffmpeg-add-text-subtitles-video.html)
- [FFmpeg subtitles / burning captions (Cloudinary guide)](https://cloudinary.com/guides/video-effects/ffmpeg-subtitles)
- Local verification: `vhs version v0.11.0`, `ffmpeg 7.1.5` (`--enable-libass --enable-libfreetype
  --enable-libfontconfig`), and a working burn-in PoC against `demos/out/sase_ace_prompt_input.mp4`.
