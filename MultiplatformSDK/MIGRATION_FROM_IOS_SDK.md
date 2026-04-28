![Catapush Logo](../AndroidSDK/images/catapush_logo.png)

# Migrating from `catapush-ios-sdk-pod` to Catapush Multiplatform SDK 1.0.0

The Catapush Multiplatform SDK is the successor to the previous `catapush-ios-sdk-pod` (and to the previous `catapush-android-sdk` on Android). On iOS it is shipped as a pre-built **XCFramework**, distributed via Swift Package Manager.

This guide covers the migration from the previous CocoaPods-based iOS SDK. The migration on iOS is more involved than its Android counterpart because the API surface has been modernized for Swift-first usage: the `Catapush` static methods, the dual `CatapushDelegate` / `MessagesDispatchDelegate` protocols, the `MessageIP` CoreData entity and the `CatapushNotificationServiceExtension` subclass have all been replaced. The functional model — connect, exchange messages with attachments, receive APNs wakeups, render via a Notification Service Extension, share state with the extension via an App Group — is unchanged.

## Index

* [At a glance](#at-a-glance)
* [1. Replace CocoaPods with Swift Package Manager](#1-replace-cocoapods-with-swift-package-manager)
* [2. Move the app key to `Info.plist`](#2-move-the-app-key-to-infoplist)
* [3. Update the AppDelegate](#3-update-the-appdelegate)
* [4. Adopt `ICatapushEventDelegate`](#4-adopt-icatapusheventdelegate)
* [5. Replace the Notification Service Extension](#5-replace-the-notification-service-extension)
* [6. Update message send and receive code](#6-update-message-send-and-receive-code)
* [7. Update message storage queries](#7-update-message-storage-queries)
* [8. Renamed and new APIs to know about](#8-renamed-and-new-apis-to-know-about)
* [API rename quick reference](#api-rename-quick-reference)

## At a glance

| Area | What changes |
|---|---|
| Distribution | CocoaPods (`pod 'catapush-ios-sdk-pod'`) → Swift Package Manager (`https://github.com/Catapush/catapush-sdk-native`) |
| Language idiom | Objective-C-first static methods → Swift-first singleton (`Catapush.shared.*`) |
| App key configuration | `[Catapush setAppKey:@"…"]` (code) → `Info.plist` key `CatapushAppKey` |
| Credentials | `[Catapush setIdentifier:@"…" andPassword:@"…"]` → `Catapush.shared.setCredentials(identifier:password:)` |
| Initialization | `setupCatapushStateDelegate:andMessagesDispatcherDelegate:` + `registerUserNotification:` → `Catapush.shared.doInit(eventDelegate:mobileServices:)` (the method is named `doInit` on Swift to avoid clashing with Swift's `init` keyword) |
| Start | `[Catapush start:&error]` (NSError out-param) → `Catapush.shared.start(callback:)` (`RecoverableErrorCallback` with `success` / `warning` / `failure`) |
| Lifecycle hooks | `applicationDidEnterBackground:` / `applicationDidBecomeActive:` / `applicationWillEnterForeground:withError:` / `applicationWillTerminate:` were forwarded to `Catapush` manually — these are no longer required: the SDK handles foreground/background transitions internally |
| Delegates | `CatapushDelegate` + `MessagesDispatchDelegate` → unified `ICatapushEventDelegate` (mirrors the Android event callbacks) |
| Message class | `MessageIP` (CoreData entity) → `CatapushMessage` (plain Swift-friendly data class) |
| Read receipts | `[MessageIP sendMessageReadNotification:msg]` → `Catapush.shared.notifyMessageOpened(messageId:)` |
| Send messages | `sendMessageWithText:` and 11 overloads (`andChannel:`, `andImage:`, `andData:ofType:`, `replyTo:`) → unified `Catapush.shared.sendMessage(message:subject:channel:inReplyToMessageId:attachmentUri:callback:)` |
| Message storage | CoreData (`CatapushCoreData.managedObjectContext`, `MessageIP` `NSFetchedResultsController`) → internal database exposed as `Flow<PagingData<CatapushMessage>>` |
| Notification Service Extension | subclass `CatapushNotificationServiceExtension` and override `handleMessage:withContentHandler:withBestAttemptContent:` → subclass `UNNotificationServiceExtension` directly, init Catapush inside the extension, and consume `Catapush.shared.observeNextMessage(timeoutSeconds:onMessage:onTimeout:)` |
| Catapush push detection | `[Catapush isCatapushNotificationRequest:request]` → check `userInfo["sender"] as? String == "catapush"` in your extension |
| APNs token forwarding | manual via `application(_:didRegisterForRemoteNotificationsWithDeviceToken:)` → automatic via method swizzling; manual fallback `CatapushApns.shared.handleDeviceToken(_:)` / `handleDeviceTokenError(_:)` is still available |
| Logout | `[Catapush logoutStoredUser]` / `logoutStoredUser:withCompletion:` → `Catapush.shared.logout(callback:)` |
| Logging | `[Catapush enableLog:true]` / `[Catapush enableLog:false]` → `Catapush.shared.enableLog(logger:)` / `disableLog()` (the `enableLog` method now accepts an optional `ICatapushLogger` to forward log lines to your own backend) |

The rest of this guide walks through each step.

## 1. Replace CocoaPods with Swift Package Manager

Remove the previous pod from your `Podfile`:

```ruby
# Remove this line:
pod 'catapush-ios-sdk-pod'
```

Run `pod install` to drop the dependency from the project, then close Xcode and reopen the workspace.

Add the Catapush Multiplatform SDK as a Swift package via **File → Add Package Dependencies…** in Xcode and enter:

```
https://github.com/Catapush/catapush-sdk-native
```

Select version `1.0.0` (or pin to the exact tag you intend to use). Add the **`CatapushSdk`** product to your main app target. If you also use the optional UI module (Compose Multiplatform components rendered via SwiftUI bridges), add **`CatapushUi`** as well; otherwise leave it out — it is not required.

The SDK is shipped as a pre-built XCFramework, so no build-from-source step is required on your side.

If you prefer `Package.swift` over the Xcode UI:

```swift
dependencies: [
    .package(url: "https://github.com/Catapush/catapush-sdk-native", from: "1.0.0")
]
```

The previous "manual library integration when using `use_frameworks!`" workaround documented in the previous README is no longer needed — SPM handles framework-style linkage natively.

## 2. Move the app key to `Info.plist`

The `[Catapush setAppKey:@"YOUR_APP_KEY"]` call is removed. Add the key to your `Info.plist` instead:

```xml
<key>CatapushAppKey</key>
<string>YOUR_APP_KEY</string>
```

The **App Group** configuration is unchanged — you still need to enable the same App Group on the main app target and on the Notification Service Extension target so the database can be shared between them. Keep your existing `Info.plist` block:

```xml
<key>Catapush</key>
<dict>
    <key>AppGroup</key>
    <string>group.your.app.group</string>
</dict>
```

## 3. Update the AppDelegate

The previous AppDelegate had four lifecycle methods and two delegate protocols to wire up. The new pattern is a single `doInit(…)` call followed by `setCredentials(…)` and `start(…)`. The four `application…:` lifecycle forwards are no longer needed.

**Before** (Objective-C, previous pod):

```objc
@interface AppDelegate () <CatapushDelegate, MessagesDispatchDelegate, UNUserNotificationCenterDelegate>
@end

@implementation AppDelegate

- (BOOL)application:(UIApplication *)application
        didFinishLaunchingWithOptions:(NSDictionary *)launchOptions {
    [Catapush setAppKey:@"YOUR_APP_KEY"];
    [Catapush setIdentifier:@"user@example.com" andPassword:@"secret"];
    [Catapush setupCatapushStateDelegate:self andMessagesDispatcherDelegate:self];
    [Catapush registerUserNotification:self];
    NSError *error;
    [Catapush start:&error];
    if (error != nil) { /* … */ }
    [UNUserNotificationCenter currentNotificationCenter].delegate = self;
    return YES;
}

- (void)applicationDidEnterBackground:(UIApplication *)application {
    [Catapush applicationDidEnterBackground:application];
}
- (void)applicationDidBecomeActive:(UIApplication *)application {
    [Catapush applicationDidBecomeActive:application];
}
- (void)applicationWillEnterForeground:(UIApplication *)application {
    NSError *error;
    [Catapush applicationWillEnterForeground:application withError:&error];
}
- (void)applicationWillTerminate:(UIApplication *)application {
    [Catapush applicationWillTerminate:application];
}

#pragma mark CatapushDelegate / MessagesDispatchDelegate
- (void)catapushDidConnectSuccessfully:(Catapush *)catapush { /* … */ }
- (void)catapush:(Catapush *)catapush didFailOperation:(NSString *)op
                                            withError:(NSError *)error { /* … */ }
- (void)libraryDidReceiveMessageIP:(MessageIP *)messageIP { /* … */ }
- (void)libraryDidSendMessage:(MessageIP *)messageIP { /* … */ }
- (void)libraryDidFailToSendMessage:(MessageIP *)messageIP { /* … */ }

@end
```

**After** (Swift, Multiplatform SDK):

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

        // 2. Set the user credentials
        Catapush.shared.setCredentials(identifier: "user@example.com", password: "secret")

        // 3. Start the messaging service
        Catapush.shared.start(callback: MyRecoverableErrorCallback())

        return true
    }

    // APNs token forwarding is handled automatically through method swizzling.
    // Keep these only if you have disabled swizzling (e.g. FirebaseAppDelegateProxyEnabled = NO).
    func application(_ application: UIApplication,
                     didRegisterForRemoteNotificationsWithDeviceToken deviceToken: Data) {
        CatapushApns.shared.handleDeviceToken(deviceToken)
    }

    func application(_ application: UIApplication,
                     didFailToRegisterForRemoteNotificationsWithError error: Error) {
        CatapushApns.shared.handleDeviceTokenError(error as NSError)
    }

    func application(_ application: UIApplication,
                     didReceiveRemoteNotification userInfo: [AnyHashable: Any],
                     fetchCompletionHandler completionHandler:
                         @escaping (UIBackgroundFetchResult) -> Void) {
        CatapushApns.shared.handlePushNotification(userInfo: userInfo)
        completionHandler(.newData)
    }
}
```

Notes:

* The four lifecycle forwards (`applicationDidEnterBackground:` etc.) are gone. The SDK observes app state changes internally via `UIApplication` notifications.
* `[Catapush registerUserNotification:self]` is no longer needed. Request notification authorization yourself (typically once at app start) using `UNUserNotificationCenter.current().requestAuthorization(…)`.
* The two delegate protocols collapse into a single `ICatapushEventDelegate` — see the next section.
* APNs token forwarding is automatic via method swizzling. Keep the `didRegisterForRemoteNotificationsWithDeviceToken` / `didFailToRegisterForRemoteNotificationsWithError` overrides only if you have disabled swizzling for other reasons.

## 4. Adopt `ICatapushEventDelegate`

The previous `CatapushDelegate` (connection state) and `MessagesDispatchDelegate` (incoming/outgoing messages) protocols are unified into a single `ICatapushEventDelegate` protocol that mirrors the one used on Android. The callback names follow the Android conventions (`onConnected`, `onMessageReceived`, …).

```swift
import CatapushSdk

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

Mapping from the previous protocols:

| Legacy (Obj-C) | Multiplatform 1.0.0 (Swift) |
|---|---|
| `catapushDidConnectSuccessfully:` | `onConnected()` |
| `catapush:didFailOperation:withError:` (with `WRONG_AUTHENTICATION`) | `onRegistrationFailed(error:)` |
| `catapush:didFailOperation:withError:` (with connection errors) | `onDisconnected(error:)` |
| (not exposed as a delegate callback in the previous pod) | `onConnecting()` |
| `libraryDidReceiveMessageIP:` | `onMessageReceived(message:)` (and `onMessageReceivedConfirmed(message:)` after the server ack) |
| `libraryDidSendMessage:` | `onMessageSent(message:)` (and `onMessageSentConfirmed(message:)` after the server ack) |
| `libraryDidFailToSendMessage:` | the failure surfaces through the `Callback<Boolean>` passed to `sendMessage(…)` |

The `RecoverableErrorCallback` and `Callback` protocols passed to `start(…)` and `stop(…)` follow the same shape as Android: `success`, `warning` (recoverable, with retry expected), and `failure` (irrecoverable).

## 5. Replace the Notification Service Extension

The previous SDK exposed a base class `CatapushNotificationServiceExtension` that you subclassed and overrode `handleMessage:` / `handleError:`. The new SDK does **not** ship a base extension class; instead, you implement a vanilla `UNNotificationServiceExtension`, initialize the SDK inside it, and observe the next incoming message via `Catapush.shared.observeNextMessage(timeoutSeconds:onMessage:onTimeout:)`.

The deployment-target rule still applies: **the extension target's deployment target must match the main app's deployment target**, and both must enable the same App Group.

**Before** (previous pod, Obj-C):

```objc
@interface NotificationService : CatapushNotificationServiceExtension
@end

@implementation NotificationService

- (void)handleMessage:(MessageIP * _Nullable)message
   withContentHandler:(void (^_Nullable)(UNNotificationContent * _Nullable))contentHandler
withBestAttemptContent:(UNMutableNotificationContent * _Nullable)bestAttemptContent {
    if (contentHandler != nil && bestAttemptContent != nil) {
        bestAttemptContent.body = message != nil ? message.body.copy : @"No new message";
        contentHandler(bestAttemptContent);
    }
}

- (void)handleError:(NSError * _Nonnull)error
   withContentHandler:(void (^_Nullable)(UNNotificationContent * _Nullable))contentHandler
withBestAttemptContent:(UNMutableNotificationContent * _Nullable)bestAttemptContent {
    if (contentHandler != nil && bestAttemptContent != nil) {
        if (error.code == CatapushCredentialsError) bestAttemptContent.body = @"User not logged in";
        if (error.code == CatapushNetworkError)     bestAttemptContent.body = @"Network error";
        if (error.code == CatapushNoMessagesError)  bestAttemptContent.body = @"No new message";
        contentHandler(bestAttemptContent);
    }
}

@end
```

**After** (Multiplatform SDK, Swift):

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
            contentHandler(request.content) // not ours, deliver as-is
            return
        }

        // Initialize Catapush in the extension; credentials are loaded from the shared
        // App Group database written by the main app.
        Catapush.shared.doInit(
            eventDelegate: ExtensionEventDelegate(),
            mobileServices: [CatapushApns.shared]
        )

        // Wait up to 5 seconds for the actual message to arrive, then deliver the notification.
        Catapush.shared.observeNextMessage(
            timeoutSeconds: 5,
            onMessage: { [weak self] title, body in
                guard let self = self, let best = self.bestAttemptContent else {
                    contentHandler(request.content)
                    return
                }
                best.title = title ?? "Catapush"
                if let body = body { best.body = body }
                contentHandler(best)
            },
            onTimeout: { [weak self] in
                guard let self = self, let best = self.bestAttemptContent else {
                    contentHandler(request.content)
                    return
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
```

The previous error-code branch (`CatapushCredentialsError`, `CatapushNetworkError`, `CatapushNoMessagesError`, `CatapushFileProtectionError`, `CatapushConflictErrorCode`, `CatapushAppIsActive`) is no longer surfaced in the extension. The new pattern reduces it to a single timeout fallback inside `observeNextMessage(…)`. If you previously rendered different fallback bodies depending on the error code, replace them with a single localized "No new message" string in the `onTimeout` branch.

## 6. Update message send and receive code

The 11 previous `sendMessageWithText:…` overloads collapse into a single function:

```swift
Catapush.shared.sendMessage(
    message: "Hello",
    subject: nil,                  // optional
    channel: nil,                  // optional
    inReplyToMessageId: nil,       // optional
    attachmentUri: nil,            // optional — pass a local file URL string for image/data attachments
    callback: MyCallback()
)
```

The `andImage:` / `andData:ofType:` overloads are replaced by writing the image or data to a temporary file inside the app sandbox and passing the resulting file URL via `attachmentUri`. The MIME type is inferred from the file extension; if you need to control it explicitly, write the file with the correct extension (`.jpg`, `.pdf`, …).

Marking a message as read changes class:

| Legacy | Multiplatform 1.0.0 |
|---|---|
| `[MessageIP sendMessageReadNotification:msg]` | `Catapush.shared.notifyMessageOpened(messageId: msg.id)` |

The previous `MessageIP` properties (`body`, `sentTime`, `status`, `originalMessageId`, …) map onto `CatapushMessage` properties with mostly the same names. The `MESSAGEIP_STATUS` enum (`MessageIP_NOT_READ`, `MessageIP_READ`, `MessageIP_SENT`, `MessageIP_ERROR`, …) is replaced by `MessageState` on `CatapushMessage`.

## 7. Update message storage queries

The previous SDK exposed messages through CoreData (`CatapushCoreData.managedObjectContext`, `MessageIP` `NSFetchRequest`, `NSFetchedResultsController` for change tracking). The Multiplatform SDK exposes the local message store through `kotlinx.coroutines` `Flow`s, which Swift consumes as standard `AsyncSequence`s.

**Before** (previous pod, retrieve all messages):

```swift
let messages = Catapush.allMessages() // [MessageIP]
```

**After** (Multiplatform SDK, paged Flow):

```swift
// Create a CoroutineScope from Swift via the helper exposed by the SDK:
let scope = CreateMainScopeKt.createMainScope()

let flow = Catapush.shared.getMessages(
    cachedInScope: scope,
    pageSize: 20,
    channel: nil
)

// Consume it as a standard AsyncSequence:
Task {
    for await pagingData in flow {
        // hand `pagingData` to your list adapter
    }
}
```

Live counts (badge updates, unread indicators) are exposed as `Flow<Int64>`:

```swift
Task {
    for await unreadCount in Catapush.shared.countUnreadMessages(channel: nil) {
        await MainActor.run {
            UIApplication.shared.applicationIconBadgeNumber = Int(unreadCount)
        }
    }
}
```

The CoreData-based `NSFetchedResultsController` pattern from the previous `MessageIP` / `NSPredicate` approach is replaced by collecting the corresponding `Flow`. There is no longer an exported managed object model.

## 8. Renamed and new APIs to know about

**Renamed (semantics preserved):**

* `[Catapush logoutStoredUser]` and `logoutStoredUser:withCompletion:` → `Catapush.shared.logout(callback:)`.
* `[MessageIP sendMessageReadNotification:msg]` → `Catapush.shared.notifyMessageOpened(messageId: msg.id)`.
* `[Catapush enableLog:true]` / `enableLog:false]` → `Catapush.shared.enableLog(logger:)` / `disableLog()`.
* `[Catapush isCatapushNotificationRequest:request]` → no built-in helper; check `userInfo["sender"] as? String == "catapush"` (see [step 5](#5-replace-the-notification-service-extension)).

**New in 1.0.0:**

* `Catapush.shared.setMessageTtlDays(days:)` / `getMessageTtlDays()` — automatic deletion of messages older than `N` days. Defaults to 90; pass `0` to disable.
* `Catapush.shared.setMessageTransformation(messageTransformation:)` — installs a hook that transforms every incoming message before it is persisted (useful for end-to-end decryption or placeholder substitution).
* `Catapush.shared.clearDataAndCredentials()` — wipes the local message database, attachments and stored credentials in one call.
* `Catapush.shared.deleteMessage(messageId:)` / `deleteMessages()` / `deleteMessagesInChannel(channel:)` — programmatic message removal.
* `CatapushApns.shared.handleVoipToken(voipToken:)` — forward VoIP push tokens from your `PKPushRegistryDelegate` (only relevant if you ship a PushKit/VoIP push integration).

**Removed:**

* The four `applicationDidEnterBackground:` / `applicationDidBecomeActive:` / `applicationWillEnterForeground:withError:` / `applicationWillTerminate:` lifecycle forwards — handled internally; remove them from your AppDelegate.
* `[Catapush registerUserNotification:self]` — request notification authorization yourself with `UNUserNotificationCenter.current().requestAuthorization(…)`.
* `CatapushNotificationServiceExtension` base class — replaced by the `observeNextMessage(…)` pattern shown in [step 5](#5-replace-the-notification-service-extension).
* CoreData accessors (`CatapushCoreData.managedObjectContext`, the `MessageIP` entity, the `MESSAGEIP_STATUS` enum) — replaced by `Flow`-based queries on `CatapushMessage`.
* `[Catapush allMessages]` — replaced by `Catapush.shared.getMessages(cachedInScope:pageSize:channel:)`.
* The `+sendMessageWithText:` family of 11 overloads — replaced by the single `sendMessage(message:subject:channel:inReplyToMessageId:attachmentUri:callback:)`.
* The previous "Manual library integration when using `use_frameworks!`" workaround — Swift Package Manager handles framework-style linkage natively.

## API rename quick reference

A flat list of the public-API renames you will encounter while porting an existing pod-based integration. Things not listed here either keep the same name and meaning, or have been removed (see the section above).

| Legacy pod | Multiplatform 1.0.0 |
|---|---|
| `pod 'catapush-ios-sdk-pod'` | SwiftPM dependency on `https://github.com/Catapush/catapush-sdk-native` |
| `[Catapush setAppKey:@"…"]` | `Info.plist` key `CatapushAppKey` |
| `[Catapush setIdentifier:@"…" andPassword:@"…"]` | `Catapush.shared.setCredentials(identifier:password:)` |
| `[Catapush setupCatapushStateDelegate:s andMessagesDispatcherDelegate:m]` + `[Catapush registerUserNotification:s]` | `Catapush.shared.doInit(eventDelegate: …, mobileServices: [CatapushApns.shared])` + `UNUserNotificationCenter.current().requestAuthorization(…)` |
| `NSError *err; [Catapush start:&err];` | `Catapush.shared.start(callback: RecoverableErrorCallback)` |
| `applicationDidEnterBackground:` / `applicationDidBecomeActive:` / `applicationWillEnterForeground:withError:` / `applicationWillTerminate:` forwards | (removed — handled internally) |
| `CatapushDelegate` + `MessagesDispatchDelegate` | `ICatapushEventDelegate` |
| `MessageIP` | `CatapushMessage` |
| `MESSAGEIP_STATUS` | `MessageState` |
| `+sendMessageWithText:` (11 overloads) | `Catapush.shared.sendMessage(message:subject:channel:inReplyToMessageId:attachmentUri:callback:)` |
| `[MessageIP sendMessageReadNotification:msg]` | `Catapush.shared.notifyMessageOpened(messageId: msg.id)` |
| `[Catapush allMessages]` | `Catapush.shared.getMessages(cachedInScope:pageSize:channel:)` returning `Flow<PagingData<CatapushMessage>>` |
| `CatapushCoreData.managedObjectContext` / `MessageIP` `NSFetchedResultsController` | `Flow<PagingData<CatapushMessage>>` and `Flow<Int64>` count helpers |
| `[Catapush logoutStoredUser]` / `logoutStoredUser:withCompletion:` | `Catapush.shared.logout(callback:)` |
| `[Catapush isCatapushNotificationRequest:req]` | check `userInfo["sender"] as? String == "catapush"` |
| `CatapushNotificationServiceExtension` (subclass + `handleMessage:` / `handleError:`) | plain `UNNotificationServiceExtension` + `Catapush.shared.observeNextMessage(timeoutSeconds:onMessage:onTimeout:)` |
| `[Catapush enableLog:true]` / `[Catapush enableLog:false]` | `Catapush.shared.enableLog(logger:)` / `disableLog()` |

For the full up-to-date public API surface, see the [Multiplatform SDK iOS guide](DOCUMENTATION_MULTIPLATFORM_SDK_IOS.md).
