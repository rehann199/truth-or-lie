# Architecture

Conceptual view of the Android TV app. Names are descriptive, not production identifiers.

![System overview](../assets/diagrams/system-overview.svg)

## Layers

| Layer | Responsibility |
|---|---|
| Input | Focus search for menus; an activity-level key handler and a focus anchor on the vote screen; the Back key routed through the back-pressed dispatcher |
| Screens | Splash, home, how to play, settings, purchase, categories, rounds, player count, player names, statement, vote, reveal, final standings, leaderboard (14 activities) |
| Game session | One in-memory object: category, rounds, players with scores, round and turn counters |
| Data | Bundled category files, preferences, a local leaderboard table |
| Services | Music, ads, billing, analytics |
| Native | Kotlin only; no JNI |

## Startup

```mermaid
sequenceDiagram
    participant App as Application
    participant An as Analytics
    participant Mu as Music
    participant Bill as Billing
    participant Sp as Splash
    participant Home as Home
    App->>App: set default settings on first run
    App->>An: init wrapper, session tracker, user properties
    App->>Mu: init, register lifecycle observer
    App->>Bill: connect, query products and purchases
    App->>Bill: re-query on every return to foreground
    Sp->>Sp: title animation, short delay
    Sp->>Home: open home
    Home->>Home: init ads, start music, focus Start
```

## A game

```mermaid
stateDiagram-v2
    [*] --> Categories
    Categories --> Locked: locked category selected
    Locked --> Categories: Back, or unlocked by ad or purchase
    Categories --> Rounds: unlocked category selected
    Rounds --> PlayerCount
    PlayerCount --> Names
    Names --> Statement: scores and counters reset
    Statement --> Vote
    Vote --> NextPlayer: vote locked, more players
    NextPlayer --> Vote: Select
    Vote --> Reveal: last player voted
    Reveal --> Statement: more rounds
    Reveal --> Final: last round
    Vote --> Final: End Game from pause
    Final --> [*]
```

## A vote

```mermaid
stateDiagram-v2
    [*] --> Idle: focus on anchor, timer running
    Idle --> Selected: Left or Right
    Selected --> Selected: Left or Right again
    Selected --> Locked: Select
    Idle --> Locked: timer ends (recorded as Lie)
    Selected --> Locked: timer ends (choice kept)
    Idle --> Paused: Up or Back
    Selected --> Paused: Up or Back
    Paused --> Idle: Resume (timer restarts from remainder)
    Paused --> [*]: End Game
    Locked --> [*]: score added, next player or reveal
```

Input is ignored while a selection animation is running and after the lock.

## Statement selection

```mermaid
flowchart TD
    L[Load category file] --> S[Split into true and false pools<br/>excluding used ids]
    S --> E{Either pool empty?}
    E -- yes --> R[Clear used ids, use full file]
    E -- no --> B
    R --> B[Build a sequence for remaining rounds<br/>random, no immediate repeat,<br/>opposite type after two of the same]
    B --> P[Take the first entry]
    P --> U[Mark it used]
    U --> V[Pass to every player's vote screen]
```

## Services

| Service | Notes |
|---|---|
| Music | One `MediaPlayer` at a time, looped; stops when the app is backgrounded; paused around votes and ads |
| Ads | Unity Ads. Interstitial after a game, rewarded ad for a session-long category unlock. Preload, cooldown, entitlement check. Failures fall through |
| Billing | Play Billing client with in-app and subscription queries merged, pending purchases enabled, reconnect after a delay, acknowledgement of unacknowledged purchases |
| Analytics | One wrapper over Firebase Analytics, a pure validator, screen, session, input and ad trackers |

## Data

| Data | Where | Notes |
|---|---|---|
| Statements | App assets, JSON | Read on each statement screen; nothing cached across rounds |
| Settings | Preferences | Timer length, music, track |
| Entitlements | Preferences | Ads removed, all categories unlocked, per-category unlock flag and timestamp |
| Leaderboard | SQLite table | Name, cumulative points, last-played time |
| Game session | Memory | Lost if the process is recreated |

## Design observations

- The activity-per-screen structure with a shared singleton is simple and keeps each screen self-contained, at the cost of state that does not survive process recreation.
- Billing, ads, music and analytics are singletons initialised from the application class or the home screen, which keeps them out of the screens.
- Shared focus feedback is repeated in each screen rather than extracted.
