# API Specifications (Apple Server API v2)

## Overview

This document defines the API contracts and specifications for the modern Apple In-App Purchase system using Apple's Server API v2 with server notifications and CRM integration for one-time purchases.

## 1. Apple Server Notifications v2 Webhook

### Endpoint: `POST /api/v1/webhooks/apple-notifications`

Handles Apple's Server-to-Server Notifications v2 for purchase updates.

#### Request Headers (from Apple)

```http
x-apple-receipt-signature: {signature}
Content-Type: application/json
```

#### Request Body (from Apple)

```json
{
  "signedPayload": "eyJhbGciOiJFUzI1NiIsIng1YyI6W...JWS_PAYLOAD"
}
```

#### Decoded JWS Payload Structure

```json
{
  "notificationType": "PURCHASE|CANCEL|REFUND",
  "subtype": "INITIAL_BUY",
  "notificationUUID": "string",
  "data": {
    "appAppleId": 123456789,
    "bundleId": "com.yourapp.bundle",
    "bundleVersion": "1.0.0",
    "environment": "Sandbox|Production",
    "signedTransactionInfo": "eyJhbGciOiJFUzI1NiIsIng1YyI6W...TRANSACTION_JWS"
  },
  "version": "2.0",
  "signedDate": 1234567890000
}
```

#### Response

```json
{
  "status": "processed",
  "timestamp": "ISO-8601-timestamp"
}
```

## 2. Transaction Verification API

### Endpoint: `POST /api/v1/transactions/verify`

Internal API to fetch and verify transaction details from Apple Server API v2.

#### Request

```json
{
  "transaction_id": "string",
  "environment": "Sandbox|Production",
  "user_id": "string"
}
```

#### Transaction Processing Response

```json
{
  "status": "success",
  "transaction_valid": true,
  "transaction_details": {
    "transaction_id": "string",
    "original_transaction_id": "string",
    "product_id": "string",
    "purchase_date": "ISO-8601-timestamp",
    "quantity": 1,
    "type": "Non-Consumable|Consumable"
  },
  "content_unlocked": [
    {
      "content_id": "string",
      "access_granted": true
    }
  ]
}
```

## 3. Content Access API

### Endpoint: `GET /api/v1/content/{content_id}/access`

Checks user access to specific content and returns secure access URLs.

#### Content Access Request Headers

```http
Authorization: Bearer {jwt_token}
X-Device-ID: {device_id}
X-App-Version: {version}
```

#### Success Response (200)

```json
{
  "has_access": true,
  "access_url": "string",
  "expires_at": "ISO-8601-timestamp",
  "content_metadata": {
    "title": "string",
    "description": "string",
    "duration": 3600,
    "file_size": 1048576,
    "format": "mp4|pdf|html"
  }
}
```

#### Access Denied Response (403)

```json
{
  "has_access": false,
  "reason": "CONTENT_NOT_PURCHASED",
  "purchase_options": [
    {
      "product_id": "string",
      "price": "string",
      "currency": "USD"
    }
  ]
}
```

## 4. User Purchases API

### Endpoint: `GET /api/v1/users/{user_id}/purchases`

Retrieves current user purchases and owned content.

#### User Purchases Response

```json
{
  "user_id": "string",
  "purchases": [
    {
      "product_id": "string",
      "transaction_id": "string", 
      "purchase_date": "ISO-8601-timestamp",
      "content_ids": ["course_123", "video_456"]
    }
  ],
  "last_updated": "ISO-8601-timestamp"
}
```

## 5. Purchase Notification APIs

### Push Notification Payload

#### Purchase Complete Notification

```json
{
  "aps": {
    "alert": {
      "title": "Purchase Complete!",
      "body": "Your content is now available"
    },
    "sound": "default"
  },
  "purchase_data": {
    "product_id": "course_123",
    "content_ids": ["course_123", "video_456"],
    "transaction_id": "txn_abc123"
  }
}
```

## 6. CRM Integration API

### Endpoint: `POST /api/v1/crm/purchase`

Updates CRM system with purchase information.

#### CRM Purchase Request

```json
{
  "user_id": "string",
  "transaction_id": "string",
  "product_id": "string",
  "purchase_date": "ISO-8601-timestamp",
  "amount": 9.99,
  "currency": "USD",
  "platform": "ios",
  "notification_data": "jws-payload"
}
```

#### CRM Integration Response

```json
{
  "status": "success",
  "crm_customer_id": "string",
  "sync_timestamp": "ISO-8601-timestamp"
}
```

## 7. Product Catalog API

### Endpoint: `GET /api/v1/products`

Retrieves available products and their metadata.

#### Product Query Parameters

- `platform`: ios|android|web
- `user_id`: string (optional, for personalized pricing)
- `locale`: string (for localized content)

#### Product Catalog Response

```json
{
  "products": [
    {
      "product_id": "string",
      "title": "string",
      "description": "string",
      "price": 9.99,
      "currency": "USD",
      "product_type": "consumable|non_consumable",
      "content_included": [
        {
          "content_id": "string",
          "title": "string",
          "type": "course|video|document"
        }
      ]
    }
  ]
}
```

## Authentication & Security

### JWT Token Structure

```json
{
  "sub": "user_id",
  "iss": "your-service",
  "iat": 1234567890,
  "exp": 1234571490,
  "scopes": ["content:read", "purchases:read"],
  "device_id": "string"
}
```

### API Rate Limits

| Endpoint | Rate Limit | Window |
|----------|------------|--------|
| Apple Notifications | Unlimited | Apple managed |
| Content Access | 500 req/min | Per User |
| Product Catalog | 1000 req/min | Global |

### Error Codes

| Code | Description | Action |
|------|-------------|--------|
| 2001 | Invalid JWS Signature | Check Apple certificates |
| 2002 | Transaction Not Found | Retry with Apple API |
| 2003 | Network Error | Retry with backoff |
| 2004 | Server Error | Retry later |
| 2005 | User Not Found | Register user first |

## Implementation Examples

### React Native Implementation

```javascript
// Purchase notification handler for React Native
import { useEffect } from 'react';
import messaging from '@react-native-firebase/messaging';

const PurchaseNotificationManager = () => {
  useEffect(() => {
    // Handle purchase complete notifications
    const unsubscribe = messaging().onMessage(async remoteMessage => {
      if (remoteMessage.purchase_data) {
        await handlePurchaseComplete(remoteMessage.purchase_data);
      }
    });

    return unsubscribe;
  }, []);

  const handlePurchaseComplete = async (purchaseData) => {
    const { product_id, content_ids, transaction_id } = purchaseData;
    
    // Update local state - content is now owned
    await markContentOwned(content_ids);
    
    // Refresh UI to show owned content
    console.log(`Purchase complete: ${product_id}, Content unlocked: ${content_ids}`);
  };
};

// In-App Purchase handling with react-native-iap
import { 
  initConnection, 
  purchaseUpdatedListener,
  purchaseErrorListener,
  finishTransaction,
  getProducts,
  requestPurchase
} from 'react-native-iap';

class IAPManager {
  constructor() {
    this.purchaseUpdateSubscription = null;
    this.purchaseErrorSubscription = null;
  }

  async initialize() {
    try {
      await initConnection();
      
      // Set up purchase listeners
      this.purchaseUpdateSubscription = purchaseUpdatedListener(
        (purchase) => this.handlePurchaseUpdate(purchase)
      );
      
      this.purchaseErrorSubscription = purchaseErrorListener(
        (error) => this.handlePurchaseError(error)
      );
    } catch (error) {
      console.error('IAP initialization failed:', error);
    }
  }

  async handlePurchaseUpdate(purchase) {
    try {
      // Finish the transaction - Apple handles validation
      await finishTransaction(purchase);
      
      // Content access will be available immediately after processing
      console.log('Purchase completed:', purchase.productId);
      
    } catch (error) {
      console.error('Error finishing transaction:', error);
    }
  }

  async makePurchase(productId) {
    try {
      await requestPurchase(productId);
    } catch (error) {
      console.error('Purchase failed:', error);
    }
  }

  cleanup() {
    if (this.purchaseUpdateSubscription) {
      this.purchaseUpdateSubscription.remove();
    }
    if (this.purchaseErrorSubscription) {
      this.purchaseErrorSubscription.remove();
    }
  }
}
```

### Backend Service Example (Node.js)

```javascript
// Apple Server Notification handler for one-time purchases
async function handleAppleNotification(signedPayload) {
    // Verify JWS signature
    const notification = verifyJWS(signedPayload);
    
    // Extract transaction info
    const transactionInfo = verifyJWS(notification.data.signedTransactionInfo);
    
    // Update user's purchased content
    await updateUserPurchases(transactionInfo);
    
    // Send simple purchase confirmation
    await sendPurchaseConfirmation(transactionInfo.userId, transactionInfo.productId);
    
    return { status: 'processed' };
}
```

## Testing & Validation

### Test Scenarios

1. **Apple Notification Processing**
   - Valid PURCHASE notifications
   - CANCEL/REFUND handling

2. **Purchase Flow**
   - Successful purchase completion
   - Content access after purchase
   - Purchase restoration

3. **Edge Cases**
   - Invalid JWS signatures
   - Duplicate notifications
   - Network failures

### Mock Data

```json
{
  "test_notifications": {
    "valid_purchase": "eyJhbGciOiJFUzI1NiIs...",
    "invalid_signature": "invalid-jws-payload"
  },
  "test_users": [
    {
      "user_id": "test_user_1",
      "owned_products": ["course_123"],
      "purchase_date": "2024-12-31T23:59:59Z"
    }
  ]
}
```
