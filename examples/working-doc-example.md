# Add User Notification Preferences

## Goal
Allow users to configure which notifications they receive and through which channels (email, in-app, push). Currently all users get all notifications with no way to opt out or customize.

## Context
- Users have complained about notification spam (GitHub issue #142)
- We currently send notifications via a single `sendNotification()` function in `src/lib/notifications.ts`
- No user preferences exist — every event triggers every notification type
- We use Supabase for auth and database

## Current State
- `src/lib/notifications.ts` — single `sendNotification(userId, event, message)` function
- `src/app/api/notifications/route.ts` — POST endpoint that calls sendNotification
- No `notification_preferences` table exists
- Users table: `public.users` with columns: id, email, name, created_at

## How It Should Work
1. New DB table `notification_preferences` with columns: user_id, channel (email/push/in_app), event_type, enabled (boolean)
2. Default: all notifications enabled for new users (insert defaults on first login)
3. API endpoint: GET/PUT `/api/notifications/preferences` — read and update preferences
4. UI: Settings page with toggle matrix (event types x channels)
5. `sendNotification` checks preferences before sending — skips disabled channels
6. If no preference row exists for a user+event+channel combo, treat as enabled (backward compatible)

## Technical Approach
- Add Supabase migration for the new table with RLS (users can only read/write their own prefs)
- Modify `sendNotification` to query preferences first
- New React component `NotificationSettings` using the existing settings page layout
- Follow the pattern of the existing `ProfileSettings` component for layout and data fetching

## What NOT to Change
- Don't touch the notification sending infrastructure (email provider, push service)
- Don't change the events themselves — only whether they're delivered
- Don't add notification grouping or scheduling — that's a separate feature

## Scope
- IN: preferences table, API, UI settings page, sendNotification filter
- OUT: notification templates, delivery infrastructure, notification history/log

## Edge Cases
- User deletes account -> cascade delete preferences (FK constraint)
- Concurrent preference updates -> last-write-wins is fine
- Missing preference row -> treat as enabled
