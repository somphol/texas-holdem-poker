# AGENTS.md — NIGHT SHIFT · TEXAS HOLD'EM

> Handoff for the next maintainer or agent. Rebuild detail is in
> [docs/REPRODUCTION.md](docs/REPRODUCTION.md); the technical map is
> [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md).

## 1. What this is

A single-file **no-limit Texas Hold'em tournament** against 1–5 AI opponents: 52-card
deck, full betting rounds, proper side pots, a 7-card evaluator, Monte Carlo bots, and a
Web Audio sound layer. Everything — markup, CSS, game logic — lives in `index.html`
(3,011 lines). No build, no dependencies, no network calls, no persistence.

It is **not** real-money gambling, not networked multiplayer, and not a poker trainer
with hand history. A refresh starts a new tournament. Live copy:
<https://somphol.github.io/texas-holdem-poker/>. Local master:
`~/Projects/texas-holdem-poker`. Current release: **v1.0.0**.

## 2. Commands

```bash
cd ~/Projects/texas-holdem-poker

# run (no install, no build)
open index.html
python3 -m http.server 8000        # http://localhost:8000/

# verify (macOS, Chrome installed)
"/Applications/Google Chrome.app/Contents/MacOS/Google Chrome" \
  --headless=new --disable-gpu --virtual-time-budget=3000 \
  --dump-dom "file://$PWD/index.html" | grep 'id="setupTitle"'
curl -s -o /dev/null -w '%{http_code}\n' https://somphol.github.io/texas-holdem-poker/

# deploy — Pages serves main / root; no workflow
git push origin main
```

Headless checks cover the setup screen and rendering only; a full hand must be played by
hand. The matrix is in [docs/TESTING.md](docs/TESTING.md).

## 3. Architecture

| Lines | Block |
| --- | --- |
| 1–9 | head, meta, Open Graph |
| 10–1324 | CSS: table, seats, cards, HUD, modals, responsive rules |
| 1343–1504 | body markup: table, HUD, setup and end modals |
| 1505–3009 | one script: helpers, data, state, engine, UI, wiring |

Script groups, in order:

1. **Helpers** — `$`, `$$`, `clamp`, `delay`, `money`, card labels, second-person grammar.
2. **Data** — `SUITS`, rank tables, `BOT_PROFILES` (8 bots with avatar, colour,
   `aggression`, `bluff`), `SEAT_LAYOUTS` (desktop and compact, 1–5 opponents).
3. **Audio** — `initAudio`, `tone`, `sfx`; an `AudioContext` created on the first
   gesture, guarded by `G.sound`.
4. **Evaluator** — `makeDeck`, `shuffle` (Fisher–Yates), `score5`, `evaluate7` (best of
   21 five-card combinations), `compareScores`, `describeScore`.
5. **Equity** — `monteCarloEquity(hole, board, opponents, iterations)`: random rollouts
   against live opponents, half-credit for ties.
6. **State** — `G` is the whole game (players, board, pot, street, dealer, config), plus
   `humanResolver`, `lastConfig`, `lastSound`.
7. **Engine** — `startGame`, `playHand`, `bettingRound`, `getAction`, `applyAction`,
   `commit`, `botDecide`, `buildPots`, `finishHand`, `endHand`.
8. **UI** — `buildSeats`, `applySeatLayout`, `renderAll`, `renderHud`, card rendering,
   action log, modals, setup parsing, key bindings, `boot()`.

Critical paths:

- **Betting** — `bettingRound(gen)` loops `nextActionIndex`, pausing for the human via
  `humanResolver`; `applyAction` validates and applies fold/check/call/raise/all-in, then
  `commit` moves chips and reconciles.
- **Bots** — `botDecide` computes equity (110 rollouts pre-flop, 90 after), compares it
  with pot odds (softened pre-flop), then raises/calls/folds/bluffs by profile.
- **Pots** — `buildPots` tiers contributions by `totalBet` and assigns eligibility per
  tier, so short all-ins can only win what they covered.
- **Tournament** — busted players are `eliminated` and sit out; the last stack wins and
  `showEnd('champion')` renders; a human bust shows `showEnd('bust')`.

## 4. Decisions and constraints

- **One file is the product.** No modules, assets or fetches. Cards are CSS/HTML, not
  images; sound is synthesised.
- **Play money only.** Never add real-money features, accounts or payment links.
- **No persistence is deliberate.** There is no save, and `localStorage` is unused; a
  reload is a new tournament. Adding a save changes the product.
- **Chip conservation is a hard invariant.** Every action must leave
  `sum(balances) + sum(bets) + pot` unchanged. `buildPots` must return the full pot.
- **Monte Carlo is bounded.** 90–110 rollouts keeps bot decisions under a frame budget;
  raising it makes the table feel sluggish, lowering it makes bots erratic.
- **Evaluator is exhaustive.** 21 combinations per 7-card hand is fast enough and avoids
  the class of bugs that shortcut evaluators introduce.
- **Audio starts on a gesture.** Browsers block autoplay; `initAudio()` is called from the
  Sit Down handler.
- **Bot identities are cosmetic.** `BOT_PROFILES` order is shuffled per game, so the same
  bot can appear at any seat.

## 5. Known issues

- No save/resume; closing the tab ends the tournament.
- Bots are heuristic, not solvers; they can be exploited by consistent over-betting.
- `monteCarloEquity` uses `Math.random()`, so two identical situations can produce
  different actions. That is expected.
- The README claims testing in Chrome, Safari, Firefox and Edge; this documentation pass
  re-verified Chromium only. Treat the other browsers as unverified.
- No automated gameplay tests. Headless checks stop at the setup modal.
- The 3D edition linked from the setup modal (`texas-holdem-3d`) is a separate project
  with its own repo and docs.

## 6. Continuity

- Live: <https://somphol.github.io/texas-holdem-poker/> (HTTP 200 at the time of writing).
- Pages: `main` branch, repository root, `.nojekyll`, no Actions workflow.
- Local master copy: `~/Projects/texas-holdem-poker`.
- First build: `8803f42` "Night Shift: single-file Texas Hold'em poker game".
- Changelog: `CHANGELOG.md`.
