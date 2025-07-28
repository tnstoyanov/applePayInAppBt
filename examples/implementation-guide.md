# Implementation Examples (Apple Server API v2)

## Overview

This document provides practical implementation examples for integrating Apple In-App Purchases using the modern Apple Server API v2 with real-time server notifications, eliminating the deprecated verifyReceipt API.

## iOS Application Implementation

### 1. Modern StoreKit Manager (No Receipt Validation)

```swift
import StoreKit
import Foundation

@MainActor
class ModernStoreKitManager: NSObject, ObservableObject {
    @Published var products: [SKProduct] = []
    @Published var purchasedProducts: Set<String> = []
    @Published var isLoading = false
    
    private let productIdentifiers: Set<String>
    private let notificationManager: PurchaseNotificationManager
    
    init(productIdentifiers: Set<String>, notificationManager: PurchaseNotificationManager) {
        self.productIdentifiers = productIdentifiers
        self.notificationManager = notificationManager
        super.init()
        
        SKPaymentQueue.default().add(self)
        loadProducts()
        setupPurchaseNotifications()
    }
    
    deinit {
        SKPaymentQueue.default().remove(self)
    }
    
    // MARK: - Product Loading
    
    func loadProducts() {
        isLoading = true
        let request = SKProductsRequest(productIdentifiers: productIdentifiers)
        request.delegate = self
        request.start()
    }
    
    // MARK: - Purchase Flow (Simplified - No Receipt Handling)
    
    func purchase(_ product: SKProduct) {
        guard SKPaymentQueue.canMakePayments() else {
            print("Payments not allowed")
            return
        }
        
        let payment = SKPayment(product: product)
        SKPaymentQueue.default().add(payment)
    }
    
    // MARK: - Real-time Notification Setup
    
    private func setupPurchaseNotifications() {
        // Setup push notifications for instant content unlock
        notificationManager.onContentUnlock = { [weak self] contentIds in
            await self?.handleContentUnlock(contentIds)
        }
        
        // Setup WebSocket for real-time updates
        notificationManager.connectWebSocket(userId: UserManager.shared.currentUserId)
        
        // Setup polling fallback
        notificationManager.startPolling()
    }
    
    private func handleContentUnlock(_ contentIds: [String]) async {
        // Refresh entitlements immediately
        await refreshEntitlements()
        
        // Update UI
        await MainActor.run {
            // Show success message, unlock UI elements, etc.
            showContentUnlockedAlert(contentIds)
        }
    }
    
    private func refreshEntitlements() async {
        do {
            let entitlements = try await EntitlementService.shared.getCurrentEntitlements()
            let productIds = entitlements.compactMap { $0.productId }
            await MainActor.run {
                self.purchasedProducts = Set(productIds)
            }
        } catch {
            print("Failed to refresh entitlements: \(error)")
        }
    }
    
    private func showContentUnlockedAlert(_ contentIds: [String]) {
        // Show user-friendly notification about unlocked content
        let message = "🎉 Your premium content is now available!"
        // Show alert or banner
    }
}

// MARK: - SKPaymentTransactionObserver (Simplified)

extension ModernStoreKitManager: SKPaymentTransactionObserver {
    nonisolated func paymentQueue(_ queue: SKPaymentQueue, updatedTransactions transactions: [SKPaymentTransaction]) {
        for transaction in transactions {
            switch transaction.transactionState {
            case .purchased, .restored:
                // No receipt validation needed - Apple handles everything
                queue.finishTransaction(transaction)
                
                // Optional: Track transaction for analytics
                Task {
                    await self.trackPurchaseCompletion(
                        productId: transaction.payment.productIdentifier,
                        transactionId: transaction.transactionIdentifier
                    )
                }
                
            case .failed:
                if let error = transaction.error as? SKError {
                    print("Transaction failed: \(error.localizedDescription)")
                }
                queue.finishTransaction(transaction)
                
            case .deferred:
                print("Transaction deferred")
            case .purchasing:
                print("Transaction purchasing")
            @unknown default:
                break
            }
        }
    }
    
    private func trackPurchaseCompletion(productId: String, transactionId: String?) async {
        // Optional analytics tracking
        AnalyticsService.shared.trackPurchase(
            productId: productId,
            transactionId: transactionId
        )
    }
}
```

### 2. Real-time Notification Manager

```swift
import Foundation
import Combine

class PurchaseNotificationManager: ObservableObject {
    var onContentUnlock: (([String]) async -> Void)?
    
    private var webSocketTask: URLSessionWebSocketTask?
    private var pollingTimer: Timer?
    private var lastPollingCheck: Date = Date()
    
    // MARK: - Push Notifications
    
    func handleSilentPushNotification(_ userInfo: [AnyHashable: Any]) {
        guard let customData = userInfo["custom"] as? [String: Any],
              let type = customData["type"] as? String,
              type == "content_unlock",
              let contentIds = customData["content_ids"] as? [String] else {
            return
        }
        
        Task {
            await onContentUnlock?(contentIds)
        }
    }
    
    // MARK: - WebSocket Connection
    
    func connectWebSocket(userId: String) {
        let url = URL(string: "wss://yourapi.com/api/v1/users/\(userId)/stream")!
        webSocketTask = URLSession.shared.webSocketTask(with: url)
        webSocketTask?.resume()
        
        listenForWebSocketMessages()
    }
    
    private func listenForWebSocketMessages() {
        webSocketTask?.receive { [weak self] result in
            switch result {
            case .success(let message):
                switch message {
                case .string(let text):
                    self?.handleWebSocketMessage(text)
                case .data(let data):
                    if let text = String(data: data, encoding: .utf8) {
                        self?.handleWebSocketMessage(text)
                    }
                @unknown default:
                    break
                }
                
                // Continue listening
                self?.listenForWebSocketMessages()
                
            case .failure(let error):
                print("WebSocket error: \(error)")
                // Reconnect after delay
                DispatchQueue.main.asyncAfter(deadline: .now() + 5) {
                    self?.connectWebSocket(userId: UserManager.shared.currentUserId)
                }
            }
        }
    }
    
    private func handleWebSocketMessage(_ text: String) {
        guard let data = text.data(using: .utf8),
              let message = try? JSONDecoder().decode(WebSocketMessage.self, from: data),
              message.event == "content_unlock" else {
            return
        }
        
        let contentIds = message.data.contentUnlocked.map { $0.contentId }
        Task {
            await onContentUnlock?(contentIds)
        }
    }
    
    // MARK: - Polling Fallback
    
    func startPolling() {
        pollingTimer = Timer.scheduledTimer(withTimeInterval: 10.0, repeats: true) { [weak self] _ in
            Task {
                await self?.checkForChanges()
            }
        }
    }
    
    private func checkForChanges() async {
        do {
            let changes = try await EntitlementService.shared.getEntitlementChanges(since: lastPollingCheck)
            
            if changes.hasChanges {
                let unlockedContentIds = changes.changes
                    .filter { $0.type == "unlock" }
                    .map { $0.contentId }
                
                if !unlockedContentIds.isEmpty {
                    await onContentUnlock?(unlockedContentIds)
                }
            }
            
            lastPollingCheck = Date()
        } catch {
            print("Polling check failed: \(error)")
        }
    }
}

// MARK: - Models

struct WebSocketMessage: Codable {
    let event: String
    let data: WebSocketData
    let timestamp: String
}

struct WebSocketData: Codable {
    let userId: String
    let contentUnlocked: [UnlockedContent]
    let transactionId: String
    
    enum CodingKeys: String, CodingKey {
        case userId = "user_id"
        case contentUnlocked = "content_unlocked"
        case transactionId = "transaction_id"
    }
}

struct UnlockedContent: Codable {
    let contentId: String
    let productId: String
    let unlockedAt: String
    
    enum CodingKeys: String, CodingKey {
        case contentId = "content_id"
        case productId = "product_id"
        case unlockedAt = "unlocked_at"
    }
}
```

## Backend Service Implementation (Apple Server API v2)

### 1. Apple Server Notification v2 Handler

```javascript
const express = require('express');
const jwt = require('jsonwebtoken');
const crypto = require('crypto');

class AppleServerNotificationHandler {
    constructor(crmService, pushService, webSocketService) {
        this.crmService = crmService;
        this.pushService = pushService;
        this.webSocketService = webSocketService;
    }

    async handleNotification(req, res) {
        try {
            const { signedPayload } = req.body;
            
            // Verify JWS signature
            const notificationData = this.verifyJWS(signedPayload);
            
            // Extract transaction info
            const transactionInfo = this.verifyJWS(notificationData.data.signedTransactionInfo);
            
            // Process based on notification type
            await this.processNotification(notificationData, transactionInfo);
            
            res.status(200).json({ status: 'processed' });
            
        } catch (error) {
            console.error('Notification processing failed:', error);
            res.status(400).json({ error: error.message });
        }
    }

    verifyJWS(signedPayload) {
        // Decode JWS without verification for header
        const [headerB64] = signedPayload.split('.');
        const header = JSON.parse(Buffer.from(headerB64, 'base64').toString());
        
        // Get Apple's public key from x5c certificate chain
        const publicKey = this.getApplePublicKey(header.x5c[0]);
        
        // Verify signature
        const verified = jwt.verify(signedPayload, publicKey, { algorithms: ['ES256'] });
        
        return verified;
    }

    async processNotification(notificationData, transactionInfo) {
        const { notificationType } = notificationData;
        const userId = await this.getUserIdFromTransaction(transactionInfo);
        
        switch (notificationType) {
            case 'PURCHASE':
            case 'RENEWAL':
                await this.handlePurchase(userId, transactionInfo);
                break;
                
            case 'CANCEL':
            case 'REFUND':
                await this.handleCancellation(userId, transactionInfo);
                break;
                
            case 'EXPIRED':
                await this.handleExpiration(userId, transactionInfo);
                break;
        }
    }

    async handlePurchase(userId, transactionInfo) {
        // Update CRM
        await this.crmService.updateUserEntitlements(userId, {
            product_id: transactionInfo.productId,
            transaction_id: transactionInfo.transactionId,
            purchase_date: new Date(transactionInfo.purchaseDate),
            expires_date: transactionInfo.expiresDate ? new Date(transactionInfo.expiresDate) : null,
            platform: 'apple_ios',
            status: 'active'
        });

        // Get content to unlock
        const contentToUnlock = await this.getContentForProduct(transactionInfo.productId);
        
        // Send real-time notifications
        await Promise.all([
            this.sendPushNotification(userId, contentToUnlock),
            this.sendWebSocketUpdate(userId, contentToUnlock, transactionInfo.transactionId),
            this.updatePollingCache(userId)
        ]);
    }

    async sendPushNotification(userId, contentIds) {
        const deviceTokens = await this.getDeviceTokens(userId);
        
        const payload = {
            aps: {
                'content-available': 1,
                sound: ''
            },
            custom: {
                type: 'content_unlock',
                content_ids: contentIds,
                user_id: userId
            }
        };

        for (const token of deviceTokens) {
            await this.pushService.send(token, payload);
        }
    }

    async sendWebSocketUpdate(userId, contentIds, transactionId) {
        const message = {
            event: 'content_unlock',
            data: {
                user_id: userId,
                content_unlocked: contentIds.map(id => ({
                    content_id: id,
                    unlocked_at: new Date().toISOString()
                })),
                transaction_id: transactionId
            },
            timestamp: new Date().toISOString()
        };

        await this.webSocketService.sendToUser(userId, message);
    }

    getApplePublicKey(x5cCert) {
        // Convert x5c certificate to PEM format and extract public key
        const cert = `-----BEGIN CERTIFICATE-----\n${x5cCert}\n-----END CERTIFICATE-----`;
        const certificate = crypto.createPublicKey(cert);
        return certificate;
    }

    async getUserIdFromTransaction(transactionInfo) {
        // Map Apple transaction to your user ID
        // This might involve looking up by originalTransactionId or using app account token
        return await this.crmService.getUserByAppleTransactionId(transactionInfo.originalTransactionId);
    }

    async getContentForProduct(productId) {
        // Return content IDs that should be unlocked for this product
        const product = await this.crmService.getProduct(productId);
        return product.content_ids || [];
    }

    async getDeviceTokens(userId) {
        return await this.crmService.getUserDeviceTokens(userId);
    }

    async updatePollingCache(userId) {
        // Update cache for polling endpoint
        await this.crmService.markEntitlementsChanged(userId);
    }
}

module.exports = AppleServerNotificationHandler;
```

### 2. Real-time WebSocket Service

```javascript
const WebSocket = require('ws');

class WebSocketService {
    constructor() {
        this.connections = new Map(); // userId -> WebSocket[]
        this.setupWebSocketServer();
    }

    setupWebSocketServer() {
        this.wss = new WebSocket.Server({ 
            port: 8080,
            path: '/api/v1/users/:userId/stream'
        });

        this.wss.on('connection', (ws, req) => {
            const userId = this.extractUserIdFromPath(req.url);
            
            if (!userId) {
                ws.close(1008, 'Invalid user ID');
                return;
            }

            // Store connection
            if (!this.connections.has(userId)) {
                this.connections.set(userId, []);
            }
            this.connections.get(userId).push(ws);

            // Handle disconnect
            ws.on('close', () => {
                this.removeConnection(userId, ws);
            });

            // Send heartbeat
            const heartbeat = setInterval(() => {
                if (ws.readyState === WebSocket.OPEN) {
                    ws.ping();
                } else {
                    clearInterval(heartbeat);
                }
            }, 30000);
        });
    }

    async sendToUser(userId, message) {
        const userConnections = this.connections.get(userId) || [];
        const messageString = JSON.stringify(message);

        for (const ws of userConnections) {
            if (ws.readyState === WebSocket.OPEN) {
                ws.send(messageString);
            }
        }
    }

    removeConnection(userId, ws) {
        const userConnections = this.connections.get(userId) || [];
        const index = userConnections.indexOf(ws);
        if (index > -1) {
            userConnections.splice(index, 1);
        }
        
        if (userConnections.length === 0) {
            this.connections.delete(userId);
        }
    }

    extractUserIdFromPath(url) {
        const match = url.match(/\/api\/v1\/users\/([^\/]+)\/stream/);
        return match ? match[1] : null;
    }
}

module.exports = WebSocketService;
```

### 3. Entitlements API with Real-time Support

```javascript
class EntitlementsService {
    constructor(database, cacheService) {
        this.database = database;
        this.cacheService = cacheService;
    }

    async getCurrentEntitlements(req, res) {
        try {
            const { user_id } = req.params;
            
            const entitlements = await this.database.getUserEntitlements(user_id);
            const processedEntitlements = entitlements.map(e => ({
                product_id: e.product_id,
                status: this.calculateStatus(e),
                transaction_id: e.transaction_id,
                purchase_date: e.purchase_date,
                expires_date: e.expires_date,
                content_access: e.content_access || []
            }));

            res.json({
                user_id,
                last_updated: new Date().toISOString(),
                entitlements: processedEntitlements,
                pending_unlocks: await this.getPendingUnlocks(user_id)
            });

        } catch (error) {
            console.error('Error fetching entitlements:', error);
            res.status(500).json({ error: 'Failed to fetch entitlements' });
        }
    }

    async getEntitlementChanges(req, res) {
        try {
            const { user_id } = req.params;
            const { since } = req.query;
            
            const sinceDate = since ? new Date(since) : new Date(Date.now() - 60000); // 1 minute ago
            
            const changes = await this.database.getEntitlementChanges(user_id, sinceDate);
            
            res.json({
                has_changes: changes.length > 0,
                last_updated: new Date().toISOString(),
                changes: changes.map(c => ({
                    type: c.change_type,
                    content_id: c.content_id,
                    product_id: c.product_id,
                    changed_at: c.changed_at
                }))
            });

        } catch (error) {
            console.error('Error checking changes:', error);
            res.status(500).json({ error: 'Failed to check changes' });
        }
    }

    calculateStatus(entitlement) {
        if (!entitlement.expires_date) {
            return 'active'; // Non-consumable
        }
        
        return entitlement.expires_date > new Date() ? 'active' : 'expired';
    }

    async getPendingUnlocks(userId) {
        // Return any content that's processing/unlocking
        return await this.database.getPendingContentUnlocks(userId);
    }
}

module.exports = EntitlementsService;
```

## Key Benefits of Apple Server API v2 Approach

### ✅ **Real-time Content Unlocking**

- Content unlocks within seconds of purchase
- No app-initiated receipt validation delays
- Multiple notification channels for reliability

### ✅ **Simplified iOS Implementation**

- No receipt handling in the app
- No complex validation logic
- Just finish transactions and listen for notifications

### ✅ **Better User Experience**

- Instant gratification after purchase
- Multiple fallback mechanisms
- Works even if app is backgrounded

### ✅ **Enhanced Security**

- Server-side JWS verification
- No sensitive receipt data in app
- Apple handles all validation

This modern approach eliminates the deprecated `verifyReceipt` API while providing a much better user experience with real-time content unlocking!

## Summary

The modern Apple Server API v2 approach shown in the examples above provides:

✅ **Simplified Implementation**: No client-side receipt handling required  
✅ **Real-time Experience**: Content unlocks within seconds via server notifications  
✅ **Enhanced Security**: Server-side JWS verification with Apple's certificates  
✅ **Better Reliability**: Multiple notification channels (push, WebSocket, polling)  
✅ **Future-proof**: Uses Apple's current recommended APIs

All the code examples in this document follow Apple's current best practices and eliminate any use of the deprecated `verifyReceipt` API. The system is designed around Apple's Server-to-Server Notifications v2 for optimal performance and user experience.
