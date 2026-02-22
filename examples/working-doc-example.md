# Working Doc: Real-Time Collaborative Comments

## Goal

Add a threaded comment system to project tasks with real-time updates via WebSocket. Users should be able to comment on any task, reply to comments (one level deep), and see new comments appear instantly without refreshing. Mentions (@user) should trigger in-app notifications.

## Why This Matters

Comments are the #1 requested feature (GitHub issue #287, 43 upvotes). Users currently work around this by pasting notes into the task description field, which creates a mess — no attribution, no timestamps, no threading. We're losing teams to competitors that have this built in.

## Current Architecture

```
Frontend (React + Vite)
  └── src/
      ├── components/task-detail.tsx      # Task detail panel (right sidebar)
      ├── components/task-list.tsx        # Task list with filters
      ├── hooks/use-task.ts              # Fetches single task by ID
      ├── hooks/use-realtime.ts          # Existing WebSocket hook (used for task status updates)
      ├── lib/api.ts                     # Axios instance, all API calls
      └── lib/ws.ts                      # WebSocket client (connects to /ws, handles reconnect)

Backend (Express + Prisma + PostgreSQL)
  └── src/
      ├── routes/tasks.ts               # CRUD for tasks
      ├── routes/notifications.ts        # GET /notifications, PATCH /notifications/:id/read
      ├── middleware/auth.ts             # JWT extraction, req.user population
      ├── services/notification.ts       # createNotification(), sends to WebSocket + stores in DB
      ├── ws/index.ts                    # WebSocket server, room-based (room = project:{projectId})
      └── prisma/schema.prisma          # Current schema (User, Project, Task, Notification)
```

### Current Data Flow (Task Updates)

```
User edits task → PUT /api/tasks/:id → DB update
  → ws.broadcast(room: "project:{projectId}", event: "task:updated", payload)
  → All connected clients in that project receive the event
  → use-realtime.ts handler updates React Query cache
```

The comment system should follow this exact same pattern for real-time delivery.

### Existing WebSocket Protocol

The WebSocket server uses a simple JSON protocol:

```json
// Client → Server (join room on connect)
{ "type": "join", "room": "project:abc123" }

// Server → Client (broadcast)
{ "type": "event", "event": "task:updated", "payload": { ... } }
```

We'll add `comment:created` and `comment:updated` events to this — no protocol changes needed.

## Changes By File

### Database — `prisma/schema.prisma`

Add two models:

```prisma
model Comment {
  id        String   @id @default(cuid())
  taskId    String
  task      Task     @relation(fields: [taskId], references: [id], onDelete: Cascade)
  authorId  String
  author    User     @relation(fields: [authorId], references: [id])
  parentId  String?
  parent    Comment? @relation("Replies", fields: [parentId], references: [id], onDelete: Cascade)
  replies   Comment[] @relation("Replies")
  body      String
  mentions  String[]  // Array of mentioned user IDs (extracted on write)
  editedAt  DateTime?
  createdAt DateTime @default(now())

  @@index([taskId, createdAt])
}
```

**Constraints:**
- `parentId` must reference a root comment (no nested replies — one level deep only)
- `body` max 5000 chars, enforced at API layer
- Cascade delete: deleting a task removes all its comments; deleting a root comment removes its replies

### Backend — New file: `src/routes/comments.ts`

```
POST   /api/tasks/:taskId/comments          # Create comment (or reply if parentId sent)
GET    /api/tasks/:taskId/comments           # List comments (root + replies, paginated)
PATCH  /api/comments/:id                     # Edit comment body (author only, within 15 min)
DELETE /api/comments/:id                     # Delete comment (author only)
```

**Create comment flow:**
1. Validate `body` (1–5000 chars), validate `taskId` exists and user has project access
2. If `parentId` provided: validate it exists, belongs to same task, and is a root comment (parentId === null)
3. Extract @mentions from body via regex (`/@(\w+)/g`), resolve to user IDs against project members
4. Insert comment
5. For each valid mention: call `notificationService.createNotification()` with type `comment_mention`
6. Broadcast `comment:created` to room `project:{projectId}` via WebSocket

**List comments response shape:**
```json
{
  "comments": [
    {
      "id": "clx...",
      "body": "Looks good, @alice should review the edge case though",
      "author": { "id": "...", "name": "Bob", "avatarUrl": "..." },
      "mentions": ["alice-user-id"],
      "replies": [
        {
          "id": "clx...",
          "body": "On it!",
          "author": { "id": "...", "name": "Alice", "avatarUrl": "..." },
          "createdAt": "2026-02-20T14:30:00Z"
        }
      ],
      "replyCount": 1,
      "createdAt": "2026-02-20T14:00:00Z"
    }
  ],
  "nextCursor": "clx...",
  "hasMore": true
}
```

Pagination: cursor-based on `createdAt` of root comments, 20 per page. Replies are loaded inline (max 100 per root — this is a practical limit, not a hard constraint).

### Backend — Modify: `src/services/notification.ts`

Add notification type `comment_mention` to the existing `createNotification()` function. Payload:

```json
{
  "type": "comment_mention",
  "taskId": "...",
  "taskTitle": "Fix auth redirect",
  "commentId": "...",
  "commentPreview": "Looks good, @alice should review the...",  // First 100 chars
  "mentionedBy": { "id": "...", "name": "Bob" }
}
```

This follows the existing pattern — `notification.ts` already handles `task_assigned` and `task_due` types.

### Frontend — New file: `src/components/comment-thread.tsx`

The comment UI lives inside the existing `task-detail.tsx` panel, below the task description.

**Component tree:**
```
<TaskDetail>
  ... existing task fields ...
  <CommentThread taskId={task.id}>        # New component
    <CommentComposer />                    # Textarea + submit (root level)
    <CommentList>
      <CommentItem>                        # Single comment with avatar, name, timestamp, body
        <ReplyList />                      # Nested replies (visually indented)
        <ReplyComposer />                  # Inline reply textarea (shown on "Reply" click)
      </CommentItem>
    </CommentList>
  </CommentThread>
</TaskDetail>
```

**@mention autocomplete:** When user types `@` in the composer, show a dropdown of project members filtered by what they type next. Use the existing `GET /api/projects/:id/members` endpoint. Debounce 200ms.

### Frontend — New file: `src/hooks/use-comments.ts`

- `useComments(taskId)` — React Query hook, fetches `GET /api/tasks/:taskId/comments`, infinite scroll pagination
- `useCreateComment(taskId)` — mutation, optimistic update (insert comment at top with pending state)
- `useDeleteComment()` — mutation, optimistic removal

### Frontend — Modify: `src/hooks/use-realtime.ts`

Add handler for `comment:created` event:
- If the comment's `taskId` matches the currently viewed task, inject it into the React Query cache for `useComments`
- This gives real-time updates without polling

### Frontend — Modify: `src/components/task-detail.tsx`

- Import and render `<CommentThread>` below the task description section
- Add comment count badge next to the task title (fetch from `replyCount` sum)

## What NOT to Change

- **Don't add email notifications** — that's a separate feature with its own delivery infrastructure
- **Don't add reactions/emoji** — keep scope tight, this is comments only
- **Don't add file attachments to comments** — future scope
- **Don't change the WebSocket protocol** — use the existing event format
- **Don't add comment editing history/audit trail** — just update `editedAt` timestamp
- **Don't refactor the existing notification service** — extend it, don't rewrite it

## Edge Cases

| Scenario | Expected Behavior |
|----------|-------------------|
| User mentions themselves | No self-notification |
| User mentions someone not in the project | Silently ignored (no error, no notification) |
| Comment on deleted task | 404 — cascade delete handles orphan prevention |
| Edit after 15 min window | 403 with message "Comments can only be edited within 15 minutes" |
| Delete root comment with replies | Cascade delete all replies |
| @mention in a reply | Same behavior as root comment mentions |
| WebSocket disconnected when comment posted | Client reconnects and React Query refetch catches up |
| Empty comment body (whitespace only) | 400 validation error |
| Rapid duplicate submissions | Debounce on frontend (300ms), no server-side dedupe needed |

## Verification Checklist

- [ ] `npx prisma migrate dev` runs clean, new table created
- [ ] `POST /api/tasks/:taskId/comments` creates comment and broadcasts WebSocket event
- [ ] `GET /api/tasks/:taskId/comments` returns paginated results with nested replies
- [ ] Replies enforce one-level-deep constraint (reject reply-to-reply)
- [ ] @mention autocomplete shows project members, resolves to notifications
- [ ] Real-time: open task in two browser tabs, comment in one, appears in other
- [ ] Edit window: edit succeeds at 14 min, fails at 16 min
- [ ] Delete cascade: delete root comment removes replies from DB and UI
- [ ] Optimistic updates: comment appears immediately before server confirms
- [ ] No regressions: existing task CRUD and notification flows unchanged
