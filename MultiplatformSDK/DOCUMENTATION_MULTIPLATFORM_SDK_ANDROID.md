![Catapush Logo](../AndroidSDK/images/catapush_logo.png)

# Catapush Multiplatform SDK — Android integration guide

## Index

* [Catapush Multiplatform SDK 1.0.0](#catapush-multiplatform-sdk-100)
* [Project prerequisites](#project-prerequisites)
* [Core module](#core-module)
    * [Include the Core module as a dependency](#include-the-core-module-as-a-dependency)
    * [Set the app key in `AndroidManifest.xml`](#set-the-app-key-in-androidmanifestxml)
    * [Initialization](#initialization)
    * [`ICatapushInitializer` (optional)](#icatapushinitializer-optional)
    * [Android notification runtime permission](#android-notification-runtime-permission)
    * [Android notification channels support](#android-notification-channels-support)
    * [Set credentials and start](#set-credentials-and-start)
    * [Error handling](#error-handling)
    * [Handle notification taps](#handle-notification-taps)
* [Google Mobile Services (GMS) module](#google-mobile-services-gms-module)
    * [Firebase Cloud Messaging prerequisites](#firebase-cloud-messaging-prerequisites)
    * [Include the GMS module as a dependency](#include-the-gms-module-as-a-dependency)
    * [Google Mobile Services Gradle plugin configuration](#google-mobile-services-gradle-plugin-configuration)
    * [Use the GMS module in `Catapush.init(…)`](#use-the-gms-module-in-catapushinit)
    * [OPTIONAL: integrate Catapush GMS with a pre-existent `FirebaseMessagingService`](#optional-integrate-catapush-gms-with-a-pre-existent-firebasemessagingservice)
* [Huawei Mobile Services (HMS) module](#huawei-mobile-services-hms-module)
    * [Huawei Push Kit prerequisites](#huawei-push-kit-prerequisites)
    * [Pick the right HMS artifact](#pick-the-right-hms-artifact)
    * [Huawei Mobile Services Gradle plugin configuration](#huawei-mobile-services-gradle-plugin-configuration)
    * [Use the HMS module in `Catapush.init(…)`](#use-the-hms-module-in-catapushinit)
    * [OPTIONAL: integrate Catapush HMS with a pre-existent `HmsMessageService`](#optional-integrate-catapush-hms-with-a-pre-existent-hmsmessageservice)
* [Optional UI module (`:ui`)](#optional-ui-module-ui)
* [Advanced](#advanced)
    * [Logging](#logging)
    * [Handling client-side push services errors](#handling-client-side-push-services-errors)
    * [Notification visualization](#notification-visualization)
    * [Disable received messages visualization](#disable-received-messages-visualization)
    * [Pause received messages visualization](#pause-received-messages-visualization)
    * [Hide notifications while the user is reading](#hide-notifications-while-the-user-is-reading)
    * [Sending read receipts manually](#sending-read-receipts-manually)
    * [Conversation channels](#conversation-channels)
    * [Load and display message attachments](#load-and-display-message-attachments)
    * [Automatic message deletion (TTL)](#automatic-message-deletion-ttl)
    * [Logout and clear data](#logout-and-clear-data)
    * [Transform incoming messages before persisting them](#transform-incoming-messages-before-persisting-them)
    * [Background data restrictions (Android 7.0+)](#background-data-restrictions-android-70)
    * [Background execution restrictions (Android 9.0+)](#background-execution-restrictions-android-90)
    * [App standby buckets (Android 9.0+)](#app-standby-buckets-android-90)
* [FAQ](#faq)

## Catapush Multiplatform SDK 1.0.0

On Android the SDK is shipped as a regular set of `aar` artifacts hosted on the public Catapush Maven repository.

This release targets Android 16 (API 36) and requires a minimum of Android 7.0 (API 24).

If you are coming from `catapush-android-sdk` 16.0.x, follow the dedicated [Android migration guide](MIGRATION_FROM_ANDROID_SDK.md) instead of reading this document end-to-end — most of your existing integration is preserved with only a handful of source-code changes.

## Project prerequisites

The Catapush Multiplatform SDK assumes that your Android project:

1. Has `compileSdk` set to **36** ([Android 16](https://developer.android.com/tools/releases/platforms#16)).
2. Has `minSdk` greater than or equal to **24** ([Android 7.0](https://developer.android.com/tools/releases/platforms#7.0)).
3. Builds with **JDK 21** as its toolchain.
4. Uses **Kotlin 2.0** or later (only relevant if your app code is in Kotlin; the SDK itself is compiled and shipped as Java-friendly bytecode).

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

## Core module

The Core module is the heart of the SDK. It opens and maintains the persistent connection to the Catapush messaging service, persists messages and attachments locally, and dispatches lifecycle events to your app.

When the app is in the foreground, the Core module alone is enough to send and receive messages. To receive messages while the app is in the background or stopped, add a push notification provider — Google Mobile Services (GMS) or Huawei Mobile Services (HMS) — as documented later.

### Include the Core module as a dependency

Add the Catapush Maven repository to the `repositories` block of your project (either at the root via `allprojects { repositories { … } }` or inside the consuming module's `build.gradle`):

```groovy
repositories {
    maven { url "https://s3.eu-west-1.amazonaws.com/m2repository.catapush.com/" }
}
```

Then add the Core module to your app's `dependencies`:

```groovy
implementation 'com.catapush.sdk:core:1.0.0'
```

### Set the app key in `AndroidManifest.xml`

Declare your Catapush app key as a `<meta-data>` element inside the `<application>` node:

```xml
<meta-data
    android:name="com.catapush.library.APP_KEY"
    android:value="YOUR_APP_KEY" />
```

`YOUR_APP_KEY` is the AppKey of your Catapush App. You can find it on your [Catapush App configuration dashboard](https://www.catapush.com/panel/dashboard) under "App details".

### Initialization

Initialize the SDK in your `Application.onCreate()`. The `Catapush` singleton is a Kotlin `object`, so you call its methods directly without `getInstance()`.

`Catapush.init(…)` takes six parameters:

1. The application `Context`.
2. An `ICatapushEventDelegate` that receives lifecycle and messaging events.
3. The list of mobile-services adapters (`CatapushGms`, `CatapushHms`, both, or empty for foreground-only delivery).
4. An `ICatapushNotificationIntentProvider` that builds the `PendingIntent` launched when the user taps a notification.
5. The main `NotificationTemplate` used to render incoming messages on the default notification channel.
6. An optional `Collection<NotificationTemplate>` if you want to associate different templates with different notification channels (or `null` if you only use the default).

A typical Kotlin `Application.onCreate()` looks like this:

```kotlin
class MyApplication : Application() {

    private val NOTIFICATION_CHANNEL_ID = "your.app.package.CHANNEL_ID"

    override fun onCreate() {
        super.onCreate()

        val eventDelegate = object : ICatapushEventDelegate {
            override fun onRegistrationFailed(error: CatapushAuthenticationError) { /* … */ }
            override fun onConnecting() { /* … */ }
            override fun onConnected() { /* … */ }
            override fun onDisconnected(error: CatapushConnectionError) { /* … */ }
            override fun onPushServicesError(error: PushServicesException) { /* … */ }
            override fun onMessageReceived(message: CatapushMessage) { /* … */ }
            override fun onMessageReceivedConfirmed(message: CatapushMessage) { /* … */ }
            override fun onMessageOpened(message: CatapushMessage) { /* … */ }
            override fun onMessageOpenedConfirmed(message: CatapushMessage) { /* … */ }
            override fun onMessageSent(message: CatapushMessage) { /* … */ }
            override fun onMessageSentConfirmed(message: CatapushMessage) { /* … */ }
        }

        val intentProvider = ICatapushNotificationIntentProvider { message, context ->
            val intent = Intent(context, MainActivity::class.java).apply {
                // A unique data URI prevents Android from reusing the extras Bundle across
                // PendingIntent instances for different messages.
                data = Uri.parse("catapush://${context.packageName}/message/${message.id()}")
                putExtra("message", message)
                flags = Intent.FLAG_ACTIVITY_NEW_TASK or Intent.FLAG_ACTIVITY_SINGLE_TOP
            }
            PendingIntent.getActivity(
                context, 0, intent,
                PendingIntent.FLAG_ONE_SHOT or PendingIntent.FLAG_IMMUTABLE
            )
        }

        // Create the Android notification channel BEFORE Catapush.init(…)
        val nm = getSystemService(Context.NOTIFICATION_SERVICE) as NotificationManager
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O &&
            nm.getNotificationChannel(NOTIFICATION_CHANNEL_ID) == null
        ) {
            nm.createNotificationChannel(
                NotificationChannel(
                    NOTIFICATION_CHANNEL_ID,
                    "Catapush messages",
                    NotificationManager.IMPORTANCE_HIGH
                )
            )
        }

        val template = NotificationTemplate.Builder(NOTIFICATION_CHANNEL_ID)
            .title("Your notification title")
            .iconId(R.drawable.ic_stat_notify_default)
            .vibrationEnabled(true)
            .soundEnabled(true)
            .soundResourceUri(RingtoneManager.getDefaultUri(RingtoneManager.TYPE_NOTIFICATION))
            .circleColor(ContextCompat.getColor(this, R.color.primary))
            .ledEnabled(true)
            .ledColor(Color.BLUE)
            .ledOnMS(2000)
            .ledOffMS(1000)
            .build()

        Catapush.init(
            context = this,
            eventDelegate = eventDelegate,
            mobileServices = emptyList(), // Will be replaced with GMS / HMS adapters below
            notificationIntentProvider = intentProvider,
            mainNotificationChannelTemplate = template,
            otherNotificationChannelsTemplates = null
        )
    }
}
```

If you are defining a custom `Application` class for the first time, register it in `AndroidManifest.xml`:

```xml
<application
    android:name=".MyApplication"
    android:icon="@mipmap/ic_launcher"
    android:label="@string/app_name"
    android:theme="@style/AppTheme">
```

### `ICatapushInitializer` (optional)

In some edge cases the Android system can wake the SDK up before your `Application.onCreate()` has run (for example, when a push notification arrives while the process is being recreated). To handle this transparently, implement `ICatapushInitializer` and register it with the SDK:

```kotlin
class MyApplication : Application(), ICatapushInitializer {

    override fun onCreate() {
        super.onCreate()
        initCatapush()
    }

    override fun initCatapush() {
        // Move all the Catapush.init(…) / setSecureCredentialsStoreCallback / etc. setup
        // into this method, then invoke it from onCreate().
    }
}
```

Then register the initializer with the SDK so it can be invoked on demand:

```kotlin
Catapush.setInitializer(this)
```

### Android notification runtime permission

Starting from Android 13.0 (API 33), apps must explicitly request permission to display notifications.

Declare the permission in your `AndroidManifest.xml`:

```xml
<uses-permission android:name="android.permission.POST_NOTIFICATIONS" />
```

Then prompt the user at the appropriate time using the standard `ActivityResultContracts.RequestPermission` API. See the Android [notification runtime permission documentation](https://developer.android.com/develop/ui/views/notifications/notification-permission) for details.

### Android notification channels support

Since Android 8.0 (API 26), every notification must be associated with a channel ([Create and Manage Notification Channels](https://developer.android.com/training/notify-user/channels)).

To deliver Catapush messages to different channels:

1. Create each `NotificationChannel` via `NotificationManager.createNotificationChannel(…)` in your `Application.onCreate()`, **before** calling `Catapush.init(…)`.
2. Build a `NotificationTemplate` for each channel using `NotificationTemplate.Builder(notificationChannelId)`. The channel ID must match the one you used when creating the `NotificationChannel`.
3. Pass the main/default template as the 5th argument of `Catapush.init(…)` and the others as a `Collection<NotificationTemplate>` in the 6th argument.

To test this, send a Catapush message with the `channel` attribute set to the same value as your `NotificationChannel` ID — the SDK will pick the matching `NotificationTemplate` to render it. Messages received without a `channel`, or with one that does not match any template, are posted using the main/default template.

To update a template at runtime (e.g. to change the icon after a theme switch), use:

```kotlin
Catapush.updateNotificationTemplate(template)
```

### Set credentials and start

Once initialization is complete, call `setCredentials(…)` followed by `start(…)`:

```kotlin
Catapush
    .setCredentials(CATAPUSH_USER, CATAPUSH_PASSWORD)
    .start(object : RecoverableErrorCallback<Boolean> {
        override fun success(response: Boolean) {
            // Connected to the Catapush messaging service
        }
        override fun warning(recoverableError: Throwable) {
            // The SDK detected a transient issue (e.g. background data restrictions)
            // but will continue to retry. Surface the issue to the user if it makes
            // sense for your UX.
        }
        override fun failure(irrecoverableError: Throwable) {
            // Permanent failure (wrong credentials, missing manifest entry, …).
            // The SDK will not retry. See `Error handling` below for the codes you
            // might receive here and in onDisconnected.
        }
    })
```

`CATAPUSH_USER` and `CATAPUSH_PASSWORD` are the user identifier and password from the Users section of your Catapush App dashboard.

You can call `setCredentials(…)` and `start(…)` either:

* As soon as your app starts, if the user is already logged in and you have already obtained their Catapush credentials from your backend.
* As soon as the user logs in, once your app obtains the credentials.

The credentials are persisted in the Android secure store (Android Keystore-backed) so subsequent app launches can call `start(…)` without re-providing them.

### Error handling

The `onDisconnected(error: CatapushConnectionError)` callback may be invoked with the following reason codes:

| reasonCode | Description | Suggested strategy |
|---|---|---|
| 3 = `BACKEND_NOT_AVAILABLE` | The SDK could not establish a connection to the Catapush remote messaging service. It might be blocked by a firewall, or the service might be temporarily disrupted. | Check your internet connection and retry in a few minutes. |
| 4 = `INTERNET_NOT_AVAILABLE` | The device is not connected to the internet. | Inform the user about the lack of connectivity. |
| 6 = `SERVICE_STOPPED` | The Android system stopped the messaging service due to power-saving policies, or the local messaging service stopped following a network error. | Expected on Android 8.0+: this code is delivered shortly after the app goes to background. The SDK will reconnect automatically when the app returns to the foreground. |
| 7 = `SERVICE_STOPPED_AS_REQUESTED` | The local messaging service stopped after explicitly invoking `Catapush.stop()`. | No action needed. |
| 8 = `CONNECTION_PARAMETERS_LOAD_FAILED` | The stored credentials are invalid. | Check the credentials passed to `setCredentials(…)`. |
| 9 = `CONNECTION_TIMEOUT` | The connection attempt timed out, possibly due to connectivity issues. | Check your internet connection and retry in a few minutes. |
| 10 = `START_TIMEOUT` | The SDK could not complete the start command in a timely fashion, possibly due to connectivity issues. | Check your internet connection and retry in a few minutes. |

The `onRegistrationFailed(error: CatapushAuthenticationError)` callback may be invoked with the following reason codes:

| reasonCode | Description | Suggested strategy |
|---|---|---|
| 14011 = `API_UNAUTHORIZED` | The credentials have been rejected. | The user might have been deleted from the Catapush app, or the password has changed. Stop retrying and provide new credentials. |
| 15001 = `API_INTERNAL_ERROR` | Internal error of the remote messaging service. | Likely a temporary disruption. Retry in a few minutes. |
| 24042 = `REGISTRATION_NOT_FOUND_USER` | Register API: user not found. | The user has probably been deleted. Stop retrying, clear stored credentials and provide a new identifier. |
| 44043 = `UPDATE_PUSH_TOKEN_NOT_FOUND_USER` | Update push token API: user not found. | Same as above. |

### Handle notification taps

When the user taps a Catapush notification, the SDK launches the `PendingIntent` returned by your `ICatapushNotificationIntentProvider`. Handle the resulting `Intent` in the target `Activity`:

```kotlin
override fun onCreate(savedInstanceState: Bundle?) {
    super.onCreate(savedInstanceState)
    if (intent.hasExtra("message")) {
        handleCatapushMessageIntent(intent)
    }
}

override fun onNewIntent(intent: Intent) {
    super.onNewIntent(intent)
    if (intent.hasExtra("message")) {
        handleCatapushMessageIntent(intent)
    }
}

private fun handleCatapushMessageIntent(intent: Intent) {
    val message = IntentCompat.getParcelableExtra(intent, "message", CatapushMessage::class.java)
    // Navigate to the conversation view, mark the message as opened, …
}
```

Customize the extras and data URI in your `ICatapushNotificationIntentProvider` to fit the navigation model of your app.

## Google Mobile Services (GMS) module

The GMS module integrates the SDK with Google Mobile Services / Firebase Cloud Messaging. It is required to receive Catapush messages while the app is in the background on devices that have Google Play Services installed.

### Firebase Cloud Messaging prerequisites

The GMS module needs a Firebase project. See the [FCM configuration guide](DOCUMENTATION_PLATFORM_GMS_FCM.md) to set up your Firebase project and your Catapush dashboard.

Once configured:

1. Open the [Firebase Console](https://console.firebase.google.com) and select your project.
2. Click the gear icon and select **Project settings**.
3. In the **General** tab scroll to **Your apps** and select your Android app.
4. Download the `google-services.json` file.
5. Copy `google-services.json` into the `app/` subfolder of your Android Studio project.

### Include the GMS module as a dependency

```groovy
implementation 'com.catapush.sdk.android:gms:1.0.0'
```

### Google Mobile Services Gradle plugin configuration

The Google Mobile Services Gradle plugin parses `google-services.json` and configures its client libraries. Add it to your project `build.gradle`:

```groovy
buildscript {
    repositories {
        google()
        mavenCentral()
    }
    dependencies {
        classpath 'com.android.tools.build:gradle:8.9.1'
        classpath 'com.google.gms:google-services:4.4.2'
    }
}
```

Apply it at the bottom of `app/build.gradle`:

```groovy
apply plugin: 'com.google.gms.google-services'
```

Add the metadata tag to the `<application>` block in your `AndroidManifest.xml`:

```xml
<meta-data
    android:name="com.google.android.gms.version"
    android:value="@integer/google_play_services_version" />
```

### Use the GMS module in `Catapush.init(…)`

Pass `CatapushGms` to the `mobileServices` parameter:

```kotlin
Catapush.init(
    context = this,
    eventDelegate = eventDelegate,
    mobileServices = listOf(CatapushGms),
    notificationIntentProvider = intentProvider,
    mainNotificationChannelTemplate = template,
    otherNotificationChannelsTemplates = null
)
```

### OPTIONAL: integrate Catapush GMS with a pre-existent `FirebaseMessagingService`

If your app already declares its own `FirebaseMessagingService`, forward Catapush wakeups and token refreshes manually:

```kotlin
class YourFirebaseMessagingService : FirebaseMessagingService() {

    override fun onMessageReceived(remoteMessage: RemoteMessage) {
        if (remoteMessage.data.containsKey(ICatapush.KEY_CATAPUSH_WAKEUP)) {
            CatapushGms.handleFcmWakeup(remoteMessage)
        } else {
            // Your own push handling logic
        }
    }

    override fun onNewToken(refreshedToken: String) {
        CatapushGms.handleFcmNewToken(refreshedToken)
        // Your own token handling logic
    }
}
```

The wakeup data key is `ICatapush.KEY_CATAPUSH_WAKEUP` (the value `"catapush_wake"`).

## Huawei Mobile Services (HMS) module

The HMS module integrates the SDK with Huawei Mobile Services / Push Kit. It is required to receive Catapush messages while the app is in the background on Huawei devices that do not have Google Play Services.

### Huawei Push Kit prerequisites

The HMS module needs Huawei Push Kit. See the [HMS configuration guide](DOCUMENTATION_PLATFORM_HMS_PUSHKIT.md) to set up your AppGallery project and your Catapush dashboard.

Once configured:

1. Open AppGallery Connect and select your project.
2. Go to **Project settings → General information** and download the `agconnect-services.json` file.
3. Copy `agconnect-services.json` into the `app/` subfolder of your Android Studio project.

### Pick the right HMS artifact

Huawei Push Kit only allows **one** `HmsMessageService` per app. The SDK ships two distinct artifacts so you can integrate Catapush in either of these scenarios:

* **`com.catapush.sdk.android:hms`** — your app does **not** declare its own `HmsMessageService`. The SDK ships its own `HmsMessageService` implementation and does all the wiring for you. This is the most common case.

  ```groovy
  implementation 'com.catapush.sdk.android:hms:1.0.0'
  ```

* **`com.catapush.sdk.android:hms-base`** — your app **already** declares its own `HmsMessageService`. Catapush does not register its own service; you forward the wakeups manually from yours (see the OPTIONAL section below).

  ```groovy
  implementation 'com.catapush.sdk.android:hms-base:1.0.0'
  ```

### Huawei Mobile Services Gradle plugin configuration

The Huawei Mobile Services Gradle plugin parses `agconnect-services.json` and configures its client libraries. Add it to your project `build.gradle`:

```groovy
buildscript {
    repositories {
        google()
        mavenCentral()
        maven { url 'https://developer.huawei.com/repo/' }
    }
    dependencies {
        classpath 'com.android.tools.build:gradle:8.9.1'
        classpath 'com.huawei.agconnect:agcp:1.9.1.303'
    }
}
```

Apply it at the top of your `app/build.gradle`:

```groovy
apply plugin: 'com.huawei.agconnect'
```

### Use the HMS module in `Catapush.init(…)`

Pass `CatapushHms` to the `mobileServices` parameter:

```kotlin
Catapush.init(
    context = this,
    eventDelegate = eventDelegate,
    mobileServices = listOf(CatapushHms),
    notificationIntentProvider = intentProvider,
    mainNotificationChannelTemplate = template,
    otherNotificationChannelsTemplates = null
)
```

To support both GMS and HMS in the same APK, list them in priority order:

```kotlin
mobileServices = listOf(CatapushGms, CatapushHms)
```

The SDK evaluates the adapters in order and elects the first available one for the current device. See the [FAQ](#how-does-the-sdk-choose-between-gms-and-hms) for guidance on which to prioritize.

### OPTIONAL: integrate Catapush HMS with a pre-existent `HmsMessageService`

If your app already declares its own `HmsMessageService`, depend on `:hms-base` instead of `:hms` and forward Catapush wakeups and token refreshes manually:

```kotlin
class YourHmsMessageService : HmsMessageService() {

    override fun onMessageReceived(remoteMessage: RemoteMessage) {
        if (remoteMessage.dataOfMap.containsKey(ICatapush.KEY_CATAPUSH_WAKEUP)) {
            CatapushHms.handleHmsWakeup(remoteMessage)
        } else {
            // Your own push handling logic
        }
    }

    override fun onNewToken(refreshedToken: String) {
        CatapushHms.handleHmsNewToken(refreshedToken)
        // Your own token handling logic
    }
}
```

## Optional UI module (`:ui`)

The `:ui` module ships ready-to-use Compose Multiplatform message-list and conversation views built on top of `:core`. Most apps do not use it — they integrate Catapush into their own existing message UI — but it can be a fast way to bring up a working chat screen.

If you want to use it, add the dependency:

```groovy
implementation 'com.catapush.sdk:ui:1.0.0'
```

Then initialize the UI module **after** `Catapush.init(…)`:

```kotlin
Catapush.init(/* … */)
CatapushUi.init()
```

The two main composables exposed by the module are:

* **`ChatList`** — a paged list of `CatapushMessage` items rendered as bubbles, with an integrated `MessageComposer` for sending new messages and attachments. Bound to a `ChatListState` created via `rememberChatListState(channelId, …)` and to a `MessageComposerState` created via `rememberMessageComposerState()`.
* **`ConversationsList`** — a list of conversation channels (one row per channel), useful for an inbox/landing screen.

Bridge `CatapushUi.kamelConfig` to [Kamel](https://github.com/Kamel-Media/Kamel) (the image loader used internally) via a `CompositionLocalProvider` once at the root of your composition. Then drop `ChatList` into your screen and react to its `ChatListAction`s:

```kotlin
class MainActivity : ComponentActivity() {

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContent {
            CompositionLocalProvider(LocalKamelConfig provides CatapushUi.kamelConfig) {
                ChatScreen()
            }
        }
    }
}

@Composable
private fun ChatScreen() {
    val coroutineScope = LocalLifecycleOwner.current.lifecycle.coroutineScope

    // Bind the list to the default channel; pass a non-null channelId to scope it to one channel.
    val chatListState = rememberChatListState(
        channelId = null,
        coroutineScope = coroutineScope
    )
    val messageComposerState = rememberMessageComposerState()

    val filePickerLauncher = rememberLauncherForActivityResult(
        ActivityResultContracts.OpenDocument()
    ) { uri: Uri? ->
        uri?.also { messageComposerState.setAttachment(it.path) }
    }

    ChatList(
        state = chatListState,
        messageComposerState = messageComposerState,
        modifier = Modifier.fillMaxSize(),
        onAction = { action ->
            when (action) {
                is ChatListAction.OnAttach ->
                    filePickerLauncher.launch(arrayOf("image/*", "application/pdf", "text/plain"))
                is ChatListAction.OnMessageDelete ->
                    Catapush.deleteMessage(action.message.id)
                is ChatListAction.OnAttachedImageTap,
                is ChatListAction.OnAttachedPdfTap,
                is ChatListAction.OnAttachedTxtTap -> {
                    // Open the file with an external viewer using the URI on action.file
                }
                is ChatListAction.OnMessageTap,
                is ChatListAction.OnMessageLongPress,
                is ChatListAction.OnMessageReply -> {
                    // Wire these to your own UX (selection, reply composer, …)
                }
            }
        }
    )
}
```

`rememberChatListState(channelId, …)` reads from `Catapush.getMessages(…)` internally, so no additional wiring is needed: messages flow into the list as they arrive and the composable handles paging, scroll position and read-receipt dispatch automatically.

A complete working example — including file picker plumbing, FileProvider URIs and the dropdown menu that drives `Catapush.logout()` — is available in the sample app at [`androidApp/src/main/java/com/catapush/sdk/android/sample/MainActivity.kt`](https://github.com/Catapush/catapush-sdk-native/blob/main/androidApp/src/main/java/com/catapush/sdk/android/sample/MainActivity.kt).

The `:ui` module is fully optional: omit the dependency and its initialization call if you don't use it.

## Advanced

### Logging

Enable SDK logging to the Android system log:

```kotlin
Catapush.enableLog()
```

To route log lines to a custom destination (Crashlytics, Sentry, internal log file, …) pass an `ICatapushLogger`:

```kotlin
Catapush.enableLog(object : ICatapushLogger {
    override fun verbose(tag: String, msg: String) { /* … */ }
    override fun debug(tag: String, msg: String) { /* … */ }
    override fun info(tag: String, msg: String) { /* … */ }
    override fun warn(tag: String, msg: String, t: Throwable?) { /* … */ }
    override fun error(tag: String, msg: String, t: Throwable?) { /* … */ }
})
```

Disable logging at any time:

```kotlin
Catapush.disableLog()
```

Logs are only emitted when your app is built in debug mode.

### Handling client-side push services errors

If the GMS or HMS client libraries on the device are unavailable or out of date, Catapush cannot receive messages while the app is not running. The `onPushServicesError(error: PushServicesException)` callback notifies you about these problems; you can guide the user to update Google Play Services or HMS Core via the system tray:

```kotlin
override fun onPushServicesError(error: PushServicesException) {
    if (PushPlatformType.GMS.name == error.platform && error.isUserResolvable) {
        val gmsAvailability = GoogleApiAvailability.getInstance()
        gmsAvailability.setDefaultNotificationChannelId(this@MyApplication, NOTIFICATION_CHANNEL_ID)
        gmsAvailability.showErrorNotification(this@MyApplication, error.errorCode)
    } else if (PushPlatformType.HMS.name == error.platform && error.isUserResolvable) {
        HuaweiApiAvailability.getInstance().showErrorNotification(
            this@MyApplication, error.errorCode
        )
    }
}
```

If you only depend on one of the two adapters, drop the corresponding branch.

### Notification visualization

By default the SDK builds a status-bar notification for every received message and posts it to the Android `NotificationManager` using the `NotificationTemplate` you provided.

### Disable received messages visualization

Permanently disable status-bar notifications:

```kotlin
Catapush.disableNotifications()
```

Re-enable them:

```kotlin
Catapush.enableNotifications()
```

This state is persisted across app restarts and Catapush starts/stops.

> Note: disabling notifications **does not** stop the SDK from receiving messages. You are still notified via `onMessageReceived(…)` and you can present them to the user yourself.

### Pause received messages visualization

Temporarily disable status-bar notifications:

```kotlin
Catapush.pauseNotifications()
```

Resume them:

```kotlin
Catapush.resumeNotifications()
```

This state is **not** persisted: restarting the app or stopping/starting Catapush returns to the resumed state.

### Hide notifications while the user is reading

Pause notifications when your conversation `Activity` is in the foreground and resume them when it goes back to the background:

```kotlin
override fun onResume() {
    super.onResume()
    Catapush.pauseNotifications()
}

override fun onPause() {
    super.onPause()
    Catapush.resumeNotifications()
}
```

### Sending read receipts manually

If you build your own message list UI, mark messages as read manually:

```kotlin
Catapush.notifyMessageOpened(catapushMessage.id())
```

### Conversation channels

Catapush messages can be delivered through different conversation channels. To send a message to a specific channel, set the [`channel` property](https://www.catapush.com/docs-api?php#2.1-post---send-a-new-message) on the send-message API call.

The SDK exposes the local channels and messages as `Flow`s:

```kotlin
// Observe the list of channels that have at least one message stored locally
viewModelScope.launch {
    Catapush.getChannelList().collect { channels ->
        // channels is a List<String> with one entry per channel; "" represents the default channel
    }
}

// Page through messages in a specific channel
val messages: Flow<PagingData<CatapushMessage>> =
    Catapush
        .getMessages(viewModelScope, pageSize = 20, channel = "orders")
        .cachedIn(viewModelScope)

// Page through messages without a channel (default channel)
val defaultMessages: Flow<PagingData<CatapushMessage>> =
    Catapush
        .getMessages(viewModelScope, pageSize = 20, channel = null)
        .cachedIn(viewModelScope)
```

Counts are also exposed as `Flow<Long>`:

```kotlin
viewModelScope.launch {
    Catapush.countUnreadMessages(channel = null).collect { count ->
        // Update your unread badge
    }
}
```

### Load and display message attachments

If a message includes an attachment, its remote URL is available at `message.file().remoteUri()`.

For images, load it with your image loader of choice (Glide, Coil, …):

```kotlin
Glide.with(context).load(message.file().remoteUri()).into(imageView)
```

Outgoing messages with attachments expose a local URI (`message.file().localUri()`) — prefer that when available.

To open a non-image attachment (e.g. a PDF) in an external app:

```kotlin
val intent = Intent(Intent.ACTION_VIEW).apply {
    setDataAndType(uri, "application/pdf")
    addFlags(Intent.FLAG_ACTIVITY_NEW_TASK or Intent.FLAG_GRANT_READ_URI_PERMISSION)
}
runCatching { context.startActivity(Intent.createChooser(intent, "Open file")) }
    .onFailure { Toast.makeText(context, "No app available to open this file", Toast.LENGTH_SHORT).show() }
```

### Automatic message deletion (TTL)

Messages older than a configurable TTL are deleted automatically the next time `Catapush.start(…)` runs successfully. The default TTL is **90 days**. Configure it explicitly:

```kotlin
Catapush.setMessageTtlDays(30) // delete messages older than 30 days
Catapush.setMessageTtlDays(0)  // disable automatic deletion entirely
```

The current value is available via `Catapush.getMessageTtlDays()`. The setting is persisted across app restarts.

### Logout and clear data

Use `logout(…)` when the user signs out of your app. It stops the messaging service, deregisters the device on the Catapush backend, and clears the stored credentials:

```kotlin
Catapush.logout(object : Callback<Boolean> {
    override fun success(response: Boolean) { /* signed out */ }
    override fun failure(irrecoverableError: Throwable) { /* … */ }
})
```

Use `clearDataAndCredentials()` for a more destructive reset that also wipes the locally persisted messages and attachments (e.g. on app uninstall via `BACKUP_DELETE`, or when the user switches accounts):

```kotlin
Catapush.clearDataAndCredentials()
```

### Transform incoming messages before persisting them

Install a `CatapushMessageTransformation` to intercept every incoming message before it is stored in the local database. Typical use cases include:

* end-to-end decryption of the message body before it lands in the local store;
* substitution of server-side placeholders (`{{firstName}}`, `{{orderId}}`, …) with values that only the client can resolve;
* attachment of derived metadata (e.g. routing tags) to drive your UI's grouping or filtering;
* sanitization or redaction of fields you do not want to persist on disk in their raw form.

```kotlin
Catapush.setMessageTransformation(object : CatapushMessageTransformation() {
    override fun transform(input: Model): Model {
        // input.originalMessage is the read-only original message;
        // input.body and input.previewText are the mutable values that will be persisted.

        // Example 1 — end-to-end decryption: the server delivers an envelope and the client
        // unwraps it with a key that never leaves the device.
        val decryptedBody = MyCrypto.decrypt(input.body, MyKeystore.userKey)

        // Example 2 — placeholder substitution: replace tokens with values from local state.
        input.body = decryptedBody
            .replace("{{firstName}}", LocalProfile.firstName)
            .replace("{{orderId}}", LocalCart.currentOrderId.orEmpty())

        // Optionally update the preview shown in the messages list.
        input.previewText = input.body.take(80)

        // Return the same instance with the values you want to persist.
        return input
    }
})
```

Modify `input.body` and `input.previewText` in place — the SDK persists the values present on the returned `Model`. Other fields (id, sender, recipient, timestamps) are read-only via `input.originalMessage`.

The transformation runs synchronously on the SDK's IO scheduler. Keep it fast — long-running work blocks message persistence.

### Background data restrictions (Android 7.0+)

Android 7.0 introduced background data restrictions. When background data is restricted for your app, you receive a `SystemConfigurationException` with `reasonCode = 103` (`REASON_SYSCFG_BACKGROUND_DATA_RESTRICTIONS`) as a warning from `start(…)`.

Detect the state via `ConnectivityManager.getRestrictBackgroundStatus()` and, if appropriate, send the user to the relevant Settings page:

```kotlin
Catapush
    .setCredentials("user", "password")
    .start(object : RecoverableErrorCallback<Boolean> {
        override fun success(response: Boolean) { /* … */ }
        override fun failure(irrecoverableError: Throwable) { /* … */ }
        override fun warning(recoverableError: Throwable) {
            if (recoverableError is SystemConfigurationException &&
                recoverableError.reasonCode == SystemConfigurationException.REASON_SYSCFG_BACKGROUND_DATA_RESTRICTIONS
            ) {
                Intent(Settings.ACTION_IGNORE_BACKGROUND_DATA_RESTRICTIONS_SETTINGS).also {
                    it.data = Uri.fromParts("package", packageName, null)
                    it.addFlags(Intent.FLAG_ACTIVITY_NEW_TASK)
                    startActivity(it)
                }
            }
        }
    })
```

### Background execution restrictions (Android 9.0+)

Android 9.0 added background execution restrictions. When they are enabled for your app, you receive a `SystemConfigurationException` with `reasonCode = 104` (`REASON_SYSCFG_BACKGROUND_EXECUTION_RESTRICTIONS`) as a warning from `start(…)`.

Detect the state via `ActivityManager.isBackgroundRestricted()` and prompt the user to disable the restriction in the device settings.

### App standby buckets (Android 9.0+)

Android 9.0 introduced [app standby buckets](https://developer.android.com/topic/performance/appstandby). When your app is demoted below `STANDBY_BUCKET_FREQUENT`, network access is throttled and you receive a `SystemConfigurationException` with `reasonCode = 105` (`REASON_SYSCFG_STANDBY_BUCKET_RESTRICTED`) as a warning.

Detect the bucket via `UsageStatsManager.getAppStandbyBucket()` and educate the user to use the app more often. The SDK cannot lift the restriction on its own.

## FAQ

### How does the SDK choose between GMS and HMS?

When you pass more than one mobile-services adapter to `Catapush.init(…)`, the SDK evaluates them in the order you provided. The first available one is elected for the current session.

Most apps should prioritize GMS:

```kotlin
mobileServices = listOf(CatapushGms, CatapushHms)
```

Prioritize HMS only if your app has been [explicitly approved by Huawei for high-priority data messages](https://developer.huawei.com/consumer/en/doc/development/HMSCore-Guides/faq-0000001050042183#EN-US_TOPIC_0000001124288055__section037425218509) or if your user base is mainly on Huawei devices in regions where Google Play Services are unavailable.

If neither is available, the error from the first adapter is delivered to `onPushServicesError(…)`.

### Which push services provider should I prioritize?

For users in the EU and US, prioritize GMS. For users in regions where Google Play Services are blocked (e.g. China) or who almost exclusively own Huawei devices, prioritize HMS. If you ship via different stores, use product flavors: GMS-only for Google Play, HMS-only for AppGallery.

### Can I use Catapush without push notifications providers (foreground-only delivery)?

Yes. Pass an empty list to `mobileServices`:

```kotlin
mobileServices = emptyList()
```

You only need the `:core` dependency in your `build.gradle`:

```groovy
implementation 'com.catapush.sdk:core:1.0.0'
```

In this configuration, messages are received only while the app is in the foreground and the SDK is connected.

### Do I need to configure ProGuard / R8 for Catapush?

No. All Catapush modules ship a `consumer-rules.pro` file that is merged into your app's R8 configuration at build time.

### Does the SDK use RxJava?

No. The public API uses `kotlinx.coroutines.Flow` for streams and plain `Callback` interfaces for one-shot operations. Your app does not need to depend on RxJava.

### Is there an example project available?

The Catapush Multiplatform SDK repository includes a working sample at [`androidApp/`](https://github.com/Catapush/catapush-sdk-native/tree/main/androidApp) demonstrating Core + GMS + optional UI module integration.
