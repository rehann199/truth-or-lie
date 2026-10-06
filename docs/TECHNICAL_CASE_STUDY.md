# Technical Case Study

This is an engineering write-up of Truth or Lie TV, an Android TV party game. Statements describe behaviour observed in the production code, manifest and build files during a read-only review. Nothing here is a performance measurement; no metrics are claimed. Diagram labels are conceptual and do not match production identifiers.

## Shape of the project

| Area | Observation |
|---|---|
| Kotlin | 34 files, about 6,600 lines, native Android with XML layouts |
| Screens | 14 activities and 14 layouts, one activity per screen |
| Largest files | Settings (about 790 lines), billing manager (about 565), voting (about 465), ad manager (about 400) |
| Analytics layer | 8 files, about 1,100 lines, plus 4 test files |
| Content | 15 bundled JSON files with about 3,550 statements |
| Tests | 55 JVM unit tests in 4 files: the analytics sanitiser (29), the D-pad key throttle and focus debounce (12), analytics safety (8) and screen-tracker de-duplication (6). There are no tests around game logic or focus handling |
| Release | Minification and resource shrinking enabled for release builds; keep rules for billing, ads, lifecycle and the app's own classes |
| Native code | None beyond Kotlin. There is no JNI or C++ |
| Build | Gradle with a version catalogue, Java 11 source and target, minimum SDK 24, target and compile SDK 36 |

## 1. The TV platform contract

The manifest declares the Leanback feature as required and the touchscreen feature as not required, registers a Leanback launcher category on the splash activity (along with the standard launcher category), and provides a banner drawable for the TV launcher. No screen orientation is declared. The window background is a dark colour and the app uses a bundled font family, so nothing is fetched at runtime for typography.

## 2. Input model: arrows and select

Input is central to the design because the remote is the only controller. The project uses two mechanisms side by side.

| Screens | Mechanism |
|---|---|
| Home, categories, rounds, player count, names, settings, leaderboard, purchase, results | Android's focus system. Controls are focusable and focus search moves between them. Each screen sets its initial focus explicitly. Where the default search is not enough, next-focus targets are set in code (leaderboard rows, the pause overlay buttons) |
| Vote screen | An activity-level key handler plus a focus anchor, a 1dp invisible focusable view, so that Left and Right reach the handler instead of moving focus |

Focus feedback is shared in spirit across screens: on gaining focus a control scales up slightly and gains elevation over 150 ms, and on losing focus it returns. The scale factor differs by screen (roughly 1.01 for list rows up to 1.06 for buttons); the code repeats this listener on each screen rather than sharing a helper.

## 3. The vote screen in detail

The vote screen is where the remote is used as a controller rather than a pointer.

- **Mapping.** Left selects Truth, Right selects Lie, Up pauses, Select (centre key or gamepad A) locks a selected answer. Back is intercepted through the activity's back-pressed dispatcher and also pauses.
- **Focus anchor.** The Truth and Lie panels are not focusable. An invisible 1dp view holds focus so the key handler receives the arrows. The pause button is focusable.
- **Re-entrancy guards.** Two flags, locked and selection-animating, stop input while the 300 ms panel animation is running or after the vote is locked. Keys are consumed while locked so they do not leak to focus search.
- **Selection animation.** The two panels share a horizontal layout weight; selecting a side animates the weights so the chosen side widens. Locking dims the other side with an alpha animation.
- **Timer.** A countdown ticks every 100 ms, updates a seconds label and scales a bar by the remaining fraction, and turns the label red in the last three seconds. Pausing cancels the timer and keeps the remaining milliseconds; resuming starts a new countdown from them.
- **Timeout.** When the timer ends with no choice made, the vote is recorded as Lie; if a choice was made but not locked, it is locked.
- **Hand-off.** After a lock, the next player is named in an overlay and the Continue button takes focus. The screen then relaunches itself for the next player, which gives that player a fresh timer and fresh state. After the last player of a round the app moves to the reveal screen.
- **Pause overlay.** Resume and End Game are chained with explicit up and down targets, wrapping in both directions. Pausing also pauses the music.

## 4. Game state

State is one in-memory singleton, not a view model or a saved state:

- Chosen category, number of rounds, player count
- A list of players with a name, a score and the last answer
- A round counter and a turn counter; the current player is the turn counter modulo the player count

A new game resets scores, the turn counter and the used-statement set. Because the state is in memory, an activity or process recreated by the system loses an in-progress game; this is acceptable for a short session on a TV but is a trade-off (see the challenges document).

Scoring is fixed at 100 points per correct vote, added when a vote is locked.

## 5. Statements and truth/lie logic

Each category is a bundled JSON file with a header (category, version, total) and a list of statements, each with an id, a text and a boolean answer.

For every round the facts screen:

1. Loads the category file and splits it into true and false pools, excluding statement ids already used in this game.
2. If either pool is empty, clears the used set and starts again from the full file.
3. Builds a sequence for the remaining rounds, picking randomly without immediate repeats and forcing the opposite type after two of the same, then takes the first entry for the current round and marks it used.

The same statement is then passed to every player's vote screen through intent extras. The answer is a boolean carried alongside it.

Two observations from the review:

- The pick is random over the union of unused statements, so the share of true statements a player sees follows the file's own mix, which differs by category.
- The sequence builder produces a list for all remaining rounds but only the first entry is used each round, so the "avoid three in a row" rule has little effect in practice. The used-id set is what prevents repeats.

## 6. Persistence

Local only:

| Data | Storage |
|---|---|
| Timer length, music on/off and track, first-run flag | `SharedPreferences` |
| Purchase entitlements and time-limited category unlocks | `SharedPreferences` |
| All-time leaderboard (name, points, last-played time) | A single SQLite table through a hand-written `SQLiteOpenHelper` |

The leaderboard adds new points to an existing row with the same name, so identical names are the same player. Scores are saved when the final results screen opens, and the leaderboard screen reads them on a background coroutine.

## 7. Audio and lifecycle

- A `MediaPlayer` loops one of five bundled tracks at half volume. It is released when music stops.
- A process-level lifecycle observer stops music when the app goes to the background and starts it again when it returns, if music is enabled.
- Screens pause and resume music around the vote and around ads.
- The home screen's floating title animations are cancelled when the screen pauses.
- A second process-level observer re-queries purchases whenever the app returns to the foreground.
- Another observer feeds session start, foreground and background events to analytics.

## 8. Ads

- Unity Ads is initialised from the home screen and preloads an interstitial and a rewarded ad.
- Entitlement check: if either the subscription flag or the ads-removed flag is set, ads are skipped. The default for both is unset, so a new install sees ads.
- The interstitial is requested when the final results screen opens. It is skipped on a 3-minute cooldown, when the SDK is not initialised, or when an ad is already showing. Its failure callback only logs; the screen carries on.
- The rewarded ad unlocks one category for the current session only. If the reward is not earned or no ad loads, the category stays locked and a message is shown.
- Music is paused while an ad is on screen and resumed afterwards, and ad outcomes are reported to analytics.

## 9. Billing

- A manager owns the Play Billing client, enables pending purchases for one-time products, reconnects after a short delay when disconnected, queries in-app and subscription purchases separately and merges the results, and acknowledges unacknowledged purchases.
- A thin repository exposes it as a singleton, and the application class translates listener callbacks into analytics events and local flags.
- Per-category purchases map product id to category key and are stored as an unlock flag with a timestamp; the category manager treats a category as unlocked for 24 hours from that timestamp and clears the flag on read after expiry.
- A switch exists in the manager for a simulated purchase path. In the reviewed build it is a constant that is off.

## 10. Analytics

- A single wrapper is the only code that talks to Firebase Analytics. All methods swallow exceptions and do nothing before initialisation.
- A pure Kotlin validator enforces event-name rules, truncates strings, caps parameters at 25 per event and drops empty values. It is tested on the JVM.
- A screen tracker records enter and exit with durations, a session tracker records foreground and background, and an ad tracker wraps ad outcomes.
- D-pad key events on the vote screen are throttled; focus events on the 15-card grid and on the leaderboard are debounced to one event after half a second of idle.
- The reviewed call sites identify players by position and report scores in buckets.

## 11. Offline behaviour

Statements and music are bundled, so the game plays without a network. Ads, billing, analytics and the policy links need a connection; each is written so that its absence does not stop a game.

## 12. Release configuration

- Release builds minify and shrink resources, and keep line numbers for stack traces.
- The ProGuard rules keep the app's own classes in full, along with billing, ads, lifecycle, coroutines and confetti libraries.
- Signing configuration is read from environment variables with a local fallback (details are not published).

## What was not found

- No crash-reporting SDK among the declared dependencies; Firebase is used for analytics only.
- No server or account system; all game data is on the device.
- No tests for game logic, focus handling or billing.
- No instrumentation or UI tests.

## Reading order

1. [Architecture](ARCHITECTURE.md) for structure and flows
2. [Engineering challenges](ENGINEERING_CHALLENGES.md) for the decisions behind this write-up
3. [Product overview](PRODUCT_OVERVIEW.md) for the player's view
