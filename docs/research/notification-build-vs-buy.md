# Notification infrastructure for Cadence: free / open-source options vs building

Research for [#2](https://github.com/YoelDruxman/cohort-005-project/issues/2), part of the map [#1](https://github.com/YoelDruxman/cohort-005-project/issues/1). Researched **2026-11-12**. This area changes fast, so every version, price and limit below is as of that date. Check it again before you rely on it.

This document compares the options. It makes no decision; the build-vs-buy call is [#4](https://github.com/YoelDruxman/cohort-005-project/issues/4).

## Cadence's constraints

- React Router 7 (v8.3.0 packages) on Node, with **one SQLite file through Drizzle** (ADR 0006). There is no Postgres, Mongo or Redis, and no job runner or worker process.
- Email goes out through Postmark's HTTP API in `app/lib/mailer.server.ts`, with a `.outbox/` JSON transport for dev and tests.
- What the human wants: recipient rules per Notification Event and per role; preferences per Notification Category × Channel that can later go down to a single event; mandatory and optional Categories; batching that sends the first Notification immediately and batches later ones inside a window; an in-app bell/feed with real-time updates; reliable delivery; and a free or OSS option if one fits.

## Summary table

| | **Novu self-hosted (Community)** | **Novu Cloud Free** | **Knock Free** | **SuprSend Free** | **Courier Developer (free)** | **MagicBell Builder (free)** | **Build in-house** |
|---|---|---|---|---|---|---|---|
| Licence / model | MIT core, commercial `enterprise/` folder | Hosted SaaS | Hosted SaaS (MIT SDKs) | Hosted SaaS (MIT SDKs) | Hosted SaaS (MIT SDKs) | Hosted SaaS | Ours |
| Free limit | Limited by your infrastructure | 10k workflow runs/mo, **5 workflows**, **24 h feed retention** | 10k messages/mo, unlimited workflows | 10k notifications/mo | 10k sends/mo | **1k deliveries/mo** | None |
| Recipient rules | Cadence computes the recipients and calls trigger per subscriber/topic | same | Cadence computes the recipients | same | same | same | Cadence |
| Category × Channel prefs | Per workflow × channel plus global per channel. **No server-side Category**: tags only group the UI | same | **Native**: `categories` × `channel_types`, per-workflow as well | **Native**: sections → categories, per-channel opt-out, "Can't Unsubscribe" | Subscription topics (OPTED_IN/OUT/REQUIRED), channel choice per topic | Categories × channels | Ours to model |
| Mandatory | "Critical" workflow | same | "Override recipient preferences" per workflow | "Can't Unsubscribe" + mandatory channels | `REQUIRED` topic | n/a found | Ours |
| First-immediate-then-batch | **Yes** ("When events repeat" digest start) | Yes (plan limits on window unclear) | **Yes** ("Immediately flush leading item") | Yes ("Flush first item immediately"), **but batching is not on Free** | Digest/batching **Enterprise only** | Not documented | Ours |
| React inbox UI | `@novu/react` 3.19.2 (Inbox, Bell, Preferences) | same | `@knocklabs/react` 0.15.1 | `@suprsend/react` 1.5.0 | `@trycourier/courier-react` 9.3.5 | `@magicbell/react` 1.14.0 | Build (shadcn/Radix already in repo) |
| Real-time | Yes, its own `ws` service | Yes | Yes (sockets) | Yes | Yes | Yes | Build (SSE or polling) |
| Provider retries | **None**: one attempt per provider send | **None** | 8 retries over 30 min, exponential backoff | Not checked in depth | Not checked in depth | Not checked | Ours (outbox + retry) |
| Postmark | Yes | Yes | Yes (with delivery webhooks) | Yes (with webhooks) | Yes | **Not listed** (SMTP only) | Already have it |
| Self-host needs | Mongo + Redis (AOF) + S3-compatible storage + 4 containers; 4 vCPU / 8 GB recommended for a single VM | n/a | n/a (Enterprise only) | n/a (Enterprise only) | n/a | n/a | Nothing new: SQLite + Node |
| Branding on free | Novu branding stays in Community | stays | Knock branding stays | SuprSend branding on the feed | — | — | none |

## Self-hostable open source

### Novu (the only credible OSS fit)

- **Repo and licence.** [novuhq/novu](https://github.com/novuhq/novu) has about 40.3k stars. The README says the core is MIT, while the root `enterprise/` folder is under a separate commercial licence (LICENSE-ENTERPRISE). GitHub reports the repo licence as `NOASSERTION` because of that split. The latest server release is **v3.19.0 (2026-08-07)**; `@novu/react` is at **3.19.2** (2026-09-07).
- **What self-hosting needs.** The [community docker-compose](https://github.com/novuhq/novu/blob/next/docker/community/docker-compose.yml) runs `redis:alpine`, `mongo:8.0.17` and four Novu images (`api`, `worker`, `ws`, `dashboard`, all 3.19.0). The [self-hosting overview](https://docs.novu.co/community/self-hosting-novu/overview) recommends 4 vCPU / 8 GB RAM for a single-VM setup, Redis with AOF persistence for the queues, MongoDB (2 GB RAM / 20 GB disk), and at least 10 GB of S3-compatible object storage. That is a separate stack Cadence would have to run and back up: Mongo, Redis and object storage next to one SQLite file.
- **Community vs Cloud** ([comparison](https://docs.novu.co/community/self-hosted-and-novu-cloud)). Self-hosted includes digest, the Inbox, Bell, Notifications and Preferences components, the block and HTML email editors, and code-first Framework workflows. It does **not** include: Snooze, removing Novu branding, email activity tracking (delivery and engagement events), workflow translations, more than 2 environments, environment variables, SSO, RBAC or MFA, and it allows **1 team member**. Auth is single-user login, or "bring-your-own Clerk", which is Enterprise ([v2.2.0 changelog, 2025-05-18](https://novu.co/changelog/novu-community-edition-v220-self-hosting)). The [pricing page](https://novu.co/pricing) offers supported "On-Prem Self-Hosted" only on Enterprise.
- **Preferences** ([docs](https://docs.novu.co/platform/concepts/preferences)). Workflow-level channel defaults, subscriber global per-channel preferences, and subscriber per-workflow per-channel preferences. "The most specific layer that defines a channel wins." A workflow marked **critical** cannot be opted out of and is hidden from the preferences UI. `preferenceGroups` (SDK ≥ 3.4.0) groups workflows into sections **in the UI only**. Preferences are "stored per workflow", and [the Inbox preferences docs](https://docs.novu.co/platform/inbox/configuration/preferences) mention no server-side Category-level preference. A Category × Channel toggle therefore has to be emulated by flipping every workflow in that Category.
- **Batching** ([digest docs](https://docs.novu.co/platform/workflow/add-and-configure-steps/configure-action-steps/digest)). The "When events repeat" start option does exactly what the human asked for: if no similar notification went out recently, Novu delivers immediately and skips digesting; otherwise it starts a digest. The docs say it is for "when you want the first notification delivered instantly, while batching subsequent events that occur close together." Digests can be grouped per subscriber plus one payload key, and scheduled digests are also available.
- **Reliability** ([delivery retry policy](https://docs.novu.co/platform/developer/delivery-retries)). **Provider sends are not retried.** Novu makes "a single call to the provider", and "provider rejections are treated as terminal regardless of the status code", including 429 and 503. Failed steps have no dead-letter queue; they show up only in the Activity Feed and the `messages.failed` webhook, and there is no fallback provider. Only bridge (code-step) calls are retried (3 times, 1/2/4 s), along with Novu's outbound webhooks. Cadence would have to build retries itself by listening to `messages.failed` and re-triggering. The [How Novu Works](https://docs.novu.co/platform/how-novu-works) page vaguely says failed steps are retried. The dedicated retry policy page is more specific, so this document follows it.
- **Postmark.** Supported as an email integration ([docs](https://docs.novu.co/platform/integrations/email/postmark)).
- **Real-time.** The Inbox updates in real time through Novu's own `ws` service.
- **Lock-in.** Workflows, templates, subscribers, preferences and the notification history all live in Novu's Mongo, and the Inbox UI is Novu's. Code-first workflows (`@novu/framework` v2.14.0) keep the workflow definitions in Cadence's repo, which softens the lock-in a little.

### Other OSS projects checked and not shortlisted

- **Dittofeed** ([repo](https://github.com/dittofeed/dittofeed), MIT, v0.23.0 released 2025-12-01). This is customer-engagement and marketing journeys, not a product in-app inbox. Its [lite compose](https://github.com/dittofeed/dittofeed/blob/main/docker-compose.lite.yaml) needs Postgres 15, ClickHouse and Temporal, which is heavier than Novu.
- **Notifo** ([repo](https://github.com/notifo-io/notifo), MIT). It is .NET and MongoDB, and its last release was 1.3.0 on **2022-11-30**, so it looks effectively dormant.
- **Notifuse** (newsletter and email marketing) and **Apprise** (BSD-2, push-to-many-services) solve different problems.
- **notifme-sdk** (MIT, last release v1.16.25 on 2025-03-25) only covers delivery to providers. It adds nothing that Cadence's mailer doesn't already do.

No OSS option runs on Node + SQLite. Every credible one brings its own database and queue stack.

## Hosted free tiers

### Knock

- **Free plan** ([pricing](https://knock.app/pricing)): $0, up to **10,000 messages/month**, unlimited workflows, in-app feeds, batching and preferences. Knock branding stays on Free. Knock says it doesn't hard-cap and will contact you if you go over. The next tier, Starter, is $250/month.
- **Preferences** ([docs](https://docs.knock.app/preferences/overview)). The `PreferenceSet` has `categories`, `workflows`, `channel_types` and `channels`, and they nest, so you can express **Category × Channel type and per workflow**. That is the closest match to the human's model. Knock sends only "if all preference combinations that exist on the recipient evaluate to `true`", and unset values default to true. For mandatory notifications there is a per-workflow "Override recipient preferences".
- **Batching** ([docs](https://docs.knock.app/designing-workflows/batch-function)). Fixed, sliding (with a max window) and dynamic windows, plus a batch key. **"Immediately flush leading item"** sends the first item immediately and batches the later ones, which is exactly the reference behaviour.
- **In-app** ([React feed docs](https://docs.knock.app/in-app-ui/react/feed)). `@knocklabs/react` 0.15.1 provides `NotificationIconButton`, `NotificationFeedPopover` and the feed, updated in real time over sockets, with seen and read states.
- **Reliability** ([docs](https://docs.knock.app/send-notifications/delivering-notifications)). Up to **8 retries over 30 minutes** with exponential backoff and jitter.
- **Postmark** ([docs](https://docs.knock.app/integrations/email/postmark)). Supported, using the `outbound` stream by default, with delivery and bounce webhooks back into Knock.
- **Lock-in.** High. Workflows, templates, preferences and feed history all live in Knock. There is no self-hosting below Enterprise.

### SuprSend

- **Free plan** ([pricing](https://www.suprsend.com/pricing)): 10k notifications/month, unlimited channels, an in-app feed with SuprSend branding, and branching. The page lists **batching/digest and user preferences under Business ($250/month)**, not Free. Self-hosting is Enterprise only.
- **Preferences** ([docs](https://docs.suprsend.com/docs/user-preferences)). Sections → categories, each with On, Off or **Can't Unsubscribe**, plus mandatory channels and per-category channel opt-outs. Conceptually this is a strong fit, but per the pricing page it is not free.
- **Batching** ([docs](https://docs.suprsend.com/docs/batch)) has "Flush first item immediately". **Postmark** is supported, with webhooks ([docs](https://docs.suprsend.com/docs/postmark)). React inbox: `@suprsend/react` 1.5.0. The older `suprsend-react-inbox` repo is archived.

### Courier

- **Developer plan** ([pricing](https://www.courier.com/pricing)): free up to 10k sends/month. **"Digest and batching" is listed under Enterprise only.** Adding a card moves you to pay-as-you-go at $0.005 per send, including the first 10k.
- **Preferences** ([docs](https://www.courier.com/docs/platform/preferences/preferences-overview)): subscription topics set to OPTED_IN, OPTED_OUT or **REQUIRED**. A user can pick channels per topic when custom routing is enabled.
- **Inbox** ([docs](https://www.courier.com/docs/platform/inbox/inbox-overview)): `@trycourier/courier-react` v9 (9.3.5) with real-time updates and read/archive. The old `courier-react` repo is archived. **Postmark** is supported ([docs](https://www.courier.com/docs/external-integrations/email/postmark)).

### MagicBell

- **Builder (free)** ([pricing](https://www.magicbell.com/pricing)): only **1,000 deliveries/month**, then Startup at $249/month.
- The [docs](https://www.magicbell.com/docs) list SES, Mailgun, SMTP and SendGrid as email providers, but **not Postmark** (Postmark does offer SMTP). It has categories × channel preferences, but no batching or digest documentation turned up. Not a fit.

## Building in-house on Cadence's stack (a sketch at the same level)

| Concern | What it would take |
|---|---|
| Event catalogue and recipient rules | A typed catalogue in code: each Notification Event has a payload type, a Category, and a recipient resolver (role → users) that queries Drizzle. Every hosted option needs this too, because none of them knows Cadence's roles. |
| Notifications table | A `notifications` table (recipient, event, payload, created/read/archived timestamps) serves as both the In-app Channel and its history. |
| Preferences | A `notification_preferences` table keyed by (user, category, channel), with an optional event column added later for finer control. Mandatory Categories are a flag in code. About 2 tables plus one resolution function. |
| Delivery and reliability | A transactional **outbox** in the same SQLite write as the domain change, so the notification cannot be lost. A worker loop drains it with retries and backoff. With no job runner, it can be an in-process interval in the server, or an SQLite-backed queue library such as [`liteque`](https://www.npmjs.com/package/liteque) (MIT, 0.10.0, updated 2026-10) or `plainjob` (MIT, 0.0.14, last updated 2024-10, stale). SQLite allows one writer, which is fine at Cadence's volume. |
| Batching (first now, rest in a window) | On each event, look up a recent delivery for (recipient, event or batch key) inside the window. If there is none, send now and record it. If there is one, append to an open batch row whose `flush_at` is the window end, and let the worker flush it. A small, testable state machine. |
| Email Channel | Wrap the existing `MailTransport` (Postmark, or `.outbox/` in dev) as a Channel. |
| In-app UI and real-time | A bell and dropdown built with the existing Radix/shadcn and lucide components. Real-time via polling a loader/`useRevalidator`, or Server-Sent Events from a resource route. SSE is single-process friendly and needs no new infrastructure. |
| Preferences UI | A Category × Channel grid of toggles on the settings page. |
| Costs | No vendor, no new infrastructure, data stays in SQLite, and the dev/test doubles are trivial. In exchange, Cadence owns retries, batching edge cases, the UI and monitoring. |

## Lock-in and fit, side by side

- **Infrastructure fit:** build > hosted (no infrastructure, but an external dependency) > Novu self-hosted (Mongo + Redis + S3 + 4 containers, alongside SQLite).
- **Model fit (Category × Channel, mandatory, first-then-batch):** Knock (native and free) ≈ SuprSend (native, but batching and preferences are paid) > Novu (batching is native; Category has to be emulated) > Courier (no batching on free) > MagicBell.
- **Reliability out of the box:** Knock (documented retries) > build (whatever we write) > Novu (no provider retries).
- Every option still needs Cadence to compute recipients from roles and to keep its own source of truth for who the users are (subscriber sync).

## Shortlist for the build-vs-buy decision (#4)

1. **Build in-house** on SQLite with an outbox, a worker and SSE. It fits the stack and the preferences model exactly, at the cost of writing it.
2. **Novu Community, self-hosted.** The only credible OSS option, and it does first-then-batch natively. Against it: a heavy separate stack, no provider retries, no native Category preference, single-user admin, and Novu branding.
3. **Knock Free.** The best model fit and the best reliability of the hosted tiers, and free up to 10k messages/month. Against it: SaaS lock-in and Knock branding.

Not shortlisted: Novu Cloud Free (5-workflow cap and 24 h feed retention), SuprSend Free (batching and preferences are paid), Courier Developer (digest/batching is Enterprise only), MagicBell (1k/month, no Postmark), Dittofeed and Notifo (wrong shape, or dormant).

## Findings that affect other tickets

- **Reliability:** buying does not automatically mean reliable delivery. Novu does not retry provider sends at all. Whichever option is chosen, Cadence still needs some form of durable hand-off (an outbox, or retry on trigger failure), because there is no job runner today.
- **Preferences:** only Knock (free) and SuprSend (paid) store Category-level preferences natively. Novu stores preferences per workflow × channel, so Category would be an app-side concept either way.
- **Batching:** "first immediately, then batch in a window" is a named, built-in feature in Novu, Knock and SuprSend, which confirms it is a standard pattern and gives a reference for its semantics if Cadence builds it.
- **Real-time:** every vendor ships sockets in its inbox SDK. If Cadence builds, SSE or polling is the main choice, and that is cheap on one Node process.
- **Recipient rules:** no vendor models Cadence's roles. The role → recipient resolution is Cadence code in every option.
