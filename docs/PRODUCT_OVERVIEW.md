# Product Overview

Truth or Lie TV is a party game for Android TV. A statement appears on the television, and each player in the room votes on whether it is true or a lie using the remote. Descriptions here come from a read-only review of the production project; no app was run to collect them.

No screenshots are included in this repository. The images are product artwork, and each is labelled as such.

## The idea

One screen, one remote, any number of friends in the room. The game is a hot-seat quiz: the same statement is shown to every player, each player votes in turn, and the TV reveals the answer and awards points. The game is designed around the remote because it is the only controller on a television.

## How a game plays

1. **Home.** Start, How to Play, Leaderboard and Settings, with the Start button focused by default.
2. **Category.** A grid of 15 category cards. Three are open; the other twelve show a lock and open an unlock overlay with a purchase option and a watch-an-ad option.
3. **Rounds.** Choose 3, 5, 7, 10 or 15 rounds (5 is focused by default).
4. **Players.** Choose 1 to 6 players, then optionally type a name for each (3 to 10 characters). An empty name becomes "Player N".
5. **Statement.** The category, the round number and the statement are shown, with a button to start voting.
6. **Vote.** The first player is addressed by name. Left selects Truth, Right selects Lie, Select locks the choice. A countdown runs for the length set in Settings.
7. **Next player.** After a vote is locked, an overlay names the next player and waits for Select. Each player gets a fresh timer.
8. **Reveal.** After the last player of a round, the screen shows TRUTH or LIE and a tick or cross with +100 or +0 for every player.
9. **Final standings.** After the last round, a winner screen (or a tie notice) with a ranked list and confetti. An interstitial ad may be shown when this screen opens.
10. **Play again.** Returns to the splash screen.

## Controls

| Key | On the vote screen | Elsewhere |
|---|---|---|
| Left / Right | Select Truth / select Lie | Move focus |
| Up | Open the pause overlay | Move focus |
| Down | Not mapped to a game action | Move focus |
| Select (centre or gamepad A) | Lock the selected answer | Activate the focused control |
| Back | Open the pause overlay | Go back, or close an overlay |

In the pause overlay, Up and Down switch between Resume and End Game and Select activates the focused button.

## Rules and scoring

- 100 points for each correct vote. A wrong vote scores nothing.
- If the timer ends with no choice made, the vote is recorded as Lie. If a choice was made but not locked, that choice is locked.
- There is no time bonus, streak bonus or penalty.
- Highest total wins. Equal top scores are shown as a tie; there is no tiebreaker round.

## Content

| Item | Observation |
|---|---|
| Categories | 15, each a bundled data file of statements marked true or false |
| Size | About 3,550 statements in total (roughly 2,450 true and 1,100 false) |
| Balance | The true/false mix varies by category; some are close to even, others are heavily weighted to true statements |
| Free | 3 categories (general knowledge, science, animals) |
| Unlockable | 12 categories (history, space, technology, human body, crazy facts, money and luxury, movies, party mode, relationships, survival, travel, family mode) |
| Selection | A random statement from those not yet used in the current game; see the case study |

Statement text is not reproduced in this repository.

## Settings

- Vote timer length: 10, 20, 30, 40 or 50 seconds
- Music on or off, and one of five looping tracks
- Remove ads and restore purchases
- Links to a privacy policy and terms of service, opened in an external browser

## The player leaderboard

At the end of every game each player's score is added to a local table keyed by name, so the leaderboard shows cumulative points per name on that device, ordered by points and then by most recent play. It is a household scoreboard, not an online ranking.

## Ways to unlock a category

| Path | Behaviour in the code |
|---|---|
| Rewarded ad | Unlocks the category for the current app session only |
| Per-category purchase | A one-time product per category, granted as a time-limited unlock (24 hours in the code) |
| All-categories purchase | Permanent unlock of every category and removal of ads |
| Subscription | Defined in the billing code; grants all categories and removes ads |

Prices and product identifiers are not published here.

## What the artwork shows

- The **app icon** and the **TV launcher banner** are product artwork. The banner is the image shown on the Android TV home screen.
- The marketing strip at the bottom of the banner lists a player range that differs from the 1 to 6 players the reviewed build allows. The code is the source of truth for this write-up.
