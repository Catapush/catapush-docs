![Catapush Logo](../AndroidSDK/images/catapush_logo.png)

# Migrating from Catapush Android SDK 16.0.x to Catapush Multiplatform SDK 1.0.0

The Catapush Multiplatform SDK is the successor to the previous `catapush-android-sdk` (and to the previous `catapush-ios-sdk-pod` on iOS). It is built with Kotlin Multiplatform so a single codebase powers both Android and iOS.

This guide covers the migration **from version 16.0.x of the previous Android SDK** to version 1.0.0 of the Multiplatform SDK. The conceptual model is unchanged — the same `Catapush` singleton, the same `ICatapushEventDelegate`, the same `NotificationTemplate`, the same GMS/HMS modules — but the artifact coordinates and a handful of method signatures have changed and require source-code updates. There is **no Maven artifact relocation**, so simply bumping the version is not enough.

If you are still on a version older than 16.0.x, first follow the [previous migration guides](../AndroidSDK/DOCUMENTATION_ANDROID_SDK.md#migration-from-catapush-150x) to bring your integration up to 16.0.x, then come back here.

## Index

* [At a glance](#at-a-glance)
* [1. Update project prerequisites](#1-update-project-prerequisites)
* [2. Confirm the Maven repository declaration](#2-confirm-the-maven-repository-declaration)
* [3. Replace the Catapush dependencies](#3-replace-the-catapush-dependencies)
* [4. Update your `Application.onCreate()`](#4-update-your-applicationoncreate)
* [5. Update calls to `start()` and credential management](#5-update-calls-to-start-and-credential-management)
* [6. Update message queries to the new Paging 3 API](#6-update-message-queries-to-the-new-paging-3-api)
* [7. Renamed and new APIs to know about](#7-renamed-and-new-apis-to-know-about)
* [API rename quick reference](#api-rename-quick-reference)

## At a glance

The migration consists of one build-time change (artifact coordinates) and a small number of source-code changes:

| Area | What changes |
|---|---|
| `minSdk` | `23` → `24` |
| Maven artifact group | `com.catapush.catapush-android-sdk:*` → `com.catapush.sdk:*` and `com.catapush.sdk.android:*` |
| `Catapush` access | `Catapush.getInstance().…` → `Catapush.…` (Kotlin singleton object) |
| Credentials method | `setUser(id, pwd)` → `setCredentials(id, pwd)` |
| `init(…)` signature | drops the `ICatapushInitializer` and the trailing `Callback<Boolean>` parameters; init is now synchronous. The `ICatapushInitializer` is registered via the new `Catapush.setInitializer(…)` method when needed |
| Message queries | callback-based / RxJava `Flowable` queries are replaced by `kotlinx.coroutines` `Flow`s. `getMessagesWithoutChannelAsList(…)` / `getMessagesFromChannelAsList(…)` / `getMessagesWithoutChannelAsPagingDataFlowable(…)` / `getMessagesFromChannelAsPagingDataFlowable(…)` → unified `getMessages(scope, pageSize, channel)` returning `Flow<PagingData<CatapushMessage>>` |
| Counts and channel list | callback-based `getChannelList(callback)`, `countMessages(callback)`, `countUnreadMessages(callback)` → `Flow`-returning equivalents (`getChannelList()`, `countMessages()`, `countUnreadMessages(channel?)`) |
| Send and attachments | the dedicated `sendFile(…)` overloads are removed; attachments are now passed via the `attachmentUri` parameter of `sendMessage(…)` |
| Wakeup data key | `Catapush.KEY_CATAPUSH_WAKEUP` → `ICatapush.KEY_CATAPUSH_WAKEUP` (relevant only if you have a custom `FirebaseMessagingService` or `HmsMessageService` that forwards wakeups to Catapush) |

The rest of this guide walks through each step.

## 1. Update project prerequisites

The Multiplatform SDK requires:

* `minSdk` **24** (Android 7.0). If your app still targets API 23, raise `minSdk` to 24 before adopting the new SDK.
* `compileSdk` 36 (Android 16) — same as 16.0.x.
* JDK **21** for the build toolchain.

```groovy
android {
    compileSdk 36
    defaultConfig {
        minSdk 24
        targetSdk 36
    }
    compileOptions {
        sourceCompatibility JavaVersion.VERSION_21
        targetCompatibility JavaVersion.VERSION_21
    }
}
```

If you are still using Kotlin in your app, make sure you are on Kotlin 2.0 or later.

## 2. Confirm the Maven repository declaration

The Catapush Maven repository is the same as 16.0.x — the public, read-only S3 bucket. No change is required if your project already declares it. If you need to add it (for example because you are wiring up a fresh module), declare it either inside the consuming module's `build.gradle` or at the root level via `allprojects { repositories { … } }`:

```groovy
repositories {
    maven { url "https://s3.eu-west-1.amazonaws.com/m2repository.catapush.com/" }
}
```

No credentials are required.

## 3. Replace the Catapush dependencies

Replace the previous `com.catapush.catapush-android-sdk:*` artifacts with the new ones. The old dependency lines:

```groovy
dependencies {
    implementation 'com.catapush.catapush-android-sdk:core:16.0.0'
    implementation 'com.catapush.catapush-android-sdk:gms:16.0.0'
    // and/or:
    implementation 'com.catapush.catapush-android-sdk:hms:16.0.0'
    // or, if you keep your own HmsMessageService:
    // implementation 'com.catapush.catapush-android-sdk:hms-base:16.0.0'
}
```

become:

```groovy
dependencies {
    implementation 'com.catapush.sdk:core:1.0.0'
    implementation 'com.catapush.sdk.android:gms:1.0.0'
    // and/or:
    implementation 'com.catapush.sdk.android:hms:1.0.0'
    // or, if you keep your own HmsMessageService:
    // implementation 'com.catapush.sdk.android:hms-base:1.0.0'

    // Optional: ready-to-use Compose Multiplatform UI components
    // implementation 'com.catapush.sdk:ui:1.0.0'
}
```

Notes on artifact naming:

* The pure KMP modules (`core`, `ui`) live under the **`com.catapush.sdk`** group.
* Android-only platform modules (`gms`, `hms`, `hms-base`) live under **`com.catapush.sdk.android`**.
* The HMS module split is the same as in 16.0.x, just under the new group. Pick **`:hms`** when you do not have a `HmsMessageService` of your own (the SDK ships its own implementation), or **`:hms-base`** when you keep your existing `HmsMessageService` and forward push wakeups via `CatapushHms.handleHmsWakeup(…)` and `CatapushHms.handleHmsNewToken(…)`. The HMS Push Kit only allows one messaging service per app, which is why the split exists.

The corresponding Google/Huawei plugin and `google-services.json` / `agconnect-services.json` configuration is unchanged from 16.0.x — see the [GMS configuration guide](DOCUMENTATION_PLATFORM_GMS_FCM.md) and the [HMS configuration guide](DOCUMENTATION_PLATFORM_HMS_PUSHKIT.md).

## 4. Update your `Application.onCreate()`

The `Catapush` singleton is now a Kotlin `object`, so all the `Catapush.getInstance()` calls become direct calls on `Catapush`. The `init(…)` method has also been simplified: it no longer takes the `ICatapushInitializer` as the second parameter (use the new `Catapush.setInitializer(…)` method instead, when you actually need it), and it is now **synchronous** so the trailing `Callback<Boolean>` is gone.

**Before** (16.0.x):

```kotlin
Catapush.getInstance().init(
    /* context                              */ this,
    /* initializer                          */ this,
    /* eventDelegate                        */ eventDelegate,
    /* mobileServices                       */ listOf(CatapushGms.INSTANCE),
    /* notificationIntentProvider           */ { message, context ->
        val intent = Intent(context, MainActivity::class.java).apply {
            data = Uri.parse("catapush://message/${message.id()}")
            putExtra("message", message)
            flags = Intent.FLAG_ACTIVITY_NEW_TASK or Intent.FLAG_ACTIVITY_SINGLE_TOP
        }
        PendingIntent.getActivity(
            context, 0, intent,
            PendingIntent.FLAG_ONE_SHOT or PendingIntent.FLAG_IMMUTABLE
        )
    },
    /* mainNotificationChannelTemplate      */ notificationTemplate,
    /* otherNotificationChannelsTemplates   */ null,
    /* callback                             */ object : Callback<Boolean> {
        override fun success(result: Boolean) { /* … */ }
        override fun failure(t: Throwable) { /* … */ }
    },
)
```

**After** (Multiplatform 1.0.0):

```kotlin
val intentProvider = ICatapushNotificationIntentProvider { message, context ->
    val intent = Intent(context, MainActivity::class.java).apply {
        data = Uri.parse("catapush://message/${message.id()}")
        putExtra("message", message)
        flags = Intent.FLAG_ACTIVITY_NEW_TASK or Intent.FLAG_ACTIVITY_SINGLE_TOP
    }
    PendingIntent.getActivity(
        context, 0, intent,
        PendingIntent.FLAG_ONE_SHOT or PendingIntent.FLAG_IMMUTABLE
    )
}

Catapush.init(
    context = this,
    eventDelegate = eventDelegate,
    mobileServices = listOf(CatapushGms),
    notificationIntentProvider = intentProvider,
    mainNotificationChannelTemplate = notificationTemplate,
    otherNotificationChannelsTemplates = null,
)
```

If you used the `ICatapushInitializer` pattern in 16.0.x (passing your `Application` as the second parameter of `init(…)`), register it via the new dedicated method instead:

```kotlin
Catapush.setInitializer(this) // 'this' implements ICatapushInitializer
```

The mobile services constants are now plain Kotlin `object`s rather than Java-style singletons, so:

* `CatapushGms.INSTANCE` becomes `CatapushGms`.
* `CatapushHms.INSTANCE` becomes `CatapushHms`.

The Multiplatform SDK also exposes a separate `Catapush.setNotificationIntentProvider(provider)` method that you can call **after** `init(…)` to replace the intent provider at runtime — useful when the destination `Activity` depends on app state that is not known at startup.

## 5. Update calls to `start()` and credential management

`setUser(identifier, password)` is renamed to `setCredentials(identifier, password)`. The semantics are identical — the credentials are persisted in the secure store and reused on subsequent app launches.

**Before:**

```kotlin
Catapush.getInstance()
    .setUser(CATAPUSH_USER, CATAPUSH_PASSWORD)
    .start(object : RecoverableErrorCallback<Boolean> { /* … */ })
```

**After:**

```kotlin
Catapush
    .setCredentials(CATAPUSH_USER, CATAPUSH_PASSWORD)
    .start(object : RecoverableErrorCallback<Boolean> {
        override fun success(response: Boolean) { /* connected */ }
        override fun warning(recoverableError: Throwable) { /* transient, will retry */ }
        override fun failure(irrecoverableError: Throwable) { /* permanent */ }
    })
```

The `Catapush.isRunning()` / `Catapush.isConnected()` distinction introduced in 12.0.x is preserved unchanged. The new `isConnecting()`, `isStopping()`, and `isStopped()` accessors are also available if your UI cares about the intermediate states.

## 6. Update message queries to the Flow-based API

In 16.0.x the same data was reachable via two parallel sets of methods: the callback-based `…AsList(callback)` family and the RxJava 3 `…AsPagingDataFlowable(scope)` family. Both have been collapsed into a single `kotlinx.coroutines` `Flow`-based API in 1.0.0:

* `getMessagesWithoutChannelAsList(…)` and `getMessagesFromChannelAsList(…)` (and their `…AsPagingDataFlowable(…)` siblings) → `getMessages(scope, pageSize, channel)` returning `Flow<PagingData<CatapushMessage>>`.
* `getChannelList(callback)` → `getChannelList(): Flow<List<String>>`.
* `countMessages(callback)` → `countMessages(): Flow<Long>`.
* `countUnreadMessages(callback)`, `countUnreadMessagesWithoutChannel(callback)`, `countUnreadMessagesFromChannel(channel, callback)` → `countUnreadMessages(channel?): Flow<Long>` (pass `null` to count across the default channel).

**Before:**

```kotlin
Catapush.getInstance()
    .getMessagesWithoutChannelAsList(callback)

Catapush.getInstance()
    .getMessagesFromChannelAsList("orders", callback)
```

**After:**

```kotlin
// Inside a ViewModel:
val messages: Flow<PagingData<CatapushMessage>> =
    Catapush
        .getMessages(viewModelScope, pageSize = 20, channel = "orders")
        .cachedIn(viewModelScope)

// In a Compose UI:
val items = messages.collectAsLazyPagingItems()
LazyColumn {
    items(items) { message -> /* render … */ }
}
```

Pass `channel = null` to `getMessages(…)` to query messages from the default (unnamed) channel. The flow emits a new `PagingData` snapshot whenever the underlying database changes, so list and count UIs stay in sync without manual refreshes.

## 7. Renamed and new APIs to know about

Most of the public API is preserved with the same name and meaning. A handful of methods have been renamed or merged for consistency, and a couple of features are introduced in 1.0.0.

**Renamed:**

* `clean(callback)` → **`clearDataAndCredentials()`** — wipes all locally stored messages, attachments, and credentials, and resets the SDK to its initial state. Now synchronous.
* `clearMessages(callback)` → **`deleteMessages()`** — removes all messages from the local database. Now synchronous.
* `deleteMessagesInChannel(callback, channel)` → **`deleteMessagesInChannel(channel)`** — argument order is reversed to match the rest of the API; now synchronous.
* `sendFile(path, …)` and `sendFile(uri, …)` overloads → **`sendMessage(message, subject?, channel?, inReplyToMessageId?, attachmentUri?, callback?)`** — attachments are now passed via the `attachmentUri` parameter of the unified `sendMessage(…)`.

**New in 1.0.0:**

* **`setMessageTtlDays(days)` / `getMessageTtlDays()`** — configure automatic deletion of messages older than `N` days. Defaults to 90 days; pass `0` to disable.
* **`setNotificationIntentProvider(provider)`** — replace the notification intent provider at runtime (the 16.0.x SDK only allowed setting it via `init(…)`).
* **`logout(callback?)` accepts a nullable callback** — pass `null` if you do not need a completion notification.
* **`enableLog(logger?)`** — accepts an optional `ICatapushLogger` to redirect SDK log output to a custom destination (Crashlytics, Sentry, etc.).
* **`deleteMessage(messageId)`** — single-message deletion by ID.

**Removed:**

* `setAppKey(appKey)` and `setSenderId(senderId)` — these were already deprecated in 16.0.x. Configure the app key through the `com.catapush.library.APP_KEY` `<meta-data>` in your `AndroidManifest.xml`, exactly as in the last 16.0.x recommendation.
* `enableModalNotification()`, `disableModalNotification()`, `isModalNotificationEnabled()` — these were already no-ops in 15.x. If you need bubble-style chat notifications, use Android's [Bubbles API](https://developer.android.com/develop/ui/views/notifications/bubbles).
* `enableStatusBarNotification()`, `disableStatusBarNotification()`, `isStatusBarNotificationEnabled()` — replaced by the existing pair `enableNotifications()` / `disableNotifications()`, which already covered the same toggle persistently.
* `getIdentifier(callback)` and `getPassword(callback)` — direct read-back of the credentials is no longer offered. Keep your own copy of the identifier in app preferences if you need to display it.
* `getMessagesAsList(callback)`, `getMessagesAsPagingDataFlowable(scope)`, `getMessageById(id, callback)` — replaced by `getMessages(…)` filtering on `channel = null` to list across all channels, and by reading messages directly from the local database `Flow` if you need a single message by ID.
* `rebuildSecureCredentialsStore(callback)` — the SDK now handles secure-store recovery internally; surface the failure path via `setSecureCredentialsStoreCallback(…)` instead.
* `isStarted()` — the `isConnecting()` / `isConnected()` / `isStopping()` / `isStopped()` accessors cover the same intent more precisely.

## API rename quick reference

A flat list of the public-API renames and signature changes you will encounter while porting an existing 16.0.x integration. Methods not listed here keep the same name and meaning.

| 16.0.x | Multiplatform 1.0.0 |
|---|---|
| `Catapush.getInstance()` | `Catapush` (Kotlin `object`) |
| `Catapush.setUser(id, pwd)` | `Catapush.setCredentials(id, pwd)` |
| `Catapush.init(ctx, init, eventDelegate, services, intentProvider, tpl, otherTpls, callback)` | `Catapush.init(ctx, eventDelegate, services, intentProvider, tpl, otherTpls)` (synchronous, no callback; the `ICatapushInitializer` is registered via `setInitializer(…)` when needed) |
| `CatapushGms.INSTANCE` | `CatapushGms` |
| `CatapushHms.INSTANCE` | `CatapushHms` |
| `Catapush.getMessagesAsList(callback)` / `getMessagesAsPagingDataFlowable(scope)` | `Catapush.getMessages(scope, pageSize, channel = null)` returning `Flow<PagingData<CatapushMessage>>` |
| `Catapush.getMessagesWithoutChannelAsList(callback)` / `getMessagesWithoutChannelAsPagingDataFlowable(scope)` | `Catapush.getMessages(scope, pageSize, channel = null)` |
| `Catapush.getMessagesFromChannelAsList(channel, callback)` / `getMessagesFromChannelAsPagingDataFlowable(channel, scope)` | `Catapush.getMessages(scope, pageSize, channel)` |
| `Catapush.getChannelList(callback)` | `Catapush.getChannelList(): Flow<List<String>>` |
| `Catapush.countMessages(callback)` | `Catapush.countMessages(): Flow<Long>` |
| `Catapush.countUnreadMessages(callback)` | `Catapush.countUnreadMessages(channel?): Flow<Long>` |
| `Catapush.countUnreadMessagesWithoutChannel(callback)` | `Catapush.countUnreadMessages(channel = null): Flow<Long>` |
| `Catapush.countUnreadMessagesFromChannel(channel, callback)` | `Catapush.countUnreadMessages(channel): Flow<Long>` |
| `Catapush.clean(callback)` | `Catapush.clearDataAndCredentials()` (synchronous) |
| `Catapush.clearMessages(callback)` | `Catapush.deleteMessages()` (synchronous) |
| `Catapush.deleteMessagesInChannel(callback, channel)` | `Catapush.deleteMessagesInChannel(channel)` (synchronous; argument order reversed) |
| `Catapush.sendMessage(text, channel, originalMessageId, callback)` | `Catapush.sendMessage(message, subject?, channel?, inReplyToMessageId?, attachmentUri?, callback?)` |
| `Catapush.sendFile(path, message, channel, originalMessageId, callback)` / `sendFile(uri, …)` | `Catapush.sendMessage(message, …, attachmentUri = uri.toString(), …)` |
| `Catapush.enableLog()` | `Catapush.enableLog(logger?)` (accepts an optional `ICatapushLogger`) |
| `Catapush.KEY_CATAPUSH_WAKEUP` | `ICatapush.KEY_CATAPUSH_WAKEUP` (constant moved from the `Catapush` class to the `ICatapush` companion) |

For the full up-to-date public API surface, see the [Multiplatform SDK Android guide](DOCUMENTATION_MULTIPLATFORM_SDK_ANDROID.md).
