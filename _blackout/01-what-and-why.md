---
layout: post
title: "Blackout, Part 1: What Is This Thing and Why Did I Build It"
chapter: 1
date: 2026-07-24
categories: [blackout, flutter, devlog]
tags: [flutter, habit-tracker, architecture]
---

This is the first post in a series documenting **Blackout**, a habit-tracking
Android app I built in Flutter. The audience for this series is future-me —
someone who's about to open this codebase again after a long gap and needs
to remember what it does and why it's built the way it is, before touching
a single line. If that's you: welcome back. Here's the lay of the land.

## What Blackout does

At its core, Blackout is a habit tracker: you define habits, set how often
they're due (daily, every other day, specific weekdays, specific days of
the month, or a custom N-day interval), and check them off as you complete
them. Some habits can require multiple completions per day rather than a
single checkbox.

But it's not _just_ a checklist app. Three other pieces are bolted on:

- **A daily mood quiz** — a short multi-question check-in (mood, rest,
  motivation, a couple of free-text prompts) that produces a single mood
  score for the day.
- **A journal** — partly a manual notes log, partly an auto-generated
  record of your quiz answers each day.
- **Daily quotes** — a rotating quote (fetched from a couple of public
  APIs, cached locally for offline use) shown alongside your habits.

Tying it together is a **universal streak**, one number representing how
many consecutive days you've fully completed everything that was due,
across _all_ habits at once, not per-habit. That single number, and keeping
it alive, is the main psychological hook of the app.

I made this app because I was annoyed at all the habit apps that offered very
little customisation and hid basic features (like having more than 3 habits)
behind a paywall. I wanted a central system that acted like a sort of diary and
I was in my stoicism phase, so the app became an amalgamation of different ideas
and needs.

## A tour of the screens

The app has three main tabs, plus a handful of screens reached by
navigation:

**Today** — the home screen. Shows the current streak, today's daily
quote, the mood quiz entry point (once per day), and a checklist of
whatever habits are due today, each with a tap-to-complete button that
fills in as you log completions.

**Journey** — the reflective view. A horizontally-scrollable week strip of
mood icons going back to a fixed epoch, a mood-trend comparison ("better
than yesterday", "above your average"), and a per-habit streak history
(a bubble-per-day visualization, tap in for a full calendar view of that
habit).

**Profile** — account-level settings and stats (not deeply explored yet in
this series, flag for a later post if there's anything non-obvious here).

**Off the tab bar:**

- **Add/Edit Habit** — the habit creation form: name, description, color,
  icon, times-per-day, frequency rules, start date.
- **Journal** — full list of journal entries, manual and quiz-derived.
- **Favorite Quotes** — saved quotes (a separate mechanism from the
  quote-of-the-day caching, see the reflection-subsystem post).
- **Settings** — theme, language, notification preferences.

## The stack, in one paragraph

Flutter + Provider for state (several independent `ChangeNotifier`s rather
than one global store), `SharedPreferences` for almost all persisted state
(habits included, serialized as JSON), `sqflite` used narrowly for a local
quote cache, `flutter_local_notifications` for reminders and streak
milestones, and a home-screen widget synced on every save. Runtime-loaded
JSON theme files (via `json_theme`) rather than an in-code `ThemeData`.
Each of these is its own post, this one is just the map.

## What's next

The next post covers the app's entry point and how all these providers get
wired together, plus the two decisions that don't look obvious from the
code alone: why themes are loaded from JSON at runtime, and how the custom
language-preference system differs from just following the device locale.
