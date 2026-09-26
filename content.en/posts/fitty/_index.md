---
title: Fitty
---

# Fitty

With [Fitty](https://github.com/xndrdev/fitty), I am building a personal fitness companion
for iOS and the browser. The idea: one chat per day for food, workouts, walking pad sessions,
and everyday questions. Describe a meal, attach a photo, or log some exercise — and build
a daily overview that I can check and revisit.

## The idea

AI helps turn messages into food and activity entries and estimate calories and nutrients.
Photos of meals, nutrition labels, or workout displays can also serve as input. Estimates
stay clearly marked, and entries can be corrected later through the chat or directly in
the overview.

Personal goals and dietary preferences provide context. Optional daily targets for
calories, protein, carbohydrates, and fat show progress. Daily totals are calculated from
stored entries, with activity calories shown separately. A dedicated photo history is
intended to help compare physical changes over time.

## Tech stack

| Area | Technology and purpose |
| --- | --- |
| iOS and web | React Native with Expo, TypeScript, and Expo Router — one shared codebase for the app and browser. |
| Backend | Go with pgx — API, application logic, and PostgreSQL access. |
| Database | PostgreSQL within Supabase — profiles, chats, targets, and tracking entries. |
| Login and images | Supabase Auth for login and sessions, Supabase Storage for private photos. |
| AI | OpenAI Responses API through the Go backend — text and image analysis with a configurable model. |
| Hosting, planned | Coolify to run the web interface, backend, and Supabase on my own infrastructure. |

AI analysis uses the external OpenAI API, which receives the chat, profile, and image data
needed for processing. The separate progress photos are excluded from that analysis.

## Status and next steps

According to the [project README](https://github.com/xndrdev/fitty/blob/main/README.md),
login, profiles, daily chats, text and photo tracking, daily targets, and progress photo
comparisons are already implemented. Testing on a physical iPhone and deployment through
Coolify are still pending. Without an OpenAI key, Fitty remains usable as a diary.

This is where I will document how the app holds up in everyday use and what I build next.
