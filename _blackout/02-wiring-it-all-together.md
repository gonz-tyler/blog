---
layout: post
title: "Blackout, Part 2: Wiring It All Together"
chapter: 2
date: 2026-07-24
categories: [blackout, flutter, devlog]
tags: [flutter, architecture, provider, theming, i18n]
---

Part 1 covered what Blackout does. This post covers how the app boots up
and how its foundational pieces, state management, theming, and
localization, are wired together. These are the decisions that are
invisible once the app is running, which makes them exactly the kind of
thing worth writing down before they're forgotten.

## State management: several small stores, not one big one

Blackout uses `provider`, but rather than one global app-state object,
state is split into independent `ChangeNotifier`s, each owning one concern:

- `HabitViewModel` — habits, completions, streaks (the core domain, its
  own post is coming)
- `AppPreferencesProvider` — mood-quiz toggle, quotes toggle, mood history
- `ThemeSettingsProvider` — theme mode + seed color
- `LanguageProvider` — active locale + user override
- `MyAppState` — a grab-bag leftover from the Flutter counter/word-pair
  starter template, repurposed to hold favorite quotes. Worth noting: the
  name is misleading now, it isn't really "the" app state, just wherever
  favorites ended up.

All five are created in `main()` and handed to the widget tree via a
single `MultiProvider`. Two of them (`HabitViewModel` and
`AppPreferencesProvider`) have async load methods that are kicked off
_before_ `runApp` and raced with `Future.wait`, and the resulting future is
threaded down into `MyApp` so the UI can show a loading state until both
finish. This means habit data and preferences are guaranteed loaded by the
time `MainScreen` renders, nothing downstream needs to null-check "is this
loaded yet."

## Why themes are loaded from JSON at runtime

This is the one decision in the whole app that looks strangest out of
context: `ThemeData` isn't built with `ColorScheme.fromSeed` or defined in
Dart at all. Instead, `_MyAppState._loadTheme()` reads a JSON file like
`assets/themes/appainter_green_light.json` from the asset bundle and
decodes it into a `ThemeData` via `json_theme`'s `ThemeDecoder`, with a
static in-memory cache so each theme file is only decoded once per app
session.

The practical effect: theme files were (presumably) generated externally —
`json_theme`'s naming convention strongly suggests the
[Appainter](https://appainter.dev) tool, and swapping or adding a theme
is a matter of dropping in a new JSON file and pairing it with a
`seedColor` string, rather than writing Dart code. The seed color and
light/dark pairing are resolved reactively via `Consumer2` on
`ThemeSettingsProvider`, so both files are loaded and cached before the
`MaterialApp` itself is built, meaning a theme switch means loading (or
serving from cache) two JSON files and rebuilding, not a simple property
flip.

## Localization: not just following the system locale

`LanguageProvider` is doing more than `flutter_localizations` gives you
for free. The nuance: it tracks _two_ things, the resolved active
`Locale` the app is rendering with, and a separate nullable
`_savedLanguageCode` representing "did the user explicitly pick a
language, or are they following the system?"

- If `_savedLanguageCode` is `null`, the app follows the device locale,
  and a `WidgetsBindingObserver` hook (`didChangeLocales`) means changing
  your phone's language while the app is installed updates it live.
- If the user picks a language via `setLanguage()`, that overrides the
  system and persists across restarts and locale changes, the observer
  checks `_savedLanguageCode == null` before reacting, so an explicit
  choice is never silently clobbered by a system change.
- `resetLanguage()` clears the override and falls back to system-following
  behavior again.

This is the kind of three-state logic ("system", "explicit choice",
"reverted to system") that's easy to get subtly wrong and easy to forget
the reasoning behind, hence documenting it here rather than relying on
future-me to re-derive it from the code.

## Routing and the main screen shell

Navigation is a flat set of named routes (`/today`, `/journey`, `/profile`,
etc.) registered on `MaterialApp`, with one exception: `/edit_habit` reuses
the `/add_habit` screen, passing an existing `Habit` in as route
arguments to switch it into edit mode.

The three main tabs (`Today`, `Journey`, `Profile`) live inside an
`IndexedStack` rather than being pushed/popped, this keeps all three
screens alive and their scroll position/state intact when switching tabs,
at the cost of building all three widgets up front. `MainScreen` also
owns a listener on a global broadcast `StreamController` that notification
taps write into, when a notification is tapped, its payload comes through
this stream and switches the visible tab back to `Today`, regardless of
what screen was open when the tap happened.

## What's next

Next up: the habit domain model and the universal streak algorithm, the
actual core logic of the app, including the "what happens if today isn't
finished yet" edge case that makes the streak calculation non-trivial.
