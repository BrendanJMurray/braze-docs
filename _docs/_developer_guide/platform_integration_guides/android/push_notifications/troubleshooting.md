---
nav_title: Troubleshooting
platform: Android
page_order: 2
search_rank: 5
---
## Troubleshooting

### Understanding the Braze Workflow
The Firebase Cloud Messaging (FCM) service is Google's infrastructure for push notifications sent to Android applications. Here is the simplified structure of how push notifications are enabled for your users' devices and how Braze is able to send push notifications to them:

#### Step 1: Configuring Your Google Cloud API Key
In the development of your app, you'll need to provide the Braze Android SDK with your Firebase Sender ID. Additionally, you'll need to provide an API Key for server applications to the Braze dashboard. Braze will use this API key when we attempt to send messages to your devices. You will need to ensure that FCM service is enabled in the Google Developer's console as well. __Note__: A common mistake in this step is using an API key for Android applications. This is a different, incompatible API key for the type of access Braze needs.

#### Step 2: Devices Register for FCM and Provide Braze with Push Tokens
In typical integrations, the Braze Android SDK will handle the process of registering devices for FCM capability. This will usually happen immediately upon opening the app for the first time. After registration, Braze will be provided with a FCM Registration ID, which is used to send messages to that device specifically. We will store the Registration ID for that user and that user will become "Push Registered" if they previously did not have a push token for any of your apps.

#### Step 3: Launching a Braze Push Campaign
When a push campaign is launched, Braze will make requests to FCM to deliver your message. Braze will use the API key copied in the dashboard to authenticate and verify that we are allowed to send push notifications to the push tokens provided.

#### Step 4: Removing Invalid Tokens
If FCM informs us that any of the push tokens we were attempting to send a message to are invalid, we remove those tokens from the user profiles they were associated with. If that user has no other push tokens, they will no longer show up as "Push Registered" under the Segments page.

Google has more details about FCM in their [Developers page][6].

### Utilizing the Push Error Logs
Braze provides a log of Push Notification Errors within the "Message Activity Log". This error log provides a variety of warnings which can be very helpful for identifying why your campaigns aren't working as expected.  Clicking on an error message will redirect you to relevant documentation to help you troubleshoot a particular incident.

![Push Error Log][11]

### Troubleshooting Scenarios

#### No "Push Registered" Users Showing in the Braze Dashboard (Prior to Sending Messages)

Ensure that your app is correctly configured to allow push notifications. Common failure points to check include:

##### 1. Incorrect Sender Id

Ensure that the correct FCM Sender ID is included in the `appboy.xml` file. An incorrect Sender ID will lead to `MismatchSenderID` errors reported in the dashboard's Message Activity Log.

##### 2. Braze Registration Not Occurring

Since FCM registration is handled outside of Braze, failure to register can only occur in two places:

1. During registration with FCM
2. When passing the FCM-generated push token to Braze

We recommend setting a breakpoint or logging to ensure that the FCM-generated push token is being sent to Braze. If a token is not being generated correctly or at all, we recommend consulting the [FCM documentation][1].

##### 3. Google Play Services not present

For FCM push to work, Google Play Services must be present on the device. If Google Play Services isn't on a device, push registration will not occur.

__Note:__ Google Play Services is not installed on Genymotion emulators or Android emulators without Google APIs installed.

##### 4. Device not connected to internet

Ensure your device has good internet connectivity and that it isn't sending network traffic through a proxy.

#### Push Notifications Bounced

If a push notification isn't delivered, make sure it didn't bounce by looking in the [developer console][2]. The following are descriptions of common errors that may be logged in the developer console:

##### Error: MismatchSenderID

`MismatchSenderID` indicates an authetication failure. Ensure sure your Firebase Sender ID and FCM API key are correct. See our section on [debugging push registration][3] for more information.

##### Error: InvalidRegistration

`InvalidRegistration` can be caused by a malformed push token.

1. Make sure to pass a valid push token to Braze from FCM by calling [`FirebaseInstanceId.getToken()`][4].

##### Error: NotRegistered

1. `NotRegistered` typically occurs when an app has been deleted from a device. Braze uses `NotRegistered` internally as a signal that an app has been uninstalled from a device.

2. `NotRegistered` may also occur when multiple registrations are occurring and a second registration is invalidating the first token.

#### Push Notifications Sent But Not Displayed on Users' Devices

There are a few reasons why this could be occurring:

##### 1. Application was Force Quit

If you force-quit your application through your system settings, your push notifications will not be sent. Launching the app again will re-enable your device to receive push notifications.

##### 2. AppboyFcmReceiver Not Registered

The AppboyFcmReceiver must be properly registered in `AndroidManifest.xml` for push notifications to appear:

```
  <receiver android:name="com.appboy.AppboyFcmReceiver" android:permission="com.google.android.c2dm.permission.SEND">
  <intent-filter>
    <action android:name="com.google.android.c2dm.intent.RECEIVE" />
    <category android:name="Your Application's Package Name" />
  </intent-filter>
</receiver>
```

For an implementation example, please check out our sample application's [AndroidManifest.xml][15]

**Note** Before Braze Android SDK `2.7.0`, `AppboyFcmReceiver` is known as `AppboyGcmReceiver`. Their functionality is equivalent and integration instructions remain the same, however, all references to `AppboyFcmReceiver` in your `AndroidManifest.xml` and code will need to be replaced by references to `AppboyGcmReceiver`.

##### 3. Firewall is Blocking Push

If you are testing push over Wi-Fi, your firewall may be blocking ports necessary for FCM to receive messages. Please ensure that ports 5228, 5229 and 5230 are open. Additionally, since FCM doesn't specify its IPs, also allow your firewall to accept outgoing connections to all IP addresses contained in the IP blocks listed in [Google's ASN of 15169] [14].

##### 4. Custom Notification Factory Returning Null

If you have implemented a [custom notification factory][16], ensure that it is not returning `null`. This will cause notifications not to be displayed.

#### "Push Registered" Users No Longer Enabled After Sending Messages

There are a few reasons why this could be happening:

##### 1. Application was Uninstalled

Users have uninstalled the application. This will invalidate their FCM push token.

##### 2. Invalid Cloud Messaging API Key
The Cloud Messaging API key provided in the Braze dashboard is invalid. You will need to verify:

- The API key is for server applications. It should look like this in your Google Developers Console:
![Server apps key][9]

- The API key provided is for the same Sender Id that is referenced in your app's `appboy.xml` file.

#### Push Clicks Not Logged

Braze logs push clicks automatically, so this scenario should be comparatively rare.

If push clicks are not being logged, it is possible that push click data has not been flushed to Braze's servers yet. Braze throttles the frequency of its flushes based on the strength of the network connection. With a good network connection, push click data should arrive at the server within a minute in most circumstances.

#### Deep Links Not Working

##### 1. Verify Deep Link configuration

Deep links can be [tested with ADB][17]. We recommend testing your deep link with the following command:

`adb shell am start -W -a android.intent.action.VIEW -d "THE_DEEP_LINK" THE_PACKAGE_NAME`

If the deep link fails to work, the deep link may be misconfigured. A misconfigured deep link will not work when sent through Braze push.

##### 2. Verify Custom Handling Logic

If the deep link [works correctly with ADB][17] but fails to work from Braze push, check whether any [custom push open handling][18] has been implemented. If so, verify that the custom handling code is properly handling the incoming deep link.

[1]: https://firebase.google.com/docs/cloud-messaging/android/client
[2]: #utilizing-the-push-error-log
[3]: #scenario-1-no-push-registered-users-showing-in-the-appboy-dashboard-prior-to-sending-messages
[4]: https://firebase.google.com/docs/reference/android/com/google/firebase/iid/firebaseinstanceid.html#gettoken(java.lang.string, java.lang.string)
[6]: https://firebase.google.com/docs/cloud-messaging/
[9]: {% image_buster /assets/img_archive/serverappskey.png %}
[11]: {% image_buster /assets/img_archive/message_activity_log.png %}
[14]: http://tcpiputils.com/browse/as/15169
[15]: https://github.com/Appboy/appboy-android-sdk/blob/master/droidboy/src/main/AndroidManifest.xml
[16]: #custom-displaying-notifications
[17]: https://developer.android.com/training/app-indexing/deep-linking.html#testing-filters
[18]: #custom-handling-push-receipts-and-opens