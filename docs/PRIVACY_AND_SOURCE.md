# Privacy and Source

## What this repository is

A portfolio showcase for a production application. It describes the engineering of Truth or Lie TV on Android TV; it is not a source distribution.

## What is published

- Written descriptions of architecture, design decisions and trade-offs
- An original diagram created for this showcase
- The app icon, resized
- The TV launcher banner, labelled as artwork

## What is intentionally not published

- Production source code, in whole or in part
- Statement text and category data files
- Build and signing configuration, certificates and keystores
- Firebase and Google service configuration, API keys and tokens
- Ad network identifiers and placement IDs
- Billing product IDs and prices
- Links to policy pages used inside the app
- User data of any kind
- Version control history of the production repository

## Images

- **There are no screenshots in this repository.** No captures of the app running on an Android TV or an emulator were available, and none have been mocked up or composed.
- **Everything shown is artwork.** The two images are the app icon and the TV launcher banner. Each is captioned as product artwork.
- **The banner's marketing text** lists a player range that differs from the reviewed build. It is shown as it ships and flagged where it appears.
- **Not republished.** The in-app category and interface icons are vector drawables, and some come from third-party icon collections whose licence terms could not be confirmed from the project.
- Images were re-encoded from raw pixels so no embedded metadata remains.
- **Diagram** is an original vector illustration.

## How the descriptions were produced

The production project was inspected read-only. Statements describe behaviour observed in code, manifests and build files. Where behaviour could not be confirmed it is worded conservatively or left out. No app was run to collect measurements, and no figure here is a performance measurement.

## Data the app handles (as observed)

- Player names typed by users are kept on the device (the leaderboard table). In the analytics call sites reviewed, players are identified by position, not by name.
- Settings and purchase entitlements are stored in local preferences.
- Analytics is Firebase Analytics. No account system and no server for game data were found in the project.

## Publication checks

Before publishing, every file was checked for source code, credentials, signing material, personal data, email addresses, local file-system paths and embedded image metadata. None remain.

## Access

Source access for hiring or due-diligence purposes can be discussed at the owner's discretion.
