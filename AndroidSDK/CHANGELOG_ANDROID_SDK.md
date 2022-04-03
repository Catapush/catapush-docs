![Catapush Logo](images/catapush_logo.png)

# Catapush Android SDK Changelog

## Catapush 11.2.x

Catapush 11.2.x targets Android 11.0 (API 30) and requires Android 5.0 (API 21).

This release adds the ability to specify different notification templates for different Android native notification channels.

To deliver a Catapush message to a specific Android native notification channel just set its channel ID string into the `channel` field of the Catapush message you are sending.
Then initialize the Catapush instance in your app providing a list of `NotificationTemplate` instances, one for each native notification channel you'd like to deliver Catapush messages to.

See the [Catapush API docs](DOCUMENTATION_ANDROID_SDK.md#migration-from-catapush-111x) for details on how to migrate your code.

#### 11.2.1 (03/04/2022)

- Update com.huawei.hms:push dependency to version 6.3.0.304 to avoid Google Play Store policy violations

#### 11.2.0 (08/03/2022)

- Adds message delivery to different Android native notification channels

## Catapush 11.1.x

Catapush 11.1.x targets Android 11.0 (API 30) and requires Android 5.0 (API 21).

This release contains updated dependencies and a new methods in the Catapush interface.

#### 11.1.5 (02/04/2022)

- Update com.huawei.hms:push dependency to version 6.3.0.304 to avoid Google Play Store policy violations

#### 11.1.4 (29/12/2021)

- Improve message opened notifications delivery to Catapush servers to avoid duplicated notifications
- Added a workaround to a known Android 11 network-related issue, see the [issue on Google issue tracker](https://issuetracker.google.com/issues/175055271) for more details

#### 11.1.3 (19/10/2021)

- Fix Proguard configuration to avoid runtime crashes caused by missing `org.xmlpull.v1.*` classes

#### 11.1.2 (23/09/2021)

- Added `Catapush.getMessageById(…)` to retrieve  a `CatapushMessage` given its ID

#### 11.1.1 (22/09/2021)

- Catapush UI components: minor improvements and bug fixes

#### 11.1.0 (15/09/2021)

- Migrated the methods `Catapush.getMessagesAsDataSourceFactory()`, `Catapush.getMessagesWithoutChannelAsDataSourceFactory()` and `Catapush.getMessagesFromChannelAsDataSourceFactory(…)` to AndroidX Paging 3.x and renamed them to `Catapush.getMessagesAsPagingDataFlowable(…)`, `Catapush.getMessagesWithoutChannelAsPagingDataFlowable(…)` and `Catapush.getMessagesFromChannelAsPagingDataFlowable(…)` respectively
- Added `Catapush.logout(…)` to delete the current user and messages from the local SDK database and invalidate the authentication session server-side
- Updated AndroidX Paging to version 3.0.1
- Updated Dagger to version 2.39.1
- Update Kotlin to version 1.5.30

## Catapush 11.0.x

Catapush 11.0.x targets Android 11.0 (API 30) and requires Android 5.0 (API 21).

#### 11.0.5 (03/09/2021)

- Improvements to the `Catapush.notifyMessageOpened(…)` implementation

#### 11.0.4 (04/08/2021)

- Added `Catapush.countMessages(…)` to get the message count of the current user
- Added `Catapush.sendMessage(…)` to send a message from the current user to the backend

#### 11.0.3 (29/06/2021)

- Update com.huawei.hms:push dependency to version 5.3.0.304 to address an implicit PendingIntent vulnerability

#### 11.0.2 (24/06/2021)

- Update com.huawei.hms:push dependency to version 5.3.0.301

#### 11.0.1 (08/06/2020)

- Improve disconnection and reconnection performance after a network loss
- Catapush UI components: minor improvements and bug fixes

#### 11.0.0 (18/05/2021)

- Updated Android target SDK level to 30 and minimum SDK level to 21
- The `CatapushReceiver.onNotificationClicked(…)` and `CatapushTwoWayReceiver.onNotificationClicked(…)` callbacks have been removed and replaced with the new `Catapush.setNotificationIntent(IIntentProvider)` to avoid delays on your app launch after a user tap on the Catapush notifications. See the [Catapush API docs](DOCUMENTATION_ANDROID_SDK.md#migration-from-catapush-102x) for details
- Updated 3rd party library dependencies and removed deprecated/unmaintained libraries