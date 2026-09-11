# Anders und Sehr Demo — Design

**Date:** 2026-04-26
**Demo target:** Anders und Sehr GmbH (Stuttgart). Contact: Nelson "Javier".
**Slot:** Thursday 2026-04-30, 60 min, in their office.
**Audience framing:** prospective customer + prospective partner / reseller.

## 1. Pitch & framing

**One-line pitch.** "One Slack message starts an autonomous engineer that
keeps shipping PRs on your repo until the backlog is empty. The product
it builds runs entirely on the customer's own server — no data leaves
the box."

This dual angle — autonomy on the dev side, locality on the runtime side —
is the spine of the demo. It addresses both halves of an agency's reality:
"how do my devs ship faster" and "how do I deliver AI to clients who can't
use the cloud."

**Why the dual angle is the right one for this audience.** German
Mittelstand and industrial clients dominate the Stuttgart agency portfolio,
and DSGVO posture is non-negotiable for those clients. Showing local-LLM
runtime closes the most common objection before it's raised. Showing
autonomous dev throughput closes the cost objection ("our devs are
expensive").

## 2. The artefact: TYPO3 extension "AI Editorial Helper"

**Why TYPO3.** TYPO3 is the dominant CMS in German agency / enterprise.
Stuttgart sits in the middle of that ecosystem. Picking TYPO3 signals
"we understand your world" before the first command runs.

**Why a greenfield extension (not a fix in TYPO3 core).** TYPO3 core uses
Gerrit, not GitHub PRs natively, and the codebase is enormous. A greenfield
TYPO3 extension is right-sized for kairos to build end-to-end during a
rehearsal-and-live window, while still being TYPO3-native and demonstrably
useful.

**Scope: 6 features, one per GitHub issue.**

1. SEO meta description + page title generator
2. Auto-tag / category suggester
3. Teaser / excerpt generator
4. URL slug suggester
5. DE↔EN translation stub
6. Content quality flags (sentence length, missing headings, tone)

These features share a single `LmStudioClient` (OpenAI-compatible HTTP). Each lands as one PR.
Dropped from earlier consideration: comment/spam moderation (requires
comments infrastructure no longer in TYPO3 v13 core).

**Runtime LLM: LM Studio with Qwen 3 14B (model id `qwen/qwen3-14b`).**
- Strong German↔English multilingual capability; meaningful step up from
  Qwen 2.5 7B in coherence and instruction-following.
- LM Studio handles Qwen 3's thinking blocks internally — responses come out
  clean (no `<think>` leakage to the extension), verified locally.
- Bigger model means slower per-token output but the demo isn't latency-bound.
  Generation completes in 1-3 seconds for the editorial tasks (meta descr,
  teaser, translation stub) — well within demo pacing.
- Memory: 14B at typical quantization needs ~12-18 GB RAM available — already
  loaded on the demo laptop.
- Wire format: HTTP to `localhost:1234/v1` — LM Studio exposes an
  OpenAI-compatible API (`/v1/chat/completions`, `/v1/completions`,
  `response_format: { type: "json_object" }` for structured output).
- Demo bonus: LM Studio is a polished GUI app, so the audience can *see* the
  model loaded in its window — visual proof of locality without opening a
  terminal. The agency aesthetic likes this.
- Operationally honest: same OpenAI-compatible wire format works against
  llama.cpp's HTTP server or Ollama's OpenAI-compat mode, so the extension's
  `LmStudioClient` is portable to other local runtimes via a config flip.

**Two LLMs to keep separate in the audience's mind.**
- Agent loop (kairos workers): Anthropic, via Claude Code, under your real
  subscription. Writes the code.
- Runtime (the extension): LM Studio + Qwen 3 14B, local on the customer's
  machine. Operates on the customer's content.

The network-cable moment in Act 3 only proves the second one is offline —
that's still the GDPR pitch, because the customer's *content* is what never
leaves.

## 3. The 60-minute narrative arc

**Opening — 5 min.** Coffee, small talk. One-sentence frame. No slides.

**Act 1 — "What happened overnight." 10 min.** Open Slack on the projector.
Scroll to Wednesday night's thread. Show the seed message
(`/warroom plan "..."`), the planning output, the issue list, the
follow-up `/warroom kairos start typo3-ai-helper --turbo`. Walk the
event stream as kairos cycled through observe → judge → act → log → next.

**Act 2 — "What it built." 15 min.** GitHub. Six PRs from `kairos-bot`,
all green CI. Read two diffs together — the meta description feature and
the translation feature. Highlight the `LmStudioClient`, the prompts,
the unit tests. This is where the partner / reseller pitch lands ("this
is the code your team would inherit").

**Act 3 — "What it does." 15 min.** Switch to a pre-warmed DDEV TYPO3 v13.
Extension already merged via `composer require kairos/typo3-ai-helper:dev-main`.
TYPO3 backend → page with German content → AI Editorial Helper sidebar.
Click "Vorschlag generieren" (meta description) → result. "Übersetzen → EN"
→ result. "Qualität prüfen" → flags.

**The network-cable moment.** Pull the network cable mid-sidebar-use.
Generate one more suggestion. It still works. ("No internet. The AI is
running on this MacBook. Imagine this is the customer's server.") Plug
back in.

**Act 4 — "Watch it happen live." 10 min.** Slack. Type live:
`/warroom plan "add SEO score widget to AI Editorial Helper"` →
`/warroom kairos start typo3-ai-helper --turbo`. Slack starts streaming
events. Use the wait time for Q&A. When a PR notification fires
mid-conversation, that *is* the close.

**Q&A and close — 5 min.** Leave-behind: link to the public repo,
the Slack thread export, and a Loom of Tuesday's rehearsal as a
reference recording.

**Deliberate choices in this arc.**
- Slack first, slides never. Earn explanation by showing artefacts.
- Network-cable moment is the GDPR anchor and the memorable beat.
- Live trigger in Act 4 neutralises the "this is fake" reflex.
- Q&A overlaps the live agent run, so dead air is filled by the
  system itself, not by us.

## 4. Architecture

```
SLACK
  │
  │ /warroom plan "..."     /warroom kairos start <repo> [--turbo]
  ▼
WARROOM BOT (Mac Mini)
  │   ├── Planner       → drafts spec, files 6 GitHub issues
  │   ├── Launcher      → spawns ONE backant-kairos daemon at the workspace
  │   └── Log forwarder → tails kairos's structured log → posts to Slack
  ▼
KAIROS DAEMON  (one process, persistent, autonomous)
  │   backant.io subscription auth ($20/mo seat) — gates skill fetch only
  │   pulls skills from backant API → temp skill dir → spawns Claude Code
  │   loop: observe → judge → act → log → next issue → repeat
  │   ships PRs sequentially; stays alive between them
  │   LLM cost: Anthropic per-token billing (separate, your account)
  ▼
GITHUB  (PRs land into typo3-ai-helper one after another)

─────────────────────────────────────────
DEMO ARTIFACT (separate path, runs on laptop):
  TYPO3 v13 in DDEV → AI Editorial Helper extension → HTTP (OpenAI-compat)
                                                          ↓
                                       LM Studio @ localhost:1234 (Qwen 3 14B)
                                                          ←local, offline
```

**Integration is intentionally thin and one-directional.** Warroom invokes
kairos. Kairos has no knowledge of warroom. Both products remain shippable
independently.

### What's already built (used as-is)

- **kairos:** daemon, subscription auth, skill fetch, observe/judge/act/log/
  retry/dream loop, status/watch/stop. Verified in
  `backant-kairos/src/{commands,daemon}` — daemon owns one workspace,
  state in `~/.claude/kairos/`, sleep duration emitted by each cycle as
  `KAIROS_SLEEP:<seconds>` (max 1800), default 120s, per-workspace
  `min_sleep` floor in `.backant.toml`.
- **warroom:** Slack bot, `/warroom plan`, GitHub issue creation,
  SQLite tracking. Verified in `backant-warroom/src/{bot,core}`.
- **Two distinct billing relationships, both real, no bypass:**
  - backant.io subscription ($20/mo) — gates kairos skill fetch.
  - Anthropic per-token — pays for the Claude Code calls the loop makes.

### What's new — only inside warroom

1. `/warroom kairos start <repo-alias> [--turbo]` slash command. Resolves
   alias to workspace path, runs `backant-kairos start <workspace>` (with
   `--turbo` if requested) as child process, records launch in warroom
   SQLite.
2. Log forwarder. Tails kairos log output (under `~/.claude/kairos/logs/`)
   and reformats key events into Slack messages. Filter list:
   `act started`, `PR opened`, `error`, `cycle complete` go to channel;
   verbose `observe` / `judge` deltas go to a thread or are suppressed.
3. `/warroom kairos stop` and `/warroom kairos status` — thin wrappers
   over `backant-kairos stop|status`. Status is a nice-to-have for
   Thursday, stop is needed for the live Act-4 control.

### What's new — only inside kairos

1. `--turbo` flag on `backant-kairos start`. Sets `KAIROS_TURBO=1` env var
   passed to the daemon. In `kairos-daemon.sh`, after the existing
   `extract_sleep_duration`, override `sleep_duration=1` when
   `KAIROS_TURBO=1`, regardless of cycle output or `min_sleep`.
2. Startup warning printed in yellow:
   > Turbo mode: kairos will not sleep between cycles. Token usage will
   > rise sharply. Stop the daemon at any time with `backant-kairos stop`.
3. Same warning surfaced in Slack when launched via warroom.

Turbo serves both the demo and any solo kairos user racing a deadline,
so it's a real product feature, not a demo hack.

### What we explicitly don't touch

- Kairos's autonomy semantics. Kairos picks issue order on its own.
- Worker fan-out / parallel kairos workers / issue-bound mode. Kairos is
  a single-workspace daemon by design; we don't fight that.
- Subscription auth. Real billing account, real subscription, no bypass.
- Persistence semantics. Kairos already runs across PRs; we let it run.
- Issue priority labels. Issue list is shaped so any starting point makes
  narrative sense.

## 5. Pre-demo prep & rehearsal

Four working days (Sun–Wed), sequenced.

**Sun (today).** Build integration & extension stub.
- Land warroom changes: `/warroom kairos start|stop|status`, log forwarder
  with filter list.
- Land kairos `--turbo` (CLI parse → env var → daemon override → warning).
- Bootstrap `typo3-ai-helper` repo: minimal valid TYPO3 v13 extension
  scaffold. Kairos fills in features.
- Bootstrap DDEV TYPO3 v13: `ddev config --project-type=typo3
  --php-version=8.3` + `ddev composer create
  typo3/cms-base-distribution:^13`. Verify backend loads.
- LM Studio installed, Qwen 3 14B loaded, local server running on port 1234
  (OpenAI-compatible API). Confirmed live via `curl /v1/models`.
  **Test offline (airplane mode).** The network-cable moment dies if
  this fails.

**Mon.** Dry run #1, single-feature scope.
- File one issue manually (meta description).
- Run `/warroom kairos start typo3-ai-helper --turbo` end-to-end.
- Watch Slack stream. Watch the PR. Read the diff. Composer require it
  into DDEV. Click through the backend.
- Tune: Slack log filter (signal vs noise), kairos awareness/context
  files so it knows it's working on a TYPO3 extension.

**Tue.** Dry run #2, full 6-issue scope.
- File all 6 issues via `/warroom plan "..."`.
- Run full demo flow. Time it. Note stalls.
- Save Slack thread, issue list, PR set as the artefact for Act 1.
- Verify all 6 PRs together compose into a working extension in DDEV.
  If one PR breaks another's tests, fix the issue prompts now.

**Wed afternoon.** Final rehearsal.
- Reset demo environment. Trigger `/warroom plan` and
  `/warroom kairos start --turbo` Wednesday evening.
- Wake up Thursday to the artefacts you demo. This *is* the demo.
- Backup: keep Tuesday's full artefact set as known-good fallback.

**Thu morning.** Smoke checks.
- DDEV up, extension installs, LM Studio model loaded and server running on :1234.
- Charge laptop. Charge hotspot. Pack adapter for their projector.
- Loom of Tuesday's full run on laptop as last-resort fallback.

**Estimated effort.** Sunday build is the bulk: ~1–1.5 dev-days for
warroom commands + log forwarder + turbo flag combined. If Sunday slips,
Monday absorbs the overflow and dry run #1 moves to Tuesday morning.

## 6. Risks & fallbacks

| Failure | Likelihood | Mitigation |
|---|---|---|
| Office wifi flaky / locked-down | High | Personal hotspot tethered before the meeting starts. Test in lobby. |
| Wifi dies mid-Act-4 | Medium | Pivot to Tuesday's Loom — same point, lower risk. |
| Kairos picks confusing first issue | Medium | Wednesday's overnight is the demo. Live Act-4 is small and contained. |
| Kairos stalls/retries forever | Low–Medium | Worst case: `backant-kairos stop` mid-demo, narrate "you can interrupt at any time." |
| LM Studio server not running or no model loaded | Low | LM Studio open with model loaded before leaving. Smoke test before leaving. Extension surfaces a clear "load a model" message if it happens. |
| Local LLM gives garbage German | Medium-Low | Tuesday rehearsal validates. Llama 3.1 8B as one-line-config fallback. |
| Anthropic API outage | Low | No reliable mitigation. Pivot to Loom. |
| GitHub outage | Low | Pivot to local artefacts. Screenshot PRs Wednesday night as static fallback. |
| DDEV containers broken Thu morning | Medium | Smoke test before leaving. `ddev restart` fixes 80%. Drop DDEV finale, narrate screenshots, end on Slack — lose the network-cable moment, keep the rest. |
| Token bill from turbo concerning | Certain to be noticed | Stop daemon at end of Act 4. Don't leave running through Q&A. Mention turbo cost upfront if pricing comes up. |
| Unexpected stakeholders attend | High | Demo plays to both customer and partner angles. Have one anecdote ready for client-deliverable / per-client-mac-mini. |
| Nelson runs TYPO3 v11/v12, not v13 | Medium | Extension v12-compatible with one composer.json line. If it comes up: "kairos can rewrite for v12 in 20 min — should we?" Don't do it live unless asked. |
| Question we genuinely don't know | High | "Good question, follow up Friday." Don't bullshit. Take the note. |

**Single biggest mitigation, repeated.** Wednesday's overnight run *is*
the artefact you demo. Live Act-4 is proof-of-life on top, not the main
act. If everything live breaks, you still have a complete demo.

## 7. Open items / out of scope

These are deliberately deferred, not forgotten.

- **Multi-repo orchestration in one warroom command.** One `/warroom kairos
  start` per repo for now.
- **Kairos session federation across multiple workspaces.** One workspace
  per daemon instance, by current kairos design.
- **Per-client billing federation.** Each client running kairos has their
  own subscription; demo runs on yours. Federation is a real product
  question for the partner pitch and gets a "let's discuss after Thursday."
- **Web log / TUI / dashboard for Slack thread.** The JSONL-or-text log
  forwarder is the minimum; richer surfaces come later.
- **LM Studio → llama.cpp / Ollama swap.** Same OpenAI-compat wire format with thin adapter; left
  to ops conversation post-demo.
- **TYPO3 v12 backport.** Possible, deferred unless Nelson asks.

## 8. Success criteria for Thursday

- Nelson sees at least 4 of the 6 PRs from Wednesday's overnight run as
  real artefacts (not slides, not mockups).
- The DDEV TYPO3 backend demonstrably uses the extension on real German
  content.
- The network-cable moment lands without scrambling.
- The live Act-4 trigger produces at least Slack activity within ~60s
  of typing the command, and ideally one merged or open PR before the
  meeting ends.
- We leave with a clear next step: pilot proposal, follow-up call, or
  scoped paid engagement.
