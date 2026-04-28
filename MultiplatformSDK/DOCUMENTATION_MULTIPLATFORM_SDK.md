![Catapush Logo](../AndroidSDK/images/catapush_logo.png)

# Catapush Multiplatform SDK

The **Catapush Multiplatform SDK** is the unified messaging SDK for Android and iOS. It exposes the same API surface on both platforms — idiomatic Kotlin/Java on Android, idiomatic Swift on iOS — so most of your integration code is portable between the two.

This release supersedes the previous platform-specific SDKs (`catapush-android-sdk` and `catapush-ios-sdk-pod`), which remain available for existing integrations.

## Where to start

| You are… | Start here |
|---|---|
| Integrating Catapush into a brand-new Android app | [Android integration guide](DOCUMENTATION_MULTIPLATFORM_SDK_ANDROID.md) |
| Integrating Catapush into a brand-new iOS app | [iOS integration guide](DOCUMENTATION_MULTIPLATFORM_SDK_IOS.md) |
| Migrating an Android app from `catapush-android-sdk` 16.0.x | [Android migration guide](MIGRATION_FROM_ANDROID_SDK.md) |
| Migrating an iOS app from `catapush-ios-sdk-pod` | [iOS migration guide](MIGRATION_FROM_IOS_SDK.md) |
| Migrating an Android app from a version older than 16.0.x | First follow the [previous migration steps](../AndroidSDK/DOCUMENTATION_ANDROID_SDK.md#migration-from-catapush-150x) up to 16.0.x, then continue with the [Android migration guide](MIGRATION_FROM_ANDROID_SDK.md) |
| Configuring Firebase / Huawei AppGallery dashboards | [FCM configuration guide](DOCUMENTATION_PLATFORM_GMS_FCM.md) · [HMS Push Kit configuration guide](DOCUMENTATION_PLATFORM_HMS_PUSHKIT.md) |
| Reviewing what changed in this release | [Changelog](CHANGELOG_MULTIPLATFORM_SDK.md) |

## Modules at a glance

The SDK is split into a small number of artifacts so you only ship what you actually need. Most apps depend on the **core** plus one **push-platform** module; the **UI** module is fully optional.

| Module | Platform | Required? | Purpose |
|---|---|---|---|
| `com.catapush.sdk:core` | Android, iOS | required | Authentication, messaging, local storage, push token registration. |
| `com.catapush.sdk.android:gms` | Android | required for FCM (Google) devices | Firebase Cloud Messaging integration. |
| `com.catapush.sdk.android:hms` | Android | required for HMS (Huawei) devices, when you don't have your own `HmsMessageService` | Huawei Push Kit integration with a built-in `HmsMessageService`. |
| `com.catapush.sdk.android:hms-base` | Android | required for HMS (Huawei) devices, when you already have your own `HmsMessageService` | Huawei Push Kit integration that lets you forward wakeups from your existing service. |
| APNs adapter (`CatapushApns.shared`) | iOS | always available | APNs integration; included with the iOS XCFramework, no extra dependency. |
| `com.catapush.sdk:ui` (Android) / `CatapushUi` (iOS) | Android, iOS | optional | Ready-to-use chat list, conversation views and message composer. |

Push-platform modules are thin: they handle token retrieval and wakeup forwarding only. All the messaging logic lives in the core module.

## Requirements

| | Android | iOS |
|---|---|---|
| Minimum OS | Android 7.0 (API 24) | iOS 15.0 |
| Build target | `compileSdk` 36, `targetSdk` 36 | Xcode 15+ |
| Toolchain | JDK 21 | — |
| Distribution | Maven (S3) | Swift Package Manager (XCFramework) |
| Push platform | Google Play Services (FCM) and/or Huawei Mobile Services (Push Kit) | APNs (token-based authentication key) |

## Feature parity

Most of the public API is shared verbatim between the two platforms. The table below lists what differs.

| Feature | Android | iOS |
|---|---|---|
| `Catapush` singleton | `Catapush.…` (Kotlin `object`) | `Catapush.shared.…` |
| Initialization method | `Catapush.init(context, eventDelegate, mobileServices, intentProvider, mainTpl, otherTpls)` | `Catapush.shared.doInit(eventDelegate:, mobileServices:)` |
| App key configuration | `<meta-data>` in `AndroidManifest.xml` | `Info.plist` key `CatapushAppKey` |
| Push platforms | GMS (FCM) and/or HMS (Push Kit) | APNs |
| Notification rendering | `NotificationTemplate` per Android notification channel + `ICatapushNotificationIntentProvider` | system-level (UNUserNotificationCenter) + Notification Service Extension |
| Notification channels | required since Android 8.0 (one `NotificationTemplate` per channel) | not applicable (iOS does not have channels) |
| Notification Service Extension | not applicable (notifications rendered in-process) | required for production apps |
| Optional UI module | `com.catapush.sdk:ui` (Compose) | `CatapushUi` (Compose, hosted in SwiftUI via `UIViewControllerRepresentable`) |
| Background restrictions | `SystemConfigurationException` warnings on `start(…)` (background data, execution restrictions, app standby buckets) | not applicable |

Everything else — `setCredentials(…)`, `start(…)`, `stop(…)`, `logout(…)`, `clearDataAndCredentials()`, `sendMessage(…)`, `notifyMessageOpened(…)`, `getMessages(…)`, `getChannelList()`, `countMessages()` / `countMessagesInChannel(…)` / `countUnreadMessages(…)`, `setMessageTtlDays(…)`, `setMessageTransformation(…)`, `enableLog(…)` / `disableLog()`, `pauseNotifications()` / `resumeNotifications()`, `enableNotifications()` / `disableNotifications()`, `updateNotificationTemplate(…)`, the `ICatapushEventDelegate` callbacks — has the same name and meaning on both platforms.

## Document index

* [Android integration guide](DOCUMENTATION_MULTIPLATFORM_SDK_ANDROID.md) — full Android setup walkthrough.
* [iOS integration guide](DOCUMENTATION_MULTIPLATFORM_SDK_IOS.md) — full iOS setup walkthrough.
* [Android migration guide](MIGRATION_FROM_ANDROID_SDK.md) — from `catapush-android-sdk` 16.0.x.
* [iOS migration guide](MIGRATION_FROM_IOS_SDK.md) — from `catapush-ios-sdk-pod`.
* [FCM configuration guide](DOCUMENTATION_PLATFORM_GMS_FCM.md) — Firebase project and Catapush dashboard setup for Android.
* [HMS Push Kit configuration guide](DOCUMENTATION_PLATFORM_HMS_PUSHKIT.md) — AppGallery Connect project and Catapush dashboard setup for Huawei devices.
* [Changelog](CHANGELOG_MULTIPLATFORM_SDK.md) — release notes.

## Sample apps

The SDK repository includes a working sample on each platform:

* Android — [`androidApp/`](https://github.com/Catapush/catapush-sdk-native/tree/main/androidApp) — uses `:core`, `:gms` and the optional `:ui` module.
* iOS — [`iosApp/`](https://github.com/Catapush/catapush-sdk-native/tree/main/iosApp) — main app target plus a Notification Service Extension target.
