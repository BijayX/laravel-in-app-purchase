# Laravel In-App Purchase Verification Package

A production-ready Laravel package for server-side verification of in-app purchases and subscriptions for both **Apple App Store (iOS)** and **Google Play Store (Android)**.

## Features

- ✅ **Apple App Store** verification (receipt validation)
- ✅ **Google Play Store** verification (subscriptions & one-time purchases)
- ✅ Automatic sandbox fallback for Apple (status 21007)
- ✅ Subscription status tracking (active, expired, cancelled)
- ✅ Database storage for subscriptions
- ✅ Webhook support for real-time updates
- ✅ Clean architecture with SOLID principles
- ✅ Type-safe DTOs and contracts
- ✅ Comprehensive error handling and logging

## Requirements

- PHP ^8.1
- Laravel ^10.0|^11.0|^12.0
- Google API Client Library (automatically installed)

## Installation

### 1. Install via Composer

```bash
composer require bijay/laravel-in-app-purchase
```

### 2. Publish Configuration

```bash
php artisan vendor:publish --tag=iap-config
```

### 3. Publish Migrations

```bash
php artisan vendor:publish --tag=iap-migrations
php artisan migrate
```

## Configuration

### Environment Variables

Add these to your `.env` file:

```env
# Apple App Store
IAP_APPLE_SHARED_SECRET=your_shared_secret_from_app_store_connect
IAP_APPLE_VERIFY_URL=https://buy.itunes.apple.com/verifyReceipt
IAP_APPLE_SANDBOX_URL=https://sandbox.itunes.apple.com/verifyReceipt

# Google Play Store
IAP_GOOGLE_SERVICE_ACCOUNT_PATH=/path/to/service-account.json
IAP_GOOGLE_PACKAGE_NAME=com.yourcompany.yourapp

# Optional
IAP_TABLE_NAME=iap_subscriptions
IAP_USER_MODEL=App\Models\User
```

### Apple App Store Setup

1. Go to [App Store Connect](https://appstoreconnect.apple.com/)
2. Navigate to your app → App Information → App-Specific Shared Secret
3. Generate or copy your shared secret
4. Add it to your `.env` file as `IAP_APPLE_SHARED_SECRET`

### Google Play Store Setup

1. Go to [Google Cloud Console](https://console.cloud.google.com/)
2. Create a new project or select an existing one
3. Enable the **Google Play Android Developer API**
4. Create a **Service Account**:
   - Go to IAM & Admin → Service Accounts
   - Create a new service account
   - Download the JSON key file
5. Link the service account to your Google Play Console:
   - Go to Google Play Console → Settings → API access
   - Link your service account
   - Grant **View financial data** permission
6. Place the JSON file in `storage/app/private/google-service-account.json`
7. Update `.env` with the path: `IAP_GOOGLE_SERVICE_ACCOUNT_PATH=storage/app/private/google-service-account.json`

## Usage

### Basic Verification

#### Using the Facade

```php
use Bijay\Iap\Facades\Iap;

// Verify Apple purchase
$result = Iap::verify('ios', [
    'receipt_data' => $receiptData,
    'password' => config('iap.apple.shared_secret'), // Optional, uses config if not provided
]);

// Verify Google purchase
$result = Iap::verify('android', [
    'package_name' => 'com.yourcompany.yourapp',
    'product_id' => 'premium_subscription',
    'purchase_token' => $purchaseToken,
    'is_subscription' => true, // true for subscriptions, false for one-time purchases
]);

// Check result
if ($result->valid && $result->isActive()) {
    // Subscription is active
    echo "Product: {$result->productId}";
    echo "Expires: {$result->expiresAt}";
    echo "Status: {$result->status}";
}
```

#### Using Dependency Injection

```php
use Bijay\Iap\Services\IapManager;

class SubscriptionController extends Controller
{
    public function __construct(
        protected IapManager $iapManager
    ) {}

    public function verify(Request $request)
    {
        $result = $this->iapManager->verify(
            $request->input('platform'),
            $request->input('payload')
        );

        return response()->json($result->toArray());
    }
}
```

### API Endpoint

The package provides a ready-to-use API endpoint:

**POST** `/api/iap/verify`

**Request Body:**
```json
{
    "platform": "ios",
    "user_id": 1,
    "payload": {
        "receipt_data": "base64_encoded_receipt_data"
    }
}
```

**Response:**
```json
{
    "success": true,
    "message": "Purchase verified successfully",
    "data": {
        "verification": {
            "valid": true,
            "status": "active",
            "expires_at": "2024-12-31T23:59:59+00:00",
            "platform": "ios",
            "product_id": "premium_monthly",
            "original_transaction_id": "1000000123456789",
            "raw_data": {...}
        },
        "subscription": {
            "id": 1,
            "user_id": 1,
            "platform": "ios",
            "product_id": "premium_monthly",
            "transaction_id": "1000000123456789",
            "status": "active",
            "expires_at": "2024-12-31 23:59:59",
            "created_at": "2024-01-01 00:00:00",
            "updated_at": "2024-01-01 00:00:00"
        }
    }
}
```

### Flutter Integration Example

#### iOS (Apple)

```dart
import 'package:in_app_purchase/in_app_purchase.dart';

Future<void> verifyPurchase(ProductDetails product) async {
  final PurchaseDetails purchase = await _inAppPurchase.buyNonConsumable(
    purchaseParam: PurchaseParam(productDetails: product),
  );

  if (purchase.status == PurchaseStatus.purchased) {
    // Get receipt data
    final receiptData = await _getReceiptData();
    
    // Send to Laravel backend
    final response = await http.post(
      Uri.parse('https://your-api.com/api/iap/verify'),
      headers: {'Content-Type': 'application/json'},
      body: jsonEncode({
        'platform': 'ios',
        'user_id': userId,
        'payload': {
          'receipt_data': receiptData,
        },
      }),
    );

    final result = jsonDecode(response.body);
    if (result['success']) {
      print('Purchase verified!');
    }
  }
}

Future<String> _getReceiptData() async {
  // Get receipt data from iOS
  // This depends on your Flutter IAP plugin
  // For in_app_purchase plugin, you may need to use app_store_receipt
  return base64Encode(receiptBytes);
}
```

#### Android (Google)

```dart
import 'package:in_app_purchase/in_app_purchase.dart';

Future<void> verifyPurchase(ProductDetails product) async {
  final PurchaseDetails purchase = await _inAppPurchase.buyNonConsumable(
    purchaseParam: PurchaseParam(productDetails: product),
  );

  if (purchase.status == PurchaseStatus.purchased) {
    // Get purchase details
    final purchaseId = purchase.purchaseID;
    final productId = product.id;
    final verificationData = purchase.verificationData;
    
    // Send to Laravel backend
    final response = await http.post(
      Uri.parse('https://your-api.com/api/iap/verify'),
      headers: {'Content-Type': 'application/json'},
      body: jsonEncode({
        'platform': 'android',
        'user_id': userId,
        'payload': {
          'package_name': 'com.yourcompany.yourapp',
          'product_id': productId,
          'purchase_token': verificationData.serverVerificationData,
          'is_subscription': true,
        },
      }),
    );

    final result = jsonDecode(response.body);
    if (result['success']) {
      print('Purchase verified!');
    }
  }
}
```

### React Native Integration Example

#### iOS

```javascript
import { requestPurchase, getReceiptIOS } from 'react-native-iap';

const verifyIOSPurchase = async (productId) => {
  try {
    await requestPurchase({ sku: productId });
    
    const receipt = await getReceiptIOS();
    
    const response = await fetch('https://your-api.com/api/iap/verify', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        platform: 'ios',
        user_id: userId,
        payload: {
          receipt_data: receipt,
        },
      }),
    });

    const result = await response.json();
    if (result.success) {
      console.log('Purchase verified!');
    }
  } catch (error) {
    console.error('Purchase verification failed:', error);
  }
};
```

#### Android

```javascript
import { requestPurchase, getPurchaseHistory } from 'react-native-iap';

const verifyAndroidPurchase = async (productId) => {
  try {
    await requestPurchase({ sku: productId });
    
    const purchases = await getPurchaseHistory();
    const purchase = purchases.find(p => p.productId === productId);
    
    const response = await fetch('https://your-api.com/api/iap/verify', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        platform: 'android',
        user_id: userId,
        payload: {
          package_name: 'com.yourcompany.yourapp',
          product_id: productId,
          purchase_token: purchase.purchaseToken,
          is_subscription: true,
        },
      }),
    });

    const result = await response.json();
    if (result.success) {
      console.log('Purchase verified!');
    }
  } catch (error) {
    console.error('Purchase verification failed:', error);
  }
};
```

## Webhooks

The package includes webhook endpoints for real-time subscription updates from Apple and Google.

### Apple Webhook

**POST** `/api/iap/webhook/ios` or `/api/iap/webhook/apple`

Configure this URL in App Store Connect:
1. Go to App Store Connect → Your App → App Information
2. Set the **Server Notification URL** to: `https://your-api.com/api/iap/webhook/ios`

### Google Webhook

**POST** `/api/iap/webhook/android` or `/api/iap/webhook/google`

Configure this URL in Google Play Console:
1. Go to Google Play Console → Your App → Monetization → Subscriptions
2. Set the **Real-time developer notifications** URL to: `https://your-api.com/api/iap/webhook/android`

### Webhook Payload Examples

#### Apple Notification Types

- `INITIAL_BUY` - First purchase
- `DID_RENEW` - Subscription renewed
- `DID_RECOVER` - Subscription recovered
- `DID_FAIL_TO_RENEW` - Renewal failed
- `DID_CANCEL` - Subscription cancelled
- `EXPIRED` - Subscription expired

#### Google Notification Types

- `SUBSCRIPTION_PURCHASED` (4)
- `SUBSCRIPTION_RENEWED` (2)
- `SUBSCRIPTION_RECOVERED` (1)
- `SUBSCRIPTION_CANCELED` (3)
- `SUBSCRIPTION_EXPIRED` (12)

## Database Schema

The package creates an `iap_subscriptions` table with the following structure:

```sql
- id (bigint, primary key)
- user_id (bigint, foreign key to users)
- platform (enum: 'ios', 'android')
- product_id (string)
- transaction_id (string, unique)
- status (enum: 'active', 'expired', 'cancelled')
- expires_at (timestamp, nullable)
- raw_data (json, nullable)
- created_at (timestamp)
- updated_at (timestamp)
```

## VerificationResult DTO

The `VerificationResult` DTO provides useful methods:

```php
$result = Iap::verify('ios', $payload);

// Check if valid
if ($result->valid) { ... }

// Check if active
if ($result->isActive()) { ... }

// Check if expired
if ($result->isExpired()) { ... }

// Get properties
$result->status; // 'active', 'expired', 'cancelled'
$result->expiresAt; // Carbon instance or null
$result->platform; // 'ios' or 'android'
$result->productId; // Product identifier
$result->originalTransactionId; // Original transaction ID
$result->rawData; // Raw response from platform

// Convert to array
$array = $result->toArray();
```

## Security Best Practices

1. **Always verify on the server**: Never trust client-side purchase data
2. **Use HTTPS**: All API endpoints should use HTTPS
3. **Authenticate requests**: Add authentication middleware to protect endpoints
4. **Validate user ownership**: Ensure the user_id matches the authenticated user
5. **Store credentials securely**: Never commit service account files or shared secrets to version control
6. **Rate limiting**: Implement rate limiting on verification endpoints
7. **Logging**: Monitor logs for suspicious activity

### Adding Authentication Middleware

```php
// In routes/api.php or your service provider
Route::prefix('iap')->middleware(['auth:sanctum'])->group(function () {
    Route::post('/verify', [VerifyPurchaseController::class, 'verify']);
});
```

## Error Handling

The package handles various error scenarios:

- **Invalid receipts/tokens**: Returns `valid: false` with error details
- **Network errors**: Logs errors and returns error status
- **Sandbox fallback**: Automatically retries with sandbox URL for Apple (status 21007)
- **Missing credentials**: Logs warnings and returns error status

Check the `raw_data` field in `VerificationResult` for detailed error information.

## Testing

### Apple Sandbox Testing

1. Use sandbox test accounts in App Store Connect
2. Test purchases will automatically fallback to sandbox URL
3. Sandbox receipts expire after a short period

### Google Testing

1. Use test accounts in Google Play Console
2. Test purchases are automatically handled
3. Use the Google Play Console to manage test subscriptions

## Troubleshooting

### Apple Verification Issues

- **Status 21007**: Automatically handled (sandbox fallback)
- **Status 21008**: Receipt is from production but sent to sandbox
- **Status 21010**: Receipt data is malformed
- Check your shared secret is correct

### Google Verification Issues

- **401 Unauthorized**: Check service account JSON file path and permissions
- **403 Forbidden**: Ensure service account has proper permissions in Play Console
- **404 Not Found**: Verify package name and product ID are correct

## License

MIT

## Support

For issues, questions, or contributions, please visit the [GitHub repository](https://github.com/bijay/laravel-in-app-purchase).

## Changelog

### 1.0.0
- Initial release
- Apple App Store verification
- Google Play Store verification
- Webhook support
- Database storage
- Comprehensive documentation

