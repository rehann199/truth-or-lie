# Engineering Challenges

Six problems that stand out in the Android TV app. Each is drawn from behaviour visible in the code. Outcomes describe what the code does; no metrics are claimed.

---

## 1. Using four arrows as a game controller

**Problem**
On the vote screen, Left and Right must mean "Truth" and "Lie". In a standard Android layout the same keys move focus between views, so the framework would take the key before the game could.

**Context**
The rest of the app is menus, where the default focus search is exactly what is wanted. Only the vote screen needs the keys for itself.

**Approach**
- The Truth and Lie panels are not focusable, and an invisible 1dp view takes focus on the vote screen.
- The activity's key handler maps Left to Truth, Right to Lie, Up to pause and Select (centre key or gamepad A) to lock. Left and Right are consumed so they cannot fall through to focus search.
- A locked flag and a selection-animating flag stop input while a vote is settling, so a double press cannot select twice or score twice.
- Keys are consumed while locked.

**Trade-offs**
The focus anchor is an invisible element and a reader of the layout has to know why it exists. The handler is one function on the activity, with no unit tests.

**Outcome**
Each press maps to one action, and the vote state machine has a single place where input is accepted.

---

## 2. Pausing a timed vote on a TV

**Problem**
Pausing has to stop the countdown, keep the time that remains, quiet the music, give the remote a place to land and return everything on resume, using only the remote.

**Approach**
- Up, and also Back through the back-pressed dispatcher, cancels the countdown, stores the remaining milliseconds, pauses the music and shows an overlay with focus on Resume.
- Resume and End Game have explicit up and down targets pointing at each other, so Up and Down wrap.
- Select is handled in the activity and acts on whichever overlay button holds focus.
- Resume restarts a new countdown from the stored remainder and returns focus to the anchor.
- When the timer reaches zero with nothing chosen, the vote is recorded as Lie.

**Trade-offs**
Pausing is unlimited and has no cost, so a group can stop the clock indefinitely. Recording a timeout as Lie is a design choice that favours one answer when nobody votes.

**Outcome**
Pause and resume are symmetric and focus is always set to something the remote can act on.

---

## 3. Keeping focus predictable on a 10-foot UI

**Problem**
Without a pointer, the user needs a clear sign of where they are, and every screen and overlay needs a defined place for focus to start and to return to.

**Approach**
- Each activity requests focus on its main control when it opens: Start on the home screen, 5 rounds on the rounds screen, the first card on the category grid, the Play Again button on the final screen.
- A common focus treatment scales a control up slightly with elevation over 150 ms and back on loss.
- The leaderboard sets next-focus targets so its rows chain to each other and to the back button, and it scrolls the focused row toward the middle of the list.
- The category screen shows its unlock overlay with focus on Buy and, on Back, fades it out and returns focus to the first card.

**Trade-offs**
The focus listener is copied into each screen with slightly different constants, and the category overlay returns to the first card rather than the card that opened it.

**Outcome**
Every screen and overlay in the reviewed code has an entry point for focus and a way out.

---

## 4. Choosing statements that do not repeat

**Problem**
A game of up to 15 rounds should not show the same statement twice, and each category has a different mix of true and false statements.

**Context**
Statements are bundled data. Ids are unique within a file but are reused across files, so the used set is only meaningful for the category being played.

**Approach**
- A process-wide set records used statement ids, cleared when a new game starts.
- Each round splits the category into true and false pools without used ids, and if either pool runs out it clears the set and starts again from the full file.
- A random statement is chosen, with a rule that forces the opposite type after two of the same.

**Trade-offs**
The sequence builder produces a list for all remaining rounds but only the first entry is used, so the streak rule has little effect in practice. The mix of true and false follows each file's own balance, and some categories have far fewer false statements than true ones. A small number of statements in some files are repeated word for word, so the id-based check does not catch them.

**Outcome**
Within a game the same statement id is not shown twice unless a pool is exhausted.

---

## 5. Billing, entitlements and ads that must not block play

**Problem**
Play Billing is an asynchronous client with reconnects, pending purchases and acknowledgement. Ads can fail to load or never call back. A TV owner should still be able to play.

**Approach**
- A billing manager reconnects after a short delay, queries in-app products and subscriptions separately and merges the results, enables pending purchases, and acknowledges purchases that are not yet acknowledged.
- The application class turns billing callbacks into analytics events and local entitlement flags, and re-queries purchases whenever the app returns to the foreground.
- Ads are initialised after the home screen, preloaded, and gated by a 3-minute cooldown and an entitlement check. Each failure branch calls a failure callback and the calling screen carries on.
- The rewarded ad's unlock lives only in memory for the session.

**Trade-offs**
Entitlement flags are written to local preferences when a purchase record is processed. The restore path sets them but never clears the ads-removed and all-categories flags, so lapsed subscriptions are not revoked in the reviewed code. The per-category unlock timestamp is written each time a purchase record is processed, so the length of the window depends on how the store product is configured, which was not checked. Ad and billing identifiers are constants in code.

**Outcome**
An ad or billing failure degrades to "no ad", a message or "try again"; the vote and results screens do not depend on either.

---

## 6. Analytics that cannot crash a TV app

**Problem**
Firebase Analytics drops events with invalid names or oversized values without warning, and a D-pad produces bursts of focus and key events that would flood the data.

**Approach**
- One wrapper is the only caller of the SDK. Every method catches exceptions and does nothing before initialisation.
- A validator written in plain Kotlin cleans names, strips reserved prefixes, truncates long values, drops empty ones and caps parameters per event. It has its own unit tests.
- Vote key events pass through a 200 ms filter that suppresses identical repeats but always passes committing actions such as select, back and pause. Focus events on the 15-card grid and the leaderboard are debounced into one event after 500 ms of idle.
- Screen tracking uses lifecycle observers with de-duplication, and a session tracker avoids colliding with GA4's own session event name.
- Reviewed call sites identify players by position and report the winning score in a bucket.

**Trade-offs**
The tests cover the validator, the throttle and the trackers' de-duplication, not the game or the focus code. Booleans are sent as strings.

**Outcome**
Event volume is bounded and malformed events are cleaned up rather than silently lost, and an analytics fault cannot take down a screen.
