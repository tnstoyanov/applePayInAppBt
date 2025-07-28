# API Specifications (Apple Server API v2)

## Overview

This document defines the API contracts and specifications for the modern Apple In-App Purchase system using App      "reason": "CONTENT_NOT_PURCHASED",e's Server API v2 with real-time server notifications and CRM integration.

## 1. Apple Server Notifications v2 Webhook

### Endpoint: `POST /api/v1/webhooks/apple-notifications`

Handles Apple's Server-to-Server Notifications v2 for real-time purchase updates.

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

#### Notification Processing Response

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
  "entitlements_updated": true,
  "content_unlocked": [
    {
      "content_id": "string",
      "access_granted": true
    }
  ]
}
```

## 3. Real-time Content Unlock API

### Endpoint: `GET /api/v1/users/{user_id}/entitlements/live`

Provides real-time entitlement status for immediate content unlocking.

#### Live Entitlements Response

```json
{
  "user_id": "string",
  "last_updated": "ISO-8601-timestamp",
  "entitlements": [
    {
      "product_id": "string",
      "status": "active|cancelled",
      "transaction_id": "string",
      "purchase_date": "ISO-8601-timestamp",
      "content_access": [
        {
          "content_id": "string",
          "access_level": "full|preview|none",
          "unlock_timestamp": "ISO-8601-timestamp"
        }
      ]
    }
  ],
  "pending_unlocks": [
    {
      "content_id": "string",
      "expected_unlock": "ISO-8601-timestamp"
    }
  ]
}
```

## 4. Content Access API

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

## 5. User Entitlements API

### Endpoint: `GET /api/v1/users/{user_id}/entitlements`

Retrieves current user entitlements and purchase status.

#### User Entitlements Response

```json
{
  "user_id": "string",
  "entitlements": [
    {
      "product_id": "string",
      "status": "active|cancelled",
      "purchase_date": "ISO-8601-timestamp",
      "content_access": [
        {
          "content_id": "string",
          "access_level": "full|preview|none"
        }
      ]
    }
  ],
  "last_updated": "ISO-8601-timestamp"
}
```

## 6. Real-time App Notification APIs

### Push Notification Payload

#### Silent Push for Content Unlock

```json
{
  "aps": {
    "content-available": 1,
    "sound": ""
  },
  "custom": {
    "type": "content_unlock",
    "content_ids": ["course_123", "video_456"],
    "user_id": "user_789",
    "transaction_id": "txn_abc123"
  }
}
```

### WebSocket/Server-Sent Events

#### Endpoint: `GET /api/v1/users/{user_id}/stream`

Real-time stream for content unlock notifications.

#### Message Format

```json
{
  "event": "content_unlock",
  "data": {
    "user_id": "string",
    "content_unlocked": [
      {
        "content_id": "string",
        "product_id": "string",
        "unlocked_at": "ISO-8601-timestamp"
      }
    ],
    "transaction_id": "string"
  },
  "timestamp": "ISO-8601-timestamp"
}
```

### Polling API for Fallback

#### Endpoint: `GET /api/v1/users/{user_id}/entitlements/changes`

Efficient polling endpoint for apps to check for entitlement changes.

#### Query Parameters

```http
since: ISO-8601-timestamp (last check)
```

#### Entitlement Changes Response

```json
{
  "has_changes": true,
  "last_updated": "ISO-8601-timestamp",
  "changes": [
    {
      "type": "unlock",
      "content_id": "string",
      "product_id": "string",
      "changed_at": "ISO-8601-timestamp"
    }
  ]
}
```

## 7. CRM Integration API

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
  "entitlements_updated": true,
  "sync_timestamp": "ISO-8601-timestamp"
}
```

## 8. Product Catalog API

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
// Real-time notification listener for React Native
import { useEffect } from 'react';
import PushNotification from '@react-native-async-storage/async-storage';
import messaging from '@react-native-firebase/messaging';

const NotificationManager = () => {
  useEffect(() => {
    // Handle background/quit state notifications
    messaging().setBackgroundMessageHandler(async remoteMessage => {
      if (remoteMessage.data?.type === 'content_unlock') {
        await handleContentUnlock(remoteMessage.data);
      }
    });

    // Handle foreground notifications
    const unsubscribe = messaging().onMessage(async remoteMessage => {
      if (remoteMessage.data?.type === 'content_unlock') {
        await handleContentUnlock(remoteMessage.data);
      }
    });

    return unsubscribe;
  }, []);

  const handleContentUnlock = async (data) => {
    const { content_ids, user_id, transaction_id } = data;
    
    // Update local state to unlock content
    await unlockContent(content_ids);
    
    // Optionally show user notification
    PushNotification.localNotification({
      title: 'Content Unlocked!',
      message: 'Your premium content is now available',
    });
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
      // Simply finish the transaction - no receipt validation needed
      await finishTransaction(purchase);
      
      // Content will be unlocked via server notification
      console.log('Purchase completed, waiting for server notification...');
      
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
// Apple Server Notification handler
async function handleAppleNotification(signedPayload) {
    // Verify JWS signature
    const notification = verifyJWS(signedPayload);
    
    // Process notification
    const transactionInfo = verifyJWS(notification.data.signedTransactionInfo);
    
    // Update CRM and send real-time notifications
    await processTransaction(transactionInfo);
    
    return { status: 'processed' };
}
```

## Testing & Validation

### Test Scenarios

1. **Apple Notification Processing**
   - Valid PURCHASE notifications
   - CANCEL/REFUND handling

2. **Real-time Communication**
   - Push notification delivery
   - WebSocket message broadcasting
   - Polling fallback mechanisms

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
      "has_purchased_content": true,
      "purchase_date": "2024-12-31T23:59:59Z"
    }
  ]
}
```
