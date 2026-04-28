![Catapush Logo](../AndroidSDK/images/catapush_logo.png)

# Catapush Multiplatform SDK — iOS integration guide

## Index

* [Catapush Multiplatform SDK 1.0.0](#catapush-multiplatform-sdk-100)
* [Project prerequisites](#project-prerequisites)
* [Configure APNs and the Catapush dashboard](#configure-apns-and-the-catapush-dashboard)
* [Add the SDK with Swift Package Manager](#add-the-sdk-with-swift-package-manager)
* [Configure your Xcode project](#configure-your-xcode-project)
    * [Capabilities](#capabilities)
    * [Set the app key in `Info.plist`](#set-the-app-key-in-infoplist)
    * [App Group (required for the Notification Service Extension)](#app-group-required-for-the-notification-service-extension)
* [Add a Notification Service Extension](#add-a-notification-service-extension)
* [Initialize the SDK](#initialize-the-sdk)
    * [`ICatapushEventDelegate`](#icatapusheventdelegate)
    * [`Catapush.shared.doInit(…)`](#catapushshareddoinit)
    * [Set credentials and start](#set-credentials-and-start)
    * [Forward APNs callbacks](#forward-apns-callbacks)
* [Error handling](#error-handling)
* [Optional UI module (`CatapushUi`)](#optional-ui-module-catapushui)
* [Advanced](#advanced)
    * [Logging](#logging)
    * [Notification visualization toggles](#notification-visualization-toggles)
    * [Sending read receipts manually](#sending-read-receipts-manually)
    * [Conversation channels and message queries](#conversation-channels-and-message-queries)
    * [Load and display message attachments](#load-and-display-message-attachments)
    * [Automatic message deletion (TTL)](#automatic-message-deletion-ttl)
    * [Logout and clear data](#logout-and-clear-data)
    * [Transform incoming messages before persisting them](#transform-incoming-messages-before-persisting-them)
    * [VoIP push tokens (PushKit)](#voip-push-tokens-pushkit)
* [FAQ](#faq)

## Catapush Multiplatform SDK 1.0.0

On iOS the SDK is shipped as a pre-built **XCFramework** binary target distributed via Swift Package Manager.

This release targets iOS 15.0 and later, and is built with Xcode 15+.

If you are coming from `catapush-ios-sdk-pod` (the previous CocoaPods-based iOS SDK), follow the dedicated [iOS migration guide](MIGRATION_FROM_IOS_SDK.md) — the migration involves a number of API changes that are easier to apply step by step than to rediscover by reading this document end-to-end.

## Project prerequisites

The Catapush Multiplatform SDK assumes that your iOS project:

1. Targets **iOS 15.0** or later.
2. Is built with **Xcode 15** or later.
3. Has Swift Package Manager configured (the default for Xcode-managed projects).
4. Includes a **Notification Service Extension** target (see [Add a Notification Service Extension](#add-a-notification-service-extension)).
5. Has an **App Group** enabled on both the main app and the extension (used by the SDK to share the local message database and the credentials between the two targets).

## Configure APNs and the Catapush dashboard

This section is identical to the previous iOS pod. Generate a token-based APNs authentication key in your Apple Developer account and register it on the Catapush dashboard:

1. In your [Apple Developer Member Center](https://developer.apple.com/account), go to **Certificates, Identifiers & Profiles → Keys** and click **+** to create a new key.
2. Enter a description, enable **Apple Push Notifications service (APNs)**, click **Continue**, then **Register**.
3. Download the `.p8` key file. **This is a one-time download — store it securely.**
4. Note the **Key ID** shown on the key details page.
5. Note your **Team ID** under [Membership Information](https://developer.apple.com/account/#/membership).
6. Go to your Catapush app at `https://www.catapush.com/panel/apps/YOUR_APP_ID/platforms`, click **iOS Token Based** and fill in:
    * **iOS Team ID** — your Apple Developer Team ID.
    * **iOS Key ID** — the Key ID from step 4.
    * **iOS AuthKey** — the full content of the `.p8` file (including the BEGIN/END lines).
    * **iOS Topic** — the bundle identifier of your iOS app.

## Add the SDK with Swift Package Manager

In Xcode, go to **File → Add Package Dependencies…** and enter:

```
https://github.com/Catapush/catapush-sdk-native
```

Select version `1.0.0` (or pin to the exact tag you intend to use). Add the **`CatapushSdk`** product to your main app target **and** to your Notification Service Extension target. If you also use the optional UI module, add **`CatapushUi`** as well; otherwise leave it out — it is not required.

The SDK is shipped as a pre-built XCFramework, so no build-from-source step is required on your side.

If you prefer `Package.swift` over the Xcode UI:

```swift
dependencies: [
    .package(url: "https://github.com/Catapush/catapush-sdk-native", from: "1.0.0")
]
```

## Configure your Xcode project

### Capabilities

In your main app target, under **Signing & Capabilities**, add:

* **Push Notifications**.
* **Background Modes** with **Remote notifications** enabled.

In the Notification Service Extension target, you only need the App Group capability (see below).

### Set the app key in `Info.plist`

Add your Catapush app key to the main app's `Info.plist`:

```xml
<key>CatapushAppKey</key>
<string>YOUR_APP_KEY</string>
```

`YOUR_APP_KEY` is the AppKey of your Catapush App. You can find it on your [Catapush App configuration dashboard](https://www.catapush.com/panel/dashboard) under "App details".

### App Group (required for the Notification Service Extension)

Catapush requires the main app and the Notification Service Extension to share resources (the local message database, the secure credentials store, the runtime configuration). To do that, enable the **same** App Group on both targets.

1. In Xcode select your main app target → **Signing & Capabilities** → **+ Capability** → **App Groups** → **+** → enter a group identifier such as `group.com.example.app.catapush`.
2. Repeat the same operation on the Notification Service Extension target, picking the **same** group identifier.
3. Add the App Group identifier to **both** the app's `Info.plist` and the extension's `Info.plist`:

    ```xml
    <key>Catapush</key>
    <dict>
        <key>AppGroup</key>
        <string>group.com.example.app.catapush</string>
    </dict>
    ```

## Add a Notification Service Extension

Catapush delivers messages as silent APNs pushes; the Notification Service Extension is responsible for fetching the actual message body before iOS displays the notification banner.

In Xcode, go to **File → New → Target… → Notification Service Extension**. Make sure:

* The extension's **deployment target matches the main app's deployment target**.
* The extension target has the same **App Group** capability as the main app (see above).
* `CatapushSdk` (and `CatapushUi`, if you use it) is added to the extension target.

The new SDK does not ship a base extension class to subclass. Implement a vanilla `UNNotificationServiceExtension` that initializes Catapush in the extension, observes the next incoming message, and feeds the result back to the system:

```swift
import UserNotifications
import CatapushSdk

class NotificationService: UNNotificationServiceExtension {

    var contentHandler: ((UNNotificationContent) -> Void)?
    var bestAttemptContent: UNMutableNotificationContent?

    override func didReceive(
        _ request: UNNotificationRequest,
        withContentHandler contentHandler: @escaping (UNNotificationContent) -> Void
    ) {
        self.contentHandler = contentHandler
        bestAttemptContent = request.content.mutableCopy() as? UNMutableNotificationContent

        guard isCatapushNotification(userInfo: request.content.userInfo) else {
            // Not a Catapush push — deliver as-is.
            contentHandler(request.content)
            return
        }

        // The credentials are read from the shared App Group database written by the main app.
        Catapush.shared.doInit(
            eventDelegate: ExtensionEventDelegate(),
            mobileServices: [CatapushApns.shared]
        )

        // Wait up to 5 seconds for the actual message to arrive, then deliver the notification.
        Catapush.shared.observeNextMessage(
            timeoutSeconds: 5,
            onMessage: { [weak self] title, body in
                guard let self = self, let best = self.bestAttemptContent else {
                    contentHandler(request.content); return
                }
                best.title = title ?? "Catapush"
                if let body = body { best.body = body }
                contentHandler(best)
            },
            onTimeout: { [weak self] in
                guard let self = self, let best = self.bestAttemptContent else {
                    contentHandler(request.content); return
                }
                contentHandler(best)
            }
        )

        // Trigger the background message fetch.
        CatapushApns.shared.handlePushNotification(userInfo: request.content.userInfo)
    }

    override func serviceExtensionTimeWillExpire() {
        if let handler = contentHandler, let best = bestAttemptContent {
            handler(best)
        }
    }

    private func isCatapushNotification(userInfo: [AnyHashable: Any]) -> Bool {
        (userInfo["sender"] as? String) == "catapush"
    }
}

private class ExtensionEventDelegate: ICatapushEventDelegate {
    func onRegistrationFailed(error: CatapushAuthenticationError) {}
    func onConnecting() {}
    func onConnected() {}
    func onDisconnected(error: CatapushConnectionError) {}
    func onPushServicesError(error: PushServicesException) {}
    func onMessageReceived(message: CatapushMessage) {}
    func onMessageReceivedConfirmed(message: CatapushMessage) {}
    func onMessageOpened(message: CatapushMessage) {}
    func onMessageOpenedConfirmed(message: CatapushMessage) {}
    func onMessageSent(message: CatapushMessage) {}
    func onMessageSentConfirmed(message: CatapushMessage) {}
}
```

You can also factor the Catapush calls into a singleton helper such as `CatapushExtensionManager` if you prefer; see the `iosApp/CatapushNotificationService/` directory in the SDK repository for a worked example.

## Initialize the SDK

The integration in the main app boils down to three calls:

1. `Catapush.shared.doInit(eventDelegate:mobileServices:)` — wires the SDK to your event delegate and to the APNs adapter.
2. `Catapush.shared.setCredentials(identifier:password:)` — provides the user identity.
3. `Catapush.shared.start(callback:)` — opens the persistent connection to the Catapush backend.

A typical `AppDelegate` looks like this:

```swift
import UIKit
import CatapushSdk

class AppDelegate: NSObject, UIApplicationDelegate {

    func application(
        _ application: UIApplication,
        didFinishLaunchingWithOptions launchOptions: [UIApplication.LaunchOptionsKey: Any]? = nil
    ) -> Bool {
        // 1. Initialize the SDK (app key is read from Info.plist > CatapushAppKey)
        Catapush.shared.doInit(
            eventDelegate: MyCatapushEventDelegate(),
            mobileServices: [CatapushApns.shared]
        )

        // 2. Set the user credentials (typically once you obtain them from your backend)
        Catapush.shared.setCredentials(identifier: "user@example.com", password: "secret")

        // 3. Start the messaging service
        Catapush.shared.start(callback: MyRecoverableErrorCallback())

        return true
    }
}
```

Request notification authorization yourself, typically once at app start:

```swift
@main
struct MyApp: App {
    @UIApplicationDelegateAdaptor(AppDelegate.self) var appDelegate

    init() {
        UNUserNotificationCenter.current().requestAuthorization(
            options: [.alert, .sound, .badge]
        ) { granted, _ in
            if granted {
                DispatchQueue.main.async {
                    UIApplication.shared.registerForRemoteNotifications()
                }
            }
        }
    }

    var body: some Scene { WindowGroup { ContentView() } }
}
```

### `ICatapushEventDelegate`

The event delegate is a single Swift class conforming to `ICatapushEventDelegate`. All callbacks are delivered on the main thread.

```swift
class MyCatapushEventDelegate: ICatapushEventDelegate {
    func onRegistrationFailed(error: CatapushAuthenticationError) { /* … */ }
    func onConnecting()                                          { /* … */ }
    func onConnected()                                           { /* … */ }
    func onDisconnected(error: CatapushConnectionError)          { /* … */ }
    func onPushServicesError(error: PushServicesException)       { /* … */ }

    func onMessageReceived(message: CatapushMessage)             { /* … */ }
    func onMessageReceivedConfirmed(message: CatapushMessage)    { /* … */ }
    func onMessageOpened(message: CatapushMessage)               { /* … */ }
    func onMessageOpenedConfirmed(message: CatapushMessage)      { /* … */ }
    func onMessageSent(message: CatapushMessage)                 { /* … */ }
    func onMessageSentConfirmed(message: CatapushMessage)        { /* … */ }
}
```

The pair `*Sent` / `*SentConfirmed` and `*Received` / `*ReceivedConfirmed` distinguish between local persistence and server-side acknowledgement: implement only the variant you need.

### `Catapush.shared.doInit(…)`

`Catapush.shared.doInit(eventDelegate:mobileServices:)` is the iOS counterpart of `Catapush.init(…)` on Android. The method is named `doInit` on Swift to avoid clashing with Swift's own `init` keyword.

The `mobileServices` parameter accepts a list of push platform adapters. On iOS the only adapter is `CatapushApns.shared`. Pass an empty list to disable push notifications entirely (foreground-only delivery):

```swift
// Foreground-only delivery — useful for short-lived sessions where you don't need background pushes
Catapush.shared.doInit(eventDelegate: MyCatapushEventDelegate(), mobileServices: [])
```

### Set credentials and start

```swift
Catapush.shared.setCredentials(identifier: "user@example.com", password: "secret")

Catapush.shared.start(callback: MyRecoverableErrorCallback())
```

The `start(callback:)` callback exposes three branches:

* **`success(response:)`** — the connection has been established.
* **`warning(recoverableError:)`** — the SDK detected a transient problem (network reachability, system restrictions, …) but will continue to retry.
* **`failure(irrecoverableError:)`** — permanent failure (wrong credentials, missing `CatapushAppKey`, …); the SDK will not retry automatically.

Credentials are persisted in the iOS Keychain so subsequent app launches can call `start(…)` without providing them again.

```swift
class MyRecoverableErrorCallback: RecoverableErrorCallback {
    func success(response: Any?) { /* connected */ }
    func warning(recoverableError: KotlinThrowable) { /* transient, will retry */ }
    func failure(irrecoverableError: KotlinThrowable) { /* permanent */ }
}
```

### Forward APNs callbacks

APNs token forwarding is handled automatically through method swizzling: in most cases you do **not** need to implement `application(_:didRegisterForRemoteNotificationsWithDeviceToken:)`. If you have disabled swizzling (for example by setting `FirebaseAppDelegateProxyEnabled = NO`), forward the callbacks manually:

```swift
func application(_ application: UIApplication,
                 didRegisterForRemoteNotificationsWithDeviceToken deviceToken: Data) {
    CatapushApns.shared.handleDeviceToken(deviceToken)
}

func application(_ application: UIApplication,
                 didFailToRegisterForRemoteNotificationsWithError error: Error) {
    CatapushApns.shared.handleDeviceTokenError(error as NSError)
}
```

When a push notification arrives in background, forward it to Catapush so it can fetch the corresponding message body. Both `application(_:didReceiveRemoteNotification:fetchCompletionHandler:)` and `UNUserNotificationCenterDelegate.userNotificationCenter(_:didReceive:withCompletionHandler:)` are valid entry points:

```swift
func application(_ application: UIApplication,
                 didReceiveRemoteNotification userInfo: [AnyHashable: Any],
                 fetchCompletionHandler completionHandler:
                     @escaping (UIBackgroundFetchResult) -> Void) {
    CatapushApns.shared.handlePushNotification(userInfo: userInfo)
    completionHandler(.newData)
}
```

## Error handling

The `onDisconnected(error: CatapushConnectionError)` callback may be invoked with the following reason codes:

| reasonCode | Description | Suggested strategy |
|---|---|---|
| 3 = `BACKEND_NOT_AVAILABLE` | The SDK could not reach the Catapush remote messaging service. | Check internet connectivity and retry in a few minutes. |
| 4 = `INTERNET_NOT_AVAILABLE` | The device is offline. | Inform the user about the lack of connectivity. |
| 6 = `SERVICE_STOPPED` | The local messaging service stopped (e.g. the app was suspended by the system). | Expected when the app goes to background; the SDK reconnects automatically when the app returns to the foreground. |
| 7 = `SERVICE_STOPPED_AS_REQUESTED` | The local messaging service stopped after `Catapush.shared.stop(…)` was called. | No action needed. |
| 8 = `CONNECTION_PARAMETERS_LOAD_FAILED` | The stored credentials are invalid. | Check the credentials passed to `setCredentials(…)`. |
| 9 = `CONNECTION_TIMEOUT` | The connection attempt timed out. | Check connectivity and retry. |
| 10 = `START_TIMEOUT` | The SDK could not complete the start command in a timely fashion. | Check connectivity and retry. |

The `onRegistrationFailed(error: CatapushAuthenticationError)` callback may be invoked with reason codes such as `REGISTRATION_FORBIDDEN_WRONG_AUTH` (24031), `REGISTRATION_NOT_FOUND_USER` (24042), and `UPDATE_PUSH_TOKEN_NOT_FOUND_USER` (44043). Inspect `error.reasonCode` to decide whether to clear the stored credentials and prompt the user to re-authenticate.

The `onPushServicesError(error: PushServicesException)` callback fires when APNs is unavailable or unauthorized. Check `error.isUserResolvable` to decide whether to surface a banner inviting the user to enable notifications in Settings.

## Optional UI module (`CatapushUi`)

The `CatapushUi` SwiftPM product ships ready-to-use Compose Multiplatform message-list and conversation views built on top of `CatapushSdk`. Most apps integrate Catapush into their own existing message UI — but `CatapushUi` can be a fast way to bring up a working chat screen.

If you want to use it, add the dependency to both the main app and the Notification Service Extension targets, then initialize it **after** `Catapush.shared.doInit(…)`:

```swift
Catapush.shared.doInit(/* … */)
CatapushUi.shared.doInit()
```

`CatapushUi` exposes Compose Multiplatform composables. To embed them in a SwiftUI screen, wrap them in a `UIViewControllerRepresentable` that hosts the Compose entry point. The exact wrapper API is being finalised — refer to the `iosApp/` directory in the SDK repository for a worked example.

The `CatapushUi` product is fully optional: omit the dependency and its initialization call if you don't use it.

## Advanced

### Logging

Enable SDK logging:

```swift
Catapush.shared.enableLog(logger: nil)
```

Pass a custom `ICatapushLogger` to route log lines to your own backend (Sentry, OSLog wrapper, internal log file, …):

```swift
class MyLogger: ICatapushLogger {
    func verbose(tag: String, msg: String) { /* … */ }
    func debug(tag: String, msg: String) { /* … */ }
    func info(tag: String, msg: String) { /* … */ }
    func warn(tag: String, msg: String, t: KotlinThrowable?) { /* … */ }
    func error(tag: String, msg: String, t: KotlinThrowable?) { /* … */ }
}

Catapush.shared.enableLog(logger: MyLogger())
```

Disable logging:

```swift
Catapush.shared.disableLog()
```

### Notification visualization toggles

The SDK exposes two pairs of toggles that mirror the Android API:

* **Persistent**: `enableNotifications()` / `disableNotifications()`. The state is persisted across app launches and Catapush starts/stops.
* **Transient**: `pauseNotifications()` / `resumeNotifications()`. Useful while the user is reading messages in your in-app conversation view; the state is reset on the next start.

```swift
// User opened the conversation view: stop showing banners for incoming messages
Catapush.shared.pauseNotifications()

// User left the conversation view: resume normal notification delivery
Catapush.shared.resumeNotifications()
```

### Sending read receipts manually

If you build your own message list UI, mark messages as read manually:

```swift
Catapush.shared.notifyMessageOpened(messageId: message.id)
```

### Conversation channels and message queries

Catapush messages can be delivered through different conversation channels. The SDK exposes the local channels and messages as `Flow`s, which Swift consumes as standard `AsyncSequence`s.

To create a `CoroutineScope` from Swift, use the helper exposed by the SDK:

```swift
let scope = CreateMainScopeKt.createMainScope()
```

The caller is responsible for cancelling the scope when it is no longer needed (e.g. in `deinit`).

```swift
// Page through messages in a specific channel
let flow = Catapush.shared.getMessages(
    cachedInScope: scope,
    pageSize: 20,
    channel: "orders"
)

Task {
    for await pagingData in flow {
        // hand `pagingData` to your list adapter
    }
}

// Observe the list of channels stored locally
Task {
    for await channels in Catapush.shared.getChannelList() {
        // channels is a list of channel identifiers; "" represents the default channel
    }
}

// Observe an unread badge for the default channel
Task {
    for await unread in Catapush.shared.countUnreadMessages(channel: nil) {
        await MainActor.run {
            UIApplication.shared.applicationIconBadgeNumber = Int(unread)
        }
    }
}
```

### Load and display message attachments

If a message includes an attachment, its remote URL is available at `message.file.remoteUri`. Outgoing messages with attachments also expose a local URI (`message.file.localUri`) — prefer that when available.

To load an image with `URLSession`:

```swift
guard let urlString = message.file?.remoteUri,
      let url = URL(string: urlString) else { return }

URLSession.shared.dataTask(with: url) { data, _, _ in
    guard let data = data, let image = UIImage(data: data) else { return }
    DispatchQueue.main.async {
        imageView.image = image
    }
}.resume()
```

To send an attachment, write the data to a file in the app sandbox and pass the resulting file URL string to `sendMessage(…)`:

```swift
let tmp = FileManager.default.temporaryDirectory.appendingPathComponent("photo.jpg")
try imageData.write(to: tmp)

Catapush.shared.sendMessage(
    message: "Look at this",
    subject: nil,
    channel: nil,
    inReplyToMessageId: nil,
    attachmentUri: tmp.absoluteString,
    callback: MyCallback()
)
```

The MIME type is inferred from the file extension; if you need to control it explicitly, write the file with the correct extension (`.jpg`, `.pdf`, …).

### Automatic message deletion (TTL)

Messages older than a configurable TTL are deleted automatically the next time `Catapush.shared.start(…)` runs successfully. The default TTL is **90 days**. Configure it explicitly:

```swift
Catapush.shared.setMessageTtlDays(days: 30) // delete messages older than 30 days
Catapush.shared.setMessageTtlDays(days: 0)  // disable automatic deletion entirely
```

The current value is available via `Catapush.shared.getMessageTtlDays()`. The setting is persisted across app restarts.

### Logout and clear data

Use `logout(…)` when the user signs out of your app. It stops the messaging service, deregisters the device on the Catapush backend, and clears the stored credentials:

```swift
class MyCallback: Callback {
    func success(response: Any?) { /* signed out */ }
    func failure(irrecoverableError: KotlinThrowable) { /* … */ }
}

Catapush.shared.logout(callback: MyCallback())
```

Use `clearDataAndCredentials()` for a more destructive reset that also wipes the locally persisted messages and attachments (e.g. when the user switches accounts):

```swift
Catapush.shared.clearDataAndCredentials()
```

### Transform incoming messages before persisting them

Install a `CatapushMessageTransformation` to intercept every incoming message before it is stored in the local database. Typical use cases include:

* end-to-end decryption of the message body before it lands in the local store;
* substitution of server-side placeholders (`{{firstName}}`, `{{orderId}}`, …) with values that only the client can resolve;
* attachment of derived metadata to drive your UI's grouping or filtering;
* sanitization or redaction of fields you do not want to persist on disk in their raw form.

```swift
class MyTransformation: CatapushMessageTransformation {
    override func transform(input: CatapushMessageTransformation.Model) -> CatapushMessageTransformation.Model {
        // input.originalMessage is the read-only original message;
        // input.body and input.previewText are the mutable values that will be persisted.

        // Example 1 — end-to-end decryption.
        let decrypted = MyCrypto.decrypt(input.body, with: MyKeystore.userKey)

        // Example 2 — placeholder substitution.
        input.body = decrypted
            .replacingOccurrences(of: "{{firstName}}", with: LocalProfile.firstName)
            .replacingOccurrences(of: "{{orderId}}", with: LocalCart.currentOrderId ?? "")

        // Optionally update the preview shown in the messages list.
        input.previewText = String(input.body.prefix(80))

        // Return the same instance with the values you want to persist.
        return input
    }
}

Catapush.shared.setMessageTransformation(messageTransformation: MyTransformation())
```

Modify `input.body` and `input.previewText` in place — the SDK persists the values present on the returned model. Other fields (id, sender, recipient, timestamps) are read-only via `input.originalMessage`.

The transformation runs synchronously on the SDK's IO scheduler. Keep it fast — long-running work blocks message persistence.

### VoIP push tokens (PushKit)

If you ship a PushKit/VoIP integration alongside regular APNs pushes, forward the VoIP token to Catapush from your `PKPushRegistryDelegate`:

```swift
import PushKit

class VoipDelegate: NSObject, PKPushRegistryDelegate {
    func pushRegistry(_ registry: PKPushRegistry,
                      didUpdate pushCredentials: PKPushCredentials,
                      for type: PKPushType) {
        let token = pushCredentials.token.map { String(format: "%02x", $0) }.joined()
        CatapushApns.shared.handleVoipToken(voipToken: token)
    }
}
```

This is only needed if your app also uses VoIP pushes for incoming-call workflows; regular Catapush messaging does not require PushKit.

## FAQ

### Why is `init` exposed as `doInit` in Swift?

The SDK's `init` method is renamed to `doInit` on the Swift side to avoid clashing with Swift's own `init` keyword.

### How do I consume a `Flow` from Swift?

The SDK exposes `Flow`-based APIs (`getMessages`, `getChannelList`, `countMessages`, …) as standard `AsyncSequence`s. Iterate them with `for await … in flow` inside a `Task { … }`. Use `CreateMainScopeKt.createMainScope()` to create a `CoroutineScope` when one is required (for example as the `cachedInScope` parameter of `getMessages(…)`), and cancel the scope in your view controller's `deinit`.

### Can I use Catapush without push notifications (foreground-only delivery)?

Yes. Pass an empty list to `mobileServices`:

```swift
Catapush.shared.doInit(eventDelegate: MyDelegate(), mobileServices: [])
```

Messages are received only while the app is in the foreground and the SDK is connected.

### Do I need a Notification Service Extension?

Yes, for any production app. APNs delivers Catapush pushes as silent payloads; the actual message body is fetched from the server by the extension before the system displays the banner. Without an extension, users see only generic notification text.

### Is there an example project available?

The Catapush Multiplatform SDK repository includes a working sample at [`iosApp/`](https://github.com/Catapush/catapush-sdk-native/tree/main/iosApp), including the main app target and a Notification Service Extension target.
