# SimplePay Android SDK

The official Android SDK for SimplePay - a modern payment processing platform. 
This SDK enables seamless integration of SimplePay's payment services into your Android application, providing a secure and user-friendly payment experience.

## Features
- Easy integration with minimal setup
- Support for card payments, more payment methods coming soon
- Full accessibility support including font scaling and display size scaling
- 3DS two-factor authentication

## Architecture & Design

The SDK is built following industry best practices:

- **Clean Architecture** - Clear separation of concerns with distinct layers (presentation, domain, data)
- **MVVM Pattern** - Model-View-ViewModel for reactive UI and testable business logic
- **Dependency Injection** - Koin for modular and maintainable code
- **Coroutines & Flow** - Modern asynchronous programming for smooth performance
- **Type-safe navigation** - Jetpack Compose Navigation for seamless screen transitions

This architecture ensures the SDK is scalable, maintainable, and follows Android development best practices.

## Requirements

- **Minimum SDK**: 24 (Android 7.0)
- **Target SDK**: 36 (Android 16.0)

- Kotlin 2.0.21+
- Android Gradle Plugin 8.1+
- KSP for annotation processing (if using libraries like Room, Hilt, etc.)

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

Add the dependency to your app's `build.gradle.kts` (replace `<version>` with the latest version from [releases](https://github.com/otpm-green/simplepay-android-sdk/releases)):

```kotlin
dependencies {
    implementation("com.simplepay:simplepay-android-sdk:<version>")
}
```

### 2. Initialize the SDK

Initialize the SDK in your `Application` class with the base URL you receive to access SimplePay API:

```kotlin
class YourApplication : Application() {
    override fun onCreate() {
        super.onCreate()
        
        SimplePaySdk.init(
            context = this,
            baseUrl = "https://your-simplepay-api-url.com/"
        )
    }
}
```

Important: Replace ```https://your-simplepay-api-url.com/``` with an actual SimplePay API base URL provided by SimplePay. You will be able to access different environments (development, test, production) by setting their provided base URLs before initialization.

Update your `AndroidManifest.xml` to use your custom Application class:

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
                // Create a payment intent with the payment path or URL from your backend
                val intent = SimplePaySdk.getInstance()
                    .createPaymentIntent(
                        context = this,
                        paymentPath = "your-payment-path-from-backend"
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
            // Create a payment intent with the payment path or URL from your backend
            val intent = SimplePaySdk.getInstance()
                .createPaymentIntent(
                    context = this,
                    paymentPath = "your-payment-path-from-backend"
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

- **`resultCode`**: `RESULT_OK` for successful payments, or a SimplePay error code for failures
- **`transactionId`**: Unique identifier for the transaction (accessible via `SimplePaySdk.TRANSACTION_ID` extra)

Extract these values from the `ActivityResult`:

```kotlin
val resultCode = activityResult.resultCode
val transactionId = activityResult.data?.getStringExtra(SimplePaySdk.TRANSACTION_ID)
```

## Support

For issues or questions, please contact the SimplePay support team or open an issue on the GitHub repository.