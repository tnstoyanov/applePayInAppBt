# Apple In-App Purchase Flow Diagrams

## Overview

This document contains detailed flow diagrams for implementing Apple In-App Purchases with CRM integration. Each diagram represents a critical part of the system architecture.

## 1. Modern Purchase Flow (Apple Server API v2)

```mermaid
flowchart TD
    A[User Initiates Purchase] --> B{Products Available?}
    B -->|No| C[Load Products from App Store]
    C --> D[Display Product List]
    B -->|Yes| D
    D --> E[User Selects Product]
    E --> F[Initiate StoreKit Payment]
    F --> G{Payment Successful?}
    G -->|No| H[Display Error Message]
    G -->|Yes| I[StoreKit Transaction Complete]
    
    subgraph "Apple's Real-time Process"
        J[Apple sends Server Notification v2]
        K[Your webhook receives notification]
        L[Fetch transaction via API v2]
        M[Verify JWS signature]
        N[Update CRM & unlock content]
    end
    
    I --> J
    J --> K
    K --> L
    L --> M
    M --> N
    
    subgraph "Real-time App Update"
        O[Push Notification]
        P[WebSocket/SSE Update]
        Q[App polls for updates]
        R[Content unlocked in app]
    end
    
    N --> O
    N --> P
    N --> Q
    O --> R
    P --> R
    Q --> R
    
    style A fill:#e3f2fd
    style G fill:#fff3e0
    style M fill:#fff3e0
    style N fill:#fff3e0
    style R fill:#e8f5e8
```

## 2. Modern Apple Server API v2 Flow (Real-time)

```mermaid
sequenceDiagram
    participant App as iOS App
    participant Backend as Your Backend
    participant Apple as Apple Server API v2
    participant CRM as CRM System
    participant Push as Push Notification
    participant WS as WebSocket/SSE
    
    Note over App,Apple: Purchase completed in StoreKit
    Apple->>Backend: Server Notification v2 (Webhook)
    Note over Apple,Backend: PURCHASE, RENEWAL, etc.
    
    Backend->>Apple: GET /transactions/{transactionId}
    Apple->>Backend: Transaction details + JWS
    Backend->>Backend: Verify JWS signature
    
    Backend->>CRM: Update entitlements
    CRM->>Backend: Confirmation
    
    par Real-time notification
        Backend->>Push: Send push notification
        Push->>App: "Content unlocked!"
    and WebSocket update
        Backend->>WS: Send content update
        WS->>App: Live content refresh
    and Polling fallback
        App->>Backend: GET /user/entitlements
        Backend->>App: Updated entitlements
    end
```

## 3. Subscription Management Flow

```mermaid
stateDiagram-v2
    [*] --> PendingPurchase
    PendingPurchase --> Processing: Payment Initiated
    Processing --> Active: Apple Notification Received
    Processing --> Failed: Payment Failed
    Active --> Expired: Subscription Ends
    Active --> Cancelled: User Cancels
    Active --> PendingRenewal: Near Expiry
    PendingRenewal --> Active: Auto-Renewal Success
    PendingRenewal --> Expired: Auto-Renewal Failed
    Expired --> Active: Manual Renewal
    Cancelled --> Active: Resubscribe
    Failed --> PendingPurchase: Retry Purchase
    
    Active: Content Accessible
    Expired: Content Locked
    Cancelled: Grace Period
    Failed: Show Error
```

## 4. Content Delivery Architecture

```mermaid
graph LR
    subgraph "iOS App"
        A[Content Manager] --> B[Local Cache]
        A --> C[Download Queue]
    end
    
    subgraph "CDN Layer"
        D[CloudFront/CDN] --> E[Origin Server]
    end
    
    subgraph "Backend Services"
        F[Content API] --> G[Content Database]
        F --> H[Access Control]
        H --> I[User Entitlements]
    end
    
    subgraph "Storage"
        J[S3/Cloud Storage] --> K[Encrypted Content]
    end
    
    A -->|Request Content| F
    F -->|Check Access| H
    H -->|Generate URL| D
    D -->|Deliver Content| C
    E --> J
    
    style A fill:#e3f2fd
    style H fill:#fff3e0
    style D fill:#e8f5e8
```

## 5. Error Handling and Retry Logic

```mermaid
flowchart TD
    A[Transaction/Notification Error] --> B{Error Type}
    B -->|Network Error| C[Exponential Backoff]
    B -->|Invalid JWS Signature| D[Log & Alert]
    B -->|Server Error| E[Immediate Retry]
    B -->|User Cancelled| F[No Action]
    
    C --> G{Max Retries?}
    G -->|No| H[Wait & Retry]
    G -->|Yes| I[Queue for Later]
    
    E --> J{Retry Count < 3?}
    J -->|Yes| K[Retry After Delay]
    J -->|No| L[Queue for Later]
    
    I --> M[Background Sync Process]
    L --> M
    H --> N[Continue Flow]
    K --> N
    D --> O[Manual Investigation]
    F --> P[Return to Store]
    
    style D fill:#ffebee
    style O fill:#ffebee
    style M fill:#fff3e0
```

## 6. CRM Integration Pattern

```mermaid
sequenceDiagram
    participant NH as Notification Handler
    participant Auth as Auth Service
    participant CRM as CRM API
    participant User as User Service
    participant Content as Content Service
    participant Analytics as Analytics
    
    NH->>Auth: Get API Token
    Auth->>NH: JWT Token
    
    NH->>User: Link Apple ID to User
    User->>CRM: Update User Profile
    
    NH->>CRM: Create Purchase Record
    Note over NH,CRM: Purchase details + JWS payload
    
    CRM->>Content: Update Entitlements
    Content->>User: Grant Access
    
    par Analytics Tracking
        CRM->>Analytics: Purchase Event
    and Notification
        CRM->>User: Send Confirmation
    end
    
    CRM->>RV: Success Response
```

## 7. Offline Handling Strategy

```mermaid
flowchart TD
    A[App Launch] --> B{Network Available?}
    B -->|No| C[Load Cached Data]
    B -->|Yes| D[Sync with Server]
    
    C --> E[Check Local Entitlements]
    E --> F{Pending Notifications?}
    F -->|Yes| G[Queue for Sync]
    F -->|No| H[Continue Offline]
    
    D --> I[Fetch Latest Entitlements]
    I --> J[Update Local Cache]
    J --> K[Sync Complete]
    
    G --> L{Network Restored?}
    L -->|Yes| M[Process Queue]
    L -->|No| N[Continue Queuing]
    
    M --> O[Update Server]
    O --> P[Clear Queue]
    
    H --> Q[Limited Functionality]
    Q --> R[Periodic Sync Attempts]
    
    style C fill:#fff3e0
    style Q fill:#ffecb3
    style K fill:#e8f5e8
```

## 8. Security Implementation Flow

```mermaid
graph TB
    subgraph "Device Security"
        A[Certificate Pinning] --> B[Request Signing]
        B --> C[JWS Verification]
    end
    
    subgraph "Transport Security"
        D[TLS 1.3] --> E[API Authentication]
        E --> F[Rate Limiting]
    end
    
    subgraph "Server Security"
        G[Apple Notification Verification] --> H[Replay Protection]
        H --> I[Audit Logging]
    end
    
    subgraph "Content Security"
        J[Content Encryption] --> K[Access Tokens]
        K --> L[DRM Integration]
    end
    
    C --> D
    F --> G
    I --> J
    
    style A fill:#ffebee
    style G fill:#ffebee
    style J fill:#ffebee
```

## 9. Real-time Content Unlock Strategies

```mermaid
flowchart TD
    A[Apple Server Notification v2] --> B[Webhook Received]
    B --> C[Verify JWS Signature]
    C --> D[Parse Transaction Data]
    D --> E[Update CRM]
    
    E --> F{How to notify app?}
    
    F -->|Option 1| G[Push Notification]
    F -->|Option 2| H[WebSocket/SSE]
    F -->|Option 3| I[Polling Strategy]
    
    G --> J[Silent Push + Content ID]
    J --> K[App fetches new entitlements]
    
    H --> L[Real-time WebSocket message]
    L --> M[Instant content unlock]
    
    I --> N[App polls every 5-10 seconds]
    N --> O[Check entitlements API]
    
    K --> P[Content Available]
    M --> P
    O --> P
    
    style A fill:#e3f2fd
    style C fill:#fff3e0
    style P fill:#e8f5e8
```

## 10. Apple Server API v2 Integration

```mermaid
sequenceDiagram
    participant Apple as Apple Server
    participant Webhook as Your Webhook
    participant API as Apple Server API v2
    participant CRM as CRM System
    participant App as iOS App
    
    Note over Apple: User makes purchase
    Apple->>Webhook: POST /webhooks/apple-notifications
    Note over Apple,Webhook: notificationType: PURCHASE
    
    Webhook->>Webhook: Verify JWS signature
    Webhook->>API: GET /inApps/v2/transactions/{transactionId}
    API->>Webhook: Transaction details (JWS)
    
    Webhook->>Webhook: Decode transaction JWS
    Webhook->>CRM: Update user entitlements
    
    par Push notification
        Webhook->>App: Silent push with content unlock
    and WebSocket
        Webhook->>App: WebSocket message
    and Response to Apple
        Webhook->>Apple: 200 OK
    end
```

## Implementation Notes

### Modern Apple APIs Required

1. **Apple App Store Server API v2**
   - `GET /inApps/v2/transactions/{transactionId}` - Get transaction details
   - `GET /inApps/v2/history/{originalTransactionId}` - Transaction history
   - `POST /inApps/v2/notifications/test` - Test notifications

2. **Server-to-Server Notifications v2**
   - Webhook endpoint for real-time notifications
   - JWS signature verification
   - Handle notification types: PURCHASE, RENEWAL, CANCEL, etc.

3. **Real-time App Communication**
   - Push Notifications (silent + content)
   - WebSocket/Server-Sent Events
   - Polling fallback mechanism

### Critical Success Factors

1. **Proper Apple Notification Processing**
   - Always process server-side notifications
   - Handle sandbox vs production environments
   - Implement replay attack prevention with JWS verification

2. **Robust Error Handling**
   - Network failure recovery
   - Graceful degradation
   - User-friendly error messages

3. **Content Security**
   - Secure content delivery
   - Access control enforcement
   - Audit trail maintenance

4. **Performance Optimization**
   - Real-time notification processing
   - Content pre-loading
   - Efficient sync mechanisms
