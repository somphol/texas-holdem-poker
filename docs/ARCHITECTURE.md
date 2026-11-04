# Architecture

One HTML file, 3,011 lines. No framework, no build, no assets. This is the map.

## 1. File layout

| Lines | Block | Contents |
| --- | --- | --- |
| 1–9 | head | meta, Open Graph/Twitter cards, title |
| 10–1324 | CSS | felt table, seat positions, CSS playing cards, HUD, modals, action log, responsive rules |
| 1343–1504 | body | table, HUD, setup modal, end-of-tournament modal |
| 1505–3009 | script | helpers → data → audio → evaluator → equity → engine → UI → boot |

The two modal overlays live in the body and are shown/hidden via the `hidden` attribute.
There is no router and no view layer: `renderAll()` redraws from `G` after every change.

## 2. State

```js
G = {
  players: [...],      // seat 0 = human, then bots
  board: [],           // community cards
  deck: [],
  pot, street, currentBet, minRaise,
  dealerIndex, actorIndex, handNo,
  sb, bb, sound, busy, humanTurn, summary, over, ...
}
humanResolver           // resolves the promise that waits for the human action
lastConfig, lastSound   // used to prefill the next setup screen
```

Player objects carry `balance`, `hole`, `bet`, `totalBet`, `folded`, `allIn`, `acted`,
`eliminated`, `reveal`, `winner`, `lastAction`, `handDesc`, `eval`, and bot traits
(`aggression`, `bluff`).

No global persistence: `G` is rebuilt by `startGame()` and discarded on reload.

## 3. Data

- **Deck** — four suits × ranks 2–14, `makeDeck()`; `shuffle()` is Fisher–Yates.
- **Bot profiles** — 8 entries with `name`, `avatar`, `color`, `aggression`, `bluff`;
  `startGame` shuffles and takes `config.aiCount`, so line-ups vary per game.
- **Seat layouts** — percentage coordinates per opponent count, in `desktop` and
  `compact` variants, applied by `applySeatLayout()`.

## 4. Hand evaluation

`score5(cards)` returns a comparable score: category rank plus up to five tie-breaker
ranks. `evaluate7(cards)` evaluates all **21** five-card combinations and keeps the best.
`compareScores` orders them; equality means a split. `describeScore` produces the text
shown at showdown ("Full House, Kings over Nines").

This is exhaustive rather than table-based: slower than a lookup evaluator but far less
error-prone, and plenty fast at one hand at a time.

## 5. Bot equity

```js
monteCarloEquity(hole, board, opponents, iterations)
```

Builds the unknown deck, then for each rollout deals the remaining board plus two cards
per live opponent, evaluates all hands, and scores a win as 1 and a tie as 0.5. Iterations
are floored at 30; `botDecide` uses **110 pre-flop** and **90 post-flop**.

`botDecide` then:

1. Computes `toCall`, `pot` and `potOdds = toCall / (pot + toCall)`.
2. Softens the odds pre-flop (`× 0.62`) so early aggression is not automatically folded.
3. Compares equity to adjusted odds, applies the bot's `aggression` multiplier.
4. Rolls for `bluff` frequency.
5. Chooses fold / check / call / raise (sized from the pot) / all-in, clamped to legal
   limits.

Bots are heuristic, not equilibrium solvers.

## 6. Betting engine

`playHand()` is an `async` flow paced by `delay`, so the UI can animate between actions.
Every hand uses a generation counter: `playHand` increments `G.gen`, passes the value to
`bettingRound(gen)` and `finishHand(gen)`, and each loop exits when `gen !== G.gen`. A
rematch or reset therefore abandons an in-flight hand instead of racing it.

1. Reset per-hand state, move the dealer button, post blinds (`postBlind`).
2. Deal hole cards.
3. For each street (pre-flop, flop, turn, river): `bettingRound(gen)`, then deal the
   street's cards.
4. `finishHand(gen)` builds pots, evaluates, awards, logs, and either starts the next hand
   or ends the tournament.

`bettingRound` walks seats with `nextActionIndex`, skips folded/all-in players, and stops
when the action closes. For the human it awaits a Promise stored in `humanResolver`;
`humanAction()` resolves it. Bot decisions go through `botDecide`.

`applyAction` validates legality (minimum raise, all-in shortfall, check only when no bet
is owed) and calls `commit(p, amount)`, which moves chips from `balance` into `bet` and
`totalBet` and later sweeps bets into the pot.

## 7. Pots

`buildPots()` sorts distinct contribution levels, slices each tier, and assigns eligible
players (`inHand && totalBet >= level`). Identical eligibility sets are merged. This gives
correct side pots for short all-ins and leaves uncalled excess in a tier only the bettor
can win.

`finishHand` evaluates each eligible player's 7-card hand, splits each pot among the best,
and gives odd chips to the first winner clockwise from the dealer.

## 8. UI and input

- `renderAll()` → `renderHud`, `buildSeats`/`applySeatLayout`, board, pot, log, buttons.
- `renderHud` shows stack, pot, street, and to-call; action buttons enable only on the
  human's turn and only for legal actions.
- `logEntry(text, type)` appends to the scrolling action log (capped).
- Keyboard (only on the human's turn): `F` fold, `C` check/call, `R`/`Enter` raise,
  `A` all-in. When a hand summary is showing, `Enter`/`Space` deals the next hand.
- `sfx(kind)` plays short synthesised tones; `initAudio()` runs on Sit Down.
- Setup validation lives in `readSetup()`: positive balance ≤ 10,000,000, blinds ≥ 1,
  `bb > sb`, balance ≥ `bb`, `bb ≤ 1,000,000`.

## 9. Change guide

| To change… | Touch |
| --- | --- |
| Table look | CSS token blocks and `.table` / `.seat` rules |
| Bot difficulty | `BOT_PROFILES` values, `botDecide` thresholds, iteration counts |
| Betting rules | `applyAction`, `commit`, `buildPots`, `bettingRound` |
| Hand ranking text | `describeScore` |
| Setup defaults | the input values in the modal and `readSetup` validation |
| Seat positions | `SEAT_LAYOUTS` (keep both desktop and compact) |
| Pacing | the `delay`/`nap` calls in `playHand` and `bettingRound` |

Keep chips conserved. If a change can create or destroy money, the tournament is broken.
