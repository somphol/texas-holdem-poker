# Reproduction book

How to rebuild **Night Shift · Texas Hold'em** from scratch, or verify a rebuild. Read
[ARCHITECTURE.md](ARCHITECTURE.md) first.

> **The source is the truth.** `index.html` on `main` is the release. Rebuild only when
> it is lost or you are deliberately making a variant.

## 1. What is being reproduced

A single-file **no-limit Texas Hold'em tournament**:

- **Setup** — name, starting balance, 1–5 AI opponents, small/big blind, validation.
- **Play** — pre-flop, flop, turn, river, showdown; fold, check, call, raise (slider and
  field), all-in.
- **Rules** — minimum raises, short all-ins, tiered side pots, split pots, odd chips
  clockwise from the dealer, uncalled bets returned.
- **AI** — Monte Carlo equity against live opponents, pot-odds comparison, per-profile
  aggression and bluff frequency.
- **Evaluator** — best five of seven, all 21 combinations, category + tie-breakers.
- **Tournament** — busted players sit out; last stack wins; bust and champion screens;
  instant rematch. Bots start with the player's stack.
- **Presentation** — oval felt table, seat badges, CSS playing cards, pot/board, winner
  glow, scrolling log, metallic gold/cyan palette, synthesised sound effects.
- **Delivery** — one `index.html`, no build, no assets, no network, no save. GitHub
  Pages, `main` / root.
- **Release** — v1.0.0.

## 2. Requirements matrix

| # | Requirement | Implementation | Where |
| --- | --- | --- | --- |
| 1 | Table setup: name, stack, opponents, blinds | setup modal, `readSetup`, `startGame` | markup + script |
| 2 | Full betting rounds and actions | `bettingRound`, `applyAction`, `commit` | script |
| 3 | No-limit legality (min raise, short all-in) | validation in `applyAction` | script |
| 4 | Side pots and split pots | `buildPots`, `finishHand` | script |
| 5 | AI with equity and pot odds | `monteCarloEquity`, `botDecide` | script |
| 6 | Per-bot personality | `BOT_PROFILES` (`aggression`, `bluff`) | script |
| 7 | 7-card evaluation | `score5`, `evaluate7`, `compareScores` | script |
| 8 | Tournament elimination and champion | `endHand`, `showEnd` | script |
| 9 | Live table UI and log | `renderAll`, `renderHud`, `logEntry` | script + CSS |
| 10 | Sound, muteable | `initAudio`, `tone`, `sfx`, header toggle | script |
| 11 | Keyboard shortcuts | keydown handler (`F`, `C`, `R`, `A`, `Enter`) | script |
| 12 | Single-file static hosting | `.nojekyll`, `robots.txt`, `sitemap.xml` | repo root |

## 3. Build order

1. **Shell** — head/meta, CSS variables, table and seat markup, HUD, action log, modals.
2. **Cards and data** — suits, rank labels, `makeDeck`, `shuffle`; CSS card faces.
3. **Evaluator** — `score5`, `evaluate7`, `compareScores`, `describeScore`; test these
   before any table code depends on them.
4. **State and setup** — `G` skeleton, setup modal, `readSetup`, `startGame`.
5. **Betting engine** — `commit`, `applyAction`, `bettingRound`, blinds; verify chip
   conservation.
6. **Pots and showdown** — `buildPots`, `finishHand`, odd-chip rule.
7. **Bots** — `monteCarloEquity` first, then `botDecide` thresholds and profiles.
8. **Flow** — `playHand`, `endHand`, bust and champion screens, rematch.
9. **Presentation** — `buildSeats`, `applySeatLayout`, `renderAll`, animations, log.
10. **Audio and input** — `initAudio`, `sfx`, keyboard, mute.
11. **Metadata and docs** — README, CHANGELOG, social card, sitemap, this book.

## 4. Acceptance checks

A rebuild is done when, in a desktop browser:

1. The setup modal accepts a name, balance, 1–5 opponents and blinds; invalid input
   (bb ≤ sb, balance < bb, balance > 10M) is rejected with a message.
2. Sit Down deals two cards, posts blinds and opens pre-flop action.
3. Fold/check/call/raise/all-in are each only enabled when legal; the slider matches the
   min/max.
4. A short all-in creates a side pot; a split pot divides correctly and odd chips go
   clockwise from the dealer.
5. Total chips stay constant across every hand (check stack + bets + pot).
6. Bots fold weak hands, raise strong ones, and occasionally bluff; no bot acts out of
   turn.
7. The board advances pre-flop → flop → turn → river → showdown with correct card counts.
8. A busted player is marked out and not dealt in; the last stack shows the champion
   screen; rematch returns to setup with the previous values prefilled.
9. The action log describes each street and result; it scrolls and does not overflow.
10. Keyboard shortcuts work on the human's turn and are inert otherwise.
11. The table stays usable at ~390 px wide.
12. Muting stops sound immediately; sound starts only after a user gesture.

## 5. Provenance

| Item | Value |
| --- | --- |
| Initial build | `8803f42` "Night Shift: single-file Texas Hold'em poker game" |
| 3D edition link | `10275d6` |
| Social card + README header | `6403ba0` |
| Repo docs and community files | `e577442` |
| Release docs | this commit (`AGENTS.md` + `docs/`) |
| License | MIT |

`CHANGELOG.md` is the version record.
