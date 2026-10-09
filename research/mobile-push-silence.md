# Mobile push-silence implications — fact sheet

Research for [tsueri/zulip#11](https://github.com/tsueri/zulip/issues/11)
("Research: Mobile-app implications of server-side push silence"), 2026-10-09.
Base commit: `ebf693e6547fbe400d4992f336579ae2d4233883`.
Mobile sources checked: `zulip/zulip-mobile` `main` at
`d7cf6da79b4b9467e240b42f24330e97390a8aff`.

Purpose: determine whether the settled server-side DND semantics require
mobile notification-client work, what mobile users will see, and whether
notification removals, unread counts, badges, status, or presence create a
silence UX mismatch. This is source research, not a mobile implementation
proposal.

## 0. Summary

Server-side push suppression is UX-coherent without changing mobile's
notification *rendering* code: a notification UI item is produced from a push
message, so a message for which the server sends no push creates no new mobile
notification. The normal event stream remains distinct from push delivery, so
the message and its unread/mention state can still appear in the app. Android
`remove` pushes identify message IDs and only cancel a conversation notification
when its latest displayed message ID matches; a remove for an ID never pushed
therefore does not remove an unrelated notification. The server also builds
APNs remove payloads, but current iOS code does not handle them. DND does not
retract notifications already displayed before DND started. Mobile's current
status editor/model has no DND bit.

Thus, “no mobile changes needed” is defensible only for *enforcing push
silence*. If the DND spec requires mobile users to set or see the new status
attribute through the existing status picker, mobile UI/model changes are needed;
otherwise the spec must say mobile clients may silently honor DND without
showing its indicator. The spec should preserve normal unread/event state, make
clear that no deferred push is sent at DND end, and distinguish suppressing new
pushes from removing already displayed notifications.

## 1. Mobile notification pipeline

The mobile architecture doc distinguishes a **push message** (data delivered
to the device) from a **UI notification** (an item in the operating system's
notification area). On Android, one push can create or remove UI notifications;
on iOS, the operating system interprets each APNs message into one UI
notification, without app-side rendering code. Source: [mobile notification
architecture](https://github.com/zulip/zulip-mobile/blob/d7cf6da79b4b9467e240b42f24330e97390a8aff/docs/architecture/notifications.md).

### Android

`FcmListenerService.onMessageReceived` forwards incoming FCM data to
`NotificationUiManager.onReceived`. `FcmMessage.fromFcmData` dispatches
`event: message`, `event: remove`, and test events; the `RemoveFcmMessage`
parser accepts one ID (`zulip_message_id`) and/or a batch
(`zulip_message_ids`). Sources: [FcmListenerService.kt](https://github.com/zulip/zulip-mobile/blob/d7cf6da79b4b9467e240b42f24330e97390a8aff/android/app/src/main/java/com/zulipmobile/notifications/FcmListenerService.kt),
[FcmMessage.kt](https://github.com/zulip/zulip-mobile/blob/d7cf6da79b4b9467e240b42f24330e97390a8aff/android/app/src/main/java/com/zulipmobile/notifications/FcmMessage.kt),
and [FcmMessageTest.kt](https://github.com/zulip/zulip-mobile/blob/d7cf6da79b4b9467e240b42f24330e97390a8aff/android/app/src/test/java/com/zulipmobile/notifications/FcmMessageTest.kt).

`NotificationUiManager` groups notifications by Zulip identity, and creates one
messaging-style notification per conversation. A message push appends that
message and stores its Zulip ID as `lastZulipMessageId`. On a remove event, the
app scans active notifications for that identity and cancels a conversation's
notification only if its stored *last* message ID is among the remove IDs; it
then cancels the identity's summary notification if no other conversation
notification remains. It does not remove individual messages from a conversation
notification. Source: [NotificationUiManager.kt](https://github.com/zulip/zulip-mobile/blob/d7cf6da79b4b9467e240b42f24330e97390a8aff/android/app/src/main/java/com/zulipmobile/notifications/NotificationUiManager.kt).

Consequently, an unsolicited remove for a never-pushed ID does not match a
notification's last ID and cannot clear another conversation's notification.
If there is no active child notification for that identity, the attempted
summary cancellation is a no-op. If the same conversation has a prior pushed
notification followed by a DND-suppressed message, removing only the suppressed
message ID does not match the prior notification's stored last-pushed ID.

### iOS

The mobile notification architecture doc says iOS delegates display to the
operating system. The mobile APNs parser accepts `message`-type data (or legacy
data without an event type) and returns no notification for other event types;
its own comment says it has no way to handle `remove`. However, that comment's
claim that servers do not send iOS removals is stale against the current server
source: the server constructs an APNs `remove` payload and sends it through the
same delivery function. The payload has no alert, but carries `event: remove`;
the mobile client does not process it to withdraw an already delivered iOS
notification. Sources: mobile [notifications.md](https://github.com/zulip/zulip-mobile/blob/d7cf6da79b4b9467e240b42f24330e97390a8aff/docs/architecture/notifications.md),
[extract.js](https://github.com/zulip/zulip-mobile/blob/d7cf6da79b4b9467e240b42f24330e97390a8aff/src/notification/extract.js),
[notification-test.js](https://github.com/zulip/zulip-mobile/blob/d7cf6da79b4b9467e240b42f24330e97390a8aff/src/notification/__tests__/notification-test.js),
[NotificationListener.js](https://github.com/zulip/zulip-mobile/blob/d7cf6da79b4b9467e240b42f24330e97390a8aff/src/notification/NotificationListener.js),
and [ZLPNotifications.swift](https://github.com/zulip/zulip-mobile/blob/d7cf6da79b4b9467e240b42f24330e97390a8aff/ios/ZulipMobile/ZLPNotifications.swift);
server [APNs remove payload and dispatch](https://github.com/tsueri/zulip/blob/main/zerver/lib/push_notifications.py).
This is an existing read-revocation mismatch, not a DND-suppressed-message
artifact: a DND-suppressed message has no prior displayed notification to
revoke. Enabling DND also does not send a removal for notifications already
displayed before DND began.

## 2. What server-side silence leaves visible

If the server suppresses a push before delivery, neither platform receives the
push data from which its OS notification is created. There is no client-side
placeholder, “suppressed” notice, or pending notification to clear. The app's
foreground message state comes separately through Zulip's event system: mobile
polls server events, translates a `message` event to `EVENT_NEW_MESSAGE`, and
passes the event's message flags through to the Redux action. Sources:
[realtime architecture](https://github.com/zulip/zulip-mobile/blob/d7cf6da79b4b9467e240b42f24330e97390a8aff/docs/architecture/realtime.md),
[eventActions.js](https://github.com/zulip/zulip-mobile/blob/d7cf6da79b4b9467e240b42f24330e97390a8aff/src/events/eventActions.js),
[eventToAction.js](https://github.com/zulip/zulip-mobile/blob/d7cf6da79b4b9467e240b42f24330e97390a8aff/src/events/eventToAction.js),
and [messagesReducer.js](https://github.com/zulip/zulip-mobile/blob/d7cf6da79b4b9467e240b42f24330e97390a8aff/src/message/messagesReducer.js).
That is consistent with the DND decision that messages, mention flags, and
unread state remain unchanged; suppression is not a client-side mark-read.

The server's removal path is tied to the `active_mobile_push_notification`
marker: the push worker sets it while processing a push, and message-flag
cleanup queues removals when clearing it. Sources: [server push worker](https://github.com/tsueri/zulip/blob/main/zerver/lib/push_notifications.py),
[server message-flag actions](https://github.com/tsueri/zulip/blob/main/zerver/actions/message_flags.py),
and the earlier [push-notification-path fact sheet](https://github.com/tsueri/zulip/blob/research/push-notification-path/research/push-notification-path.md).
With DND checked at each push delivery attempt and suppressed messages never
marked as having an active push, normal read/delete/edit cleanup should not
produce remove events for DND-suppressed messages. The Android behavior above
is a defensive fallback, not a reason to send removals for suppressed messages.

### Unread counts and badges

Unread and message state update through events and initial data, not by creating
a system notification. The mobile `UnreadCount` component merely renders the
count its caller supplies; it does not derive or increment unread state from a
push ([UnreadCount.js](https://github.com/zulip/zulip-mobile/blob/d7cf6da79b4b9467e240b42f24330e97390a8aff/src/common/UnreadCount.js)).
Thus DND should not decrement or freeze unread/mention state in the app, and
nothing in the mobile push renderer is needed to keep those counts correct.

The platform-level notification counters are different: Android sets a
notification number from the messages in that conversation's
`MessagingStyle` notification. iOS APNs payloads include a `badge` field, but
the current server implementation explicitly returns `0` for APNs badge count
while a client-side issue is investigated. Sources: [NotificationUiManager.kt](https://github.com/zulip/zulip-mobile/blob/d7cf6da79b4b9467e240b42f24330e97390a8aff/android/app/src/main/java/com/zulipmobile/notifications/NotificationUiManager.kt),
[server APNs payload builder](https://github.com/zulip/zulip/blob/main/zerver/lib/push_notifications.py),
and [AppDelegate.mm](https://github.com/zulip/zulip-mobile/blob/d7cf6da79b4b9467e240b42f24330e97390a8aff/ios/ZulipMobile/AppDelegate.mm).
So DND does not change the server's unread counts or in-app unread badges; it
also does not cause a DND-specific OS badge update. The spec should avoid
implying that silence changes unread state or that ending DND will replay pushes.

## 3. User status and presence on mobile today

The mobile status picker is already a “Set your status” screen with text input,
emoji input, suggested text/emoji pairs, save, and clear. It submits only
`status_text` and `status_emoji` via `api.updateUserStatus`; there is no DND
control in the screen. Sources: [UserStatusScreen.js](https://github.com/zulip/zulip-mobile/blob/d7cf6da79b4b9467e240b42f24330e97390a8aff/src/user-statuses/UserStatusScreen.js)
and [updateUserStatus.js](https://github.com/zulip/zulip-mobile/blob/d7cf6da79b4b9467e240b42f24330e97390a8aff/src/api/users/updateUserStatus.js).

The status reducer stores the legacy `away`, `status_text`, and
`status_emoji` fields; updates destructure those fields, so a new DND property
is not represented in its current state. In a user list, `UserItem` renders the
user's status emoji beside their name, but not status text. Sources:
[userStatusesModel.js](https://github.com/zulip/zulip-mobile/blob/d7cf6da79b4b9467e240b42f24330e97390a8aff/src/user-statuses/userStatusesModel.js),
[userStatusesCore.js](https://github.com/zulip/zulip-mobile/blob/d7cf6da79b4b9467e240b42f24330e97390a8aff/src/user-statuses/userStatusesCore.js),
and [UserItem.js](https://github.com/zulip/zulip-mobile/blob/d7cf6da79b4b9467e240b42f24330e97390a8aff/src/users/UserItem.js).

Presence is a separate concept and display. The app aggregates presence across
clients into active/idle/offline; the user list groups users by those presence
states, and a small colored indicator overlays the avatar (green active, orange
idle, gray offline; offline is hidden in the list avatar). The presence heartbeat
reports the mobile client as active while the app is foregrounded. Sources:
[presenceModel.js](https://github.com/zulip/zulip-mobile/blob/d7cf6da79b4b9467e240b42f24330e97390a8aff/src/presence/presenceModel.js),
[userHelpers.js](https://github.com/zulip/zulip-mobile/blob/d7cf6da79b4b9467e240b42f24330e97390a8aff/src/users/userHelpers.js),
[UserAvatarWithPresence.js](https://github.com/zulip/zulip-mobile/blob/d7cf6da79b4b9467e240b42f24330e97390a8aff/src/common/UserAvatarWithPresence.js),
[PresenceStatusIndicator.js](https://github.com/zulip/zulip-mobile/blob/d7cf6da79b4b9467e240b42f24330e97390a8aff/src/common/PresenceStatusIndicator.js),
and [PresenceHeartbeat.js](https://github.com/zulip/zulip-mobile/blob/d7cf6da79b4b9467e240b42f24330e97390a8aff/src/presence/PresenceHeartbeat.js).
A DND status bit will not automatically appear as a presence state or change
these indicators; treating DND as “idle,” “offline,” or “unavailable” would be
a new product behavior, not what the current mobile app does.

## 4. Spec implications

- Server-side push suppression does not require mobile to suppress notification
  rendering: no push means no notification is rendered. Do not suppress the
  independent event stream, unread flags, mention flags, or in-app counts.
- State that the client receives no deferred/replayed push when DND ends; users
  see retained unread messages when they next view the app.
- State that DND prevents new pushes but does not retract notifications already
  displayed before DND began. The server's active-push marker should prevent
  DND-suppressed messages from generating removals; Android ID matching is a
  safe no-op for an absent notification ID. The server sends APNs remove payloads
  but current iOS does not process them to clear an existing notification.
- Separate notification silence from the status/presence UI. If mobile is an
  entry point for DND, extend its status picker and status model to set/read the
  bit; if no mobile changes are in scope, say explicitly that old/current mobile
  clients can honor server silence but cannot toggle or display the DND
  indicator. Do not imply DND changes presence.
- Do not promise a DND-updated OS badge: mobile's in-app unread state is
  independent of push, Android notification numbers reflect displayed thread
  messages, and APNs badge count is currently hardcoded to zero.

## Direct answers

(a) “No mobile changes needed” is defensible for the server-side suppression
mechanism and push display/removal pipeline, but not as an unqualified
end-to-end spec statement if mobile must provide the status-picker entry point
or show the DND indicator. State the scope explicitly.

(b) Name that events/unreads/mentions remain intact while pushes are absent and
are not replayed at DND end; DND does not retract earlier notifications; and
suppressed messages must not be marked active or later produce remove events.
Android matches remove IDs against displayed notifications; the server sends
APNs removals but current iOS does not use them to clear existing notifications.
App unread state is independent of OS notification counts, and mobile's current
status text/emoji UI and presence indicators are separate from the new DND bit.
If mobile v1 exposes DND, update the status editor/model; otherwise call out that
current clients silently honor but cannot set/show DND.
