![Catapush Logo](images/catapush_logo.png)

# Catapush Android SDK Changelog

## Catapush 13.0.x

Catapush 13.0.x is designed for Android 13.0 (API 33) and requires a minimum of Android 5.0 (API 21).

This release includes compatibility updates for the latest Android version and updates its third-party dependencies.

#### 13.0.4 (18/10/2023)
- Introduces error code `10 = START_TIMEOUT`. This code will be received in the `onDisconnected(…)` callback when the SDK is unable to establish a connection within a reasonable timeframe following the invocation of the `Catapush.start(…)` method
- Mitigates ANRs when the Catapush SDK is waken up in the background by a push notification

#### 13.0.3 (11/09/2023)
- Fix transitive test dependencies leaking to the core module Maven POM file

#### 13.0.2 (11/09/2023)
- Updates internal dependecies

#### 13.0.1 (05/09/2023)
- Adds the `Catapush.deleteMessagesInChannel` method to delete all messages in a specific channel

#### 13.0.0 (14/06/2023)
- Andorid 13 compatibility

## Catapush 12.1.x

Catapush 12.1.x targets Android 12.0 (API 31) and requires Android 5.0 (API 21).

This release overhauls the SDK initialization and reorganize the `Catapush.init(…)` method parameters.

The main change is the introduction of the `ICatapushInitializer` interface: we suggest you to implement this interface in your app's `Application` class, implement the `initCatapush()` method by moving your Catapush initialization code into its body. To complete the migration remember to invoke this new method in your `Application.onCreate(…)` method and pass the instance of the class that implements the new interface as the 2nd parameter of the `Catapush.init(…)` method.

The second change is the removal of the the `Catapush.setNotificationIntent()`, its parameter is now passed as 5th parameter of the `Catapush.init(…)` method.

#### 12.1.2 (31/01/2023)
- Adds the option to skip the updated GMS security providers installation

#### 12.1.1 (10/01/2023)

- Update GMS dependencies
- Update HMS dependencies

#### 12.1.0 (05/12/2022)

- Improved initialization sequence using the new `ICatapushInitializer` interface
- Removed the `Catapush.setNotificationIntent(…)` method and moved its functionality to the `Catapush.init(…)` method
- Fix threading and scheduling issues mainly affecting HUAWEI devices

## Catapush 12.0.x

Catapush 12.0.x targets Android 12.0 (API 31) and requires Android 5.0 (API 21).
- Updated Kotlin to version 1.6.21
- Updated AndroidX Core to 1.7.0
- Updated Dagger to version 2.42
- Updated Firebase Messaging to version 23.0.4
- Updated HMS Push Kit to version 6.3.0.304

This release adds the ability to specify different notification templates for different Android native notification channels.

To deliver a Catapush message to a specific Android native notification channel just set its channel ID string into the `channel` field of the Catapush message you are sending.
Then initialize the Catapush instance in your app providing a list of `NotificationTemplate` instances, one for each native notification channel you'd like to deliver Catapush messages to.

See the [Catapush API docs](DOCUMENTATION_ANDROID_SDK.md#migration-from-catapush-111x) for details on how to migrate your code.

#### 12.0.8 (31/01/2023)
- Adds the option to skip the updated GMS security providers installation

#### 12.0.7 (05/12/2022)
- Fix threading and scheduling issues mainly affecting HUAWEI devices

#### 12.0.6 (11/10/2022)
- Reduce the network usage when receiving a message in the background
- Suppress some unnecessary network activity logs

#### 12.0.5 (29/09/2022)
- Increase initialization timeout from 10s to 60s
- Update HMS dependencies

#### 12.0.4 (22/09/2022)
- Improve received messagge confirmations reliability
- Improve connection availability observation and reactions
- Update GMS dependencies

#### 12.0.3 (25/08/2022)
- Fix `Catapush.logout(…)` implementation

#### 12.0.2 (26/07/2022)
- Improve SDK initialization performances
- Update OkHttp dependency to version 4.10.0

#### 12.0.1 (13/07/2022)

- Adds the `onMessageReceivedConfirmed` method to the `ICatapushEventDelegate` interface, its implementation is **required**
- Improve SDK initialization performances
- Improve the messages delivery performances while the app is in background
- Update GMS and HMS dependencies to improve Android 12 compatibility

#### 12.0.0 (13/05/2022)

- Update com.huawei.hms:push dependency to version 6.3.0.304 to avoid Google Play Store policy violations