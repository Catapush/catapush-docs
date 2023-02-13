## Google Mobile Services (GMS) and Firebase Cloud Messaging (FCM) configuration

Catapush uses Firebase Cloud Messaging, the native push notification service available on GMS-enabled Android devices (Google Mobile Services), as a component of its reliable and secure message delivery system.

These instructions will guide you on how to setup your Firebase project and your Catapush dashboard to enable the background delivery on GMS-enabled Android devices.

<br/>

**1** - Visit [console.firebase.google.com](https://console.firebase.google.com) and click "Create a project"/"Add project", then type a project name and choose a country/region

![Firebase Console](images/gms_fcm_step1.png)

**2** - Select "Add app" then click on the Android logo.

![Project home page](images/gms_fcm_step2.png)

**3** - Fill the form with the requested info: type your app package in "Android package name" and add an optional "App nickname".<br/>
Finally, confirm with "Register app" button.

![Register the Android app](images/gms_fcm_step3.png)

**4** - Now click on "Download google-services.json" button and press "Next".<br/>
Add this file in your Android Studio project `/app` subfolder.<br/>
Make sure you've configured the `classpath com.google.gms:google-services:{version}` in the plugin section of the `build.gradle` file of your project.<br/>
Complete step 3 by clicking "Next" and step 4 by clicking "Skip this step" if you want to skip the app verification procedure.

![Download config file](images/gms_fcm_step4.png)

**5** - Click on the gear icon then select "Project settings".

![Project settings](images/gms_fcm_step5.png)

**6** - In the "Service accounts" tab click "Create service account".

![Service account](images/gms_fcm_step6.png)

**7** - Click "Generate new private key" then confirm clicking "Generate key".<br/>
Securely store the JSON file containing the key.

![Service account](images/gms_fcm_step7.png)

**8** - Visit [www.catapush.com/panel/dashboard](https://www.catapush.com/panel/dashboard) and login with your Catapush credentials.<br/>
Select your app from the side menu and select "Platforms".<br/>
Enable "Use Service Account JSON", then select the file you downloaded at the previous step. Press "Save".

![Catapush dashboard, configuring platform](images/gms_fcm_step8.png)

<br/>

Firebase Cloud Messaging setup is now complete.

You can now proceed to the [Catapush Android SDK Documentation](DOCUMENTATION_ANDROID_SDK.md) to integrate the Catapush SDK for Android in your app.