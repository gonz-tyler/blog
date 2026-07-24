---
layout: post
title: "Blackout, Part 4: Three Notifications and a Trick"
chapter: 4
date: 2026-07-24
categories: [blackout, flutter, devlog]
tags: [flutter, notifications, background-tasks]
---

Blackout schedules three independent kinds of local notification, each
solving a different problem, plus one notification that exists purely as
infrastructure and isn't meant to be seen as a notification at all. This
post is about `NotificationService` and how it hooks back into the app.

## Track 1: the daily habits reminder

`scheduleDailyHabitsNotification()` fires at a user-configurable time
(default 9:00 AM) if daily reminders are enabled. It re-derives the count
of incomplete-but-due habits **independently from the view model**, it
reads the raw `habits` JSON straight out of `SharedPreferences` and
re-deserializes it with `Habit.fromJson`, then runs the exact same
`isDueOn` / completions-vs-`timesPerDay` check the view model uses. If the
count is zero, no notification is scheduled at all, an empty "0 habits
to do" reminder would just be noise.

This duplication is deliberate, not accidental: the notification service
shouldn't need a live reference to app-level state (it can run detached,
e.g. from a background isolate), so it reads the same persisted source of
truth the view model also reads and writes. The trade-off is that the two
codepaths could drift if one changes and the other doesn't, worth keeping
in mind if the due-date logic is ever touched.

`matchDateTimeComponents: DateTimeComponents.time` is what makes this
repeat daily without needing to be rescheduled every day, the OS handles
recurrence once the notification is registered at a given time-of-day.

## Track 2: the daily quote

Same shape as the reminder, user-configurable time, `matchDateTimeComponents`
for daily recurrence, but the content comes from `QuoteService`, which
mixes remote APIs with the local SQLite cache described in the reflection
subsystem post. Localization is loaded manually per-schedule call, via
`AppLocalizations.delegate.load(Locale(langCode))` against a
`user_language_code` preference, notably a **separate** stored key from
the one `LanguageProvider` uses (`language_code`). These being two
different keys is worth double-checking next time notification text shows
up in the wrong language; it's a plausible source of that bug.

## Track 3: streak milestones — and why they're not scheduled directly

This is the interesting one. A streak milestone (day 7, day 30, etc.)
isn't something you can schedule ahead of time, you don't know when
someone will hit it. But you also don't want to show a "7 day streak!"
notification the instant it happens, mid-tap, mid-scroll, possibly with
the screen off.

The solution: **milestones don't get their own notification directly.**
When `HabitViewModel` detects the streak crossed a milestone, it just
writes the milestone data (`{'streakCount': N}`) into
`SharedPreferences` under `pending_streak_data` and does nothing else.

Separately, `scheduleDailyStreakCheck()` schedules an entirely different
notification, at a user-chosen time (default noon), whose _visible_
content ("Streak Milestone Check... Checking for pending streak
milestones...") is never meant to actually be seen by the user, it's
low-importance, no sound, no vibration. Its real job is carrying a special
payload, `process_streak_queue`. When the OS fires it, `_onNotificationTapped`
intercepts that specific payload and, instead of navigating anywhere,
calls `processPendingStreakNotification()`, which reads `pending_streak_data`
back out of prefs and, if present, immediately shows the _real_ milestone
notification (`showStreakNotification`), then clears the queue.

In effect: **the "check" notification is a workaround for not having a
clean "run this code in the background at time X" primitive** without
attaching it to something the OS notification scheduler understands.
It's infrastructure wearing a notification's clothes. If this ever looks
like dead code or a bug (a low-priority, silent notification with debug-y
text), it isn't. That's the mechanism working as intended.

One caveat worth flagging for later: this only fires the milestone
notification when the daily check happens to run, a milestone reached
right after that day's check time won't surface until the _next_ day's
check. That's a real gap, not a bug exactly, but worth knowing if streak
notifications ever seem to arrive "a day late."

## Wiring it together: `scheduleAllNotifications`

Called on init and whenever settings change, this cancels every existing
notification and re-schedules whichever of the three tracks are currently
enabled, reading each toggle from `SharedPreferences`. It's the single
place all three tracks converge, and the right place to look first if a
notification isn't firing, check whether its enabling preference is
actually set before debugging the scheduling logic itself.

## Notification tap routing

All taps funnel through `_onNotificationTapped`, which special-cases the
`process_streak_queue` payload (handled above) and otherwise pushes the
`/today` route, clearing the navigation stack. This is also mirrored by
`MainScreen`'s own listener on the app-wide `selectNotificationStream`
(covered in Part 2), the two mechanisms overlap somewhat (both route back
to Today on a tap) and are worth reconciling if navigation-from-notification
ever needs to become more specific than "always go to Today."

## What's next

Last post in the series: the reflection subsystem, the mood quiz, how the
journal auto-synthesizes entries from quiz answers via a same-day dedup
check rather than any explicit event, and how the quote-of-the-day system
falls back to a local cache when offline.
