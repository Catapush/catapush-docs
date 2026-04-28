![Catapush Logo](../AndroidSDK/images/catapush_logo.png)

# Catapush Multiplatform SDK Changelog

## Catapush Multiplatform SDK 1.0.0

The Catapush Multiplatform SDK 1.0.0 is the first public release of a unified SDK for Android and iOS. It is the successor to `catapush-android-sdk` (Android) and `catapush-ios-sdk-pod` (iOS), which remain available for existing integrations.

If you are coming from a previous SDK, follow the dedicated migration guide for your platform:

* [Migrating from Catapush Android SDK 16.0.x](MIGRATION_FROM_ANDROID_SDK.md)
* [Migrating from `catapush-ios-sdk-pod`](MIGRATION_FROM_IOS_SDK.md)

### 1.0.0 (Pre-release)

This release is currently in **alpha**. The latest published artifact is `1.0.0-alpha5`. APIs are stable enough to integrate against, but minor adjustments are still possible until the GA release.

#### Highlights

* Single shared API surface across Android and iOS.
* Modern, Swift-friendly iOS API (single `ICatapushEventDelegate`, no Objective-C lifecycle forwards, no CoreData entities to track).
* Coroutines `Flow`-based APIs for messages, channels and counts on both platforms.
* Android: same `Catapush` singleton model, with a Kotlin `object` access pattern (`Catapush.…`) instead of `Catapush.getInstance().…`.
* iOS: distributed as a pre-built XCFramework via Swift Package Manager (no more CocoaPods, no more `use_frameworks!` workaround).
* New optional Compose Multiplatform UI module (`com.catapush.sdk:ui` on Android, `CatapushUi` on iOS) ships ready-to-use chat list and conversation views; integration without it is fully supported.
* Automatic message deletion (TTL): messages older than a configurable number of days are removed at every successful start. Default 90 days, configurable via `setMessageTtlDays(…)`.
* Logging delegate: `enableLog(logger)` accepts an optional `ICatapushLogger` to forward SDK log lines to your own backend (Crashlytics, Sentry, OSLog, internal log file, …).

#### Android

* New Maven coordinates:
    * `com.catapush.sdk:core` — core module.
    * `com.catapush.sdk:ui` — optional UI module.
    * `com.catapush.sdk.android:gms` — Firebase Cloud Messaging integration.
    * `com.catapush.sdk.android:hms` — Huawei Push Kit integration with the SDK's own `HmsMessageService`.
    * `com.catapush.sdk.android:hms-base` — Huawei Push Kit integration for apps that already declare their own `HmsMessageService`.
* Maven repository unchanged: `https://s3.eu-west-1.amazonaws.com/m2repository.catapush.com/`.
* Minimum SDK level raised from 23 to **24** (Android 7.0).
* `compileSdk` 36 (Android 16); JDK 21 toolchain.
* `Catapush.init(…)` is now synchronous and takes 6 parameters (no trailing `Callback`); `ICatapushInitializer` moved out of the init signature and is registered separately via `Catapush.setInitializer(…)`.
* `setUser(id, password)` renamed to `setCredentials(id, password)`.
* The `KEY_CATAPUSH_WAKEUP` constant moved from `Catapush.KEY_CATAPUSH_WAKEUP` to `ICatapush.KEY_CATAPUSH_WAKEUP`.
* The previous callback-based and RxJava-based message-query overloads collapse into a single `getMessages(scope, pageSize, channel)` returning `Flow<PagingData<CatapushMessage>>`.
* `clean(callback)` renamed to `clearDataAndCredentials()`; `clearMessages(callback)` renamed to `deleteMessages()`; both are now synchronous.
* `sendFile(…)` overloads replaced by an `attachmentUri` parameter on the unified `sendMessage(…)`.
* The deprecated `setAppKey(…)` / `setSenderId(…)` / `enable*ModalNotification()` / `enable*StatusBarNotification()` methods, the `getIdentifier(…)` / `getPassword(…)` / `getMessageById(…)` / `getMessagesAsList(…)` accessors, and `rebuildSecureCredentialsStore(…)` have been removed.

#### iOS

* Distribution moved from CocoaPods (`pod 'catapush-ios-sdk-pod'`) to Swift Package Manager (XCFramework binary target).
* App key configured via `Info.plist` (`CatapushAppKey` key) instead of `[Catapush setAppKey:…]`.
* Deployment target: iOS 15.0+.
* Single unified `ICatapushEventDelegate` replaces the previous `CatapushDelegate` and `MessagesDispatchDelegate` protocols.
* The four lifecycle forwards (`applicationDidEnterBackground:`, `applicationDidBecomeActive:`, `applicationWillEnterForeground:withError:`, `applicationWillTerminate:`) are no longer required: app state changes are observed internally.
* Automatic APNs token interception via method swizzling. Manual fallback (`CatapushApns.shared.handleDeviceToken(_:)` / `handleDeviceTokenError(_:)`) remains available.
* Notification Service Extension pattern simplified: subclass `UNNotificationServiceExtension` directly, initialize Catapush in the extension and consume `Catapush.shared.observeNextMessage(timeoutSeconds:onMessage:onTimeout:)` instead of subclassing `CatapushNotificationServiceExtension`.
* `MessageIP` (CoreData entity) replaced by `CatapushMessage` (plain Swift-friendly data class). The `CatapushCoreData` accessors and the exported managed-object model are removed.
* The 11 `+sendMessageWithText:…` overloads collapse into a single `sendMessage(message:subject:channel:inReplyToMessageId:attachmentUri:callback:)`.
* VoIP push token forwarding via `CatapushApns.shared.handleVoipToken(voipToken:)`.

The full alpha-to-GA changes will be consolidated under a `1.0.0` heading once the GA release is tagged.
