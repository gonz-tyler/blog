---
layout: post
title: "Blackout, Part 3: The Habit Model and the Streak Algorithm"
chapter: 3
date: 2026-07-24
categories: [blackout, flutter, devlog]
tags: [flutter, domain-model, algorithms]
---

This is the core of the app. Everything else, screens, notifications,
the mood/journal subsystem, exists to support or react to what's in this
post. If you only read one post in this series before diving back into
the code, make it this one.

## The `Habit` model

A `Habit` is a plain data class: name, description, color, icon,
`timesPerDay`, a `FrequencyType`, and a `Map<DateTime, int>` of
`completedDates` (date → number of completions logged that day).

**Frequency is a tagged union via optional fields, not a sealed class.**
`FrequencyType` is a simple enum (`daily`, `alternate`, `weekly`, `monthly`,
`custom`), and the fields that matter depend on which one you picked:
`weeklyDays` for weekly, `monthlyDays` for monthly, `customInterval` for
custom. This isn't the most type-safe way to model it, Dart's sealed
classes or a package like Freezed would make illegal states unrepresentable
— but it keeps `toJson`/`fromJson` simple, which mattered more given how
much of this app round-trips through manual JSON serialization. If this
ever gets revisited, that trade-off is the thing to reconsider first.

**`isDueOn(date)` is memoized, and the cache survives `copyWith`.** Each
`Habit` carries a private `Map<String, bool> _dueCache` keyed by a
`"year-month-day"` string. This matters because due-date checks happen a
lot (every calendar cell, every streak-loop iteration, every "what's due
today" list) and recomputing the day-of-week/interval logic each time
adds up. The cache is deliberately threaded through `copyWith()` (passed
as `dueCache: _dueCache`) rather than reset, because a `copyWith` is used
constantly for small mutations (toggling a completion) that don't change
frequency rules, so there's no reason to invalidate. **If you ever add a
way to edit a habit's frequency after creation, this cache needs to be
explicitly cleared**, right now `copyWith` doesn't distinguish "changed
frequency" from "changed completions," so an edited frequency with a
stale cache would silently return wrong due-dates for previously-checked
dates. This is a latent bug waiting for a `updateHabit` codepath change,
not a bug today (updateHabit currently only touches completions, colors,
etc., check whether frequency edits go through the same doc-cache-bearing
constructor before changing that).

**Icons are stored as full `Icon` objects and serialized by `codePoint`.**
`toJson` writes `iconData.icon?.codePoint`, `fromJson` rebuilds an `Icon`
by codepoint against `'MaterialIcons'` specifically. This is fragile if
Blackout ever switches icon packs or bumps a Flutter version that
renumbers Material icon codepoints, worth a note if that ever happens,
since existing habits' icons would silently break or shift.

## The universal streak

This is the single number shown at the top of the Journey screen, and the
main retention mechanic of the app: how many consecutive days has
_everything due_ been fully completed, across all habits at once (not
per-habit).

The naive version, walk backward from today, day by day, checking if
everything due that day was completed, stop at the first miss, has one
problem: **it would zero out your streak every single day until you finish
today's habits**, even if yesterday was perfect. That's a bad user
experience (opening the app in the morning and seeing your 40-day streak
show as "0" until you've done anything today).

The fix, in `HabitViewModel._updateUniversalStreak()`:

1. First, check whether _today_ is already fully complete.
2. If today is **not** complete, start the backward-counting loop from
   **yesterday** instead of today, so an in-progress day doesn't count
   against you, but a fully-missed day still would once you're counting
   from it.
3. If today **is** complete, start counting from today as normal.
4. Walk backward day by day from that start point; for each day, check
   every habit that (a) had started by that date and (b) was due that
   date. If any such habit fell short of `timesPerDay` completions, stop,
   the streak ends the day before.
5. A `loopGuard` (max 1000 iterations) exists purely as a safety valve
   against an infinite loop bug, not as a meaningful business rule.

**This method is called from every single mutation** — `addHabit`,
`deleteHabit`, `updateHabit`, `toggleHabitDate`, `addHabitCompletion`,
`resetHabitDateCompletions`, `setHabitCompletionsForDate`,
`setAllHabitCompletions` — recalculating the entire streak from scratch
each time rather than incrementally updating it. Given the `_dueCache` on
each habit, this is cheap enough in practice, but it means the streak
algorithm's cost scales with total app lifetime (days since the earliest
habit's start date), not with the size of a single edit.

**Milestone notifications piggyback on streak increases.** Whenever a
completion pushes the streak past specific milestone values (1, 3, 7, 14,
21, 30, 50, 100, then every 100 after that), `checkAndQueueMilestoneNotification`
fires, but it doesn't show a notification directly. It writes the
milestone into `SharedPreferences` under `pending_streak_data` for the
notification service to pick up later. Why that indirection exists is a
notification-system decision, covered in the next post.

## There's a second, near-identical streak implementation — and it's dead

`helper/streak_helper.dart` contains `StreakHelper.getStreakInfo()`, a
pure/stateless reimplementation of almost the exact same algorithm
(same "check today first, count from yesterday if incomplete" logic),
returning a `StreakInfo` record instead of mutating instance state.
`journey_page.dart` imports it but never calls it, the screen reads
`habitViewModel.universalStreak` and `.isTodayComplete` directly instead.

This has the shape of an abandoned refactor: an attempt to pull the streak
math out of the view model into a stateless, more testable function that
never got finished plugging in. **It's safe to delete** unless something
non-obvious depends on it, grep for `StreakHelper` before removing, but
as of this writing it's dead weight. Worth doing as a small cleanup pass
rather than leaving it to confuse a future read-through.

## The habit calendar dialog

`HabitCalendarDialog` is a fully hand-rolled month-grid calendar (not a
package like `table_calendar`), a 42-cell `GridView` (6 weeks × 7 days,
padded to always start on a Monday), with per-day state distinguishing:
today, future (disabled), before the habit's start date (disabled), valid
but not-due (dimmed, not tappable), and due-and-tappable. Tapping a due
day cycles its completion count and calls back up to persist the change
through the same `HabitViewModel` methods the main list uses.

Completion count is rendered as small dots under the date (up to 5, then
a single dot plus a number), a deliberate choice to keep density readable
for habits with high `timesPerDay` values rather than showing 10 tiny
dots. Month navigation is clamped: you can't go earlier than the habit's
start date or later than the current month.

If this ever gets rebuilt, the main candidates for using a package instead
would be `table_calendar`, but the current implementation is simple
enough, and tightly coupled enough to the due/completion logic, that a
rewrite isn't obviously a win.

## What's next

Next: the notification system, three separate scheduled-notification
tracks (daily reminder, daily quote, streak milestones) and the trick used
to work around not having a clean "run code at a specific time in the
background" primitive.
