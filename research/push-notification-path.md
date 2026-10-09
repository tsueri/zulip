# Push notification decision path — fact sheet

Research for [tsueri/zulip#4](https://github.com/tsueri/zulip/issues/4)
("Research: Push notification decision path end-to-end"), 2026-10-09.
Base commit: `ebf693e6547fbe400d4992f336579ae2d4233883`.

Purpose: pin down exactly where the server decides to send a mobile push
notification, enumerate every gate where a user-level suppression check
(e.g. a "do not disturb" flag) could sit, and compare hook points. All
references are to this repo at the base commit.

## 0. One-paragraph summary

The *decision* is made twice, in different processes: `do_send_messages`
computes per-user eligibility flags and a presence-idle set synchronously
in the Django process and ships them to Tornado in the `message` event;
Tornado then, per user, rebuilds a `UserMessageNotificationsData` and —
when the user is idle (no active client or presence-idle) — calls
`maybe_enqueue_notifications`, which is the single point that actually
publishes to the `missedmessage_mobile_notifications` queue. The
`PushNotificationsWorker` consumes that queue, re-checks a few gates
against the DB, and sends via legacy (PushDeviceToken) and/or E2EE
(Device) channels, either directly (zilencer) or through the push
bouncer. `UserMessageNotificationsData.get_push_notification_trigger` is
the shared decision funnel for all three Tornado call sites (new
message, message edit, queue GC), so it is the natural home for a
push-only suppression check.

## 1. Message-send path (Django process)

### Per-recipient eligibility sets — `get_recipient_info`

`zerver/actions/message_send.py:287-631` builds a `RecipientInfoResult`
(message_send.py:239-262) with one set of user IDs per notification
concept. For streams, subscription rows feed
`user_allows_notifications_in_StreamTopic`
(`zerver/lib/notification_data.py:292-319`; visibility policy → mute →
stream-specific setting → global setting hierarchy) to produce
`stream_push_user_ids`, `followed_topic_push_user_ids`, wildcard-mention
sets, etc. (message_send.py:407-472).

The UserProfile query then annotates `has_push_device_registered` as
`Exists(Device…push_token_id__isnull=False) | Exists(PushDeviceToken…)`
(message_send.py:509-517) — i.e. it already unifies both the E2EE
`Device` and legacy `PushDeviceToken` device flows — and extracts:

- `online_push_user_ids`: users with `enable_online_push_notifications`
  (message_send.py:561-563).
- `dm_mention_push_disabled_user_ids`: users with
  `enable_offline_push_notifications` *disabled* (message_send.py:571-573).
  Despite the name it covers all default/mention push for DMs and
  mentions.
- `long_term_idle_user_ids`: soft-deactivated users (message_send.py:577-579).
- `push_device_registered_user_ids` (message_send.py:607).

### `UserMessageNotificationsData` per recipient

`do_send_messages` (message_send.py:1044) builds one
`UserMessageNotificationsData.from_user_id_sets(...)` per
`active_user_ids` member (message_send.py:1296-1318), passing the
per-message flags (from the `UserMessage` rows built by
`create_user_messages`, message_send.py:878-957) plus the sets above.

### Presence-idle computation

- `filter_presence_idle_user_ids` (message_send.py:960-977): a user is
  *not* idle iff `UserPresence.last_active_time >= now -
  settings.OFFLINE_THRESHOLD_SECS`. Presence-based only; no
  UserActivity, no UserStatus involved.
- `get_active_presence_idle_user_ids` (message_send.py:980-1005) first
  filters to users for whom `is_notifiable(sender_id, idle=True)` is
  true (message_send.py:1002) — an important optimization so large
  channels don't scan presence for thousands of inactive users — and
  bails out entirely when `realm.presence_disabled`
  (message_send.py:994).
- The result goes into the message event as
  `presence_idle_user_ids` (message_send.py:1320-1324, event at
  message_send.py:1339).

### long_term_idle handling

- At send time: `create_user_messages` skips creating UserMessage rows
  for long-term-idle users *unless* the flags are nonzero or the user is
  in any of the push/email eligibility sets (message_send.py:903-948),
  because the notification code later calls `access_message` and needs
  the row for private streams.
- At push time: `handle_push_notification` handles a missing UserMessage
  row by erroring unless `user_profile.long_term_idle`
  (`zerver/lib/push_notifications.py:1794-1804`), and soft-reactivates a
  long-term-idle user who is personally mentioned before pushing
  (`soft_reactivate_if_personal_notification`,
  push_notifications.py:1838-1841).

### The event to Tornado

`send_event_on_commit(realm, event, users)` (message_send.py:1424)
carries the wide message dict plus all of: `presence_idle_user_ids`,
`online_push_user_ids`, the push/email per-channel ID lists,
`all_bot_user_ids`, `push_device_registered_user_ids`,
`disable_external_notifications`, `realm_host`
(message_send.py:1335-1362).

## 2. Decision logic (`zerver/lib/notification_data.py`)

`UserMessageNotificationsData` (notification_data.py:12-33) holds one
boolean per channel/trigger pair plus `online_push_enabled`,
`sender_is_muted`, `disable_external_notifications`.

`from_user_id_sets` (notification_data.py:53-201):

- Bots get an all-False object (notification_data.py:76-99).
- Every push flag is multiplied by `push_device_registered`
  (notification_data.py:141-179), with a TODO/compat shim hardcoding
  True for events predating the field (notification_data.py:136-139).
- `online_push_enabled = push_device_registered and user_id in
  online_push_user_ids` (notification_data.py:175).

Gates, in evaluation order, in `get_push_notification_trigger`
(notification_data.py:230-257):

1. `if not idle and not self.online_push_enabled: return None`
   (notification_data.py:231-232). `idle` is caller-supplied (see §3);
   this is the "user is online" gate.
2. `trivially_should_not_notify` (notification_data.py:206-220):
   recipient is the sender; sender is muted by recipient; or
   `disable_external_notifications` (internal sends).
3. Trigger precedence ladder: dm → mention → followed-topic wildcard
   variants → topic/stream wildcard → followed-topic push → stream push
   (notification_data.py:240-255); `None` if no flag set
   (notification_data.py:256-257).

`is_push_notifiable` = trigger is not None (notification_data.py:227-228).
`is_notifiable` = email *or* push (notification_data.py:222-225) —
**shared** between email and push, used as a cheap pre-filter
(message_send.py:1002, event_queue.py:1297).

The email twin `get_email_notification_trigger`
(notification_data.py:262-289) has one crucial difference: it returns
`None` whenever `not idle` (notification_data.py:263-264) — email
notifications only ever fire for idle users, there is no
"online email push" analog of `online_push_enabled`.

**Every gate a user-level suppression flag could sit behind:**

| Gate | Where | Notes |
| --- | --- | --- |
| `idle`/`online_push_enabled` | notification_data.py:231 | per-message-time state, caller computes |
| `trivially_should_not_notify` | notification_data.py:234 | channel-independent; shared with email |
| `push_device_registered` | notification_data.py:141-179 | per-user, plumbable as an ID set |
| per-channel trigger flags | notification_data.py:240-255 | per-user/per-message settings |
| worker-side settings recheck | push_notifications.py:1728-1733 | `enable_offline_push_notifications or enable_online_push_notifications` |
| `push_notifications_configured` | push_notifications.py:1722 | server config, not user-level |

`NotificationTriggers` constants: `zerver/models/scheduled_jobs.py:71-82`.

## 3. Delivery path

### Tornado: from event to queue

`process_message_event` (`zerver/tornado/event_queue.py:1139-1354`):

1. Rebuilds `UserMessageNotificationsData.from_user_id_sets` per user
   (event_queue.py:1262-1281) from the event's ID lists and the user's
   message flags.
2. Copies the object (`vars(...)`, event_queue.py:1286-1291) into
   `internal_data`, which rides along in the per-user message event
   (event_queue.py:1338-1340).
3. Short-circuits users for whom the message is not notifiable even
   when idle (event_queue.py:1297-1298).
4. Computes the live idle state:
   `idle = receiver_is_off_zulip(user_id) or (user_id in
   presence_idle_user_ids)` (event_queue.py:1300).
   `receiver_is_off_zulip` (event_queue.py:1024-1034) is true when the
   user has no active, non-offline message-receiving event queues.
5. Calls `maybe_enqueue_notifications(...)` (event_queue.py:1302-1311)
   and stores its return value (`push_notified`/`email_notified`
   markers) back into `internal_data`.

`maybe_enqueue_notifications` (event_queue.py:1037-1082) is **the
enqueue point** for both notification channels: push notices go to
`mobile_notifications_queue_name(user_id)` → `missedmessage_mobile_notifications`
(`zerver/lib/queue.py:469-473`; shards when
`MOBILE_NOTIFICATIONS_SHARDS > 1`), guarded by
`already_notified.push_notified` (event_queue.py:1062-1066); email
notices go to `missedmessage_emails` (event_queue.py:1072-1080),
guarded by `already_notified.email_notified`.

The same funnel serves two other call sites:

- **Message edits**: `maybe_enqueue_notifications_for_message_update`
  (event_queue.py:1615-1676; called at event_queue.py:1561-1568) —
  skips muted senders, DMs, prior mentions, and users already covered by
  stream/followed-topic settings, then re-evaluates idle and calls the
  same `maybe_enqueue_notifications`.
- **Queue GC / offline marking** — the "hard disconnect" backstop:
  `missedmessage_hook` (event_queue.py:884-1021) runs when a user's
  last event queue is garbage-collected or marked offline, replays every
  still-queued `message` event through the internal_data record
  (`idle=True` hardcoded, event_queue.py:1005-1006; `online_push_enabled`
  hardcoded False since the user is by then idle, event_queue.py:997),
  and respects the `push_notified`/`email_notified` markers recorded at
  delivery time (event_queue.py:1010-1013). Fork-specific commits
  `b004b73778` (offline field) and `0c9ebd4e08` (mark long-lived queues
  offline so mobile clients get notifications after
  `EVENT_QUEUE_OFFLINE_TIMEOUT_SECS`, event_queue.py:57, 606-631) touch
  this path; the known duplicate-notification caveat is documented at
  event_queue.py:587-594.

### Worker

`PushNotificationsWorker` (`zerver/worker/missedmessage_mobile_notifications.py:21-72`)
dispatches three event types on that queue: `add` (default →
`handle_push_notification`), `remove` (→
`handle_remove_push_notification`), and
`register_push_device_to_bouncer`; failures are retried via `retry_event`.

`handle_push_notification` (push_notifications.py:1717-1915) gates in
order:

1. `push_notifications_configured()` (push_notifications.py:1722;
   definition at 813-834).
2. User must have at least one of
   `enable_offline_push_notifications`/`enable_online_push_notifications`
   (push_notifications.py:1728-1733; marked as a "BUG: investigate" —
   normally unreachable).
3. Access check on the message inside a durable transaction
   (push_notifications.py:1757-1782), with races (deleted/moved message)
   silently dropped.
4. Skip if already read or already has
   `active_mobile_push_notification`; otherwise set that flag
   (push_notifications.py:1784-1793) — this is what later drives
   revocation (remove) events via
   `do_clear_mobile_push_notifications_for_ids`
   (`zerver/actions/message_flags.py:211-294`), enqueued on mark-as-read,
   message deletion, or mention-removing edits (message_flags.py:43,
   131, 175, 208).
5. Trigger compat renames, mentioned-user-group resolution,
   soft-reactivation for long-term-idle users (§1), sender-access check
   (push_notifications.py:1806-1855).

### Sending: two device flows, both live

`prepare_payload_and_send_push_notifications`
(push_notifications.py:1672-1714) sends over **both** flows:

- **Legacy flow** (unless `realm.require_e2ee_push_notifications`):
  `PushDeviceToken` rows → `send_push_notifications_legacy`
  (push_notifications.py:1392-1429) → direct APNs/FCM, or
  `send_notifications_to_bouncer` (push_notifications.py:600-694) to the
  Zulip services push bouncer (`push/notify`).
- **E2EE flow**: `Device` rows with `push_token_id`
  (`zerver/models/devices.py:12-70`; `push_key`, `push_key_id`,
  `push_token_kind`) → payload encrypted per device with
  `crypto_secretbox_easy` via `get_encrypted_data`
  (push_notifications.py:1502-1513) → `send_push_notifications`
  (push_notifications.py:1516-1669) → direct via zilencer
  (`send_e2ee_push_notifications`, push_notifications.py:1575-1581) when
  `ZILENCER_ENABLED`, otherwise the bouncer at `push/e2ee/notify`
  (push_notifications.py:1582-1588). Stale tokens reported by the
  bouncer are cleared (push_notifications.py:1616-1629).

Which mode is in effect: `uses_notification_bouncer()`
(push_notifications.py:592-593) and
`sends_notifications_directly()` (push_notifications.py:596-597).

## 4. Notification-state markers (how the "already pushed" state travels)

- At enqueue time, `maybe_enqueue_notifications` returns
  `{"push_notified": True}` / `{"email_notified": True}`
  (event_queue.py:1066, 1080), which `process_message_event` merges into
  the per-user `internal_data` of the message event (event_queue.py:1302-1311,
  1338-1340).
- That state is *not* for clients: the web app never reads
  `internal_data` (no reference in `web/src/` outside an unrelated
  `presence.clear_internal_data`), and the field is not in
  `zerver/openapi/zulip.yaml`. It is tornado-internal state that
  `missedmessage_hook` reads back from the queue contents at GC time
  (event_queue.py:917-921, 1010-1013) to avoid duplicate notifications.
- The durable DB marker is `UserMessage.flags.active_mobile_push_notification`,
  set at send (push_notifications.py:1792-1793) and cleared (plus a
  `remove` notice enqueued) by
  `do_clear_mobile_push_notifications_for_ids`
  (message_flags.py:211-294).
- Web/desktop notifications are client-side (flags spliced into message
  events; `docs/subsystems/notifications.md`), unrelated to this marker.
- Email has no equivalent durable flag; its duplicates risk at GC time
  is documented at event_queue.py:587-594 (email batches through
  `ScheduledMessageNotificationEmail` rows instead,
  `zerver/models/scheduled_jobs.py:85-117`).

## 5. Test coverage — how suppression behavior would be tested

| Layer | File | Pattern to reuse |
| --- | --- | --- |
| Decision unit | `zerver/tests/test_notification_data.py:9-` (`test_is_push_notifiable` 15, `test_is_email_notifiable` 181, `test_is_notifiable` 340, `test_push_notifiability_when_no_push_device_registered` 479) | `self.create_user_notifications_data_object(user_id=..., **flags)` helper (`zerver/lib/test_classes.py:2021-2061`) — pure, no DB |
| Enqueue logic | `zerver/tests/test_event_queue.py:42-93` (`MaybeEnqueueNotificationsTest`) | mock `zerver.tornado.event_queue.queue_json_publish_rollback_unsafe`, assert queue names + notice payloads; `get_maybe_enqueue_notifications_parameters` fixture (test_classes.py:2063) |
| GC hook | `zerver/tests/test_event_queue.py:137-` (`MissedMessageHookTest`) | build a client descriptor with a queued message event, call `missedmessage_hook`, assert enqueue args |
| Worker routing | `zerver/tests/test_queue_worker.py:492-570` | `FakeClient` + `simulated_queue_client`, mock `handle_push_notification` |
| `handle_push_notification` | `zerver/tests/test_handle_push_notification.py` (`HandlePushNotificationTest` 38; end-to-end 58, read-message skip 361, soft-deactivated 911, already-active 1128) | `PushNotificationTestCase` (`zerver/lib/test_classes.py:2895-`): `get_message(...)`, `UserMessage.objects.create`, `missed_message = {"message_id": ..., "trigger": ...}` dict, `mock_fcm()`/`mock_apns()` or `responses` bouncer mock; `self.soft_deactivate_user` |
| Payload building | `zerver/tests/test_push_notifications.py` (`TestGetAPNsPayload` 1549, `TestGetGCMPayload` 1881, bouncer suites 337, 2092) | payload assertions per trigger type |
| Full integration | `zerver/tests/test_message_flags.py:904-1052` (`PushNotificationMarkReadFlowsTest`) | `self.register_push_device(user_id)` fixture (test_classes.py:2004-2011 creates a `Device` with FCM push_key; `register_push_device_token` at 2013 for legacy `PushDeviceToken`), `do_change_user_setting`, real `send_stream_message`, then assert via `get_mobile_push_notification_ids` (UserMessage rows with `active_mobile_push_notification`, test_message_flags.py:905-914) |
| Message-edit notifications | `zerver/tests/test_message_edit_notifications.py` | edit a message to add/remove a mention, assert queue behavior |

A "do not disturb" suppression would most naturally get: (1) unit tests
in `test_notification_data.py` asserting the trigger is `None` when the
flag is on; (2) integration tests in `test_message_flags.py` /
`test_message_send.py` style — register a push device, enable push
settings, enable DND, send a message, assert no
`missedmessage_mobile_notifications` publish and no
`active_mobile_push_notification` row.

## 6. Presence/status today, and email divergence

**Verified: nothing in this path consults `UserProfile` status text or
`UserStatus`.** Grep for `UserStatus`/`status_text` across
`zerver/actions/message_send.py`,
`zerver/lib/notification_data.py`, `zerver/lib/push_notifications.py`,
`zerver/tornado/event_queue.py`, `zerver/lib/email_notifications.py`,
and both workers returns no matches. The only "status" input is
`UserPresence.last_active_time` vs `OFFLINE_THRESHOLD_SECS`
(message_send.py:960-977) plus live client-connectedness via
`receiver_is_off_zulip` (event_queue.py:1024-1034). The `UserStatus`
model (status text/emoji, `zerver/models/presence.py:103-114`) is
display-only metadata today; a "busy"/DND-via-status design would be
new plumbing, not reuse of an existing hook.

**Email vs push divergence:**

- Shared: the same `UserMessageNotificationsData` object and the
  `is_notifiable` pre-filter (notification_data.py:222-225);
  `trivially_should_not_notify` gates both channels; the same
  `maybe_enqueue_notifications` enqueue point (email branch at
  event_queue.py:1072-1080); the same GC/edit call sites.
- Different: `get_email_notification_trigger` hard-requires
  `idle=True` (no online-email analog of `online_push_enabled`,
  notification_data.py:263-264); the `push_device_registered`
  multiplier is push-only (notification_data.py:141-179) — email
  notification flags do not require a registered device; the email
  worker batches into `ScheduledMessageNotificationEmail` rows per
  `email_notifications_batching_period_seconds`
  (`zerver/worker/missedmessage_emails.py:48-109`) and re-checks
  `enable_offline_email_notifications` at send
  (`zerver/lib/email_notifications.py:691-693`), while the push worker
  re-checks both push settings immediately
  (push_notifications.py:1728-1733) and sends within seconds.

Consequence for a push-only suppression flag: if it is checked inside
`is_notifiable` (notification_data.py:222-225) it will also suppress
email; to keep email unaffected, the check must live in
`is_push_notifiable`/`get_push_notification_trigger`, or be a separate
flag consulted alongside them.

## Candidate hook points

### (a) Inside the decision funnel: `get_push_notification_trigger` / `UserMessageNotificationsData`

Mechanics: add a `push_suppressed`-style boolean to the dataclass, set
from a new per-user ID set plumbed exactly like
`push_device_registered_user_ids` — `RecipientInfoResult` +
`ActiveUserDict` (message_send.py:264-272, 607) → `SendMessageRequest`
(message_send.py:836-873) → message event dict (message_send.py:1359)
→ `from_user_id_sets` (notification_data.py:74) → checked at the top of
`get_push_notification_trigger` (e.g. next to
notification_data.py:231).

- **Correctness**: strongest. Covers all three Tornado decision sites
  (new message event_queue.py:1302, message edit event_queue.py:1669,
  GC hook event_queue.py:1014) with one check, because they all funnel
  through the same method. The state is snapshotted with the message
  event, so behavior is reproducible from `internal_data` at GC time.
- **Event consistency**: the suppressed decision is recorded in
  `internal_data` like any other notifiability data, so the
  `push_notified` marker stays `False` and the GC hook will re-apply
  current suppression state 10 minutes later (a user who re-enables
  notifications in that window gets the missed push — arguably correct,
  but must be a documented stance).
- **Testability**: unit-testable with zero DB (`test_notification_data.py`
  fixture); integration-testable end to end via the
  `test_message_flags.py` pattern.
- **Costs**: touches the event schema (needs the TODO/compat `None`
  shim pattern like notification_data.py:136-139 for old queued
  events); the per-user set must be computed synchronously in the send
  path (one cheap indexed query, same as `has_push_device_registered`);
  message-edit path needs the flag carried in edit events too.
- **Already-enqueued pushes**: unaffected (none exist yet at decision
  time); push notices enqueued before the user enabled DND will still
  send unless a worker-side check also exists.

### (b) At send time in `message_send` (e.g. excluding users in `get_recipient_info` / skipping them in the eligibility sets)

- **Correctness**: weak-to-moderate. Tornado re-derives the decision
  from the event's ID lists, so omitting a user from e.g.
  `stream_push_user_ids` works for the initial message, but the
  semantic gets lost: `internal_data` would just show flags off, and
  the GC hook replays the same snapshot with `idle=True`, so an
  "eligible but suppressed" state is indistinguishable from "settings
  off" — the user's later toggle-back is invisible to the replay.
  `is_notifiable(idle=True)` filtering (message_send.py:1002) also
  keys off the *unsuppressed* eligibility, so presence computation
  would need care.
- **Event consistency**: same as (a) — snapshot-based — but with
  information loss (no explicit "suppressed" bit).
- **Testability**: `test_message_send.py`-style asserts on
  `RecipientInfoResult` contents.
- **Costs**: hot path (every message send) gains a per-user state
  query; and any suppression logic here duplicates what the decision
  funnel would do anyway.
- **Already-enqueued pushes**: none.

### (c) At worker consume time (`handle_push_notification`)

- **Correctness**: decision made with the freshest DB state, and it
  retroactively suppresses pushes already sitting in the queue — the
  only hook that can do that. Precedent: the worker already re-checks
  user-level settings (push_notifications.py:1728-1733) and read-state
  (push_notifications.py:1784-1787); `docs/subsystems/notifications.md`
  explicitly documents queue processors as the place for final
  re-checks.
- **Event consistency**: weakest. The `push_notified` marker was set at
  enqueue time regardless (event_queue.py:1066), so after a worker-side
  suppression the GC hook will consider the user already notified and
  never re-try — even if DND was toggled off before GC. A message
  whose push was suppressed at the worker is marked
  "notification sent" forever in tornado's view (no DB marker is set
  though, so no remove-flow side effects; the message just never gets
  a push unless the user later reads/unreads or edits re-trigger —
  edits won't re-trigger for the same message).
- **Testability**: `test_handle_push_notification.py` fixture is a
  direct fit (call `handle_push_notification(user_id, missed_message)`
  with the flag on, assert no FCM/APNs send mocks called).
- **Costs**: does not cover the missedmessage_emails path (email would
  still notify unless a parallel hook goes in
  `MissedMessageWorker.consume`); leaves notices traveling through the
  queue that are dropped at consume; small extra latency for every
  suppressed push (queue round-trip).
- **Already-enqueued pushes**: fully covered (that's the point).

### Recommendation

Implement the primary check as (a): a per-user suppression set plumbed
through `from_user_id_sets` and consulted at the top of
`get_push_notification_trigger` (push-only, *not* inside
`is_notifiable`, to keep email unaffected per §6). That covers new
messages, edits, and GC replay with one code path, keeps tornado's
`internal_data` bookkeeping truthful, and unit-tests with the existing
no-DB fixture. Optionally add a thin (c)-style recheck in
`handle_push_notification` for defense-in-depth and to suppress
already-enqueued notices when the flag is flipped mid-flight — this
mirrors the existing user-settings recheck at
push_notifications.py:1728-1733 and is cheap to test with the
`test_handle_push_notification.py` fixtures.

## Open questions

1. **Does suppression extend to email?** The two channels share
   `is_notifiable` and `trivially_should_not_notify`
   (notification_data.py:206-225); a push-only flag must avoid the
   shared funnel or the email branch of
   `maybe_enqueue_notifications` (event_queue.py:1072-1080) will
   silently suppress too (or vice versa, if the product wants both).
2. **Timing semantics for already-enqueued notices.** A user enabling
   DND has a window where notices are already queued (push: seconds;
   email: up to the batching period, worker at
   missedmessage_emails.py:34, 55-64). Does DND mean "no future
   enqueues", "drop queued notices too", or "suppression evaluated
   once at enqueue and never revisited at GC replay"? (a) vs (c) pick
   different answers.
3. **GC-replay stance.** With (a), the GC hook re-evaluates the flag
   from a snapshot; if the snapshot records "suppressed at send time",
   a user who re-enables within the ~10-minute window loses the push
   unless the hook re-queries the user instead of trusting
   `internal_data`. Decide and document; the existing duplicate-risk
   comment (event_queue.py:587-594) suggests upstream is already
   careful here.
4. **Message-edit path.** `maybe_enqueue_notifications_for_message_update`
   (event_queue.py:1615-1676) builds fresh notification data from edit
   events; the suppression flag must ride in edit events as well, or a
   mention added by an edit would bypass suppression.
5. **Fork-specific offline marking.** `mark_clients_offline`
   (event_queue.py:606-631, fork commit 0c9ebd4e08) fires
   `missedmessage_hook` for long-lived mobile queues; any suppression
   logic must behave identically on that path (it currently flows
   through the same hook, so (a) covers it; (c) would too, via the
   queue).
6. **Where does user-level state live?** A DB column on `UserProfile`
   is simplest for the set-based plumbing in (a) and the worker recheck
   in (c); a `UserStatus`-driven design would additionally need a
   status→suppression mapping and a decision about whether
   `filter_presence_idle_user_ids` (message_send.py:960) or
   `receiver_is_off_zulip` (event_queue.py:1024) should treat
   "status says busy" as idle — both are unlinked from `UserStatus`
   today (verified §6).
7. **Uniqueness/ownership of the `push_notified` marker.** If suppression
   is worker-side (c), should `handle_push_notification` inform tornado
   somehow (it cannot — different process, no event) or should the
   marker only be set when a push actually sends? The latter would be
   an upstream-visible behavior change to `maybe_enqueue_notifications`.
