# Hosted vendors for in-app feed, web push and email

Issue: #2 (parent map #1). Researched 2026-11-16. Facts only, taken from each vendor's own pricing and docs pages (fetched through a page-summarising tool, so quotes are as returned by it; re-verify exact numbers before relying on them). No recommendation.

Question: which hosted vendors cover (a) an in-app Notification Feed we render ourselves, (b) browser web push, (c) email, with a free tier usable in development?

Note: `app/lib/mailer.server.ts` (Postmark) is not present on the branch this research was run from, so the Postmark integration was not inspected; Postmark support is checked only against each vendor's provider list.

## Summary table

| Vendor | Free tier | In-app feed, headless | Web push | Email via Postmark |
| --- | --- | --- | --- | --- |
| Knock | 10,000 messages/month | Yes (client API, websocket, headless mode) | No first-party channel; via FCM channel | Yes |
| Novu | 10,000 workflow runs/month | Yes (`@novu/js`, `@novu/react` headless) | Via FCM (`webPush` override) | Yes |
| Courier | 10,000 sends/month | Not verified | Not verified (push providers listed, none browser-specific) | Yes |
| MagicBell | 1,000 deliveries/month | Not verified | Native web push channel (VAPID) | Not verified |
| OneSignal | 10,000 email sends/month; web push up to 10,000 subscribers per send | No notification center/feed API found on pricing page | Native, requires service worker | n/a (it sends email itself) |

## Knock

- Free plan: 10,000 messages/month, no credit card. A message is "a single message delivered to a single user on a single channel"; bounced/failed messages are not counted. Limits are soft ("We don't hard cap your messages"). Unlimited workflows, delivery channels, team members, feed retention. Log retention 30 days. Not on Free: per-tenant branding/preferences, audit logs, custom roles. Source: https://knock.app/pricing
- In-app feed: the feed channel "returns a per-user list of in-app messages to render in a feed-based UI"; fetched via client API, newest first, filterable; returns unread/read/seen counts; real-time over a Knock-managed websocket; states unread/read/seen/archived; headless mode with lower-level primitives; JS (non-React) SDK listed. Retention period not stated on the page. Message content cannot be changed after generation. Sources: https://docs.knock.app/in-app-ui/overview , https://docs.knock.app/in-app-ui/feeds/overview
- Web push: "Knock does not have a first-party web push channel." Web push goes through an FCM channel configured in the dashboard; the browser gets a token via the Firebase SDK, which is stored on the user's Knock profile as channel data. Needs a service worker (`firebase-messaging-sw.js`) and a VAPID key; the guide states FCM web push is supported in all major modern browsers; on iPhone the web app must be installed to the home screen. Expired tokens are the app's responsibility unless the opt-in token deregistration feature is used. Sources: https://docs.knock.app/tutorials/sending-web-push-notifications-with-fcm , https://docs.knock.app/integrations/push/token-deregistration
- Email: Postmark is a supported provider (also Amazon SES, Resend, SendGrid, SMTP, others; plus "Knock (testing only)"). Source: https://docs.knock.app/integrations/email/overview
- Preference model details were not researched.

## Novu

- Free plan: 10,000 workflow runs/month, no credit card. All channels (email, in-app, SMS, chat, push). Activity feed retention 24 hours. Up to 5 workflows, 2 environments (Dev + Prod), 3 team members. No option to remove Novu branding on Free. Behavior above the limit is not stated on the page. Source: https://novu.co/pricing
- In-app inbox: "Build your own" path using `@novu/js` (framework-agnostic) or `@novu/react` hooks for a fully custom Inbox. `@novu/react` needs React 18+. Method list not read. Source: https://docs.novu.co/platform/inbox/overview
- Web push: the FCM FAQ states "Desktop notifications for websites can be sent using FCM webpush." Setup: paste a Firebase service-account key into the FCM integration, save each subscriber's `deviceTokens`, trigger with a `webPush` override (use a relative `link` to return users to the site). Browser-side setup (service worker) is not covered on that page. Source: https://docs.novu.co/platform/integrations/push/fcm . Other push providers mentioned in a search summary (not read first-hand): OneSignal, Pushpad, Expo, APNs.
- Email: Postmark is a supported provider. Source: https://docs.novu.co/platform/integrations/email

## Courier

- Free (Developer) plan: 10,000 sends/month; one template delivered over several channels counts as one send. Above the limit, adding a card moves to the Business plan at $0.005/send including the first 10,000. Excluded from free: EU data residency, RBAC, enterprise SLA, AI features. The page does not name which channels the free plan includes. Source: https://www.courier.com/pricing
- Push providers listed: APNs, FCM, Expo, OneSignal, Airship, Amazon SNS, Pusher Beams, Pusher, MagicBell, Pushbullet, Beamer, NowPush. The page does not say whether browser web push is supported. Source: https://www.courier.com/docs/external-integrations/push/intro
- Email: Postmark is a supported provider. Source: https://www.courier.com/docs/external-integrations/email/intro
- Headless in-app inbox: not verified.

## MagicBell

- Free (Builder) plan: 1,000 deliveries/month, counted per channel (one notification on two channels = two deliveries); 1 project; unlimited users and team members. Overage billed per use; the Builder rate is not stated. Channels not named for the plan. Source: https://www.magicbell.com/pricing
- Web push: dedicated Web Push channel configured with a VAPID public/private key pair; browser subscriptions stored as channel tokens (`endpoint`, `keys.p256dh`, `keys.auth`) via `PUT /v2/channels/web_push/tokens`. Source: https://www.magicbell.com/docs/web-push . A search summary of the docs (not read first-hand) says the `@magicbell/webpush` SDK registers a service worker (default `/sw.js`) and that Inbox supports web push on Chrome, Firefox and Safari.
- Headless feed API and email provider support: not verified (the email integration docs URL returned 404).

## OneSignal

- Free plan: email 10,000 sends/month (shared IPs); web push "Max 10,000 subscribers per send"; in-app messaging unlimited for orgs up to 1,000 MAU (carousels excluded). The pricing page mentions no in-app notification center/feed API; the Notification History API is not on Free. 3 active journeys, 6 segments; no frequency capping or message throttling on Free. Source: https://onesignal.com/pricing
- Web push: requires HTTPS with a valid certificate and the ability to add the OneSignal service worker; supported browsers Chrome, Firefox, Safari, Edge; Safari on macOS and iOS 16.4+ ("iOS requires extra setup"). Localhost behavior not stated. Source: https://documentation.onesignal.com/docs/en/web-push-setup

## Cross-cutting facts

- Web push on every vendor that offers it uses the browser Push API plus a service worker. Knock and Novu route it through Firebase Cloud Messaging, so a Firebase project is needed in addition to the vendor account. MagicBell and OneSignal document native web push channels with no Firebase account described.
- iOS Safari web push needs the site installed to the home screen (Knock FCM tutorial; OneSignal says iOS 16.4+ with extra setup).
- Free-tier units are not directly comparable: Knock counts per message per channel; Novu counts workflow runs; Courier counts sends per template across channels; MagicBell counts deliveries per channel.

## Not verified / gaps

- Courier: free-plan channels, headless inbox, browser web push.
- MagicBell: headless feed API, email provider support.
- OneSignal: whether any in-app feed API exists beyond in-app messaging.
- Per-user preference models, delivery retry/failure behavior, data residency and privacy terms (other than Courier's EU residency exclusion), and local-dev/test behavior with no network were not researched for any vendor.
