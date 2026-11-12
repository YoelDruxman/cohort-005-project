# What Cadence sends and does today that notifications would hook into

Research for [#3](https://github.com/YoelDruxman/cohort-005-project/issues/3), part of the map [#1](https://github.com/YoelDruxman/cohort-005-project/issues/1). The only source is this codebase, at commit `cfd16d5`. Paths are relative to the repo root. This is a list of facts, not a proposed catalogue. Choosing the catalogue belongs to [#5](https://github.com/YoelDruxman/cohort-005-project/issues/5).

## Summary

- Cadence sends **three emails**, all to the buyer: Refund Request raised, Refund Request approved, and Refund Request declined. Nothing else sends anything.
- `sendEmail` is called directly in two route actions, **after** the DB write has committed. It is awaited, and there is no try/catch, retry, queue, or outbox table.
- A failed send therefore turns into a **500 after the state change has already been saved**. When the user retries they get a 409, and the email is never sent.
- **Nothing runs outside a request.** There is no cron job, worker, queue, or scheduler. The only script is `scripts/seed.ts`.
- There are about 15 other state changes a Notification Event could hook onto: Purchases, Seat claims, comments and replies, quiz attempts, course status changes, ratings and more. None of them notify anyone. One has a dead `sendEmail` flag with a "Would send welcome email here" stub.

## 1. Email today

### Transport: `app/lib/mailer.server.ts`

- `Email = { to, subject, body }`. The body is plain text only (`TextBody`). There is no HTML, no template system, and no per-recipient name in the From address. `FROM_ADDRESS` is hard-coded as `"Cadence <hello@cadence.dev>"`.
- `MailTransport` is an interface with `send(email): Promise<void>`. There are two implementations:
  - `postmarkTransport(token)`: a `fetch` POST to `https://api.postmarkapp.com/email` on `MessageStream: "outbound"`. A non-2xx response **throws** `Postmark rejected the email to <to>: <status>`. There is no timeout, retry or idempotency key, and network errors also throw.
  - `outboxTransport(dir)`: writes `${Date.now()}-${to}.json` into the directory with a synchronous `writeFileSync`. Two emails to the same address in the same millisecond would overwrite each other. `.outbox/` is gitignored (`.gitignore:35`).
- `defaultTransport()` is chosen **for each call**: it uses Postmark if `process.env.POSTMARK_SERVER_TOKEN` is set, and `.outbox` otherwise. There is no DI container. `sendEmail(email, transport = defaultTransport())` takes the transport as an optional argument, which is how `app/lib/mailer.server.test.ts` injects an outbox in a temp directory. No `.env` / `.env.example` exists in the repo.
- So `MailTransport` already works as a small Channel-like seam: one interface with interchangeable providers. It covers email only.

### Where emails are built and sent

All three emails are built in `app/lib/refunds.server.ts` as plain `Email` values and returned in the domain result. The routes do the sending. `refunds.server.ts` also declares its own duplicate `Email` type.

| Email | Built in | Sent from | Recipient |
| --- | --- | --- | --- |
| "Refund requested — {course}" | `raiseRefundRequest` | `app/routes/courses.$slug.refund.ts` `action` (line 75) | buyer (Student or Team Admin) |
| "Refund approved — {course}" | `approveRefundRequest` → `buildDecisionEmail` | `app/routes/admin.refunds.tsx` `action` (line 130) | buyer |
| "Refund declined — {course}" | `declineRefundRequest` → `buildDecisionEmail` | `app/routes/admin.refunds.tsx` `action` (line 130) | buyer |

Nobody else is emailed. For example, no Platform Admin is told when a Refund Request is opened, and the Team Admin's teammates are not told either. `buildDecisionEmail` falls back to `to: ""` if the buyer row is missing.

### Failure handling and transactions

- **Raise** (`raiseRefundRequest`): the checks and the `insert(refunds)` are **not** wrapped in a transaction. The insert is a single statement that autocommits. The route then runs `await sendEmail(result.email)` with no try/catch.
- **Approve / decline**: the work runs inside `db.transaction(...)`, which covers the Enrollment delete or Seat `refundedAt` update and the `refunds` status update. The email *content* is built inside the transaction, but the transaction **commits before** the route sends. better-sqlite3 transactions are synchronous, so an async `fetch` could not run inside one anyway.
- What happens when a send fails, in both flows:
  - The action throws, and React Router returns a 500 or error boundary to the user.
  - The DB change is already saved. The Refund Request is open or decided, and the Enrollment may already be gone.
  - A retry by the user hits the 409 guard ("You already have an open refund request…" / "This refund request was already approved"). The email is never sent and nothing records that it failed.
  - There is no retry, no logging beyond the thrown error, and no delivery record.
- In dev and test there is no token, so the outbox write effectively always succeeds. This fragility would only appear in production.

### Stale documentation

`docs/adr/0010-refund-emails-go-to-the-server-log.md` still says refund emails go to `console.log` and that "no learner actually receives the confirmation". The code has moved on to Postmark/outbox. The ADR was not superseded and says nothing about Postmark.

### Other signals

- `README.md` ("Signing Up Later") tells course-takers to create a free **[Knock](https://knock.app/)** Developer account "when you build notifications", with keys in `.env`. Knock is a hosted notification service, not open source. This matters for the build-vs-buy question.
- `_internal/feature-ideas.md` items 23 (in-app notifications with a bell) and 29 (email notifications: welcome, progress reminders, completion, new content).
- `refunds.server.ts`'s request email promises "we will email you again when it is done". The decision email keeps that promise.

## 2. Work outside a request

There is none. A search for cron, queues, workers, `setInterval` and `setTimeout` finds only:

- `app/components/youtube-player.tsx`: a client-side `setInterval` that posts watch progress to `/api/video-tracking`.
- `app/routes/team.tsx`: a client-side `setTimeout` that resets a "Copied" UI state.
- `scripts/seed.ts` (`npm run db:seed`), a one-off script.

The app runs as `react-router-serve`, and all DB work happens in loaders and actions. Nothing time-based exists either. For example, nothing fires when a refund window closes or a question goes stale. `instructor.questions.tsx` computes "stale after 3 days" only when the page is rendered. Anything time-triggered, any batching window, and any retry would all need new infrastructure, unless a library supplies it.

## 3. Candidate Notification Events visible in the code

"Roles plausibly interested" is an inference from the domain, not a decision. Roles follow `GLOSSARY.md`. The code's `UserRole.Admin` is the Platform Admin, and `TeamMemberRole.Admin` is the Team Admin. Unless noted, the writes are bare autocommitted statements in a route file, with no transaction and no service layer.

| # | Candidate event | Where it happens | Plausibly interested | Emails today? |
| --- | --- | --- | --- | --- |
| 1 | Individual Purchase made (which also creates an Enrollment) | `app/routes/courses.$slug.purchase.tsx` `action`, intent `confirm-purchase` → `createPurchase` + `enrollUser(…, sendEmail=false, …)`. These are two separate inserts with no transaction. | Student (receipt, welcome); Instructor (new Student); Platform Admin (sales) | No. `enrollUser` has a dead `sendEmail` param and a `// Would send welcome email here` stub. |
| 2 | Team Licence bought (Seats granted) | same `action`, intent `confirm-team-purchase` → `createTeamPurchase` (Purchase, team and team member if new, then N `coupons` rows, all separate inserts) | Team Admin (receipt, Seat links); Instructor; Platform Admin | No |
| 3 | Seat claimed (and an Enrollment created) | `app/routes/redeem.$code.tsx` `redeemCoupon`: updates `coupons.redeemedAt`/`redeemedByUserId`, then inserts the Enrollment. No transaction. | Team Admin (the human's example); claiming Student (welcome); Instructor | No |
| 4 | Refund Request raised | `app/lib/refunds.server.ts` `raiseRefundRequest`, called from `courses.$slug.refund.ts` | buyer (Student or Team Admin); Platform Admin (has work to review) | **Yes**, buyer only |
| 5 | Refund Request approved | `refunds.server.ts` `approveRefundRequest` (transaction), from `admin.refunds.tsx` | buyer (Student or Team Admin) | **Yes**, buyer only |
| 6 | Refund Request declined | `refunds.server.ts` `declineRefundRequest` (transaction), from `admin.refunds.tsx` | buyer | **Yes**, buyer only |
| 7 | Enrollment ended by an approval (the Student becomes Lapsed) | inside `approveRefundRequest` (`tx.delete(enrollments)` unless they hold a Claimed Seat) | the Student (the same person as the buyer for an Individual Purchase); Instructor (roster change) | Covered only by the approval email |
| 8 | Unclaimed Seats refunded | inside `approveRefundRequest` (`tx.update(coupons).set({ refundedAt })`) | Team Admin (the same person as the buyer) | Covered only by the approval email |
| 9 | Pending / Issued / Failed Refund | **Not in code.** These terms exist only in `GLOSSARY.md`. There is no payment provider. Approval sets `amountRefunded` but nothing tracks the money being sent. | buyer; Platform Admin (Failed Refund) | No (no such event exists) |
| 10 | Comment posted (a top-level question on a lesson) | `app/routes/courses.$slug.lessons.$lessonId.tsx` `createComment` (`parentId === null`), intent `create-comment` | Instructor of the course (it joins their `/instructor/questions` queue); Platform Admin (who also sees the queue) | No |
| 11 | Reply posted | the same `createComment` with `parentId` set (only one level deep). Staff replies also go through a **duplicate** `createComment` in `app/routes/instructor.questions.tsx` `action`, intent `reply` | author of the parent comment (Student); Instructor if a Student replied | No |
| 12 | Comment edited / deleted (soft-deleted, possibly by staff) | `updateComment` / `deleteComment` in the lesson route | author, if staff deleted it | No |
| 13 | Quiz attempt submitted (score, passed if > 0.7) | lesson route `computeResult`, intent `submit-quiz`. It inserts `quizAttempts` and `quizAnswers` and swallows errors with `console.log` → 500 | Student (result); Instructor (maybe) | No |
| 14 | Lesson completed | lesson route `markLessonComplete`, intent `mark-complete` | Student (progress, course completion, which is not modelled as its own event); Instructor | No |
| 15 | Course status changed (Draft / Published / Archived) | `app/routes/instructor.$courseId.tsx` `updateCourseStatus` (intent `update-status`, by the Instructor or Platform Admin), **and** a duplicate in `app/routes/admin.courses.tsx` (Platform Admin) | on publish: Platform Admin, possibly future Students; on archive or unpublish: enrolled Students, and the Instructor if a Platform Admin did it | No |
| 16 | Course content added (module or lesson added, lesson updated) | `instructor.$courseId.tsx` intents `add-module`, `add-lesson`, etc.; `instructor.$courseId.lessons.$lessonId.tsx` `update-lesson` | enrolled Students ("new content", feature idea 29) | No |
| 17 | Price / PPP changed | `instructor.$courseId.tsx` `update-price`, `update-ppp-enabled` | Platform Admin (maybe) | No |
| 18 | Course created | `app/routes/instructor.new.tsx` `action` (Instructor only) | Platform Admin | No |
| 19 | Course rated / re-rated | `app/routes/courses.$slug.tsx` `rateCourse` (upsert, enrolled Students only) | Instructor | No |
| 20 | Quiz created / deleted (deleting a quiz deletes Students' attempts) | `app/routes/instructor.$courseId.lessons.$lessonId.quiz.tsx` `save-quiz` / `delete-quiz` | Students who had attempts (on delete) | No |
| 21 | User signed up | `app/routes/signup.tsx` `action`: inserts a Student. There is no password or email verification, and an existing email is silently logged in. | the new Student (welcome); Platform Admin | No |
| 22 | User role changed / profile edited by an admin | `app/routes/admin.users.tsx` `updateUserRole` / `updateUser` (Platform Admin) | the affected user | No |
| 23 | Video watched / progress | `app/routes/api.video-tracking.ts` (inserts `videoWatchEvents`, posted on an interval). It is high volume. Its only bearing on notifications is the Watch Limit (refund eligibility lost). | Student (maybe, e.g. "you've passed the Watch Limit") | No |

Not candidates, or no matching code found:

- **Coupon redemption** other than a Seat claim. The `coupons` table *is* the Seat table, and there are no discount coupons.
- **Team membership** changes beyond the Team Admin row that `getOrCreateTeamForUser` creates. Claiming a Seat does not add the Student to `team_members`.
- Category CRUD (`admin.categories.tsx`) and settings and profile edits by the user themselves (`settings.tsx`).

## 4. Facts that will shape the design

- Every candidate event is an inline DB write in a route action or a route-local helper. There is **no domain event layer and no service module**, except `refunds.server.ts`. Hooking in means touching each action. Some helpers are duplicated across routes (`createComment` ×2, `updateCourseStatus` ×2), so one event has more than one origin.
- Only the refund approve/decline flows use a transaction. A transactional outbox would need those flows, and the others, to write the Notification in the same transaction as the state change. better-sqlite3's synchronous transactions allow that for DB writes, but not for an async send.
- Recipients are not stored anywhere. The buyer is `purchases.userId`, the Instructor is `courses.instructorId`, the Team Admin is the `team_members` row with role `admin`, and Platform Admins are `users.role = 'admin'`.
- The only existing "notification-like" UI is pulled when a page loads. Examples are the questions queue and the Refund Request lists on `/settings`, `/team` and `/admin/refunds`. There is no push or real-time mechanism.
