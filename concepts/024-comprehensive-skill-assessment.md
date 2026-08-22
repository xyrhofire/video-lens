# 2026-08-22 — Comprehensive assessment: video-lens + video-lens-gallery

**Scope.** Full read of both skills — `SKILL.md` ×2, all six `video-lens/scripts/`, both
gallery scripts, `template.html` (1976 lines), `index.html` (1295 lines), `Taskfile.yml`,
`scripts/raycast-video-lens.sh`, `requirements.txt`, and the 101-test suite.

**Method.** Static read plus empirical verification: the fast suite was run (100 passed,
1 skipped, 2 deselected in 4.75 s), individual defects were reproduced against the real
scripts, and — most usefully — the **live library of 180 reports** in
`~/Downloads/video-lens` was mined as a corpus to check whether each suspected defect
actually occurs in practice. Several suspicions died there; those are recorded in
§5 rather than deleted, because a negative result is the expensive half of an audit.

**Headline.** The pipeline is in good shape. The security work from 009/013 holds, the
mechanical-glue relocation from 019 removed most of the agent's chances to get arithmetic
wrong, and the recent Whisper fallback is well-tested. What is left is mostly **one
genuine correctness bug that silently makes a whole class of videos unprocessable**,
a handful of cosmetic and consistency defects, one real gap in test coverage, and a
library that has now crossed the scale (180 reports, 151 distinct videos) where
*curation* features matter more than *generation* features.

---

## 1. Summary table

Severity: 🔴 breaks a run · 🟠 wrong output · 🟡 cosmetic/inconsistent · ⚪ latent
Effort: **S** ≤1 h · **M** ~half-day · **L** 1–3 days · **XL** spec first
Complexity: Low = mechanical · Med = design choices · High = architectural

| ID | Finding | Category | Sev | Effort | Cplx | Status |
|---|---|---|---|---|---|---|
| **B1** | Renderer hard-fails on any `{{UPPER_CASE}}` token in agent content | Bug | 🔴 | S | Low | ✅ done |
| **B2** | `_format_duration(None)` emits `"0 min"` instead of blank | Bug | 🟠 | S | Low | ✅ done |
| **B3** | HTML-scraped title/channel are never unescaped | Bug | ⚪ | S | Low | ✅ done |
| **B4** | Empty `META_LINE` renders a dangling leading `·` | Bug | 🟡 | S | Low | ✅ done |
| **B5** | `build_index` sorts by storage location before date | Bug | 🟡 | S | Low | ✅ done |
| **B6** | Report ignores `prefers-color-scheme`; gallery honours it | Bug | 🟡 | S | Low | ✅ done |
| **B7** | Markdown export: revoked blob URL + all emphasis dropped | Bug | 🟠 | S | Low | ✅ done |
| **B8** | Raycast/Taskfile pin two-generation-stale model IDs; `m.youtube.com` rejected | Bug | 🟡 | S | Low | ✅ done |
| **H1** | Local server exposes `.tmp/payload-*/payload.json` and never stops | Hardening | 🟠 | S | Low | ✅ done |
| **H2** | Output-path clamp is disabled by a `PYTEST_CURRENT_TEST` env var | Hardening | ⚪ | M | Low | — next |
| **H3** | Raycast path runs the agent `--dangerously-skip-permissions` over untrusted transcripts | Hardening | ⚪ | M | Med | — next |
| **H4** | `_download_audio` has no subprocess timeout | Hardening | 🟠 | S | Low | ✅ done |
| **T1** | `fetch_transcript.py` has zero tests — and it owns the error contract | Testing | 🟠 | M | Low | — next |
| **T2** | `backfill_meta.py` untested; error codes split across stdout/stderr | Testing | 🟡 | S | Low | — next |
| **U1** | 16 % of the library is redundant re-runs; gallery shows each as a separate card | UX | 🟠 | M | Med | — later |
| **U2** | 206 of 305 tags are used exactly once | UX | 🟡 | M | Med | deferred |
| **U3** | Half the library predates `agentModel`/`generatedAt`/`durationSeconds` | UX | 🟡 | M | Med | — later |
| **U4** | Outline entries expand on click but are not keyboard-reachable | UX/a11y | 🟠 | S | Low | ✅ done |
| **U5** | No print stylesheet, no `prefers-reduced-motion` | UX/a11y | 🟡 | S | Low | — next |
| **U6** | `<html lang="en">` hardcoded on non-English reports (023-F2, still open) | UX/a11y | 🟡 | S | Low | — next |
| **U7** | No back-link from a report to the gallery (023-I7, still open) | UX | 🟡 | S | Low | — next |
| **N1** | Transcript sidecar — one artifact that unlocks four features | Feature | — | M–L | Med | — later |
| **N2** | Whisper progress output + `large-v3-turbo` | Feature | — | S–M | Low | — later |
| **N3** | `preflight.py --doctor` | Feature | — | S | Low | — next |
| **N4** | Deterministic payload lint before render | Feature | — | M | Med | — later |
| **N5** | Batch / playlist mode | Feature | — | L | Med | spec first |
| **N6** | Non-YouTube sources via yt-dlp + Whisper | Feature | — | L | High | spec first |

---

## 2. Bugs

### B1. The renderer hard-fails on any `{{UPPER_CASE}}` token in agent content 🔴

`_render_clean` substitutes placeholders with a loop of `str.replace`, then asserts that
no `\{\{[A-Z_]+\}\}` pattern survives (`render_report.py:505-513`). Agent-authored content
is substituted *into* the template before that check runs, so any such token in a summary,
key point, or outline entry is scanned as if it were a template placeholder — and the
render dies.

Reproduced:

```
SUMMARY = "The speaker mentions the {{FOO_BAR}} template variable."
→ ValueError: RENDER_UNREPLACED_PLACEHOLDERS ['{{FOO_BAR}}']
```

There is a second path with the same root cause: `VIDEO_LENS_META` is substituted **last**,
and its own JSON embeds a truncated copy of `SUMMARY`. A summary containing the literal
string `{{VIDEO_LENS_META}}` therefore re-injects the token after its only substitution
pass has already run — same crash, from a different direction.

**Why this one leads the list.** It is the only defect found that makes an entire *category*
of input unprocessable rather than merely ugly. Any video about Jinja, Handlebars, Mustache,
Vue, Angular, or GitHub Actions — squarely inside this library's subject matter, where
`developer tools` (31) and `ai agents` (25) are top tags — can put `{{TITLE}}` or
`{{USER_NAME}}` in front of the model, which will faithfully quote it. The run then dies at
Step 4 with `ERROR:RENDER_UNREPLACED_PLACEHOLDERS`, which matches the `ERROR:RENDER_*` row
of SKILL.md's error table and instructs the agent to **stop** — after the transcript fetch,
the analysis, and the whole token spend. The user gets nothing and no actionable message.

**Fix.** Compute the template's own placeholder set *before* substituting, and do the
substitution in a single pass so injected text is never rescanned:

```python
placeholders = set(re.findall(r"\{\{([A-Z_]+)\}\}", html))
missing = placeholders - data.keys()
if missing: raise ValueError(f"RENDER_UNREPLACED_PLACEHOLDERS {sorted(missing)}")
html = re.sub(r"\{\{([A-Z_]+)\}\}", lambda m: data[m.group(1)], html)
```

`re.sub` with a function replacement never re-examines what it inserted, which fixes both
paths at once. Add two regression tests (`{{FOO_BAR}}` in `SUMMARY`, `{{VIDEO_LENS_META}}`
in `SUMMARY`).

### B2. `_format_duration(None)` emits `"0 min"` 🟠

`fetch_metadata.py:37-40` coerces a missing duration with `int(dur_s or 0)` and returns
`"0 min"` — a value that then flows into `META_LINE` and the gallery card as if it were
real. Live streams, premieres, and any yt-dlp response without a `duration` field hit this.
Reproduced directly; 0/180 current reports are affected, so it is latent but free to fix.
Note the sibling `_format_views(None)` already returns `""` — this is an inconsistency
between two adjacent functions, not a considered choice.

**Fix.** `if not dur_s: return ""`. One line, one test.

### B3. HTML-scraped title and channel are never unescaped ⚪

`_fetch_html_metadata` (`fetch_transcript.py:21-27`) pulls the title straight out of
`<title>…</title>` and the channel out of a JSON blob with a regex — neither is decoded:

```
'<title>Rock &amp; Roll: &quot;Best&quot; Hits - YouTube</title>'
  → 'Rock &amp; Roll: &quot;Best&quot; Hits'
'"channelName":"Café Müsik"'  → 'Caf\\u00e9 M\\u00fcsik'
```

The renderer escapes `VIDEO_TITLE` again, so a passed-through entity would surface in the
`<h1>` as the literal text `Rock &amp; Roll`.

**Measured: 0 of 180 reports are affected.** The reason is worth stating plainly — the model
reads `TITLE: Rock &amp; Roll` and writes `Rock & Roll` into the payload of its own accord.
Correctness here currently rests on model behaviour rather than on code, which is exactly
the kind of dependency the 019 mechanical-glue work set out to remove. A smaller or
differently-tuned model is entitled to copy the string verbatim.

**Fix.** `html.unescape()` the title; decode the channel properly (or drop the regex and
parse the embedded JSON). Better still, have `fetch_metadata.py` emit `YTDLP_TITLE` — yt-dlp
already returns a clean, unescaped title, and 2b's values are documented as preferred for
every *other* field. Title is the one metadata field with no yt-dlp path, which is an
oversight rather than a decision.

### B4. Empty `META_LINE` leaves a dangling `·` 🟡

`template.html:1114` is `<p class="meta-line">{{META_LINE}} · <a …>Open on YouTube ↗</a></p>`.
When yt-dlp is absent *and* the HTML scrape fails, `META_LINE` composes to `""` and the
header renders as ` · Open on YouTube ↗`. Present in the wild:
`2026-05-18-122925-video-lens_nepKKz-MzFM_ffmpeg_th.html` (1/180).

**Fix.** Move the separator into the renderer — have `_maybe_compose_meta_line` append the
link fragment, or make the template emit the `·` only when `META_LINE` is non-empty.

### B5. `build_index` sorts by storage location before date 🟡

`build_index.py:154` sorts the merged list on `filename`, but phase-1 entries were rewritten
to `"reports/" + name` (line 129) while phase-2 legacy entries keep a bare name. Since
`"r" > "2"`, every `reports/` entry sorts ahead of every legacy one regardless of date.
Today that coincides with the right answer — the new location *is* newer — so the gallery
looks correct. It stops being correct the moment a legacy file is touched or re-added.

**Fix.** Sort on `pathlib.Path(filename).name`, or better on `generationDate` with the
filename as tiebreak.

### B6. The report ignores the OS colour scheme; the gallery honours it 🟡

Gallery (`index.html:642`):

```js
localStorage.getItem(THEME_KEY) || (matchMedia('(prefers-color-scheme: dark)').matches ? 'dark' : 'light')
```

Report (`template.html:1367`):

```js
var saved = localStorage.getItem('theme');
applyTheme(saved === 'dark');
```

Both use the same `'theme'` key and the same origin, so once the user has toggled anywhere
they agree. But on a first visit — including every fresh browser profile — a dark-mode user
gets a white report and a dark gallery side by side.

**Fix.** Copy the gallery's one-liner into the template.

### B7. Markdown export: revoked blob URL, and all emphasis dropped 🟠

`template.html:1869-1874`:

```js
a.href = URL.createObjectURL(blob);
a.click();
URL.revokeObjectURL(a.href);   // synchronous — may cancel the download
```

The anchor is also never attached to the document. Both are long-standing causes of silently
failing downloads outside Chrome; on macOS, Safari is the likely victim. Defer the revoke
(`setTimeout(…, 0)` or an `onclick` handler) and append/remove the anchor.

Separately, the export builds every line from `textContent`/`innerText`, so the
`<strong>`/`<em>` structure that SKILL.md spends three rules specifying is thrown away —
exported Key Points arrive as flat prose. Walking the nodes and wrapping `strong` in `**`
and `em` in `*` is ~15 lines and makes the export worth the button.

### B8. Stale model pins and a too-narrow URL guard 🟡

`scripts/raycast-video-lens.sh:36-39` and the `install-raycast` task in `Taskfile.yml` pin
`claude-opus-4-6` / `claude-sonnet-4-6` / `claude-haiku-4-5-20251001`. The library's own
`agentModel` values show what is actually being used today: `claude-opus-4-8`,
`claude-fable-5`, `claude-opus-4-7`, `qwen3.6`. The Raycast entry point is therefore pinned
two generations behind, and the Copilot branch of the Taskfile carries a third variant of
the same stale IDs. This is 023-F9 ("standing maintenance tax") coming due.

**Fix.** Drop the pin and let the CLI use its configured default, keeping the argument as
an optional pass-through (`--model "$2"` when non-empty). That removes the maintenance
obligation permanently instead of resetting its clock.

Same file, line 22: the guard is `^https?://(www\.)?(youtube\.com|youtu\.be)/`, which rejects
`m.youtube.com` — the exact form the iOS share sheet produces — even though `preflight.py`
accepts it (`YOUTUBE_HOSTS` includes `m.youtube.com`). Add `m\.` to the alternation.

---

## 3. Hardening

### H1. The local server exposes run payloads and never shuts down 🟠

`serve_report.sh` is invoked with `~/Downloads/video-lens` as the root so that reports,
`index.html`, and `manifest.json` are all reachable. That root also contains
`.tmp/payload-*/payload.json` — the complete generated content of every run from the last
seven days. The payload dirs are `0700`, but `http.server` runs *as the user*, so the
loopback serves them happily, directory listing included. The server then stays up
indefinitely (by design — the PID file exists to reuse it).

The realistic exposure is other local processes and other local accounts, not the open
internet (the bind is `127.0.0.1`, and CORS blocks a hostile web page from reading
responses). It is still avoidable for free.

**Fix.** Move `PAYLOAD_BASE_DIR` out of the served tree to
`${XDG_CACHE_HOME:-~/.cache}/video-lens/payloads` — which is already where `serve_report.sh`
keeps its PID file, so no new location is introduced. `preflight.py` emits the absolute
`PAYLOAD_PATH` anyway, so nothing downstream changes. Consider also an idle-timeout wrapper
so a forgotten server does not outlive the session by days.

### H2. The path clamp is disabled by an environment variable ⚪

```python
if os.environ.get("PYTEST_CURRENT_TEST") or os.environ.get("VIDEO_LENS_ALLOW_ANY_PATH") == "1":
    return resolved   # skips the ~/Downloads/video-lens/reports clamp entirely
```

This is a documented decision (013), and the traversal test correctly scrubs both vars. The
residual concern is narrow but real: the clamp — one of the two controls the 013 threat
model rests on — is switched off by an env var whose name is well-known and whose *value* is
not even checked. Anything that can set an environment variable in the renderer's process
can write HTML anywhere the user can.

**Fix.** Delete the env sniffing and let tests inject the root instead — either monkeypatch
`render_report.ALLOWED_OUTPUT_ROOT` (in-process tests) or pass an explicit
`--allow-output-root` flag (subprocess tests). Roughly six tests touch this; the change is
mechanical and removes a production-code branch that exists only for tests.

### H3. The Raycast path runs the agent with permissions fully disabled ⚪

`raycast-video-lens.sh` builds `claude --dangerously-skip-permissions --allowedTools
"Bash,Read" …`. The tool list is scoped, but `Bash` under skip-permissions is unrestricted
shell. The content being processed is a YouTube transcript plus a creator-controlled
description — untrusted by construction — and the only defence is the *Untrusted input*
paragraph in SKILL.md, i.e. a prose instruction to a model.

I am not claiming an exploit exists; the skill's own hardening docs already accept this
trade-off knowingly. It is listed because it is the highest-consequence residual item in the
system and because a cheaper mitigation now exists: permission rules support command-scoped
patterns, so the Raycast invocation could allow only what the skill actually runs —
`Bash(python3 *preflight.py*)`, `Bash(python3 *fetch_transcript.py*)`, `Bash(python3
*fetch_metadata.py*)`, `Bash(python3 *render_report.py*)`, `Bash(bash *serve_report.sh*)`,
`Bash(python3 *build_index.py*)` — instead of blanket `Bash`. A hijacked run would then be
confined to the six scripts it was always supposed to call. Worth an experiment; the
per-agent flag variations in the Taskfile make it a half-day, not an hour.

### H4. `_download_audio` has no timeout 🟠

`transcribe_local.py:108-113` calls `subprocess.run` with no `timeout=`. `--socket-timeout
30` bounds individual socket reads but not total runtime, so a throttled or stalling
download runs until the agent's 600 000 ms Bash cap kills the *whole* step — consuming the
entire budget the user was told to reserve for transcription, and returning no output at
all rather than a structured `ERROR:AUDIO_DOWNLOAD_FAILED`.

**Fix.** Pass a `timeout=` (a fixed 900 s, or a value derived from `YTDLP_DURATION`), catch
`TimeoutExpired`, and emit `ERROR:AUDIO_DOWNLOAD_FAILED: download timed out after Ns` so the
existing error row handles it. `_ytdlp_version()` already models the pattern.

---

## 4. Testing and infrastructure

### T1. `fetch_transcript.py` has zero tests — and it owns the error contract 🟠

101 tests cover the renderer, preflight, the gallery index builder, and the Whisper fallback.
`fetch_transcript.py` is imported by nothing in `tests/`. That is precisely inverted risk
allocation: it is the most failure-prone script in the repo — live network, HTML scraping
with four brittle regexes, two incompatible `youtube-transcript-api` call shapes
(`.list()` vs `.list_transcripts()`), nine exception classes imported defensively behind
`try/except ImportError`, and a three-tier language-selection fallback.

And it is the **source of the error contract**: `CAPTIONS_DISABLED`, `NO_TRANSCRIPT`,
`IP_BLOCKED`, `PO_TOKEN_REQUIRED`, `REQUEST_BLOCKED`, `AGE_RESTRICTED`,
`VIDEO_UNAVAILABLE`, `INVALID_VIDEO_ID`, `LIBRARY_MISSING`, `NETWORK_ERROR`,
`TRANSCRIPT_FETCH_FAILED`, plus `LANG_WARN:`. Every row of SKILL.md's Error Handling table
routes on these strings, and the Whisper fallback is triggered by five of them. If an
upstream rename silently drops one of the defensive imports to `None`, every error in that
class collapses into `TRANSCRIPT_FETCH_FAILED` — which SKILL.md says to **stop** on, so the
Whisper fallback would never be offered again and nothing would fail loudly.

**Fix.** `tests/test_fetch_transcript.py`, using the pattern `test_transcribe.py` already
established for mlx-whisper: put a stub `youtube_transcript_api` package on `sys.path` and
run the script as a subprocess. Assert (a) each exception class maps to its documented code,
(b) the three-tier language selection picks native → exact → fallback and emits `LANG_WARN:`
on the third, (c) the `.list()` → `.list_transcripts()` shim, (d) the header block shape
that `transcribe_local.py` is documented as byte-compatible with, (e) `[M:SS]` vs `[H:MM:SS]`
timestamp switching at the hour boundary. No network needed.

### T2. Smaller coverage and consistency gaps 🟡

- `backfill_meta.py` has no tests, and it is the one script that **rewrites existing report
  files in place**. It has a `--dry-run` flag; nothing exercises it.
- `serve_report.sh` has two tests, neither covering the `URL_PATH="${HTML_PATH#${SERVE_DIR}/}"`
  prefix strip — which silently yields a doubled absolute path in the URL if the caller ever
  passes a relative or symlink-differing path.
- Error codes are split across streams inside one file: `fetch_transcript.py` prints
  `LIBRARY_MISSING` and all mapped codes to **stdout** (lines 70, 121) but
  `TRANSCRIPT_FETCH_FAILED` on fetch failure to **stderr** (line 157). The agent reads both,
  so nothing is broken today, but it makes the contract untestable-by-stream and would trip
  any future shell composition. Standardise on stderr, as `preflight.py` and
  `render_report.py` already do.

---

## 5. Checked and holding (negative results)

Recorded so the next audit does not re-spend the time:

- **Path traversal via `VIDEO_ID` in the derived filename.** `_derived_filename` interpolates
  `VIDEO_ID` before `sanitise_payload` validates it, which looked like an ordering bug.
  Verified safe: `validate_output_path` clamps to the reports root first, and a 300-char or
  slash-bearing id is rejected by `VIDEO_ID_RE` before any directory is created. Tested with
  `"../../../../evil"`, `"sub/dir/id"`, and `"x"*300` — all produce clean structured errors
  and touch no filesystem.
- **XSS through the sanitiser.** Attribute values are unescaped by `HTMLParser` regardless of
  `convert_charrefs`, so `&amp;t=` in an outline href still parses to a matching video id;
  `_serialize_meta` escapes `</` so the meta JSON cannot break out of its `<script>`; the
  gallery builds every card with `textContent`, never `innerHTML`, for report-derived data.
- **`onPlayerError` DOM teardown.** It wipes `.video-wrap`, then touches `#progress-track` —
  which is a *sibling*, not a child, so no null dereference.
- **Gallery build performance.** 180 reports rebuild in **0.05 s**. The incremental-indexing
  idea from 007 is not worth building; at this growth rate a full rebuild stays under a
  second past 2 000 reports. The manifest is 241 KB and is inlined into `index.html` (251 KB),
  which is the only figure worth watching.
- **Double-escaped titles in practice.** 0/180 (see B3).
- **`"0 min"` durations in practice.** 0/180 (see B2).

---

## 6. Library-scale UX findings

These come from measuring the actual 180-report library rather than from reading code —
the skill has crossed the point where the archive, not the individual report, is the product.

### U1. 16 % of the library is redundant re-runs 🟠

151 distinct videos across 180 files. **14 videos have more than one report; 29 files are
duplicates.** `preflight.py` detects this and prints `DUPLICATE_PATH:`, but SKILL.md
explicitly makes it non-blocking ("do not ask the user to choose"), and the gallery renders
every copy as its own card. So the same talk appears three times in the grid with three
different summaries and no indication that they are the same video.

**Fix (023-I5).** Group manifest entries by `videoId` in the gallery; show the newest as the
card with a "3 versions" affordance revealing the others. Keeps every file, removes the
visual duplication. `videoId` is already in every meta block, so this is gallery-only.

### U2. 206 of 305 tags are used exactly once 🟡

Distribution: 64 tags used ≥3×, 35 used twice, **206 used once**. Average 4.4 tags per
report. The chip row caps at 50 (cut-off lands at count 3) and hides the rest behind
"+255 more", so the long tail is already mostly out of the way — and only **6 reports** are
reachable *exclusively* through a singleton tag, so the tail costs almost nothing in
discoverability either.

This is a smaller problem than 023-F4 assumed, and the `EXISTING_TAGS` feedback loop is the
right long-term fix already in place — it only steers *new* reports, so the ratio will
improve on its own. The remaining cost is that ~68 % of the model's tag-writing effort is
wasted and the manifest carries 206 dead strings. A one-off merge pass over the manifest
(fold singletons into their nearest ≥3-count neighbour, human-reviewed) would clean history
without touching report HTML. **Downgraded to a nice-to-have** on the evidence.

### U3. Half the library predates its own metadata schema 🟡

Missing fields: `agentModel` 98/180 (54 %), `generatedAt` 86/180 (47 %), `durationSeconds`
89/180 (49 %). Not a defect — those fields were added later — but any gallery feature that
sorts or filters on them is unreliable across half the collection, and `agentModel` values
are free-text and un-normalised (`qwen3.6`, `qwen3.6:35b-a3b-coding-mxfp8`, `Codex (GPT-5)`,
`claude-sonnet-4-6`). Decide explicitly: backfill what is derivable (`generationDate` →
`generatedAt`), normalise the model string at write time, and have the gallery hide columns
that are sparse rather than showing half-empty ones.

### U4. Outline entries are mouse-only 🟠

`template.html:1600` binds expansion to a click on the `<li>`. The `<li>` has no `tabindex`,
no `role="button"`, no `aria-expanded`. Only the timestamp `<a class="ts">` is focusable, and
activating it seeks the video instead of revealing the detail sentence. The `outline-detail`
text — one of the four content sections the skill generates — is therefore unreachable by
keyboard and invisible to screen readers.

**Fix.** `tabindex="0"`, `role="button"`, `aria-expanded` toggled with the class, and a
`keydown` handler for Enter/Space. Sanitiser impact: `tabindex`/`role`/`aria-expanded` would
need adding to the OUTLINE allowlist if emitted by the model — better to have the template's
JS add them at runtime, which needs no allowlist change at all.

### U5–U7. Known-open smaller items

- **U5** — Neither page has `@media print` (005-B2) or `prefers-reduced-motion`. The report is
  a reading artifact; printing it currently emits the video frame and control bar.
- **U6** — `<html lang="en">` is hardcoded in both files while reports are deliberately written
  in the video's language (023-F2, still open). The payload has no language key, so fixing
  this properly means adding `LANG` to the payload — which also closes 023-F3 (language and
  transcript provenance missing from `VIDEO_LENS_META`) and would let the gallery filter by
  language and mark Whisper-sourced reports. One change, three wins. Currently 1 report in
  the library carries the `🎙 transcribed locally` suffix, and it is discoverable only by
  reading the meta line.
- **U7** — No back-link from a report to the gallery (023-I7, still open). The gallery links
  out; nothing links back.

---

## 7. Feature opportunities

Ranked by value-per-unit-effort against *this* library's observed usage.

### N1. Transcript sidecar — **M–L, Med complexity**

Persist the fetched transcript next to the report (`reports/<name>.transcript.txt`, or
gzipped in a `transcripts/` subdir). One artifact, four features: re-summarise with a
different focus without re-fetching (the top ask in 005-A7), quote lookup, "ask my library"
grounded in real text rather than 300-char summaries, and free re-rendering when the
template changes. It also makes the Whisper fallback economically sane — an 8-minute
transcription currently produces output that is thrown away the moment the run ends.
Storage: ~50 KB/report → ~9 MB for the whole current library. This is the single highest-
leverage item on the list.

### N2. Whisper progress + `large-v3-turbo` — **S–M, Low**

`mlx_whisper.transcribe(verbose=False)` prints nothing, so a 90-minute video produces eight
minutes of silence and SKILL.md has to tell the agent to background-and-poll. Emitting
`PROGRESS: 12:34 / 1:28:00` to **stderr** (stdout is contractually the transcript) makes the
poll meaningful. Separately, `MODEL_REPOS` lacks `large-v3-turbo`, which is roughly 4–6×
faster than `large-v3` at close to the same accuracy — a better default than `medium` for
non-English audio.

### N3. `preflight.py --doctor` — **S, Low**

One command reporting: python version, `youtube-transcript-api` version, `yt-dlp` version
(with the 2026.8.19 comparison already implemented in `transcribe_local._ytdlp_version`),
`ffmpeg`/`mlx-whisper` presence, reports-dir path and count, and whether port 8765 is free
or held. Every one of those checks already exists somewhere in the codebase; this collects
them. Turns "it didn't work" into one paste.

### N4. Payload lint before render — **M, Med**

A `--lint` mode on `render_report.py` asserting deterministic properties no unit test can
reach: outline timestamps strictly increasing and inside the video duration, every outline
`href` id equal to `VIDEO_ID`, key-point count within 3–8, tags 3–5, `TAKEAWAY` not a
substring-level restatement of `SUMMARY`, no key point shorter than N chars. The sanitiser
already parses this HTML — the traversal is nearly free. This is the only proposal here that
attacks *output quality* mechanically rather than through prompt wording, which is where the
011/014/015 audits kept landing.

### N5. Batch / playlist mode — **L, Med**

`preflight.py` accepts one URL. Given 180 reports accumulated one at a time, a playlist or
"these five links" mode with a shared server start and a single index rebuild is a real
workflow win. Needs a concurrency decision (transcript fetches parallelise; Whisper does
not) — spec first.

### N6. Non-YouTube sources — **L, High**

yt-dlp supports hundreds of sites and the Whisper path already handles caption-less media,
so conference recordings, podcast MP3s, and Vimeo links are within reach. High complexity
because `VIDEO_ID`, the embed iframe, the `&t=` deep links, and the duplicate check all
assume an 11-character YouTube id. Genuinely a v6 feature, not an increment.

---

## 8. Suggested sequencing

**Now — one session, all Low complexity, all with tests (B1 first):**
B1 single-pass substitution · B2 blank duration · B3 unescape + `YTDLP_TITLE` ·
B4 meta-line separator · B5 index sort key · B6 colour-scheme default ·
B7 blob revoke + markdown emphasis · B8 unpin models + `m.youtube.com` ·
H1 payloads out of the served tree · H4 download timeout · U4 outline keyboard access.

**Next — half a day each:**
T1 `fetch_transcript` test suite (do this before any further change to that file) ·
H2 remove the env-var clamp bypass · U6 `LANG` in the payload → `lang` attribute +
provenance in meta + gallery filter · U7 gallery back-link · N3 `--doctor`.

**Then — evaluated on merit:**
N1 transcript sidecar (highest leverage) · U1 gallery grouping by `videoId` ·
N4 payload lint · N2 Whisper progress + turbo model · H3 scoped permission experiment.

**Deliberately deferred:** U2 tag merge (evidence says the cap already contains it) ·
007 incremental indexing (0.05 s at 180 reports — not a problem) · N5, N6 (spec first).

---

## 9. Working-tree note

Five files are modified and uncommitted: the `--audio-file` escape hatch for
`transcribe_local.py`, its stale-yt-dlp 403 hint, the `_pick_audio_file` `.part` fix, the
matching SKILL.md error-table row and version bump to 5.1, the `yt-dlp>=2026.8.19` pin, and
tests for all of it. The suite is green (100 passed, 1 skipped). This work is complete and
coherent — it should be committed before any of the above lands on top of it.

## Appendix — reproductions

```bash
# B1
python3 -c "import sys;sys.path.insert(0,'skills/video-lens/scripts');import render_report as r;\
r.render_from_payload({...,'SUMMARY':'uses {{FOO_BAR}} here'},'/tmp/x.html')"
# → ValueError: RENDER_UNREPLACED_PLACEHOLDERS ['{{FOO_BAR}}']

# B2
python3 -c "import sys;sys.path.insert(0,'skills/video-lens/scripts');\
import fetch_metadata as m;print(repr(m._format_duration(None)))"   # → '0 min'

# §5 library statistics
python3 - <<'PY'
import json,collections,pathlib
m=json.loads((pathlib.Path.home()/'Downloads/video-lens/manifest.json').read_text())
ids=collections.Counter(x['videoId'] for x in m['reports'])
print(len(m['reports']),'reports |',len(ids),'videos |',sum(v-1 for v in ids.values()),'duplicates')
PY
```

---

## 10. Implementation record — 2026-08-22

The §8 "Now" batch was implemented in one session. Every item has at least one
regression test; the suite went from 103 to 122 passing, 0 failures.

| ID | Fix as shipped | Files | Tests |
|---|---|---|---|
| **B1** | Template placeholders are collected *before* substitution and replaced in one `re.sub` pass with a function replacement, so injected content is never rescanned. Both paths (`{{FOO_BAR}}` in `SUMMARY`, `{{VIDEO_LENS_META}}` in `SUMMARY`) are closed by the same change. | `render_report.py:508-520` | `test_render_allows_template_tokens_in_agent_content`, `test_render_allows_meta_token_in_summary`, `test_render_still_reports_a_genuinely_missing_placeholder` |
| **B2** | `_format_duration` returns `""` for `None`/`0`/`""`, matching the adjacent `_format_views`. | `fetch_metadata.py:37-41` | `test_format_duration_blank_when_missing`, `test_format_duration_still_formats_real_values` |
| **B3** | `<title>` is run through `html.unescape`; `channelName` is captured as a full JSON string literal and decoded via `json.loads`. `fetch_metadata.py` now also emits `YTDLP_TITLE`, and SKILL.md Step 2b prefers it over the scrape and requires copying it verbatim. | `fetch_transcript.py:15-21, 34, 38-41`, `fetch_metadata.py:121`, `SKILL.md:89` | `test_html_metadata_decodes_entities_and_escapes`, `test_fetch_metadata_emits_ytdlp_title` |
| **B4** | `META_LINE` moved into `<span class="meta-facts">`; the `·` is now a CSS `::after` on `.meta-facts:not(:empty)`, so it disappears with the facts. The Markdown export reads that span instead of filtering text nodes. | `template.html:204-207, 1124, 1840-1842` | `test_empty_meta_line_leaves_no_dangling_separator`, `test_meta_line_separator_present_when_facts_are` |
| **B5** | Merged list sorts on `PurePosixPath(filename).name`, so the `reports/` prefix no longer outranks the date. | `build_index.py:154-157` | `test_build_index_sorts_by_date_not_storage_location` |
| **B6** | First visit with no stored preference falls back to `prefers-color-scheme`, as the gallery already did. | `template.html:1378-1382` | `test_template_honours_os_colour_scheme_on_first_visit` |
| **B7** | The anchor is appended to `document.body` and the object URL is revoked in a `setTimeout(…, 0)`. A new `mdInline` walker converts `<strong>`→`**`, `<em>`→`*`, `<code>`→`` ` ``, `<br>`→newline, and is used for the summary, takeaway, key points and outline. | `template.html:1819-1835` (walker), `1846`, `1866`, `1873-1879`, `1889-1890`, `1927-1940` | `test_markdown_export_defers_blob_revoke_and_keeps_emphasis` |
| **B8** | Model IDs unpinned entirely: the Raycast argument is passed through verbatim as `--model '<value>'`, and omitted when blank. The Taskfile's `MODEL_MAPS` rewriting is gone. URL guard widened to `((www\|m)\.)?`. | `raycast-video-lens.sh:22-23, 36-44`, `Taskfile.yml:89-90` | `test_raycast_accepts_mobile_youtube_and_does_not_pin_models` |
| **H1** | `PAYLOAD_BASE_DIR` moved to `${XDG_CACHE_HOME:-~/.cache}/video-lens/payloads`, outside the tree `serve_report.sh` publishes. Payloads written to the old `.tmp/` location keep ageing out under the existing TTL and the directory is removed once empty. | `preflight.py:16, 39-44, 202-211`, `SKILL.md:229` | `test_payload_dir_is_outside_the_served_tree`, `test_preflight_sweeps_the_legacy_payload_dir` |
| **H4** | `_download_audio` passes `timeout=DOWNLOAD_TIMEOUT_SECONDS` (900) and converts `TimeoutExpired` into the existing `ERROR:AUDIO_DOWNLOAD_FAILED` contract. | `transcribe_local.py:27, 107-122` | `test_download_audio_times_out_with_a_structured_error` |
| **U4** | Outline `<li>`s with a detail get `role="button"`, `tabindex="0"` and a synced `aria-expanded`, plus an Enter/Space `keydown` handler and a `:focus-visible` ring. Attributes are set by the template's own JS, so the sanitiser allowlist is untouched. | `template.html:346-349, 1613-1655` | `test_outline_entries_are_keyboard_reachable` |

### Verification beyond the suite

- **B1/B4** — rendered two real reports through `render_from_payload`: one with
  `{{FOO_BAR}}` and `{{VIDEO_LENS_META}}` inside `SUMMARY` (renders, tokens survive as
  literal text), one with every metadata field blank (`<span class="meta-facts"></span>`,
  no `·` anywhere in the header).
- **B7** — the shipped `mdInline` was extracted from `template.html` and exercised in Node
  against DOM stand-ins: nested `**bold with *italic***`, `<br>`→newline, `<code>` spans,
  and `mdInline(null) === ""` all behave. All template `<script>` blocks pass `node --check`.
- **B8** — `install-raycast` was run for all six agents into a scratch dir: no stale model
  ID survives in any generated script, and every agent's command line is well-formed. Both
  AppleScript branches were executed via `osascript` and print the expected command.
  *Caught during verification:* folding the trailing space into `modelArg` broke the
  `-p` prompt-flag insertion for copilot and cursor, because the Taskfile keys on the space
  before `\"/video-lens`. Fixed, and the test now guards that anchor.

### Not done in this session

`H2` (env-var clamp bypass), `H3` (scoped permissions), `T1`/`T2` (test coverage),
`U5`–`U7`, and `N3` remain the §8 "Next" batch, unchanged. `T1` should still come before
any further change to `fetch_transcript.py` — note that B3 touched that file, so it now
carries one more untested behaviour than when §4 was written.
