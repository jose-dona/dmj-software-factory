# Is browser web push viable for Cadence with no mobile app?

Issue #3 (parent map #1). Facts only, from primary sources, retrieved 2026-11-16. The channel decision is a later ticket.

Sources: [MDN-Push] https://developer.mozilla.org/en-US/docs/Web/API/Push_API ; [MDN-sub] https://developer.mozilla.org/en-US/docs/Web/API/PushManager/subscribe ; [MDN-perm] https://developer.mozilla.org/en-US/docs/Web/API/Notification/requestPermission_static ; [MDN-BP] https://developer.mozilla.org/en-US/docs/Web/API/Push_API/Best_practices ; [W3C] https://www.w3.org/TR/push-api/ ; [RFC8030] https://www.rfc-editor.org/rfc/rfc8030 ; [Apple] https://developer.apple.com/documentation/usernotifications/sending-web-push-notifications-in-web-apps-and-browsers ; [WK-iOS] https://webkit.org/blog/13878/web-push-for-web-apps-on-ios-and-ipados/ ; [WK-mac] https://webkit.org/blog/12945/meet-web-push/ ; [CrUX] https://developer.chrome.com/blog/notification-permission-data-in-crux ; [USENIX] https://www.usenix.org/conference/usenixsecurity21/presentation/bilogrevic ; [OneSignal-iOS] https://documentation.onesignal.com/docs/web-push-for-ios ; [Knock] https://docs.knock.app/tutorials/sending-web-push-notifications-with-fcm ; [Novu] https://docs.novu.co/platform/integrations/push/pushpad ; [Moz-KB] https://support.mozilla.org/en-US/kb/push-notifications-firefox .

Caveats: the page fetcher returned summaries, not raw pages. [Moz-KB] and [Novu] could not be read directly (Mozilla page failed to load; Novu/Moz facts come from search-result summaries and should be re-checked). Vendor docs ([OneSignal-iOS], [Knock]) are first-party for the vendor, not for the browsers.

## Core requirements (all browsers)

- Needs an active service worker; subscribing is `PushManager.subscribe()`. The `PushSubscription` holds an endpoint and encryption key. [MDN-Push]
- HTTPS only (secure context). [MDN-sub] The spec rejects `subscribe()` with NotAllowedError if the scope is not https, or if permission is "denied". [W3C 7.1]
- `userVisibleOnly: true` is required by Chrome and Edge (promise rejects otherwise). [MDN-sub] Safari does not support invisible push; if a push is not shown immediately, Safari revokes the site's permission. [Apple]
- `applicationServerKey` is a VAPID ECDSA P-256 public key; if given, all messages must carry a VAPID JWT. [MDN-sub] Apple requires a VAPID key pair. [Apple]
- Subscribe and permission requests should follow a user gesture; Firefox enforces it since 72. [MDN-sub] Safari requires it. [MDN-BP, WK-mac] Chrome and Firefox block permission requests from cross-origin iframes. [MDN-perm]
- Permission values: `granted`, `denied`, `default` (acts as denied). [MDN-perm]
- The endpoint is a capability URL and must be kept secret; MDN warns to protect subscribe flows against CSRF. [MDN-Push]
- Push services differ per browser. Firefox desktop: Mozilla-operated; Firefox Android: Mozilla plus Google's messaging platform. [Moz-KB, search summary only]

## Browser matrix

| Platform | Facts | Source |
|---|---|---|
| Chrome / Edge (desktop, Android) | Standard Push API; VAPID and `userVisibleOnly:true` required. Chrome has a quieter permission UI that auto-enrolls low-accept sites. | MDN-sub; CrUX |
| Firefox | Opt-in. Quota on pushes that show no notification (pushes that notify are exempt). Desktop Firefox keeps a push connection while open; closed-browser delivery only on supported Windows installs via a companion program. | MDN-Push; Moz-KB (summary) |
| Safari macOS | Safari 16 on macOS 13 Ventura+. Safari need not be running (`webpushd` wakes it). Requires explicit user gesture. No Apple Developer Program needed. Allow `*.push.apple.com`. | WK-mac; Apple |
| Safari iOS/iPadOS | 16.4+. Only for Home Screen web apps (manifest `display` standalone/fullscreen). Request must follow direct user interaction. Sent via APNs. Integrates with Focus; shows on Lock Screen, Notification Center, Watch. | WK-iOS; Apple |
| Other browsers on iOS | Chrome, Edge, Firefox on iPhone/iPad follow the same Home Screen path as Safari. | OneSignal-iOS (vendor) |

### iOS specifics (from [OneSignal-iOS], vendor docs)

A browser tab cannot prompt. iOS has no install button or `beforeinstallprompt`, so users must be told to use Share > Add to Home Screen. A request not tied to a tap shows no dialog. After "Don't Allow", iOS will not prompt again for that web app (remove icon, re-add, resubscribe). On iOS 26+ the "Open as Web App" switch is on by default; if turned off, iOS saves a bookmark that cannot prompt. Removing the icon deletes the subscription. Safari revokes permission if a push does not display a notification.

## Delivery semantics

- `TTL` is mandatory (400 if missing). With TTL 0 the message is delivered only if the device is reachable now. After TTL the service MUST NOT deliver. [RFC8030 5.2]
- 201 means accepted, not delivered. [RFC8030 5]
- Storage is best effort; the push service may expire a subscription at any time; sending to an expired subscription returns 404 (Apple returns 410 "device token has expired"). [RFC8030 7.2, 7.3; Apple]
- Optional receipts; the service MAY stop retrying before expiry. [RFC8030 6.2]
- Urgency (very-low, low, normal, high) and Topic (max 32 chars, coalescing) exist. Services must accept bodies up to 4096 bytes; Apple's limit is 4 KB. [RFC8030 5.3, 5.4, 7.2; Apple]
- Apple may store undelivered pushes up to 30 days or less depending on TTL; the count stored while offline is limited. [Apple]
- Rate limiting is possible (429). [RFC8030 8.4; Apple]
- Subscriptions can change or die outside the app's control; the browser fires `pushsubscriptionchange` (new subscription null if permission was lost). [W3C 3.4, 10.4]

## Documented opt-in numbers

- Chrome telemetry (USENIX Security 2021): notifications had "only a 12% grant rate on desktop and 23% grant rate on Android" under the original prompt. The quieter UI changed acceptance by under 5% in an A/B test on 40M+ users. [USENIX]
- Chrome auto-enrolls sites with very low Accept rates into the quieter UI, decided per device type; Block rate is a strong negative signal; normal rates vary by site type (chat and email apps expected high). Google News example: 82.3% accept. [CrUX]
- MDN states "as many as 60% of push notifications are blocked" in some studies (uncited on that page). [MDN-BP]
- Not found in primary sources: iOS Home Screen web-app opt-in rates, end-to-end delivery rates.

## How hosted vendors wire it up

- Common pieces on our side: HTTPS, a service worker on our origin, a click-triggered permission flow, a stored per-browser token.
- OneSignal: Web SDK plus its worker `OneSignalSDKWorker.js` on our origin; iOS also needs manifest.json and Home Screen install. [OneSignal-iOS]
- Knock: "does not have a first-party web push channel"; uses FCM. Needs `firebase-messaging-sw.js` in `public`, a public VAPID key to get a token, token stored as channel data on the Knock user; its sample blocks iOS unless installed; unsubscribe is via browser UI or clearing the stored token. [Knock]
- Novu: web push through the Pushpad provider; Pushpad uid registered as the subscriber device token; also a generic push webhook provider. [Novu, search summary]
- Not verified here: other vendors, free tiers.

## What a Cadence Student could realistically receive

- Desktop Chrome, Edge, Firefox, Safari 16+: OS-level notifications after a click-triggered prompt, via a service worker Cadence would need to add (repo not checked for an existing one). Safari macOS delivers with the browser closed; Firefox generally needs to be open (Windows exception above).
- Android Chrome/Firefox: same flow.
- iPhone/iPad: only on iOS 16.4+ after adding Cadence to the Home Screen as a web app, opening it from the icon, and tapping a subscribe control. A denial is not re-promptable without reinstall.
- Every push must show a visible notification. Payload about 4 KB. Delivery is best effort with TTL; accepted is not delivered.
- Documented cold-prompt opt-in is low (12% desktop, 23% Android in Chrome data); no iOS figure found.
- Subscriptions can expire silently, so web push alone cannot be the record; the in-app Notification Feed (and possibly email) is the durable channel.
