# Table of Contents
- [What is the OOB-Android SDK?](#What-is-the-OOB-Android-SDK?)
- [Environments](#Environments)
- [Requirements](#Requirements)
- [Integration](#Integration)
    - [Import SDK](#Import-SDK)
    - [Register Device](#Register-Device)
    - [Authenticate Payment](#Authenticate-Payment)
    - [Here is how you should handle errors](#Here-is-how-you-should-handle-errors)
- [Other requirements](#Other-requirements)
    - [Building the UI](#Building-the-UI)
    - [Listening to webhooks](#Listening-to-webhooks)
- [Performing SCA as requested](#Performing-SCA-as-requested)
- [Contact](#Contact)
- [License](#License)
***

# What is the OOB-Android SDK?

Checkout.com’s Out-of-Band (OOB) Authentication SDK enables Android apps to authenticate users for digital transactions (3DS). Our OOB SDK enables you to offer a modern 3DS challenge method alongside existing challenge types, such as OTP. 

OOB involves utilising a secondary factor, such as your phone’s banking app, to authenticate a payment and encompasses the fundamental processes of device binding and transaction authentication. Our OOB SDK is focused on this use case. 

***

# Environments

The Android SDK supports the following environments: Sandbox and Production. You’ll define the environment to use when you initialise the SDK. 

To use Sandbox and Production environments, you must have begun onboarding for card issuing with Checkout.com. During your onboarding, you'll receive client credentials, which you'll need to handle on your back end for authentication to access the environments.

***

# Requirements

The SDK is provided as a native Android dependency. If you're working on a hybrid project, consult your hybrid platform's documentation for instructions on integrating native third-party SDKs.

It's important to ensure Strong Customer Authentication (SCA) is enabled for your users. While we handle various security measures, you're responsible for performing SCA on your users as specified.

Each authentication session has the ability to generate multiple tokens simultaneously for various systems. For instance, for sign-in purposes or to obtain an SDK session token or an internal authentication token. However, only one SDK token can be generated from each SCA flow requested.

***

# Integration

Our SDK is designed to seamlessly integrate with all our backend services for features relevant to OOB authentication. Whether you're registering a device for OOB or authenticating payments via your Android app, our SDK simplifies the process, making it effortless for you to offer this functionality to your users.

Checkout.com’s OOB SDK is available through [Maven Central](https://mvnrepository.com/artifact/com.checkout/checkout-sdk-oob-android).

## Import SDK

Use Gradle to import the SDK into your app:

```gradle
// in project build.gradle
repositories {
    mavenCentral()
}

// in app level build.gradle
dependencies {
    implementation("com.checkout:checkout-sdk-oob-android:$checkout_sdk_oob_vesrion")
}
```
To begin utilising the SDK functionality, initialise the primary object, granting access to its features:
```Kotlin
val oobManager = CheckoutOOBManager(context = context, environment = Environment.SANDBOX)
```

## Register Device

This is the first core flow whereby the device and app must be registered for OOB. Note that a given card can only be registered for OOB for a single device/app combination. If you register another device for the same card it will overwrite the previously bound device/app combination.

The register device stage can likely be done as a background process during onboarding of a user in your Android app, but you could offer a button that enables a user to do this manually if you wish. Note that before registering a device/app/card combination for OOB, you must perform SCA on the cardholder. 

To establish device binding, register your cardholder's mobile device as follows:

```Kotlin
// Prepare the device registration request
val deviceRegistrationRequest = DeviceRegistration(
    token = "CardHolder access token",
    cardId = "CardID",
    applicationID =  "Unique Application ID",
    phoneNumber = PhoneNumber(countryCode = "+230", number = "52520782"),
    cardLocale = CardLocale.EN,
)

// Initialise device registration with OOB SDK
val registerDeviceResult = oobManager.registerDevice(deviceRegistrationRequest)
registerDeviceResult.collect { result ->
    result.onSuccess {
        // Success result (Type:Unit)
    }.onFailure {
        val message = provideErrorMessage(it)
        // See below Here is how you should handle errors section
    }
}
```

## Authenticate Payment

This is the core authentication flow where the user will approve or reject the transaction within your Android app. This flow will begin from our card issuing backend, where you will need to listen to the webhook provided to obtain transaction details to inject into the SDK (more on webhooks later).

Note that before authenticating a payment using OOB, you must perform SCA on the cardholder. 


Once you have the data from your backend, verify online card transactions made by your cardholders like so:
```Kotlin
// Prepare the authentication request
val authenticationRequest = Authentication(
    transactionId = "Acs Transaction ID",
    token = "CardHolder access token",
    cardId = "cardID",
    method = Method.OOB_BIOMETRICS,
    decision = Decision.ACCEPTED,
)

// Initialise transaction authentication with OOB SDK
val authenticatePaymentResult = oobManager.authenticatePayment(data)
authenticatePaymentResult.collect { result ->
    result.onSuccess {
        // Success result (Type:Unit)
    }.onFailure {
        val message = provideErrorMessage(it)
        // See below Here is how you should handle errors section
   }
}
```

## Here is how you should handle errors

```Kotlin
private fun provideErrorMessage(throwable: Throwable): String {
    return when (throwable) {
        is ConfigurationError -> {
            when (throwable.errorCode) {
                E1000_INVALID_CARD_ID -> "Invalid card ID"
                E1001_INVALID_TOKEN -> "Invalid token"
                E1005_INVALID_TRANSACTION_ID -> "Invalid transaction ID"
                else -> throwable.errorMessage
            }
        }
       is UnprocessableContentError ->
            "Server failure with error code: ${throwable.errorCode} and error message: ${throwable.errorMessage}"
       is HttpError -> "error code ${throwable.errorCode}, message: ${throwable.errorMessage}"
       is ConnectivityError -> {
            when (throwable.errorCode) {
                E1006_NO_INTERNET_CONNECTION -> "No internet connection"
                E1007_CONNECTION_FAILED -> "Network connection failed. Please check internet connection"
                E1008_CONNECTION_TIMEOUT -> "Network connection timeout. Please try again later"
                else -> "Unknown connectivity error"
            }
        }
        else -> throwable.message ?: "Unknown error from the SDKKit. Please contact support."
    }
}
```

***

# Other requirements

## Building the UI
Our SDK remains light on UI objects so that you can create the experience you desire. You will therefore need to build the UI for the transaction authentication screen (the screen that shows during the authenticate payment stage), as well as any associated UI (if you so wish) for the register device process.

## Listening to webhooks
In order to obtain transaction information to inject into our SDK, you must be able to listen to webhooks from our issuing backend. We recommend that you implement a push notification to the mobile app and use the data from the webhook to inform the cardholder about the details of the transaction they are authenticating for.

You should provide an endpoint that receives a HTTP POST and returns a 200 OK response on receipt of the webhook. The transaction will not complete unless we receive acknowledgement.

The JSON payload you should expect looks like this:

```Kotlin
{
    "acsTransactionId": "bea260bb-c183-4d74-8bc9-421fe3aa2658",
    "cardId": "ea3ze38kfj3g6ktv6wn14aaylvmdv3qn",
    "applicationId": "4283-23344-7884ea-474752",
    "transactionDetails": {
        "maskedCardNumber": "**** **** **** 1234",
        "purchaseDate": "2024-12-24T17:32:28.000Z",
        "merchantName": "ACME Corporation",
        "purchaseAmount": 1,
        "purchaseCurrency": "GBP"
    },
    "threeDSRequestorAppURL": "https://requestor.netcetera.com?transID=bea260bb-c183-4d74-8bc9-421fe3aa2658"
}
```

The applicationId property is used to identify which mobile app to send a push notification to. 

We will also provide a webhook in the event that the transaction is cancelled, due to timeout or any other cause. We also expect a 200 OK acknowledgement for this webhook. In this case, the JSON payload will look like this:

```Kotlin
{ "transactionId": "bea260bb-c183-4d74-8bc9-421fe3aa2658" }
```
In both cases we will also include an Authorization header in the request containing the value provided to us at client onboarding. This is used to verify that the request comes from us.

***

# Performing SCA as requested

You must perform SCA (strong customer authentication) on the cardholder before device registration and payment authentication flows. This would likely be when the user enters the app, but could be at other stages.

***

# Contact

For Checkout.com issuing clients, please email issuing_operations@checkout.com for any questions.

***

# License

This SDK is released under the [MIT licence](https://github.com/checkout/oob-authentication-android/blob/main/LICENSE.txt).

