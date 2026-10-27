# SimplePay Android SDK

The official Android SDK for SimplePay - a modern payment processing platform. 
This SDK enables seamless integration of SimplePay's payment services into your Android application, providing a secure and user-friendly payment experience.

## Features
- Easy integration with minimal setup
- 3DS two-factor authentication
- Environment-aware security checks (device integrity checks enforced only in production)
- Various payment methods:
  - Bank Card (card payments)
  - Simple Wallet (saved cards, Simple App payment)
  - Qvik Transfer (through banking app, QR payment)
  - Google Pay
- Adaptive layouts, with landscape and foldable device support
- Full accessibility support including TalkBack descriptions, text and display size scaling
- Dark mode

## Architecture & Design

The SDK is built following industry best practices:

- **Clean Architecture** - Clear separation of concerns with distinct layers (presentation, domain, data)
- **MVVM Pattern** - Model-View-ViewModel for reactive UI and testable business logic
- **Dependency Injection** - Koin for modular and maintainable code
- **Coroutines & Flow** - Modern asynchronous programming for smooth performance
- **Type-safe navigation** - Jetpack Compose Navigation for seamless screen transitions

This architecture ensures the SDK is scalable, maintainable, and follows Android development best practices.

## SDK Requirements

- **Minimum SDK**: 24 (Android 7.0)
- **Target SDK**: 36 (Android 16.0)

- Kotlin 2.0.21+
- Android Gradle Plugin 8.1+
- KSP for annotation processing (if using libraries like Room, Hilt, etc.)
- OkHttp 4.12.0+/Ktor 3.0+ (required by the SDK's underlying Ktor HTTP client engine). If your application requires compatibility with an older OkHttp/Ktor version, please contact the SimplePay support team.

Note: Koin is used internally by the SDK, which handles Koin initialization itself. 
If your application already uses Koin, the SDK will attach to the existing Koin context when the `SimplePaySdk.init()` is called.

### KAPT Compatibility

KAPT has been deprecated since 2023, and only supports libraries compiled with Kotlin 2.0 or older.
This SDK uses modern Android libraries compiled with Kotlin 2.1+ and is **not compatible with KAPT-based annotation processors** like Room 2.6.x or older.

**If your project uses KAPT:**

1. **Recommended**: Migrate all annotation processors to KSP
    - Room with KSP, from version 2.7.0+
    - Dagger/Hilt with KSP, from version [2.51](https://github.com/google/dagger/releases/tag/dagger-2.51)
    - Migrate Data Binding → View Binding

2. **Alternative**: Hybrid approach (should work, further testing recommended)
    - Migrate Room/Hilt to KSP
    - Keep KAPT only for Data Binding (if needed)

See [Android KSP Migration Guide](https://developer.android.com/build/migrate-to-ksp)

## Integration Guide

### 1. Add SDK Dependency

Add the dependency to your app's `build.gradle.kts` (replace `<version>` with the latest version from [Maven Central](https://central.sonatype.com/artifact/com.simplepay/simplepay-android-sdk)):

```kotlin
dependencies {
    implementation("com.simplepay:simplepay-android-sdk:<version>")
}
```

### 2. Initialize the SDK

Initialize the SDK in your `Application` class or from your `MainActivity` with the target SimplePay environment:
Note: This initialization should only happen once, and it requires app-level `Context`, so it should be done from your base Activity or Application class.

```kotlin
class YourApplication : Application() {
    override fun onCreate() {
        super.onCreate()

        SimplePaySdk.init(
            context = this,
            environment = SimplePayEnvironment.PROD
        )
    }
}
```

Important: `SimplePayEnvironment` lets you select which SimplePay backend the SDK connects to. The SDK supports two environments:

| Environment | Use case                              |
|:------------|:--------------------------------------|
| `SANDBOX`   | Integration testing (no real charges) |
| `PROD`      | Live production payments              |

Note: Extra device security checks (root/tamper/emulator detection) are only enforced in the `PROD` environment, so you can freely test and debug your integration in `SANDBOX` without those checks blocking your flow.

If you added the initialization with a new Application class, update your `AndroidManifest.xml` to set it with the class name:

```xml
<application
    android:name=".YourApplication">
<!--    ...-->
</application>
```

### 3. Launch Payment Flow

Set up an `ActivityResultLauncher` in your Activity or Fragment:

Using Jetpack Compose:
```kotlin
class MainActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        
        val launcher = rememberLauncherForActivityResult(
            contract = ActivityResultContracts.StartActivityForResult()
        ) { activityResult ->
            val resultCode = activityResult.resultCode
            val transactionId = activityResult.data?.getStringExtra(SimplePaySdk.TRANSACTION_ID)
            
            if (resultCode == RESULT_OK) {
                // Payment successful
                Log.d("Payment", "Success! Transaction ID: $transactionId")
            } else {
                // Payment failed or cancelled
                Log.e("Payment", "Failed with code: $resultCode, Transaction ID: $transactionId")
            }
        }
        
        Button(
            onClick = {
                // Create a payment intent with the payment path and UUID from the SimplePay API start response
                val intent = SimplePaySdk.getInstance()
                    .createPaymentIntent(
                        context = this,
                        paymentPath = "your-payment-path-from-backend",
                        paymentUuid = "uuid-hash-from-backend"
                    )

                if (intent != null) {
                    launcher.launch(intent)
                } else {
                    // Handle invalid payment path
                    Log.e("Payment", "Invalid payment path")
                }
            }
        ) {
            Text("Start Payment")
        }
    }
}
```

Using XML Layouts:
```kotlin
class MainActivity : AppCompatActivity() {
    private lateinit var paymentLauncher: ActivityResultLauncher<Intent>

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)

        paymentLauncher = registerForActivityResult(
            ActivityResultContracts.StartActivityForResult()
        ) { activityResult ->
            val resultCode = activityResult.resultCode
            val transactionId = activityResult.data?.getStringExtra(SimplePaySdk.TRANSACTION_ID)

            if (resultCode == RESULT_OK) {
                // Payment successful
                Log.d("Payment", "Success! Transaction ID: $transactionId")
            } else {
                // Payment failed or cancelled
                Log.e("Payment", "Failed with code: $resultCode, Transaction ID: $transactionId")
            }
        }
        
        findViewById<Button>(R.id.startPaymentButton).setOnClickListener {
            // Create a payment intent with the payment path and UUID from the SimplePay API start response
            val intent = SimplePaySdk.getInstance()
                .createPaymentIntent(
                    context = this,
                    paymentPath = "your-payment-path-from-backend",
                    paymentUuid = "uuid-hash-from-backend"
                )
            if (intent != null) {
                paymentLauncher.launch(intent)
            } else {
                // Handle invalid payment path
                Log.e("Payment", "Invalid payment path")
            }
        }
    }
}
```

### 4. Handle Payment Result

The SDK returns two pieces of information:

- **`resultCode`**: `RESULT_OK` (code 0) for successful payments, or a SimplePay error code for failures
- **`transactionId`**: Unique identifier for the transaction (accessible via `SimplePaySdk.TRANSACTION_ID` extra - may be null for certain transaction loading issues)

Extract these values from the `ActivityResult`:

```kotlin
val resultCode = activityResult.resultCode
val transactionId = activityResult.data?.getStringExtra(SimplePaySdk.TRANSACTION_ID)
```

### 5. Error Codes

| Code        |                Meaning                 |
|:------------|:--------------------------------------:|
| 5016        |  Timeout — payment session timed out   |
| 5017        |     Transaction cancelled by user      |
| 5026        | Transaction forbidden (possible fraud) |
| 5011        |     Invalid payment path provided      |
| 5554 / 5555 |     Session initialization failure     |
| 1099        |         Internal server error          |
| 5000        |          Generic remote error          |
| 5813        |      Payment rejected by gateway       |
| 2030        |  Incorrect amount/Insufficient funds   |
| 2071        |      Card number rejected by bank      |
| 2072        |      Card expiry rejected by bank      |
| 2074        |   Card holder name rejected by bank    |
| 3012        |       3DS authentication failure       |
| 3501        |         3DS cancelled by user          |

## Support

For issues or questions, please contact the SimplePay support team or open an issue on the GitHub repository.
