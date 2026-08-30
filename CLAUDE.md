# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project overview

Birdle is a Wordle clone built with Flutter as a learning project, developed incrementally by following a
Flutter course/book (commit messages reference "Part X Chapter Y" sections). Expect the codebase to be small
and to grow feature-by-feature rather than all at once — don't assume missing functionality (tests, themes,
persistence, a real word list) is an oversight unless asked to add it.

## Commands

Run all commands from the repository root.

- Get dependencies: `flutter pub get`
- Run the app (device/emulator or `-d chrome`/`-d windows` to target a specific platform): `flutter run`
- Analyze/lint (uses `flutter_lints` via `analysis_options.yaml`): `flutter analyze`
- Run all tests: `flutter test`
- Run a single test file: `flutter test test/<file>_test.dart`
- Format code: `dart format .`

There is currently no `test/` directory in the repo — add one (`test/<name>_test.dart`) when writing tests.

## Architecture

The app has two files in `lib/`:

- `lib/game.dart` — all game logic, UI-independent. Key pieces:
  - `Game`: holds round state (`_wordToGuess`, `_guesses`, `numAllowedGuesses`). Exposes `guess()`,
    `isLegalGuess()`, `resetGame()`, and derived getters (`activeIndex`, `guessesRemaining`, `didWin`, `didLose`,
    `previousGuess`). A `Game` does not manage its own lifecycle — callers must call `resetGame()` themselves
    when starting a new round; nothing does this automatically.
  - `Word` — a `List<Letter>` wrapped with `IterableMixin`, representing one guess row (or the hidden word).
    `Letter` is a typedef record: `({String char, HitType type})`.
  - `HitType` enum: `none`, `hit`, `partial`, `miss`, `removed`. `removed` is an internal bookkeeping state used
    only during `evaluateGuess` matching (see below) and should never reach the UI.
  - `WordUtils.evaluateGuess(other)` implements Wordle-style matching in two passes over `this` (the guess) vs
    `other` (a mutable copy of the hidden word): pass 1 marks exact position matches as `hit` and marks the
    corresponding hidden-word letters `removed`; pass 2 marks remaining letter-present-but-wrong-position
    matches as `partial`, also consuming (`removed`) hidden-word letters so duplicate letters are matched at
    most once per occurrence. Both `this` and `other` are mutated in place. `evaluateGuess` asserts the guess is
    legal — always call `isLegalGuess`/`Word.isLegalGuess` before it.
  - `legalWords` is the pool a hidden word can be picked from; `allLegalGuesses` (`legalWords` + `legalGuesses`)
    is the pool of guesses a player is allowed to submit. Both lists are currently tiny placeholder word lists,
    not the full Wordle dictionary — don't treat their small size as a bug.
  - `Word.random()` vs `Word.fromSeed(int)`: the `Game(seed: ...)` constructor param exists specifically to make
    games deterministic (e.g. for tests) by picking a fixed index into `legalWords` instead of a random one.

- `lib/main.dart` — Flutter widget tree, no game logic of its own:
  - `MainApp` → `Scaffold` → `GamePage` (stateful, owns the single `Game` instance for the session).
  - `GamePage` renders one `Row` of `Tile` widgets per entry in `Game.guesses` (a fixed-length list padded with
    empty `Word`s), then a `GuessInput` below.
  - `GamePage._handleGuess` is the boundary between UI and game logic: it checks `Game.isLegalGuess` first and
    shows a `SnackBar` on failure, since `Game.guess`/`evaluateGuess` will hit an assertion failure on an
    illegal word rather than failing gracefully.
  - `Tile` maps `HitType` to color (`hit`→green, `partial`→yellow, `miss`→grey, default/`none`→white) — this is
    the only place UI encodes game-state color, so extend the `switch` here if new `HitType`s are added.
  - `GuessInput` is a self-contained stateful widget (owns its own `TextEditingController`/`FocusNode`) that
    reports finished guesses upward via an `onSubmitGuess` callback rather than reaching into `Game` directly.

All platform folders (`android/`, `ios/`, `linux/`, `macos/`, `windows/`, `web/`) are standard Flutter
scaffolding, not hand-maintained — avoid editing generated files there unless specifically working on
platform/build configuration.
