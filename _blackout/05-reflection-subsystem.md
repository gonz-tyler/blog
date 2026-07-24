---
layout: post
title: "Blackout, Part 5: Mood, Journal, and Quotes"
chapter: 5
date: 2026-07-24
categories: [blackout, flutter, devlog]
tags: [flutter, sqlite, offline-first]
---

Last post in the series. This one covers the part of the app that isn't
about habits at all: the daily mood quiz, the journal it feeds into, and
the quote-of-the-day system. These three are more loosely coupled to each
other than the habit/streak/notification core, and each makes its own
small, easy-to-forget decision.

## The mood quiz

`QuizScreen` is a short `PageView` of questions, a mix of icon-based
multiple choice (feeling today, how rested, motivation level, on a 1–5
scale) and free-text prompts (focus for the day, most important task,
something you're grateful for), plus one binary "prepared / not prepared"
question worth zero points. Points from the choice questions sum into a
single `totalScore` for the day.

On completion, `_saveQuizResult()` writes to two separate
`SharedPreferences` keys: `MoodHistory` (a `date → score` map, used for
the streak-adjacent mood trend graph) and `QuizAnswers_<date>` (the full
set of answers, choice and text both, formatted into human-readable
strings via `_getChoiceAnswerText`/`_getIconDescription`, so an icon
selection like "battery_full" gets stored as readable text like "Fully
rested," not the icon's enum name). It also pushes the score into
`AppPreferencesProvider.updateMoodHistory()` so the rest of the app
(the mood trend on the Journey screen, notably) sees it immediately without
waiting for a reload.

Today's completion is tracked via a `last_quiz_date` string compared
against today's date, simple, but means clock changes or timezone edge
cases around midnight could theoretically cause the "have I done today's
quiz" check to be wrong for a moment. Not something to lose sleep over,
just worth knowing if a "why did it ask me to redo the quiz" report ever
comes in.

## The journal — half manual log, half derived view

`JournalPage` reads and writes a `JournalEntry` list under the
`JournalEntries` key. Entries can be added two ways:

1. **Manually** — a simple mood-text + note dialog, where the mood text
   (e.g. "feeling great") is pattern-matched against a few keywords
   (`happy`/`good`/`great`, `sad`/`bad`, etc.) to pick a mood icon.
2. **Auto-synthesized from the day's quiz** — on `initState`,
   `_checkForNewQuizAnswers()` looks for `QuizAnswers_<today>` and
   `MoodHistory[today]`, and if both exist and there **isn't already** a
   quiz-derived entry for today (checked by scanning existing entries for
   `isQuizEntry && date == today`), it builds one: formats the answers
   into a bulleted note (`_formatQuizAnswers`) and picks a mood icon from
   the score bucket.

**This is a polling/dedup pattern, not an event-driven one.** The quiz
screen doesn't notify the journal that a new entry should exist; the
journal, whenever it happens to be opened, checks whether it's missing
today's quiz-derived entry and backfills it if so. That means the
auto-generated journal entry won't appear until the Journal screen is
actually visited at least once after taking the quiz that day, a
deliberate simplicity trade-off (no event bus, no provider watching quiz
state) but worth remembering if a "my journal entry didn't show up"
report ever comes in: check whether the Journal screen was opened that
day, not whether the quiz saved correctly.

Quiz-derived entries are visually distinguished (a "Daily Check-in" chip,
no delete button, they can only be removed by deleting the whole day's
`JournalEntries` blob) from manual entries (plain delete button).

## Quotes: two independent systems sharing one word

It's worth being explicit that there are **two unrelated "quote" features**
in this app, easy to conflate:

- **Quote of the day** (`QuoteService` + `DatabaseHelper`) — the rotating
  quote shown on the Today screen and in notifications.
- **Favorite quotes** (`MyAppState.favouriteQuotes`) — a user-curated
  saved list, stored separately in `SharedPreferences` as a simple
  `Map<String, String>` of quote → author, unrelated to the SQLite cache
  below.

## The quote-of-the-day cache: SQLite as an offline fallback, not a source of truth

`QuoteService.getQuote()`'s logic, in order:

1. Always fetch a random quote from the local SQLite `quotes` table first
   (`dbHelper.getRandomQuote()`), cheap, and kept as the fallback value.
2. If the device is online (checked via `connectivity_plus`), randomly
   pick one of two remote APIs (50/50 between a stoic-quotes endpoint and
   a Buddhist-quotes endpoint), fetch a fresh quote, **save it into the
   local SQLite table**, and return it.
3. If offline, or the remote call fails or errors, fall back to the quote
   already fetched from SQLite in step 1.

So the SQLite table functions purely as a growing offline cache, every
successful online fetch adds one more row, and the app never has to ship
with a bundled quote dataset because the cache builds itself over time as
the device gets online at least once. The corollary: **a fresh install
with no internet on first launch has no fallback quote available** except
the hardcoded string `"Every day is a chance to grow stronger."` baked
into the `catch` block, worth testing that path specifically if quotes
ever seem to misbehave on a clean install.

**Also worth noting:** the `QuoteService` class declares URLs for
Islamic, Hindu, Catholic, and Jewish quote APIs that are never referenced
inside `getQuote()`, dead fields, presumably scaffolding for a planned
"choose your quote tradition" feature that never got wired in. Not a bug,
just unused code worth remembering the intent behind if picked back up.

## The Journey screen mood trend

Briefly, since it ties the mood data to something user-facing:
`journey_page.dart`'s `_getMoodTrend` compares today's score against both
yesterday's and the running average across all recorded days, and returns
a two-line localized comparison ("better than yesterday" / "above your
usual mood," etc.). It's a straightforward derived computation over
`MoodHistory`, no separate storage, recalculated on each build.

## Series wrap-up

That's the full picture: habits and streaks as the core loop, notifications
as the thing that pulls you back in, and mood/journal/quotes as the softer
reflective layer wrapped around it. If you're reading this after a long
gap away from the codebase, start with Part 3 (habit model) if you're
about to touch core logic, or this post if you're about to touch anything
quiz/journal/quote-related. The known rough edges worth a cleanup pass,
gathered from across the series:

- `helper/streak_helper.dart` is dead code (Part 3)
- Two separate stored language-code keys that could drift (Part 4)
- `DailyProgressSummary` widget appears unused/unwired anywhere (flagged
  during research, not yet confirmed)
- Unused religious-quote API URLs in `QuoteService` (this post)
- `_dueCache` on `Habit` would go stale if frequency-editing is ever added
  without clearing it (Part 3)
