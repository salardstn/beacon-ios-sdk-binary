# BeaconSDK for iOS

Swift Package for receiving push notifications from a Beacon server. It registers the device and its
APNs token, keeps tags, username and phone number in sync, reports when the user taps a notification,
and hands silent (background) notifications to your app.

- iOS 15+, Swift 5.9+ (Xcode 15+)
- No third-party dependencies
- No method swizzling: you forward a few AppDelegate callbacks
- Optional Notification Service Extension library for image notifications
- Localized notifications need no code: the server picks the version for the device's language

The SDK ships as prebuilt XCFrameworks attached to each [release](https://github.com/salardstn/beacon-ios-sdk-binary/releases).
They include dSYMs, so crash reports from apps using the SDK can be symbolicated.

## Installation

Xcode → **File → Add Package Dependencies…** → enter `https://github.com/salardstn/beacon-ios-sdk-binary` → add the
`BeaconSDK` library to your app target.

Or in `Package.swift`:

```swift
.package(url: "https://github.com/salardstn/beacon-ios-sdk-binary.git", from: "0.2.1")
```

Then, on the app target under **Signing & Capabilities**:

| Capability | Needed for |
|---|---|
| **Push Notifications** | Everything |
| **Background Modes → Remote notifications** | Silent notifications only. It adds `remote-notification` to `UIBackgroundModes` in Info.plist. |

## Registration

```swift
import UIKit
import UserNotifications
import BeaconSDK

@main
class AppDelegate: UIResponder, UIApplicationDelegate, UNUserNotificationCenterDelegate {
    func application(_ application: UIApplication,
                     didFinishLaunchingWithOptions launchOptions: [UIApplication.LaunchOptionsKey: Any]?) -> Bool {
        // Must be set before this method returns, so a tap that launched the app is delivered.
        UNUserNotificationCenter.current().delegate = self

        Beacon.initialize(
            appId: "YOUR-APP-ID",
            options: BeaconOptions(apiBaseURL: URL(string: "https://push.example.com")!)
        )
        Beacon.onNotificationOpened = { opened in
            // opened.notificationId, opened.title, opened.body, opened.data
        }
        Beacon.onBackgroundNotification = { notification, completion in
            // Silent notification: refresh content, then report the result (see below).
            completion(.noData)
        }
        return true
    }

    func application(_ application: UIApplication, didRegisterForRemoteNotificationsWithDeviceToken deviceToken: Data) {
        Beacon.didRegisterForRemoteNotifications(deviceToken: deviceToken)
    }

    func application(_ application: UIApplication, didFailToRegisterForRemoteNotificationsWithError error: Error) {
        Beacon.didFailToRegisterForRemoteNotifications(error: error)
    }

    // Silent notifications (and other background pushes) arrive here.
    func application(_ application: UIApplication,
                     didReceiveRemoteNotification userInfo: [AnyHashable: Any],
                     fetchCompletionHandler completionHandler: @escaping (UIBackgroundFetchResult) -> Void) {
        if Beacon.handleBackgroundNotification(userInfo: userInfo, fetchCompletionHandler: completionHandler) { return }
        completionHandler(.noData) // not a Beacon notification: handle it or complete it yourself
    }

    // The user tapped a notification.
    func userNotificationCenter(_ center: UNUserNotificationCenter,
                                didReceive response: UNNotificationResponse,
                                withCompletionHandler completionHandler: @escaping () -> Void) {
        Beacon.handleNotificationResponse(response)
        completionHandler()
    }

    // Show banners while the app is in the foreground (optional).
    func userNotificationCenter(_ center: UNUserNotificationCenter,
                                willPresent notification: UNNotification,
                                withCompletionHandler completionHandler: @escaping (UNNotificationPresentationOptions) -> Void) {
        completionHandler(Beacon.presentationOptions(for: notification))
    }
}
```

SwiftUI apps use the same class through `@UIApplicationDelegateAdaptor(AppDelegate.self)`. The
example app in this repository's `Example/` folder shows this.

Ask for permission when it makes sense in your UX:

```swift
Beacon.requestPermission { granted in … }
```

The device appears in the admin panel's **Devices** table after the first launch. It becomes
`active` (able to receive notifications) once the user grants permission.

### Player ID

The server gives each installation a `playerId` when it first registers, a moment after
`initialize` (or once the device is back online). Read it with `Beacon.playerId`, which is `nil`
until then. To link it to your own user, for example on your backend, use the callback instead:

```swift
Beacon.onPlayerIdChanged = { playerId in
    api.linkBeaconPlayer(playerId)
}
```

It runs on the main thread once the id is known, right away if it already is, and again whenever it
changes. The id changes only when the server no longer knows the device and the SDK registers it again.

## User data

### Username and phone number

```swift
Beacon.setUsername("alice")             // your user identifier (external_user_id in the REST API)
Beacon.setPhoneNumber("09013772544")   // digits only
// On logout:
Beacon.setUsername(nil)
Beacon.setPhoneNumber(nil)
```

Phone numbers must contain only digits (at most 32), such as `09013772544`; no `+`, spaces or
dashes. The SDK logs and ignores other values instead of sending them. Both fields are shown in the admin panel's device
table, and both are searchable there. To send to a user from your backend, use
`"include_external_user_ids": ["alice"]` in the REST API.

### Tags

Tags are typed key/value pairs used to build segments. `setTags` **replaces all of the device's tags**
with the dictionary you pass: tags not in it are removed, including ones set by an earlier call or through
the REST API. Literals convert to `BeaconTagValue` automatically:

```swift
Beacon.setTags([
    "plan": "pro",                     // string
    "seats": 3,                        // integer
    "beta": true,                      // boolean
    "renewsAt": .date(renewalDate),    // date
])
Beacon.deleteAllTags()                 // removes all tags; same as setTags([:])
```

To change some tags and keep the rest, use `addTags`, `setTag` and `deleteTags`:

```swift
Beacon.addTags(["plan": "team", "seats": 5])
Beacon.setTag("seats", seatCount)      // Int, String, Bool and Date overloads
Beacon.deleteTags(["beta"])
```

| `BeaconTagValue` | Sent as | Limits |
|---|---|---|
| `.string(String)` | `"pro"` | ≤ 256 characters |
| `.int(Int)` | `3` | ±(2⁵³ − 1) |
| `.bool(Bool)` | `true` | |
| `.date(Date)` | `{"$date": "2026-11-01T08:30:00.000Z"}` | ISO 8601 in UTC, millisecond precision |

Keys may contain letters, digits, `_`, `-` and `:` (max 128 characters); a device can have up to 100
tags. Invalid tags are logged and ignored locally. Segments compare values **by type**: a segment for
`seats greaterThan 2` matches the integer `3`, but not the string `"3"`. If you used
`setTags(["level": "3"])` with SDK 0.1, switch to `"level": 3` to filter numerically.
`setTags(_: [String: String])` still compiles and sends strings.

### Language

The SDK reports the device's primary preferred language, such as `de` for German or `pt` for both
`pt-BR` and `pt-PT`. The server picks the matching translation of a localized notification, or English
when there is none, so the app needs no code for localization. In the admin panel, write translations
for the language alone (`pt`, `zh`) to reach iOS devices.

## Notification handling

### Taps

`Beacon.handleNotificationResponse(_:)` reports a tap on a Beacon notification as an **open**, then
calls `Beacon.onNotificationOpened` on the main thread with a `BeaconOpenedNotification`:

| Property | |
|---|---|
| `notificationId` | Id in the Beacon dashboard |
| `title`, `subtitle`, `body` | As shown, in the device's language |
| `data` | Custom data from the composer or the REST API's `data` field |
| `imageURL` | The image, if one was sent |

Dismissals and custom actions don't count as opens. The method returns `false` for notifications not
sent by Beacon.

### Silent notifications

A silent notification (sent with **Type → Silent** in the panel, or `"silent": true` from the REST API)
has no alert, sound or badge. It wakes your app in the background so you can refresh content:

```swift
Beacon.onBackgroundNotification = { notification, completion in
    // notification.notificationId, notification.data (e.g. ["refresh": "inbox"])
    Task {
        do {
            let changed = try await InboxStore.shared.refresh()
            completion(changed ? .newData : .noData)
        } catch {
            completion(.failed)
        }
    }
}
```

- Forward `application(_:didReceiveRemoteNotification:fetchCompletionHandler:)` to
  `Beacon.handleBackgroundNotification(userInfo:fetchCompletionHandler:)`, as shown in
  [Registration](#registration). It returns `true` when Beacon owns the completion handler; complete
  other providers' notifications yourself.
- The handler runs on the main thread. Call `completion` **once** with `.newData`, `.noData` or
  `.failed`. Extra calls are ignored. If you don't call it within 25 seconds, the SDK reports `.failed`
  for you, because iOS gives the app about 30 seconds.
- Without a handler, Beacon completes with `.noData` and logs a warning. Visible Beacon notifications
  that reach this callback are completed with `.noData` immediately.
- Requires **Background Modes → Remote notifications** (`UIBackgroundModes` contains `remote-notification`).
  Without it, iOS doesn't wake the app.

**iOS controls whether and when background work runs.** Apple delivers silent notifications at low
priority and may delay, coalesce or drop them. This is more likely when you send more than a few per
hour, in Low Power Mode, or when the battery is low. Nothing is delivered after the user force-quits
the app from the app switcher, until they open it again. Don't rely on silent notifications for
time-critical or guaranteed work. Opens aren't counted for them.

### Images (optional)

Notifications sent with an image carry `mutable-content: 1` and the image URL. To show the image,
add a **Notification Service Extension** target to your app (File → New → Target), link the
**`BeaconNotificationServiceExtension`** library to it (not `BeaconSDK`), and make its principal
class a subclass:

```swift
import BeaconNotificationServiceExtension

class NotificationService: BeaconNotificationService {}
```

The extension downloads the image (JPEG, PNG or GIF, ≤ 10 MB), attaches it, and falls back to
the plain notification if the download fails or takes longer than iOS allows (~30 s). If you
already have a service extension, call `BeaconRichContent.imageURL(from:)` and
`BeaconRichContent.attachment(from:)` from it instead. Image URLs are subject to App Transport
Security, so use HTTPS.

## API

| Call | Purpose |
|---|---|
| `Beacon.initialize(appId:options:)` | Start the SDK. Registers with APNs on every launch (configurable). |
| `Beacon.requestPermission(options:completion:)` | Show the system prompt, then register with APNs. |
| `Beacon.didRegisterForRemoteNotifications(deviceToken:)` | Forward the APNs token. |
| `Beacon.didFailToRegisterForRemoteNotifications(error:)` | Forward APNs registration errors (logged). |
| `Beacon.handleNotificationResponse(_:)` | Report a tap as an open; returns `true` for Beacon notifications. |
| `Beacon.handleBackgroundNotification(userInfo:fetchCompletionHandler:)` | Forward background pushes; returns `true` when Beacon completes them. |
| `Beacon.presentationOptions(for:)` | Banner/list/sound/badge while in foreground. |
| `Beacon.setUsername(_:)` | Your user identifier; `nil` clears it (e.g. on logout). |
| `Beacon.setPhoneNumber(_:)` | Phone number, digits only; `nil` clears it. |
| `Beacon.setTags(_:)` | Replaces all tags with typed tags (`BeaconTagValue`) used for segments. |
| `Beacon.addTags(_:)`, `setTag(_:_:)`, `deleteTags(_:)` | Change some tags, keeping the others. |
| `Beacon.deleteAllTags()` | Removes all tags. |
| `Beacon.playerId` | This installation's id (after first registration). |
| `Beacon.onPlayerIdChanged` | Callback on the main thread with the `playerId` once it is known and whenever it changes. |
| `Beacon.onNotificationOpened` | Callback on the main thread when a Beacon notification is tapped. |
| `Beacon.onBackgroundNotification` | Callback on the main thread for silent notifications; call the completion once. |
| `Beacon.logLevel` | `.none` … `.debug` (default `.warning`). Logs go to Console.app under subsystem `com.beacon.sdk`. |

### Options

```swift
BeaconOptions(
    apiBaseURL: URL(string: "https://push.example.com")!,
    apnsEnvironment: nil,                         // nil = auto-detect (see below)
    registerForRemoteNotificationsOnLaunch: true
)
```

## How it works

- **Endpoints.** The SDK uses the OneSignal-style device API: `POST /api/v1/players` to add the
  device, `PUT /api/v1/players/:id` to update it, and `PUT /api/v1/notifications/:id` with
  `"opened": true` for opens. It sends OneSignal field names: `app_id`, `device_type: 0`, `identifier`
  (the APNs token), `test_type`, `notification_types`, `language`, `timezone_id`, `game_version`,
  `external_user_id`, `phone_number` and `tags`.
- **Identity.** The server assigns a `playerId` on first registration. It's stored in the Keychain
  (this-device-only), so it usually survives reinstalls. If the server no longer knows the player
  (`404`), the SDK registers again and keeps any pending changes.
- **Sync.** Token, permission status, timezone, language, device model, OS/app/SDK version and any
  pending tag, username or phone number changes are sent in one idempotent request. This happens on
  launch, on foreground, and (debounced) after each change. Unsent changes are persisted and retried
  with exponential backoff (up to 5 minutes) and on the next launch. Changes the server rejects as
  invalid are dropped rather than retried forever.
- **Opens.** A tap on a Beacon notification (payload key `beacon.nid`) is queued, persisted and
  reported once. Duplicates are ignored on both sides.
- **Silent notifications** are recognized by `aps.content-available = 1` with no alert, plus `beacon.nid`.
- **APNs environment.** Read from `aps-environment` in the embedded provisioning profile:
  development-signed builds → `sandbox`, TestFlight/App Store/Ad Hoc → `production`, simulator →
  `sandbox`. Override with `BeaconOptions.apnsEnvironment` if needed. Sent as `test_type` (`1` for
  sandbox, `null` for production).

## Upgrading from 0.1

- `setTags`/`setTag` take typed values. Code passing `[String: String]` still compiles and still sends
  strings.
- **`setTags` now replaces all tags.** In 0.1 it merged them. Use `addTags` where you relied on merging.
- Pending changes saved by 0.1 are read on first launch and sent.
- The SDK now calls the OneSignal-style `/api/v1/players` endpoints, so update the Beacon server before
  shipping an app with SDK 0.2. The server still accepts SDK 0.1's requests.
- New: `addTags(_:)`, `deleteAllTags()`, `setPhoneNumber(_:)`, `onBackgroundNotification` and `handleBackgroundNotification(userInfo:fetchCompletionHandler:)`.

## Not included in this version

Video/audio attachments, action buttons, in-app messages.
