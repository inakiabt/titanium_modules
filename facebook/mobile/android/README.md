Titanium Facebook Module for Android
=====================================

Overview
------------
* This module is based on Facebook's latest Android SDK (3.5.2, currently), and in accordance with Facebook's samples and recommendations.
* The API is almost identical to the iOS Facebook module I wrote, differing in some places where required.
* Currently only login/logout and Share Dialog functionality is supported, but extending it is very straight forward with the current architecture.
* Unfortunately the Facebook SDK cannot be used as-is for Titanium Android due to [TIMOB-11360](https://jira.appcelerator.org/browse/TIMOB-11360), and must be hacked a bit.
I kept the modifications to a minimum, and my fork can be found here: [Facebook Android SDK fork for Titanium](https://github.com/mokesmokes/facebook-android-sdk)

Improvements over Facebook Module 3.0.x
---------------------------------------
* Much faster login, no Facebook app switch.
* More reliable: latest SDK, latest error handling, compliant with Facebook docs and samples.
* Share Dialog via the Facebook app - share links and Graph Actions, with people tagging, etc - and you don't even need the user to log in.

Installation Details
--------------------
* In tiapp.xml or AndroidManifest.xml you must declare the following two activities inside the `<application>` node
```
<activity android:name="com.facebook.LoginActivity" android:theme="@android:style/Theme.Translucent.NoTitleBar" android:label="YourAppName"/>
<activity android:name="facebook.FacebookModuleActivity" android:theme="@android:style/Theme.Translucent.NoTitleBar" android:label="YourAppName"/>
```
One of these activities is Facebook's and required for the SDK, the second is an activity spawned by the Titanium Facebook module.
This activity is required due to [TIMOB-15443](https://jira.appcelerator.org/browse/TIMOB-15443) . The Facebook SDK requires access
to the full activity (or fragment) lifecycle, and thus the module needs to spawn this activity which handles all the internal Facebook state changes.
* You must also reference the string containing your Facebook app ID, inside the `<application>` node as well: 
`<meta-data android:name="com.facebook.sdk.ApplicationId" android:value="@string/app_id"/>`
* The app id goes into the the file `/platform/android/res/values/strings.xml`, where you should define
`<resources><string name="app_id">1234567890123456</string></resources>`, where the number is of course the app ID. The app ID is not set programmatically.

Build notes
------------
* Your Java build path must include the file `android-support-v4.jar`, which is already in the Titanium SDK.
* Note that the built Facebook SDK (from my fork of course) is in the `lib` folder
* Note: I built and tested it with Titanium SDK 3.1.3

Module Versioning
-----------------

x.y.zt, where x.y.z is the Facebook iOS SDK version, t denotes the Titanium module version for this SDK.
This module version is 3.5.21 - i.e. uses Facebook Android SDK 3.5.2

Module API
----------

The module tries to stick to the original Titanium Facebook Android module API (distributed with Ti SDK 3.1.0).
However, there are some differences, so read carefully.

*	`appid` - parameter unused. Instead, see the installation notes above.
*	`forceDialogAuth` - parameter unused.
*	The login button functionality is for now removed. It makes no sense to use a button besides the Facebook branded buttons in the SDK, and will be easy to do after [TIMOB-11360](https://jira.appcelerator.org/browse/TIMOB-11360) is resolved. 
*	"reauthorize" is not implemented, requestNewReadPermissions and requestNewPublishPermissions in a future rev.
*	Added the method `fb.initialize()`, which must be called after you setup your `login` and `logout` listeners. 
This provides flexibility, and allows you to setup the listeners late in the app. 

Events and error handling
-------------------------

The `login` event behavior is normalized to behave similarly to iOS. If you're logged in after initialization,
you *will* get a `login` event. It's important to understand the reason for this: The module checks the SDK for a cached token.
However, there is a possibility this token is invalid without the SDK's knowledge (e.g. the user changed her password, etc).
Thus, the only way to verify the token's and session's validity is to make a call to Facebook's servers. 
You will not get the `login` event if there was no cached session, or if your initial requested permissions are not a subset of those of the cached session. 
In that case - the module will close the session. Below is the integrated flow for both Android and iOS:
```javascript
var fb = require('facebook');
var fb.permissions = ['email', 'user_birthday'];
// now set up listeners
fb.addEventListener('login', function(e) {
	if(e.success) {
		// do your thang.... 
	} else if (e.cancelled) {
		// login was cancelled, just show your login UI again
	} else if (e.error) {
		if (Ti.Platform.name === 'iPhone OS') {
			// For all of these errors - assume the user is logged out
			// so show your login UI
			if (e.error.indexOf('Go to Settings') === 0){
				// alert: Go to Settings > Facebook and turn ON My Cool App 
				alert(e.error + 'My Cool App')
			} else if (e.error.indexOf('Session Login Error') === 0){
				// Session was invalid - e.g. the user deauthorized your app, etc
				alert('Please login again.');
			} else if (e.error.indexOf('OTHER:') !== 0){
				// another error that may require user attention
				alert (e.error);
			} else {
				// This must be an error message that the user was already notified about
				// Due to the automatic error handling in the graph call
				// Don't surface it again
			}
		} else {
			// Android
			if (e.error.indexOf('session') >= 0){
				alert('Please make sure you are correctly logged into the Facebook app, and try again.');
			} else if (e.error.indexOf('no alert')  >=0){
				// do nothing
			} else if (e.error.indexOf('permission') >= 0){
				alert('Permission error');
			} else if (e.error.indexOf('bug') >= 0){
				alert(errorString);
			} else {
				alert('Please check your network connection and try again');
			}			
		}
	} else {
		// if not success, nor cancelled, nor error message then pop a generic message
		// e.g. "Check your network, etc" . This is per Facebook's instructions
	}
});
		
fb.initialize(); // the module will do nothing prior to this. This enabled you to set up listeners anywhere in the app		
```

Share Dialog
-------------

See the [Facebook docs](https://developers.facebook.com/docs/android/share-dialog/)
Use it! You don't need permissions, you don't even need the user to be logged into your app with Facebook!
*	First check if you can use it - check the properties `fb.canPresentShareDialog` or `fb.canPresentOpenGraphActionDialog`, depending upon your desired sharing action.
Unfortunately this is less concise than the iOS module, due to the SDK.
*	To share a user's status just call `fb.share({});` Note: this is documented for iOS but not Android, so use with caution.
*	To share a link call `fb.share({url: 'http://example.com' });`
*	To post a graph action call:

```javascript
fb.share({url: someUrl, namespaceObject: 'myAppnameSpace:graphObject', objectName: 'graphObject', imageUrl: someImageUrl, 
		title: aTitle, description: blahBlah, namespaceAction: 'myAppnameSpace:actionType', placeId: facebookPlaceId}`
```
For the graph action apparently only placeId is optional. Note: There appears to be [a bug](https://developers.facebook.com/bugs/363119770486799) currently where place tagging doesn't work for Android.


Feel free to comment and help out! :)
-------------------------------------
