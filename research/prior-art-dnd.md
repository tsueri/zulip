# Research: Prior art — upstream DND and status-steered notifications

Ticket: https://github.com/tsueri/zulip/issues/2
Date: 2026-10-09
Branch: `research/prior-art-dnd` (based on ebf693e654, "events: Refactor RealmUpdateDictEvent to use a Realm representation")

## Summary

Upstream has a decade-long trail of "do not disturb" / "quiet hours" feature
requests, but **no server-side DND or scheduled-notification suppression has
ever been implemented in the web app or server**. The only shipped DND is a
client-side toggle in the Electron desktop app (silences desktop
notifications only). Server-side, user status (`status_text` + emoji) and
"invisible mode" (`presence_enabled=false`) are purely display/presence
features; nothing in the server notification pipeline reads them. The
closest architectural prior art is the "temporary mutes / snoozing topics"
discussion (CZO #frontend, 2020), where Tim Abbott rejected a
scheduled-worker approach in favor of an incremental design: first record
*when* a mute happened, then design expiry separately. That first step
landed (`UserTopic.last_updated`, with the exact backfill value Steve
Howell proposed), but the expiry/"muted until" step has never been built.
The flagship open ticket is #29098 "Make it possible to schedule
do-not-disturb times" (alya, 2024, explicitly "no product spec yet"), which
grew out of a CZO thread where alya's central open design question was
whether notification suppression should be **tied to the user's status or
be an independent scheduled setting**.

## Feature requests (with links + status)

| Issue | Title | State | Notes |
|---|---|---|---|
| [zulip/zulip#29098](https://github.com/zulip/zulip/issues/29098) | Make it possible to schedule do-not-disturb times | OPEN (2024-02-27) | Filed by alya from [CZO thread](https://chat.zulip.org/#narrow/stream/138-user-questions/topic/Automatic.20or.20Scheduled.20Do.20not.20Disturb/near/1718803). Labels: `feedback wanted`, `area: notifications (messages)`, `new feature`. "This feature will require a specific product design to be ready to be worked on." Zero comments to date. |
| [zulip/zulip#6629](https://github.com/zulip/zulip/issues/6629) | Need a way to temporarily suppress notifications for all channels | OPEN (2017) | Umbrella ticket, timabbott: "definitely something I'd like us to work on soon. Tagging this as a priority." Kept open to track the overall experience; desktop DND covers only part. |
| [zulip/zulip#9265](https://github.com/zulip/zulip/issues/9265) | disable notifications based on time-schedule | CLOSED 2018 | Closed by alya as duplicate of #6629. Documents desktop-app DND shipping and the cross-app inconsistency problem. |
| [zulip/zulip#17008](https://github.com/zulip/zulip/issues/17008) | Feature Request: Timed Mutes | OPEN (2021) | Topic-level timed mutes (mute for 30 min / 2 hours). timabbott: priority; alya: "good follow-up to #25673" (follow topic). Includes user request for a global "mute for 30 minutes" for meetings. |
| [zulip/zulip#6139](https://github.com/zulip/zulip/issues/6139) | Allow snoozing of topics (aka expired muting) | OPEN (2016) | Renamed from "expired muting" to "snoozing" by showell/timapril; showell: needs UI plan before coding. |
| [zulip/zulip#526](https://github.com/zulip/zulip/issues/526) | Temporary mute of stream/topic (super old) | CLOSED | Closed by timabbott as duplicate of #17008 ("super old and messy"). |
| [zulip/zulip#4372](https://github.com/zulip/zulip/issues/4372) | Add option to set oneself to "Busy" or "Do not disturb" | CLOSED 2018 | timabbott: "We have had for 1.5 years an 'unavailable' status that covers this." Closed as fixed by the legacy away/unavailable status — not a notification feature. |
| [zulip/zulip#295](https://github.com/zulip/zulip/issues/295) | Configure Status | CLOSED 2019 | Closed as resolved by rishig pointing at the user-status/availability feature (https://zulipchat.com/help/status-and-availability). |
| [zulip/zulip#25398](https://github.com/zulip/zulip/issues/25398) | Set availability and status message at same time | OPEN | UX gap: availability (presence_enabled) and status_text are set in two steps today. |
| [zulip/zulip#21310](https://github.com/zulip/zulip/issues/21310) | Possible to mute Zulip audio (all channels) temporarily? | OPEN | alya: "I wonder if we can come up with a solution that's more convenient than the browser/desktop app options." |
| [zulip/zulip#24169](https://github.com/zulip/zulip/issues/24169) | Document Do Not Disturb feature in Desktop app | CLOSED | Documentation-only; PR [#24203](https://github.com/zulip/zulip/pull/24203) merged. Confirms DND is desktop-app-only. |
| [zulip/zulip-mobile#3256](https://github.com/zulip/zulip-mobile/issues/3256) | Feature request: delay notifications until business hours | OPEN | Referenced from #9265; mobile-side gap remains unaddressed. |

No upstream PR implementing server/web DND or scheduled notification
suppression exists (searches for "do not disturb", "snooze", "quiet hours",
"mute until", "notification schedule" return nothing beyond the above).

## Maintainer positions / rejected designs (quotes + links)

### CZO: "Automatic or Scheduled Do not Disturb" (#138-user-questions, 2024)
https://chat.zulip.org/#narrow/stream/138-user-questions/topic/Automatic.20or.20Scheduled.20Do.20not.20Disturb/near/1718803

- User request (Derfinsterling, [2024-01-12](https://chat.zulip.org/#narrow/stream/138-user-questions/topic/Automatic.20or.20Scheduled.20Do.20not.20Disturb/near/1718803)): notifications muted automatically based on an "availability status", possibly scheduled ("no notifications between 9pm and 7am", out sick, on vacation).
- alya ([2024-01-15](https://chat.zulip.org/#narrow/stream/138-user-questions/topic/Automatic.20or.20Scheduled.20Do.20not.20Disturb/near/1719887)): "This is not currently possible, though you can turn on 'Do Not Disturb' manually in the Desktop app."
- alya ([2024-01-15](https://chat.zulip.org/#narrow/stream/138-user-questions/topic/Automatic.20or.20Scheduled.20Do.20not.20Disturb/near/1719888)) — **the central design question, still unanswered**: "Do you think your users would prefer to have it tied to their status, or just have a separate ability to schedule when notifications are on/off?"
- Requester Ben O: "a separate ability would be ideal, but any implementation would be great."
- alya filed [#29098](https://github.com/zulip/zulip/issues/29098), noting it has no product spec yet.

### CZO: "Temporary mutes design" (#6-frontend, 2020-01-14)
https://chat.zulip.org/#narrow/stream/6-frontend/topic/Temporary.20mutes.20design

This thread contains the most concrete maintainer design thinking on
time-bounded notification suppression:

- **Steve Howell on architecture** ([msgid 808998](https://chat.zulip.org/#narrow/stream/6-frontend/topic/Temporary.20mutes.20design/near/808998)): "we would want some cron like thing to expire the mutes, and then to the rest of the codebase it would be just as if a user had manually unmuted it." I.e. expiry should *reuse the existing unmute event path*, not add a parallel suppression path.
- **Tim Abbott's incremental strategy** ([msgid 809020](https://chat.zulip.org/#narrow/stream/6-frontend/topic/Temporary.20mutes.20design/near/809020)): "I think a good first step on this front would be to extend the `MutedTopic` and `Stream` models to record when the muting happened in a `date_muted` nullable DateTime field... It's a strict subset of the work for a 'temporary mute' feature that would help us get experience with making changes to that system while avoiding the difficult problem of figuring out how to actually do that."
- **Steve Howell's schema sketch** ([msgid 809140](https://chat.zulip.org/#narrow/stream/6-frontend/topic/Temporary.20mutes.20design/near/809140)): prefer two columns — `date_muted` (always set) + `expire_delta` (nullable timedelta, e.g. 1 hour), with any cron jobs "comput[ing] the expirations on the fly"; he preferred non-nullable with a hard-coded backfill of 2020-01-01.
- **Tim Abbott scoping topic mutes only** ([msgid 811083](https://chat.zulip.org/#narrow/stream/6-frontend/topic/Temporary.20mutes.20design/near/811083)): "It'll be totally different code for the two features... So I think we should ignore `Subscription` and proceed with the `date_muted` changeset" (stream mutes live in `Subscription`).
- **Rejected/abandoned implementation**: [PR #13559](https://github.com/zulip/zulip/pull/13559) (Ryan Rehman, 2020, "[WIP] Add temporary mute functionality", closed unmerged) implemented timed topic mutes via a nullable scheduled-unmute timestamp on `MutedTopic`, a custom django-admin command popped off `run-dev.py`, and *deleting the row* on expiry. Tim Abbott redirected this work to preparatory-only PRs ([msgid 809149](https://chat.zulip.org/#narrow/stream/6-frontend/topic/Temporary.20mutes.20design/near/809149)): "our plan will be to merge that preparatory feature and then design how 'temporary muting' will work in full."
- Follow-up prep work landed as PR #13702 (record when a topic got muted); today that state lives in `UserTopic.last_updated` with the exact 2020-01-01 backfill default Steve Howell proposed. **The actual expiry feature was never designed or built.**

### zulip/zulip#6629 — direction endorsed for DND: web-app setting, desktop follows

- akashnimare ([comment](https://github.com/zulip/zulip/issues/6629#issuecomment-388191170), 2018): desktop DND "would be one way only i.e. other users won't be able to see that you are on DND mode. Ideally, we should have this functionality in the web app itself since we have more control over notifications, user status etc there. User should have an option to choose their status from — Do not Disturb / Busy / Invisible / Custom status."
- timabbott ([comment](https://github.com/zulip/zulip/issues/6629#issuecomment-388210268)): "Yeah, we should plan to add the setting in the webapp and the desktop app can then implement the correct behavior based on the setting." — i.e. DND belongs server-side/web-side, not client-side.
- Later comments (pspacek, kkmuffme) reinforce why desktop-only DND fails: no mobile coverage, no browser coverage, and DND is invisible to other users ("missed an urgent issue and got an angry message why I'm not responding").
- alya ([2023](https://github.com/zulip/zulip/issues/6629#issuecomment-1447513170)-ish exchange in thread): agrees scheduling DND "would be helpful."

### zulip/zulip#9265 — closed as duplicate; maintainer notes

- akashnimare: desktop DND is per-app, "so if you set it then it will only work in the desktop app."
- mlncn's framing worth carrying into any design: off-hours suppression should be *visible to teammates* ("@mention... will receive notification to look at the message **when their working hours begin again**"), so mentioners know not to expect an earlier response.
- timabbott pointed out mobile OSes already ship a global DND (partial mitigation).

### zulip/zulip#21178 — "unavailable" status replaced by invisible mode (closed 2022)

The legacy "unavailable"/away UI status was deliberately removed in favor of
"Invisible mode", defined as *the same functionality as the
"Display my availability to other users when online" setting*
(`presence_enabled=false`). asah: "replace the 'Set yourself as
unavailable' in the personal profile menu with 'Turn on/off invisible mode'
- same functionality as turning off the 'Display my availability to other
users when online' setting." alya approved. This is the terminal state of
the away/unavailable status line; DND is not part of it.

## Overlapping implemented features (upstream, shipped)

1. **Desktop app DND (client-side only)** — toggled in the desktop app; disables desktop notifications across all added organizations; explicitly does *not* affect mobile push, email, or web-app desktop notifications. Documented: [zulip.com/help/do-not-disturb](https://zulip.com/help/do-not-disturb) (source: `starlight_help/src/content/docs/do-not-disturb.mdx`).
2. **User status** (`status_text` + emoji) — display-only; shipped via #295/#15682 line of work; help center: [status-and-availability](https://zulip.com/help/status-and-availability). Never read by notification code.
3. **Invisible mode / presence_enabled=false** — hides presence/availability display; the successor of the deprecated "away"/"unavailable" status. No effect on notifications.
4. **Per-channel muting** — `Subscription.is_muted` (no time bound).
5. **Per-topic muting + Follow topic** — `UserTopic` visibility policies (MUTED / UNMUTED / FOLLOWED; #25673, #6027-related); no expiry/"muted until".
6. **Muted users** — `MutedUser.date_muted`, no expiry.
7. **Message reminders** (#21810, closed/implemented) — "remind me about this message later", i.e. per-message snooze, adjacent but not notification suppression.
8. **Web-app guardrail** — when a user disables all notifications, the web app warns and points at desktop DND (`web/templates/confirm_dialog/confirm_disable_all_notifications.hbs`).

## Local-code findings (file:line, at ebf693e654)

Verified on this checkout:

- **UserStatus is display-only.** `zerver/models/presence.py:103-114` — `class UserStatus(AbstractEmoji)` carries `user_profile`, `timestamp`, `client`, `emoji_name`, `emoji_code`, `status_text` only. No availability/notification field.
- **"away" is a deprecated alias for `presence_enabled=false`.** `zerver/lib/user_status.py:32-37`:
  > "Deprecated way for clients to access the user's `presence_enabled` setting, with away != presence_enabled. Can be removed when clients migrate 'away' (also referred to as 'unavailable') feature to directly use and update the user's presence_enabled setting."

  `away = not presence_enabled` (line 37); `check_update_user_status(..., away=...)` at `zerver/lib/user_status.py:149-152`.
- **`presence_enabled`** is a plain user setting: `zerver/models/users.py:242` (`models.BooleanField(default=True)`); exposed in the UI as "invisible mode off" (`web/src/settings_config.ts:757`).
- **Status does not steer notifications.** `rg "user_status|UserStatus|status_text"` over `zerver/lib/notification_data.py`, `zerver/lib/email_notifications.py` (the push/email notification pipelines) returns nothing. Status is consumed only for display.
- **No "muted until" concept exists anywhere.** `zerver/models/user_topics.py:14-22` (`UserTopic.last_updated`, backfill default `datetime(2020, 1, 1, ...)` — exactly the value Steve Howell proposed in CZO 809140), `zerver/models/streams.py:395` (`Subscription.is_muted`), `zerver/models/muted_users.py:14` (`MutedUser.date_muted`). None has an expiry field.
- **Only "Do not disturb" string in the web/server tree**: `web/templates/confirm_dialog/confirm_disable_all_notifications.hbs:7-8`, linking `/help/do-not-disturb` (the desktop-app feature).

Local grep for `do.?not.?disturb|\bdnd\b|snooze|quiet hours|muted until`
across py/ts/tsx/hbs/md finds no server-side or web-app DND implementation —
the codebase has no partial pieces waiting to be wired up.

## Open questions the design tickets must answer

1. **Status-tied or independent?** alya's unanswered question from CZO
   1719888: should notification suppression be driven by the user's status
   (so it's visible to teammates, per mlncn's #9265 framing), or be a
   separate scheduled setting? Upstream users asked for the separate
   ability; a DND *status* is what #4372-style requests and the desktop app
   imply. A hybrid (status that also steers notifications server-side) is
   the novel part any design must justify against upstream's leaning.
2. **Server-side enforcement scope.** Desktop-only DND is already shipped
   and known-insufficient (#6629). Server-side suppression must cover mobile
   push and email pipelines (`notification_data.py` / `email_notifications.py`),
   which nothing status-related touches today.
3. **Expiry mechanics.** CZO 808998/809140: expiry should behave "just as if
   a user had manually unmuted" and reuse existing event paths; schema
   preference was `date_muted` + nullable `expire_delta` (or equivalent
   `expires_at`) computed on the fly rather than a row-deleting worker (PR
   #13559's abandoned approach). Any DND/snooze design should decide where
   state lives (new field on `UserProfile`? new table? reuse
   `UserStatus`?), how the events system broadcasts it, and how it expires.
4. **Mention/teammate semantics.** If DND suppresses mentions, should
   teammates be able to see the DND state / "notify anyway" (Slack-style,
   requested in #6629), or deferred-delivery on working-hours start (mlncn,
   #9265)?
5. **Interaction with existing precedence.** How does a global DND
   interleave with per-channel/per-topic/user muting and follow-topic
   ("notify anyway") priorities — and with the `confirm_disable_all_notifications`
   guardrail?
6. **Timezones/scheduling.** Scheduled quiet hours need user-timezone
   handling (Slack "notify anyway" + local-time display comparisons in #6629).
7. **Relationship to #29098.** Upstream's open flagship ticket has no
   product spec and is tagged `feedback wanted` — a fork design here may be
   able to feed upstream, but should decide whether it targets
   compatibility with upstream's likely shape (a notification-scheduling
   setting) rather than a status-driven one.
