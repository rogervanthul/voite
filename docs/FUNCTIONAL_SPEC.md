# Project Overview: Voite

Voite is a real-time spectator scoring system for live kickboxing events: spectators score rounds from their phones alongside the official jury and see aggregated results reflected back in real time.

> Brand note: "Voite" blends "vote" with a phonetic nod to the Irish pronunciation of "fight."

## Value Proposition

Voite is sold to Promoters, and what it sells them is engagement:

- **Turn waiting time into show time**. The minute between rounds is the least engaging part of a fight night; Voite makes it the most interactive. Every spectator becomes a judge, and the crowd's verdict (average score and vote count) lands back on their phones moments later: live proof of how much of the house is playing along.
- **The crowd vs. the judges**. The crowd's score alongside the official result makes every decision a talking point: agreement validates the judges, disagreement becomes the story of the night.
- **Zero friction, any phone**. One QR scan; no app store, install, or account. Any online phone is in: the difference between a gimmick a few people try and half the house playing judge.
- **Nothing to run**. The card (roster, bouts, rounds, judges) is configured pre-event and driven ringside by a single Event Manager. Print the QR code, run the show.

## Ecosystem Components

- Spectator App: web app for scoring rounds and viewing scores.
- Event Manager App: web app for ringside/event management.
- Backend: services for data flow and aggregation.

## High-Level Use Cases

### Spectators

#### Views

- Fight Card View, a live overview of the card:
  - All planned fights, current fight highlighted
  - Previous fights: official score and average unofficial score with its vote count
  - Cancelled fights: shown in place, marked Cancelled, no scores
- Bout View, a live overview of one match:
  - All planned rounds; updates live if the Event Manager adds a round
  - Previous rounds: average unofficial score with vote count, or an indication that voting was disabled
- Voting View, to vote on a round during intermission:
  - If voting was skipped, the reason is shown: knockdowns (see Knockdowns) or early stoppage, i.e. (T)KO or disqualification, triggered automatically when the Event Manager stops the bout early
  - If voting is enabled, five fixed options: `10-10` (even round), or `10-9` / `10-8` in favor of either fighter. The vote button enables once an option is selected. After voting, a success message notes results appear once voting closes; the round's average and vote count then appear in place once tallied. Navigating back is up to the spectator
  - A vote arriving after the grace period (see Vote Legitimacy & Tallying) is not counted; the spectator sees a friendly "voting closed" notice, not an error

#### User Journey

The backend pushes event state live and every open view updates in place, but navigation is spectator-initiated: the UI never switches views on its own (auto-navigation is a V2 candidate, see Out of Scope).

- Event Start: access via QR code or weblink; no login, no dedicated late-join flow. Joining mid-event lands on the Fight Card View in the current event state
- Live cues instead of forced navigation: the current bout is highlighted, and while voting is open a persistent "Vote now" call-to-action on every view opens the Voting View on tap
- Reconnect / Refresh: lands on the Fight Card View in the current event state. A spectator's own votes persist locally across refreshes and backgrounded tabs; clearing browser storage or incognito use loses that history, with no server-side recovery
- Early End Event: un-fought bouts show no scores and become inactive, and any open voting closes. On resume (Start Event) the event continues where it left off and those bouts reactivate

### Event Managers

#### Views

- Fight Card View, all planned bouts:
  - Edit Card: reorder fights via drag-drop
  - Start Event: start fight night (also after accidentally ending it)
  - End Event: stop fight night, at any point
- Bout View, one fight:
  - Edit: update fighters
  - Start Bout: make this the live bout, highlighted on spectator views; available once the event is live and no other bout is in progress
  - Cancel: only before the bout starts (e.g. pre-fight injury); once underway, only Stop Bout can end it. A cancelled bout stays on the card, marked Cancelled, no scores
  - Stop Bout: end the fight early ((T)KO, disqualification, other); triggers voting-skipped for the round in progress (see Stopped Bouts)
  - Add Round: while the bout is underway, adds one round beyond the planned count (e.g. to break a tie); Event Manager only, no automatic trigger, no system-enforced limit
  - Add Final Scorecards: enter the judges' final totals (see Final Scorecards)
  - Tallies: everything spectators see (a live-updating vote count while voting is open is a V2 candidate, see Out of Scope)
- Round View:
  - Start Round: the round is underway; implicitly closes voting for the previous round if still open
  - End Round: the round is over; the same screen offers Start Voting as the prominent default and Skip Voting (knockdown, see Knockdowns) as the alternative: two taps per normal round
  - Stop Voting: closes voting explicitly; voting also closes implicitly when the next round or bout starts, the bout is stopped, or Final Scorecards are entered (see Vote Legitimacy & Tallying)

#### User Journey

The Event Manager drives the event: starting and stopping fight night, in-flight card changes (the card is authored pre-event, see Promoters), and controlling when spectators can vote.

Ringside mis-taps are inevitable, so the irreversible actions (Stop Bout, End Event, Cancel) require explicit confirmation, and Stop Bout offers a ~10-second undo window before the stoppage is pushed to spectators. Mis-entered scores are handled via Score Corrections.

### Promoters

Promoters have no self-service app in V1: the App Owner configures rosters and fight cards on their behalf via internal tooling (the Promoter web app is a V2 candidate, see Out of Scope). The configuration itself:

- Fighter roster: fighters (name, record, picture) belonging to the Promoter, persisting across events; teardown never touches it. The system never updates a record from bout results; it is maintained manually
- Fight card, one per fight night:
  - Set the number of judges (see Judges)
  - Set rounds per bout (see Round Count)
  - On creation the system generates the spectator weblink and QR code, visible in the Event Manager app for distribution (printing, venue screens)
  - Editable until Start Event; once live, in-flight changes are the Event Manager's alone (see Event Managers)

### App Owner

The App Owner (the party selling the app to Promoters) has no in-event views; their role is operational:

- Provisions accounts: in V1 one Event Manager account per Promoter, so the Promoter chooses who runs the show (Promoter accounts arrive with the V2 Promoter App)
- Configures each Promoter's roster and fight cards pre-event (see Promoters)
- Each fight night is a separate event with its own setup; nothing is shared between events
- Post-event teardown is a manual App Owner activity; no automatic expiry

## Event, Bout & Round Lifecycle

The backend pushes state off three state machines; spectator views update live in place with no forced navigation (see Spectators). The Event Manager (EM) drives every transition; nothing is timer-driven or automatic.

### Event

`Draft -> Live <-> Ended`; deleted only by manual App Owner teardown

- Draft: the card is authored (App Owner in V1, see Promoters); editable until Start Event
- Live: Start Event (EM). Card configuration locks; the EM runs bouts one at a time
- Ended: End Event (EM), at any point; remaining bouts become inactive. Start Event returns to Live, resuming where it left off
- Teardown (App Owner, manual): the event and all its data are deleted; the fighter roster is untouched

### Bout

```
Planned --> Live --> Completed   (went the distance + Final Scorecards)
   |          \----> Stopped     (Stop Bout: (T)KO, DQ, other)
   \------> Cancelled            (before Start Bout only)
```

- Planned: on the card; can be reordered, edited, cancelled
- Live: Start Bout (EM); at most one bout live at a time; rounds run within it
- Completed: last planned (or added) round ended and Final Scorecards entered; official result computed (see Official Bout Result)
- Stopped: Stop Bout at any time while live; the round in progress becomes voting-skipped; result is method-based (see Stopped Bouts)
- Cancelled: only while Planned; stays on the card, marked Cancelled, no scores
- A Stopped or Completed bout cannot return to Live (mis-entered scores: see Score Corrections)

### Round

`Upcoming -> In Progress -> Ended -> (Voting -> Tallied | Skipped)`

- In Progress: Start Round (EM); the round shows as live on spectator views
- Ended: End Round (EM); the same screen offers Start Voting (default) or Skip Voting (knockdown)
- Voting: the "Vote now" call-to-action appears and the Voting View accepts votes. Closes via Stop Voting, or implicitly on the next Start Round, Start Bout, Stop Bout, End Event, or Add Final Scorecards; the 2-second grace period then runs and the tally is computed and pushed (see Vote Legitimacy & Tallying)
- Skipped: Skip Voting (knockdowns) or the bout was stopped during the round; spectators see the reason instead of options

## Scoring & State Rules

- Judges: one event-level setting applying to every bout on the card: judge count (3 or 5). Final Scorecards are always entered, and spectators always see the official result once a bout ends. (Open Scoring, live per-round judge scores, is deferred to V2; see Out of Scope.)
- Round Count: set per bout at card creation, not event-level (e.g. a 5-round title fight on a 3-round card). Extendable via Add Round.
- Knockdowns: voting is skipped when a single fighter is knocked down once or twice, defaulting the round to `10-8` or `10-7` respectively; judged manually by the EM, no automatic detection. If both fighters go down equally often, voting stays open. The defaulted score stands in for the crowd's vote: shown as the round's unofficial score and included in the Unofficial Bout Result average.
- Final Scorecards: once the last round (including added rounds) ends, the EM enters each judge's final total per fighter (3 or 5 judges, as set for the event). This determines the official result and implicitly closes final-round voting, so the official result is never published while voting is open. Only for bouts that go the distance; stopped bouts get a method-based result (see Stopped Bouts).
- Official Bout Result: the fighter favored by a majority of the judges' final scorecards wins; an even card counts toward neither fighter; no majority means a Draw.
- Stopped Bouts: a round that goes the distance ends with a judge's score; one that doesn't (for whatever reason) is not scored and determines the bout's outcome, recorded as "Winner by (T)KO", "Winner by DQ", "Technical Draw", or "No Contest". No Final Scorecards are entered; unofficial averages from rounds voted before the stoppage remain visible.
- Unofficial Bout Result: the spectator vote average per fighter across all rounds in the bout (stopped bout: rounds voted before the stoppage). Each vote contributes its numeric score per fighter (a `10-9` for Fighter A counts as A: 10, B: 9); knockdown-skipped rounds contribute their defaulted score. Every average is displayed with its vote count: per round, votes cast that round; per bout, total votes across included rounds (knockdown rounds add none). Counts are pushed with the tally; no one sees a count while voting is open (an EM-only live count is a V2 candidate, see Out of Scope). Equal averages show as a tie; no unofficial winner is declared.
- Score Corrections: the EM can correct any official entry (Final Scorecards, Stop Bout method) until teardown; corrections push live like the original entry. A Stopped or Completed bout cannot be returned to Live. Votes are never editable or re-castable.
- Vote Legitimacy & Tallying: any vote submitted while the vote button is pressable is legitimate; button access is managed by the backend push mechanism (see Out of Scope: vote integrity). Tallying happens after a 2-second grace period following the voting-closed event, so in-flight votes still count; later votes are rejected with a "voting closed" notice.

## Non-Functional Targets

- Scale: at least 5,000 concurrent spectators per event for the MVP (10,000 preferred; development speed takes priority). On success the target grows to ~100,000; don't paint the architecture into a corner, but don't gold-plate for that now.
- Latency: EM actions (state changes, scores) visible on spectator phones within ~2 seconds; aggregated spectator scores within ~5 seconds after voting closes.

## Out of Scope

- Open Scoring (V2 candidate): a Promoter toggle that shows the judges' round scores live to spectators, for every round except the last. The EM would enter each judge's score after each round via quick-pick of the standard scores (`10-10`, or `10-9` / `10-8` / `10-7` in either fighter's favor) with free numeric entry as a fallback (e.g. point deductions), one column per judge, pushed to spectators immediately and correctable like any official entry. No effect on the official result, which always comes from Final Scorecards; cut from V1 to shrink the build.
- Promoter self-service app (V2 candidate): in V1 the App Owner configures rosters and fight cards on the Promoter's behalf via internal tooling (see Promoters); V2 gives Promoters their own web app and accounts.
- EM live tally (V2 candidate): a live-updating vote count on the Event Manager's Bout View while voting is open, to gauge participation before closing the vote. In V1 the EM sees counts only once the tally is pushed, like spectators. Cut from V1 to shrink the build.
- Auto-navigation in the Spectator App (V2 candidate): the MVP never force-switches a spectator's view; navigation is guided by live cues (highlighted current bout, "Vote now" call-to-action; see Spectators).
- Correction indicators (V2 candidate): corrections push live with no visual marker and can read as a glitch to spectators; a small "corrected" badge.
- Post-teardown landing page (V2 candidate): after teardown, printed QR codes and the weblink resolve to nothing; a generic "this event has ended" page.
- Vote integrity / anti-fraud: no safeguard against ballot-stuffing (multiple tabs, devices, scripted votes).
- Multiple concurrent Event Managers: single admin per event.
- Past events: no records kept (teardown is manual; see App Owner).
- Cross-event data: each fight card's event data is fully independent; no shared spectator identity or fight history. The one persistent dataset is the Promoter's fighter roster (see Promoters), never auto-updated from results.

See [TECHNICAL_SPEC.md](./TECHNICAL_SPEC.md) for the technical design.
