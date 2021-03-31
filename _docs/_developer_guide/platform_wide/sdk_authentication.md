---
nav_title: SDK Authentication
page_order: 5
hidden: true
description: "Verify the identity of SDK requests"
platform:
  - ios
  - android
  - web
---

<br><br>
{% alert important %}
This feature is in _beta_ and only available for Android, Web, and iOS SDKs.
{% endalert %}

# SDK Authentication

SDK Authentication ensures that no one can impersonate or create Braze users using your SDK API Keys. Braze SDKs, like other client-side libraries, use a public API key to identify your app. To ensure that Braze only trusts requests from authenticated users, this feature allows _you_ to supply Braze with secure, cryptographic proof. After all, your server is the only one who would know whether a user is who they claim to be in your app (using cookies or other persistent session data).

When enabled, this feature will prevent unauthorized requests that use your app's SDK API Keys for logged in users, including:
- Sending custom events, attributes, purchases, and session data
- Creating new users in your Braze App Group
- Updating standard user profile attributes
- Receiving or triggering messages

## Getting Started

There are four high-level steps to get started:

1. [Server-Side Integration][1] - Generate a public and private key-pair, and use your private key to create a JWT (_JSON Web Token_) for the current logged-in user.<br><br>
2. [SDK Integration][2] - Enable this feature in the Braze SDK and supply the JWT Token generated from your server.<br><br>
3. [Adding Public Keys][3] - Add your _public key_ to the Braze Dashboard in the "Manage App Group" page.<br><br>
4. [Toggle Enforcement within the Braze Dashboard]({{site.baseurl}}/developer_guide/platform_wide/sdk_authentication/#braze-dashboard) - Toggle this feature's enforcement within the Braze Dashboard on an app-by-app basis.

## Server-Side Integration {#server-side-integration}

### Generate a Public/Private Key-pair {#generate-keys}

Generate an RSA public/private key-pair. The Public Key will eventually be added to the Braze Dashboard, while the Private Key should be stored securely on your server.

We recommend an RSA Key with 2048 bits for use with the RS256 JWT algorithm.

{% alert warning %}
Remember to keep your private keys _private_. Never expose or hard-code your private key in your app or website. Anyone who knows your private key can impersonate or create users on behalf of your application.
{% endalert %}

### Create a JSON Web Token for the current user {#create-jwt}

Once you have your private key, your server-side application should use it to return a JWT to your app or website for the currently logged-in user.

Typically, this logic could go wherever your app would normally request the current user's profile; such as a login endpoint or wherever your app refreshes the current user's profile.

When generating the JWT, the following fields are expected:

**JWT Header**

| Field | Required | Description                         |
| ----- | -------- | ----------------------------------- |
| `alg` | **Yes**  | The supported algorithm is `RS256`. |
| `typ` | **Yes**  | The type should equal `JWT`.        |

{: .reset-td-br-1 .reset-td-br-2 .reset-td-br-3}

**JWT Payload**

| Field | Required | Description                                                                            |
| ----- | -------- | -------------------------------------------------------------------------------------- |
| `sub` | **Yes**  | The "subject" should equal the User ID you supply Braze SDKs when calling `changeUser` |
| `exp` | **Yes**  | The "expiration" of when you want this token to expire.                                |
| `aud` | No       | The "audience" claim is optional, and if set should equal `braze`                      |
| `iss` | No       | The "issuer" claim is optional, and if set should equal your SDK API Key.              |

{: .reset-td-br-1 .reset-td-br-2 .reset-td-br-3}

### JWT Libraries

To learn more about JSON Web Tokens, or to browse the [many open source libraries](https://jwt.io/#libraries-io) that simplify this signing process, check out [https://jwt.io](https://jwt.io).

## SDK Integration {#sdk-integration}

### Enable this feature in the Braze SDK.

When this feature is enabled, the Braze SDK will append the current user's last known JWT to network requests made to Braze Servers.

{% alert note %}
Don't worry, initializing with this option alone won't impact data collection in any way, until you start [enforcing authentication](#braze-dashboard).
{% endalert %}

{% tabs %}
{% tab Javascript %}
When calling `appboy.initialize`, set the optional `sdkAuthentication` property to `true`.
```javascript
appboy.initialize("YOUR-API-KEY-HERE", {
  baseUrl: "YOUR-SDK-ENDPOINT-HERE",
  sdkAuthentication: true,
});
```
{% endtab %}
{% tab Java %}
When configuring the Appboy instance, call `setIsSdkAuthenticationEnabled` to `true`.
```java
AppboyConfig.Builder appboyConfigBuilder = new AppboyConfig.Builder()
    .setIsSdkAuthenticationEnabled(true);
Appboy.configure(this, appboyConfigBuilder.build());
```
{% endtab %}
{% tab Objective-C %}
```objc
[Appboy startWithApiKey:@"YOUR-API-KEY"
            inApplication:application
        withLaunchOptions:launchOptions
        withAppboyOptions:@{ABKEnableSDKAuthenticationKey : @YES}];
```
{% endtab %}
{% tab Swift %}
```swift
Appboy.start(withApiKey: "YOUR-API-KEY",
                 in:application,
                 withLaunchOptions:launchOptions,
                 withAppboyOptions:[ ABKEnableSDKAuthenticationKey : true ])
```
{% endtab %}
{% endtabs %}

### Set the current user's JWT Token

Whenever your app calls the Braze SDK's `changeUser` method, also supply the JWT token that was [generated server-side][4].

You can also update the token to refresh the token mid-session.

{% tabs %}
{% tab Javascript %}
Supply the JWT Token when calling `appboy.changeUser`:

```javascript
appboy.changeUser("NEW-USER-ID", {
  sdkAuthenticationToken: "JWT-TOKEN-FROM-SERVER",
});
```

Or, when you have refreshed the user's token mid-session:

```javascript
appboy.setAuthenticationToken("NEW-JWT-TOKEN-FROM-SERVER");
```
{% endtab %}
{% tab Java %}

Supply the JWT Token when calling `appboy.changeUser`:

```java
Appboy.getInstance(this).changeUser("NEW-USER-ID", "JWT-TOKEN-FROM-SERVER");
```

Or, when you have refreshed the user's token mid-session:

```java
Appboy.getInstance(this).setSdkAuthenticationSignature("NEW-JWT-TOKEN-FROM-SERVER");
```
{% endtab %}
{% tab Objective-C %}

Supply the JWT Token when calling `changeUser`:

```objc
[[Appboy sharedInstance] changeUser:@"userId" sdkAuthSignature:@"signature"];
```

Or, when you have refreshed the user's token mid-session:

```objc
[[Appboy sharedInstance] setSdkAuthenticationSignature:@"signature"];
```
{% endtab %}
{% tab Swift %}

Supply the JWT Token when calling `changeUser`:

```swift
Appboy.sharedInstance()?.changeUser("userId", sdkAuthSignature: "signature")
```
Or, when you have refreshed the user's token mid-session:

```swift
Appboy.sharedInstance()?.setSdkAuthenticationSignature("signature")
```
{% endtab %}
{% endtabs %}

### Register a callback function for invalid tokens {#sdk-callback}

When this feature is set as ["Required"](#enforcement-options), the following scenarios will cause SDK requests to be rejected by Braze:
- JWT was expired by the time is was received by the Braze API
- JWT was empty or missing
- JWT failed to verify for the Public Keys you uploaded to the Braze Dashboard

When the SDK requests fail for one of these reasons, a callback function you supply will be invoked with a relevant [Error Code][9]. Failed requests will periodically be retried until your app supplies a new valid JWT.

To fix any invalid tokens and continue to successfully sync data to Braze, your app should request a new JWT from your server and supply Braze's SDK with this new valid token.

{% alert tip %}
These callback methods are a great place to add your own monitoring or error-logging service to keep track of how often your Braze requests are being rejected.
{% endalert %}

{% tabs %}
{% tab Javascript %}
```javascript
appboy.onSdkAuthenticationFailure(async (errorEvent) => {
  const updated_jwt = await getNewTokenSomehow(errorEvent);
  // TODO: optionally log to your error-reporting service
  appboy.setAuthenticationToken(updated_jwt);
});
```
{% endtab %}
{% tab Java %}
```java
Appboy.getInstance(this).subscribeToSdkAuthenticationFailures(errorEvent -> {
    String newToken = getNewTokenSomehow(errorEvent);
    // TODO: optionally log to your error-reporting service
    Appboy.getInstance(getContext()).setSdkAuthenticationSignature(newToken);
});
```
{% endtab %}
{% tab Objective-C %}
```objc
[[Appboy sharedInstance] setSdkAuthenticationDelegate:delegate];

// Method to implement in delegate
- (void)handleSdkAuthenticationError:(ABKSdkAuthenticationError *)authError {
  NSLog(@"Invalid SDK Authentication signature.");
  [[Appboy sharedInstance] setSdkAuthenticationSignature:@"signature"];
}
```
{% endtab %}
{% tab Swift %}
```swift
Appboy.sharedInstance()?.setSdkAuthenticationDelegate(delegate)

// Method to implement in delegate
func handle(_ authError: ABKSdkAuthenticationError?) {
        print("Invalid SDK Authentication signature.")
        Appboy.sharedInstance()?.setSdkAuthenticationSignature("signature")
    }

```
{% endtab %}
{% endtabs %}

The `errorEvent` argument passed to this callback will contain the following information:

| Property | Description |
| -------- | ----------- |
| `reason` | A description of why the request failed. |
| `error_code` | An internal error code used by Braze. |
| `user_id` | The user ID from which the request failed. |
| `signature` | The JWT that failed.|
{: .reset-td-br-1 .reset-td-br-2}

## Adding Public Keys {#key-management}

In the "Manage App Group" page of the dashboard, add your Public Key to a specific app in the Braze Dashboard. Each app supports up to 3 Public Keys. Note that the same Public/Private keys may be used across apps.

To add a Public Key:
1. Choose the app in the left-hand side menu
2. Click the **Add Public Key** button within the SDK Authentication settings
3. Paste in the Public Key, and add an optional description
4. After saving your changes, the key will appear in the list of Public Keys.

To delete a key, or to promote a key to the Primary key, choose the corresponding action in the overflow menu next to each key.

## Enabling in the Braze Dashboard {#braze-dashboard}

Once your [Server-side Integration][1] and [SDK Integration][2] are complete, you can begin to enable this feature for those specific apps.

Keep in mind, SDK requests will continue to flow as usual - without authentication - _unless_ the app's SDK Authentication setting is switched to **required** in the Braze Dashboard.

Should anything go wrong with your integration (i.e. your app is incorrectly passing tokens to the SDK, or your server is generating invalid tokens), simply **disable** this feature in the Braze Dashboard and data will resume to flow as usual, without verification.

### Enforcement Options {#enforcement-options}

In the Dashboard `App Settings` page, each app has three SDK Authentication states which control how Braze verifies requests.

![dashboard][8]

| Setting| Description|
| ------ | ---------- |
| **Disabled** | Braze will not verify the JWT supplied for a user. (Default Setting)|
| **Optional** | Braze will verify requests for logged-in users, but will not reject invalid requests. |
| **Required** | Braze will verify requests for logged-in users and will reject invalid JWTs.|
{: .reset-td-br-1 .reset-td-br-2}

The "**Optional**" setting is a useful way to monitor the potential impact this feature will have on your app's SDK traffic.

Invalid JWT signatures will be reported in both **Optional** and **Required** states, however only the **Required** state will reject SDK requests causing apps to retry and request new signatures.

## Error Codes {#error-codes}

| Error Code| Error Reason | Description |
| --------  | ------------ | ---------  |
| 10 | `EXPIRATION_REQUIRED` | Expiration is a required field for Braze usage.|
| 20 | `DECODING_ERROR` | Non-matching public key or a general uncaught error.|
| 21 | `SUBJECT_MISMATCH` | The expected and actual subjects are not the same.|
| 22 | `EXPIRED` | The token provided has expired.|
| 23 | `INVALID_PAYLOAD` | The token payload is invalid.|
| 24 | `INCORRECT_ALGORITHM` | The algorithm of the token is not supported.|
| 25 | `PUBLIC_KEY_ERROR` | The public key could not be converted into the proper format.|
| 26 | `MISSING_TOKEN` | No token was provided in the request.|
| 27 | `NO_MATCHING_PUBLIC_KEYS` | No public keys matched the provided token.|
| 28 | `PAYLOAD_USER_ID_MISMATCH` | Not all user ids in the request payload match as is required.|
{: .reset-td-br-1 .reset-td-br-2, .reset-td-br-3}

## Frequently Asked Questions {#faq}

#### Which SDKs support Authentication? {#faq-sdk-support}

Braze Web SDK (vX.Y), Android SDK (vX.Y), and iOS SDK (vX.Y) include support for this feature.

#### Can I use this feature on only some of my apps? {#faq-app-by-app}

Yes, this feature can be enabled for specific apps and doesn't need to be used on all of your apps.

#### What happens to users who are still on older versions of my app? {#faq-sdk-backward-compatibility}

When you begin to enforce this feature, requests made by older app versions will be rejected by Braze and retried by the SDKs. Once users upgrade their app to a supported version, those enqueued requests will begin to be accepted again.

If possible, you should push users to upgrade as you would for any other mandatory upgrade. Alternatively, you can keep the feature ["optional"][6] until you see that an acceptable percentage of users have upgraded.

#### What expiration should I use when generating JWT tokens? {#faq-expiration}

We recommend using the higher value of: average session duration, session cookie/token expiration, or the frequency at which your application would otherwise refresh the current user's profile.

#### What happens if a JWT expires in the middle of a user's session?

Should a user's token expire mid-session, the SDK has a [callback function][7] it will invoke to let your app know that a new JWT token is needed to continue sending data to Braze.

#### What happens if my server-side integration breaks and I can no longer create a JWT?

If your server is not able to provide JWT tokens or you notice some integration issue, you can always disable the feature in the Braze Dashboard.

Once disabled, any pending failed SDK requests will eventually be retried by the SDK and accepted by Braze.

#### Why does this feature use Public/Private keys instead of Shared Secrets?

When using Shared Secrets, anyone with access to that shared secret (i.e. the Braze Dashboard page) would be able to generate tokens and impersonate your end-users.

Instead, we use private Keys so that not even Braze Employees (let alone your Dashboard users) could discover your private Keys.

[1]: #server-side-integration
[2]: #sdk-integration
[3]: #key-management
[4]: #braze-dashboard
[4]: #create-jwt
[5]: #todo
[6]: #enforcement-options
[7]: #sdk-callback
[8]: {% image_buster /assets/img/sdk-auth-settings.png %}
