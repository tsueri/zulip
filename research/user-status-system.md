# Research: User status system end-to-end fact sheet

Ticket: https://github.com/tsueri/zulip/issues/3
Date: 2026-10-09
Base commit: `ebf693e6547` (research/user-status-system branch). All `file:line` references are relative to this repo checkout.

Purpose: give the design grilling a verified, source-backed picture of how user
statuses work today (data model, API, events, web frontend, rendering, docs,
notifications), so proposals about "default status messages" and
"status affecting notifications" start from facts.

## 1. Data model

- `UserStatus` is `zerver/models/presence.py:103` — a one-to-one sidecar on
  `UserProfile` (`presence.py:104`). Fields:
  - `status_text` — `CharField(max_length=255, default="")` (`presence.py:114`).
    Note the mismatch: the DB allows 255, but the API enforces 60 (see §2).
  - `emoji_name` / `emoji_code` — `TextField(default="")`, overriding the
    inherited fields to allow empty defaults (`presence.py:109-112`).
  - `reaction_type` — inherited from `AbstractEmoji`
    (`zerver/models/messages.py:349`), `CharField(default=UNICODE_EMOJI,
    choices=REACTION_TYPES)` at `messages.py:368`; choices are `unicode_emoji`,
    `realm_emoji`, `zulip_extra_emoji` (`messages.py:363-367`). So a status
    emoji can be a realm custom emoji.
  - `timestamp` — `DateTimeField()` (`presence.py:106`) and `client` —
    `ForeignKey(Client)` (`presence.py:107`) record who/when set the status.
    They are write-only bookkeeping: no API surface exposes them (§2, §3).
  - There is **no** "away"/"unavailable" column today. Migration
    `zerver/migrations/0413_set_presence_enabled_false_for_user_status_away.py`
    (comment at line 10: "Set presence_enabled to False for all users with a
    UserStatus.status of AWAY") and `0414_remove_userstatus_status.py` show the
    old `UserStatus.status` AWAY column was folded into
    `UserProfile.presence_enabled` and then dropped.
- **"Away" is not stored on UserStatus at all.** It is a derived view of
  `UserProfile.presence_enabled` (default `True`, `zerver/models/users.py:242`):
  - `zerver/lib/user_status.py:31-53` `format_user_status` computes
    `away = not presence_enabled` (`user_status.py:36-37`) and merges it into
    the status dict returned to clients. The comment there (32-35) marks this
    as "Deprecated way for clients to access the user's `presence_enabled`
    setting, with away != presence_enabled", to be removed once clients
    migrate the "away"/"unavailable" feature to the presence setting directly.
  - `format_user_status` drops empty values: `away` is only included when
    true; `status_text`/`emoji_name`+`emoji_code`+`reaction_type` only when
    non-empty (`user_status.py:43-53`). An empty dict means "no status".
- Presence (active/idle/offline) is a **separate subsystem**
  (`UserPresence`, `presence.py:11-84`), driven by `presence_enabled`
  (updates stop entirely when disabled — `zerver/actions/user_settings.py:603-625`,
  `zerver/actions/presence.py:297`). Statuses and presence only meet in the
  derived `away` flag above and in the register `user_status` dict.

## 2. API

Routes (`zproject/urls.py`):
- `POST /users/me/status` → `update_user_status_backend` (`urls.py:492`).
- `POST/GET /users/{user_id}/status` → `update_user_status_admin` /
  `get_status_backend` (`urls.py:493`).
- OpenAPI: `operationId: update-status` ("Update your status") at
  `zerver/openapi/zulip.yaml:12848` (path `/users/me/status` at 12846);
  admin variant `update-status-for-user` ("Update user status") at
  `zulip.yaml:11601`; read-only `get-user-status` at `zulip.yaml:11750`.

View logic (`zerver/views/presence.py`):
- `update_user_status_backend` (`presence.py:101-139`): `@human_users_only`,
  params `away: Json[bool]`, `emoji_name`, `emoji_code`,
  `status_text` (stripped, max 60 — `StringConstraints(strip_whitespace=True,
  max_length=60)` at `presence.py:114-116`), and `reaction_type` aliased to
  the param name `emoji_type` (`presence.py:110-113`). Calls
  `check_update_user_status` then `do_update_user_status`.
- `update_user_status_admin` (`presence.py:142-177`): same but no `away`
  param and targets another user via `access_user_by_id(..., for_admin=True)`.
- `get_status_backend` (`presence.py:94-98`) returns
  `get_user_status(target_user)` — the single-user version of the dict below.

Validation (`zerver/lib/user_status.py:149-200` `check_update_user_status`):
- Errors if `away`, `status_text`, and `emoji_name` are all None
  ("Client did not pass any new values.", `user_status.py:161-162`).
- `emoji_name == ""` clears emoji: resets `emoji_code` and sets
  `reaction_type` to `UNICODE_EMOJI` (`user_status.py:164-168`).
- With a non-empty `emoji_name` and no code/type given, the server resolves
  the code from the name via `get_emoji_data` (`user_status.py:170-181`,
  `zerver/lib/emoji.py:70`); passing `emoji_code`/`reaction_type` without
  `emoji_name` is an error (`user_status.py:183-186`).
- Any concrete emoji is validated with `check_emoji_request`
  (`user_status.py:190-193`, `zerver/lib/emoji.py:90`).
- Returns a `NewUserStatus` dataclass (`user_status.py:141-146`).

Write path (`zerver/actions/user_status.py:23-71` `do_update_user_status`):
- If `away` is not None, it **flips `presence_enabled`** via
  `do_change_user_setting(..., "presence_enabled", not away)`
  (`user_status.py:37-40`) — i.e. "away" through this endpoint is just
  invisible mode.
- Persists via `update_user_status` (`zerver/lib/user_status.py:89-119`,
  `update_or_create` at 116-119): only passed fields are overwritten;
  `timestamp` and `client_id` are refreshed on every write.
- Emits the event with **only the fields the caller explicitly passed**
  (`user_status.py:53-68`), so unset fields are absent from the event, not
  null (`UserStatusEventDict` at `user_status.py:14-20`).

## 3. Events

- Typed schema: `UserStatusCoreEvent` (`type: "user_status"`, `user_id`) and
  `UserStatusEvent` (`zerver/lib/event_types.py:1253-1264`) with optional
  `away`, `status_text`, `emoji_name`, `emoji_code`, `reaction_type`. No
  `timestamp`/`client` fields go to clients. Runtime checker:
  `zerver/lib/event_schema.py:268,710-711`.
- Recipients: `do_update_user_status` sends via
  `send_event_on_commit(realm, event, get_user_ids_who_can_access_user(user_profile))`
  (`zerver/actions/user_status.py:69-71`).
  `get_user_ids_who_can_access_user` (`zerver/lib/users.py:838-853`): if the
  user isn't access-restricted (`user_access_restricted_in_realm`,
  `users.py:711-726` — i.e. the realm's `can_access_all_users_group` is
  Everyone), all active users in the realm; otherwise active non-guest users
  of the realm plus users sharing any stream/DM-group subscription with the
  target, plus the target. API docs record this: "Event sent to all users who
  can access the modified user when the status of a user changes. **Changes**:
  Prior to Zulip 8.0 (feature level 228), this event was sent to all users in
  the organization." (`zulip.yaml:2200-2205`).
- Initial state: `fetch_initial_state_data` populates
  `state["user_status"] = get_all_users_status_dict(...)` when `user_status`
  is in `fetch_event_types` (`zerver/lib/events.py:936-942`).
  `get_all_users_status_dict` (`zerver/lib/user_status.py:56-86`) returns
  `{user_id: {status_text, emoji_name, emoji_code, reaction_type, away}}` for
  active users, excluding rows where the user has `presence_enabled=True` and
  no status set (`user_status.py:60-66`), and access-filtered by the same
  group logic (`user_status.py:68-70`). Register response docs:
  `zulip.yaml:19811-19830`.
- Event application to that state: `zerver/lib/events.py:1957-2002` — for each
  field present in the event, `""`/reset values pop keys; an empty dict for a
  user is removed entirely (`events.py:1999-2000`).
- Web client consumption: `case "user_status"` in
  `web/src/server_events_dispatch.js:1276-1295` — `status_text` →
  `user_status.set_status_text` + buddy-list redraw + compose placeholder
  update; `emoji_name` → `user_status.set_status_emoji` + redraws (user list,
  PM list, message feed status emoji). Client-side zod schema mirrors the
  server one (`web/src/user_status.ts:15-24` using
  `web/src/user_status_types.ts`); the web app requests `"user_status"` among
  its event types (`web/src/server_event_types.ts:48`) and parses the initial
  dict via `StateData` (`web/src/state_data.ts:694`).

## 4. Web frontend (own-status UI)

- `web/src/user_status.ts`:
  - `server_update_status` POSTs `/json/users/me/status` with
    `status_text`/`emoji_name`/`emoji_code`/`reaction_type`
    (`user_status.ts:29-50`). The web app never sends `away`.
  - Invisible mode is done directly against `/json/settings`
    `presence_enabled` (`server_invisible_mode_on/off`,
    `user_status.ts:52-68`) — the modern replacement for `away`.
  - Local caches `user_info` (text) and `user_status_emoji_info`
    (`user_status.ts:26-27`), getters `get_status_text` (70-72) /
    `get_status_emoji` (83-85), `initialize` seeds them from the register
    `user_status` dict (107-125).
- `web/src/user_status_ui.ts` — the "Set status" modal:
  - `default_status_messages_and_emoji_info` is a **hardcoded, client-side
    list**, not a server-fed list and not a settings constant. It is declared
    at `user_status_ui.ts:17` and built in `initialize()`
    (`user_status_ui.ts:209-239`) as seven entries — Busy, In a meeting,
    Commuting, Out sick, Vacationing, Working remotely, At the office — each a
    translated `$t(...)` string plus an emoji resolved client-side via
    `emoji.get_emoji_details_by_name` (from the emoji catalog
    `web/src/emoji.ts`), e.g. `working_on_it`, `calendar`, `bus`, `hurt`,
    `palm_tree`, `house`, `office`.
  - The modal renders this list via `web/templates/set_status_overlay.hbs`
    (options list `<ul class="user-status-options">` fed from
    `default_status_messages_and_emoji_info`, template lines 10-24). Clicking
    a preset fills the input and selects its emoji
    (`user_status_ui.ts:156-168`).
  - The modal is opened from the emoji picker / personal menus; the emoji
    sub-picker flow lives in `UserStatusSession` in `web/src/emoji_picker.ts:85-145`.

## 5. Rendering to others

All rendering reads the two client caches via `get_status_text` /
`get_status_emoji` (`web/src/user_status.ts:70-85`):

- Buddy list: status emoji + text (second line / hover title)
  `web/src/buddy_data.ts:230-231,248-249,324-332`; display depends on the
  user's own `user_list_style` setting (`buddy_data.ts:232-237`).
- User card popover: `web/src/user_card_popover.ts:325-326`.
- Personal menu (self): `web/src/popover_menus_data.ts:415-416` (also shows
  invisible mode derived from `user_settings.presence_enabled`, line 414).
- User pills in DM recipients: `web/src/user_pill.ts:58,96`.
- Compose box: placeholder/status hint `web/src/compose_ui.ts:478`; sender
  pills via `web/src/message_helper.ts:94`.
- PM list: `web/src/pm_list_data.ts:117`; recent view:
  `web/src/recent_view_ui.ts:871`; inbox: `web/src/inbox_ui.ts:485`; search
  pills: `web/src/search_pill.ts:445`; typeahead:
  `web/src/typeahead_helper.ts:112`.
- Availability dots come from the separate presence subsystem, not
  `UserStatus` (`buddy_data.ts:228` uses `get_user_circle_class`).

## 6. Docs and clients

- Help center: `starlight_help/src/content/docs/status-and-availability.mdx`.
  What users are told: a status is "a customizable emoji, along with a short
  message" shown "next to your name in the sidebars, message feed, and compose
  box" (`mdx:21-27`); set/clear via user card, personal settings, or mobile
  (`mdx:33-113`); shown on user cards and hover (`mdx:115-145`); display is
  user-configurable in the buddy list (compact vs. status + text,
  `mdx:147-166`). Availability (active/idle/offline) is described separately
  (`mdx:168-202`), and **invisible mode** is defined as never updating
  availability so you always appear offline (`mdx:204-246`); the doc never
  claims invisible mode mutes notifications.
- Mobile apps: separate repos (zulip/zulip-mobile, zulip/zulip-flutter); this
  repo only documents the shared API. The help doc assumes mobile supports set
  status / clear status / invisible toggle (`mdx:64-74,104-112,238-246`). The
  API contract web/mobile share is: register `user_status` dict
  (`zulip.yaml:19811-19830`), `user_status` events (`zulip.yaml:2200-2272`),
  and the deprecated `away` parameter/event field — both docs say it is
  "a legacy way to access the user's `presence_enabled` setting, with
  `away = !presence_enabled`" and "To be removed in a future release"
  (endpoint `away` param at `zulip.yaml:12878-12889`; event field at
  `zulip.yaml:2212-2226`).
  Whether the mobile apps have migrated off `away` is not verifiable from this
  repo (open question).

## 7. "Default status messages" configurability and notifications

- **Default status messages are not configurable and not server data.** The
  only source is the hardcoded list in `web/src/user_status_ui.ts:209-239`
  (built in the client's `initialize()` from translated strings + client emoji
  catalog lookups). A repo-wide search for `default_status` /
  `status_messages` finds no server-side equivalent, no realm setting, and no
  entry in the register response (`zerver/openapi/zulip.yaml` has no such
  field). Any server-fed or admin-configurable design would be net-new.
- **UserStatus has no effect on notifications today.** Verified:
  - `zerver/lib/notification_data.py` (the notification-builder) contains no
    reference to `UserStatus`, `user_status`, `status_text`, or even presence
    (checked `rg -i "user_status|UserStatus|status_text|presence"` — zero
    matches in that file).
  - `zerver/lib/push.py` likewise has no status references.
  - The legacy `pushable` field that mobile push decisions once used is
    hardcoded `False` and documented "This field was never used by clients of
    the legacy API" (`zerver/lib/presence.py:134-138`).
  - `presence_enabled`'s effects are confined to presence updates
    (`zerver/actions/user_settings.py:603-625` stops presence updates and
    backdates presence so the user shows offline;
    `zerver/actions/presence.py:297` skips update_user_presence). The `away`
    flag in the `user_status` dict/event is purely informational for clients.
  - Backend `UserStatus` is only touched by: the status API/actions, events
    (initial state + apply event), export/import, and tests (verified via
    `rg -l "UserStatus" zerver/ --glob '*.py'` excluding tests/migrations).

## Open questions

1. **Mobile behavior is unverified.** zulip-mobile / zulip-flutter live in
   separate repos; whether they (a) request `user_status` in
   `fetch_event_types`, (b) render status text vs. only emoji, and (c) still
   use the deprecated `away` parameter or have migrated to
   `presence_enabled` — all open, and relevant before changing event/param
   semantics.
2. **Is there upstream work on server-fed default status messages?** The web
   list is client-only; if a design grilling wants realm-configurable presets,
   we need to check upstream zulip/zulip issues + CZO for prior art
   (nothing in this repo).
3. **Status text length mismatch**: DB allows 255 chars
   (`presence.py:114`) but the API caps at 60 code points
   (`presence.py:114-116`; documented at `zulip.yaml:12875`). Any
   design touching status text should decide whether 60 stays the contract.
4. **`away` deprecation end-state**: docs promise removal "in a future
   release"; confirming no remaining client uses `away` (web does not) is a
   prerequisite for dropping the `away` field from events and
   `format_user_status`.
5. **`UserStatus.timestamp`/`client`** are stored but never surfaced; if a
   future design wants "status set via X at time Y", the data is already
   there but unexposed.
