# Testing

## Status of this release

| Evidence | When | Result |
| --- | --- | --- |
| Live Pages probe (`curl`) | 2026-11-03 | **200** |
| Headless Chromium render, `file://` | 2026-11-03 | **setup modal present, 5 AI options, Sit Down button** |
| Headless screenshot (1280 × 900) | 2026-11-03 | **rendered, 374 KB PNG** |
| Manual full-hand pass (§3) | — | **Not run** in this release |
| Safari / Firefox / Edge / mobile | — | **Not verified** |

The documentation commit changed Markdown only. Headless Chrome renders the setup screen
but cannot click Sit Down, act on a turn or finish a hand, so **gameplay is unverified by
automation**. Anything touching betting, pots or the evaluator needs §3.

There is no CI and no committed test harness.

## 1. Commands

```bash
cd ~/Projects/texas-holdem-poker

# live
curl -s -o /dev/null -w '%{http_code}\n' https://somphol.github.io/texas-holdem-poker/

# headless: expect the setup dialog title
"/Applications/Google Chrome.app/Contents/MacOS/Google Chrome" \
  --headless=new --disable-gpu --virtual-time-budget=3000 \
  --dump-dom "file://$PWD/index.html" | grep 'id="setupTitle"'

# screenshot: expect a large, non-blank PNG
"/Applications/Google Chrome.app/Contents/MacOS/Google Chrome" \
  --headless=new --disable-gpu --virtual-time-budget=3000 \
  --window-size=1280,900 --screenshot=/tmp/th.png \
  "file://$PWD/index.html"
ls -l /tmp/th.png

# play
python3 -m http.server 8000     # http://localhost:8000/
```

## 2. Static results (2026-11-03)

- Live site: HTTP `200`.
- Headless DOM: setup dialog with `id="setupTitle"` ("Take a Seat"), five `data-ai`
  buttons, `id="startBtn"` ("Sit Down & Deal"), no missing-resource errors.
- Screenshot: 1280 × 900, **374,059 bytes** — the table and modal painted.
- No network requests: the file references no external script, font or image.

## 3. Manual gameplay matrix

Play at least one full tournament, or until the checks below are covered. Use the
browser console to assert chip conservation after each hand:

```js
// paste after a hand settles
G.players.reduce((s, p) => s + p.balance + p.totalBet, 0) // stacks + pot; must stay constant
```

| # | Check | Expected |
| --- | --- | --- |
| 1 | Load the page | setup modal, defaults: balance 1000, 3 AI, blinds 10/20 |
| 2 | Invalid setups | bb ≤ sb, balance < bb, balance > 10M, blank name all rejected with a message |
| 3 | Sit Down | blinds posted, hole cards dealt, pre-flop action starts left of the big blind |
| 4 | Fold / Check / Call | only legal buttons enabled; `F`/`C` match the buttons |
| 5 | Raise | slider min/max correct; `R` and Enter submit the shown amount |
| 6 | All-In (short) | side pot created; short stack can only win the main pot |
| 7 | Split pot | chips divide evenly; odd chip goes clockwise from the dealer |
| 8 | Board timing | flop 3, turn 1, river 1, then showdown |
| 9 | Bot behaviour | no out-of-turn actions; weak hands fold, strong hands raise, bluffs appear |
| 10 | Bust | eliminated player sits out; seats and blinds adjust |
| 11 | Champion | last stack shows the champion screen; rematch prefills the last setup |
| 12 | Log | every street and result logged, scrolls, no overflow |
| 13 | Sound | starts after Sit Down, mute stops it immediately |
| 14 | Keyboard | `F`/`C`/`R`/`A` work only on the human's turn; Enter/Space deals the next hand |
| 15 | 390 px viewport | table remains usable, no horizontal scroll |
| 16 | Reload | tournament is gone and setup returns (no persistence, by design) |

## 4. Failure triage

- **Chips appear or vanish** — an action bypassed `commit` or `buildPots` dropped a tier.
  Reproduce with a short all-in.
- **Bots acting out of turn** — a stale hand is still running; check the `G.gen` guard.
- **Buttons enabled when they should not be** — `renderHud` ran before `G.currentBet` /
  `G.actorIndex` settled.
- **Winner gets the wrong pot** — eligibility in `buildPots` (uses `inHand` and
  `totalBet`) or tie-breaking in `compareScores`.
- **Audio silent** — `initAudio()` must run from the Sit Down gesture; check `G.sound`.
- **Hand never ends** — an `await` on `humanResolver` was left unresolved after a reset.

## 5. Not verified

- Safari, Firefox, Edge and real mobile devices (the README claims they were tested at
  build time; this pass did not repeat it).
- Screen-reader output; the table is visual and card faces are CSS, so screen-reader use
  is likely poor.
- Long tournaments (many hands) and memory stability of the action log.
- Statistical fairness of the shuffle or the bots beyond casual observation.
