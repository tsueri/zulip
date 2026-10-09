# Upstream notification-issue landscape (zulip/zulip)

Ticket: https://github.com/tsueri/zulip/issues/10 · Date: 2026-10-09

Scope: triage of the 161 open zulip/zulip issues matching `is:issue state:open notification`;
26 read in depth across four clusters. Companion to `research/prior-art-dnd.md` (which covers
the DND-specific issues); this file covers the broader notification landscape those issues sit in.

## Cluster A — Suppression & scheduling

### zulip/zulip#29098 — Make it possible to schedule do-not-disturb times
- Ask: schedule when notifications pause. Open, `feedback wanted`, `new feature`; no product spec, no PR.
- Maintainers: linked CZO thread — alya's open question is whether scheduling should be **tied to user
  status or separate**; a commenter (Ben O) preferred a **separate schedule**.
- Bearing: this is exactly the core decision of our map. Our lean (status-tied attribute) must engage
  this thread head-on; a separate-schedule design is a live competitor.

### zulip/zulip#6629 — Temporarily suppress notifications for all channels (umbrella)
- Ask: blanket suppression while still using Zulip. Open, `feedback wanted`, `priority: high`.
- Maintainers: Tim — web app should own the setting so desktop follows it; Akash Nimare proposed
  **DND/Busy/Invisible/custom statuses**; Tim agreed status should be set in the web app. alya (2025)
  agreed scheduled DND would help. The original request also wanted a **visible DND indicator** and
  **others not seeing the user as active**; users report the desktop-app-only DND misses mobile/browser.
- Bearing: closest precedent for blanket suppression; its desired surface (presence signaling, visual
  notifications) exceeds our push-only scope — our spec must say what it deliberately leaves out.

### zulip/zulip#17008 — Timed Mutes (topics)
- Ask: mute a topic for a fixed duration. Open, `priority: high`. Expiration discussed at length
  (`date_muted` etc.); was sequenced behind #25673 (now done), so the topic-expiry work is unblocked upstream.
- Bearing: supplies an expiration *pattern* (per-object, nullable expiry) but is topic-scoped — no
  global-DND precedent either way.

### zulip/zulip#21310 — Temporarily mute Zulip audio
- Ask: audio-only mute that preserves visual/desktop notifications. Open, `needs product spec`.
- Bearing: useful scope boundary — upstream distinguishes channel-specific muting from global DND.

### zulip/zulip#11613 / #37082 — ambient notifications / channel-overload suggestions
- Presentation-layer and onboarding ideas; tangential to a suppression gate. One-line each in resolution.

## Cluster B — Delivery mechanics

### zulip/zulip#19451 — Re-check user settings before sending notifications
- Ask: restore send-time checks of queued email/push settings. Open.
- Maintainers: Tim — **recheck push settings at send time because queue GC can delay delivery up to
  10 minutes**.
- Bearing: **directly endorses our worker-side recheck**; a trigger-time check alone is insufficient for
  already-enqueued pushes. DND's suppression check at send time follows the same principle.

### zulip/zulip#18072 — Server never sends notification if recipient recently used webapp
- Ask: don't permanently lose notifications when a recipient is active then goes idle. Open, `priority: high`.
- Maintainers: Tim agreed a robust approach must remember messages seen while active and notify on idle.
- Bearing: presence/activity gates are a separate mechanism from DND; suppression must not depend on
  activity state, and must cover the idle/GC replay path.

### zulip/zulip#16772 — receiver_is_off_zulip should consider "full clients"
- Ask: bots reading history shouldn't count as activity. Open, `priority: high`; PR zulip/zulip#38795
  open (`maintainer review`, has conflicts).
- Bearing: activity detection is being fixed independently; keep DND orthogonal to it.

### zulip/zulip#35882 — Configurable delay for mobile notifications
- Ask: 2/5/10/30-min push delay like email's. Open, `feedback wanted`; blocker is refactoring the mobile
  push worker to the `MissedMessageWorker` delayed-notification approach.
- Bearing: delayed delivery makes the **send-time** check decisive: a push queued before DND turns on can
  still be suppressed at its eventual send; "DND turned off before the send" is an explicit policy decision
  for the semantics grill.

### zulip/zulip#30316 — Fix visual/push notifications for edited messages
- Ask: update/revoke notifications after edits (e.g. mention removed). Open; Tim: notifications still
  controlled by Zulip should be revocable on deletion, including push.
- Bearing: edit-triggered notification paths must honor DND too; a delivered alert can't be un-delivered —
  define DND's interaction with edits explicitly.

### zulip/zulip#38859 — Improve tornado event-queue garbage collection
- Ask: fix missed/duplicate notifications across GC'd queues. Open, `difficult`; PR zulip/zulip#39725 open
  (`maintainer review`), proposing cross-queue aggregation + dedup before notifying.
- Bearing: GC is a delayed-notification route — DND must apply to replay; our worker-side recheck covers it.

## Cluster C — Settings & semantics

### zulip/zulip#15718 — Allow admins to control user notification settings
- Ask: force notifications for a stream's members. Open.
- Outcome: the landed version (#23873) is **`Stream.default_push_notifications`** — an admin-set *default*
  for new subscriptions; users keep per-subscription control. alya's philosophy: "give users flexibility to
  control their individual experience unless there is a clear motivation not to". Tim also mused about an
  `@all_force_notifications` mention syntax as possibly cleaner.
- Bearing: upstream model is **defaults-not-mandates**. A DND status must be user-controlled; an
  admin-unoverridable DND would be the outlier.

### zulip/zulip#19664 — Channel setting for mandatory email notifications
- Ask: admins require email for a channel. Open; PR zulip/zulip#40046 blocked on API-design sign-off
  (waiting on Chris Bobbe; dormant since 2026-09).
- Design in flight (CZO #api design): channel-level `mandatory_email_notifications`, admin-only, and — the
  key semantic choice — **enforced at send time, stored user preferences untouched**.
- Bearing: the only upstream precedent for "notification that ignores user settings" is send-time
  enforcement; if mandatory-* notifications ever extend to push, the DND-vs-mandatory interaction is a
  genuine open design question (which enforcement wins?), not an accident to leave unspecified.

### zulip/zulip#2183 — @online/@here without push/email
- Ask: mention variant that desktop-notifies only online subscribers. Open.
- Maintainers: Tim — implement as an **`online_mentioned` message flag** ("I can imagine wanting to use
  that distinction in future features").
- Bearing: upstream already separates "the mention happened" from "which delivery channels fire" —
  urgency tiers exist as a concept at the mention level.

### zulip/zulip#28777 — Revamp notifications for alert words
- Ask: replace alert-word notification configs with per-word "follow topic" ("watched phrases"). Open,
  umbrella #28779. Settled CZO design: rename to `watched_phrases`; **explicitly muted topics are never
  followed; muted streams with default topics are**. Tim's tie-break principle for corner cases: start
  with the **visible** failure mode (notify) over the invisible one (don't).
- Bearing: precedent that broad suppression loses to specific deliberate intent — and DND will need a
  **visible** "you were suppressed" surface; the "did my DND eat a mention?" question is one upstream
  has already answered in favor of visibility.

### zulip/zulip#31610 — Groups vs. individuals mention behavior
- Outcome: merged PR zulip/zulip#33591 — compose banner when none of a group's members are subscribed.
  Tim's invariant: **subscription is the gating filter for mention notifications**; gaps are handled by
  sender-side warnings, not receiver-side exceptions.
- Bearing: DND changes the delivery channel, not the audience; don't entangle audience logic with suppression.

### zulip/zulip#16160 — Silent mentions should appear in mentions view
- Ask: `@_username` messages should be searchable/visible. Open, blocked on design.
- Maintainers: Tim sketches a **three-tier mention taxonomy** — silent (no notifications, no counts) /
  light (no email/push, counts + maybe desktop) / mention (current) — and explicitly names the
  alternative: *"just not adjust the syntax and make it easy to configure a Do Not Disturb time window
  that would change how mentions work"*. Commenters (dransome, jamesmunns) reject Slack-style DND
  because it "loses messages sent during the DND period"; what they want is suppressed real-time
  delivery **but full mention counts/unreads when they return**.
- Bearing: the most design-relevant thread. Tim has already framed DND as a plausible upstream design,
  and the counter-requirement is crisp: **DND suppresses real-time channels only; mention flags, counts,
  and unread state stay intact.**

## Cluster D — Push transport & platform

### E2EE mobile push (context from #34240)
- Tracking #35368 closed with all high/medium items checked ("Rework using Device model" #37731, "Skip
  legacy notifications when E2EE is required" #38433, security/data-access docs #39055); open remainder:
  cron GC (#39180) and registration retry/backoff (#39179).
- Bearing: a server-side per-realm/per-device decision layer now exists before content leaves for
  FCM/APNs/bouncer — **DND should be one more policy in that gate, upstream of the bouncer handoff**,
  without touching payload construction or Device internals.

### zulip/zulip#626 / #13337 / #34240 — Web Push API / PWA
- #626 open, dormant; alya: "not a contributor-accessible issue". #13337 opportunistic, no plan.
- #34240: Tim (2025) — other notifications work **"will not get maintainer attention until we've gotten
  end-to-end encryption for mobile notifications fully into production"**; web desktop notifications fine
  via pinned tabs.
- Bearing: transport work is parked behind e2ee. A DND PR framed as *user-status gating at the send-time
  decision layer* is adjacent-but-distinct enough to be reviewable now; it must not entangle with
  Web Push/PWA transports.

### zulip/zulip#27246 — "Send test push notification" in settings
- Endpoint exists (#23997); settings UI pending, slotted in Settings → Notifications → Mobile message
  notifications — **next to where a DND toggle would live**.
- Bearing: DND must define its interaction with test notifications; a user-initiated diagnostic arguably
  bypasses suppression (otherwise "send test" silently fails while DND is on and looks like a bug).

### zulip/zulip#10397 — Soft deactivation and push
- Mostly resolved (#10491; message_send now creates UserMessage rows for configured long-term-idle
  users); open question on `access_message` auto-create left over a performance tradeoff.
- Bearing: **notification paths must tolerate missing UserMessage rows** — keep the DND gate stateless or
  keyed off UserProfile/UserStatus, never off UserMessage.

### zulip/zulip#27581 — Forbid duplicate email notifications for one message
- Tim's proposal: a `sent_email_notification` UserMessage flag mirroring `active_mobile_push_notification`,
  or a richer table; no decision; labeled `difficult`.
- Bearing: per-message notification state flags are considered *difficult* — **prefer a stateless
  suppress-at-send-time gate over adding notification-state flags for DND**.

### zulip/zulip#31460 — Mobile notifications banner refactor
- Banner conflates not-configured vs plan-cap; alya wants that admin-facing status out of the shared banner.
- Bearing: don't communicate user-level state (like DND) through that shared banner.

## Synthesis — implications for the design tickets

1. **The core decision is upstream's own open question.** Status-tied vs separate scheduling is asked in
   #29098's CZO thread (one preference recorded for separate); #6629's thread leans status-in-web-app
   (Tim, Akash). Our attribute-design grill must resolve this with these positions on the table.
2. **Enforcement point: the send-time decision layer.** Tim endorses send-time rechecks (#19451, GC can
   delay push up to 10 min); GC replay (#38859, #18072) and delayed-send (#35882) all route through the
   worker — suppression must apply at send time, covering already-enqueued pushes.
3. **Suppression is channel-scoped, not verdict-scoped.** Tim's silent/light/mention taxonomy (#16160,
   #2183) separates "mention happened" from "channels fired": DND keeps mention flags, counts, and
   unreads intact and suppresses only real-time delivery — the explicit anti-"Slack loses messages" rule.
4. **Visibility beats invisibility.** #28777's tie-break principle and #31610's merged banner show
   upstream answers suppression-eats-something questions with a visible indicator; DND needs a
   client-visible suppressed surface, decided in the UX prototype.
5. **User-controlled, defaults-not-mandates.** #15718→#23873 philosophy rules out admin-mandated DND;
   the only ignore-user-settings precedent (#19664) is narrow, channel-scoped, admin-set, and enforced at
   send time — DND-vs-mandatory must be explicitly reconciled in the spec.
6. **Stateless gate keyed off user state.** Don't add UserMessage notification-state flags (#27581,
   `difficult`); tolerate missing UserMessage rows (#10397); live on UserProfile/UserStatus.
7. **Timing policy decisions to make explicitly:** DND turning on with pushes already queued;
   DND turning off before a queued push's send; edited messages re-triggering or revoking (#30316);
   test notifications bypass (#27246).
8. **Transport alignment:** e2ee push is nearly landed; DND slots into the existing server-side decision
   layer upstream of the bouncer; do not touch payload/Device internals or Web Push/PWA (parked).
