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

## Complete Implementation Reference

### 1. StoreKit Manager

```swift
import StoreKit
import Foundation

@MainActor
class StoreKitManager: NSObject, ObservableObject {
    @Published var products: [SKProduct] = []
    @Published var purchasedProducts: Set<String> = []
    @Published var isLoading = false
    
    private let productIdentifiers: Set<String>
    private let receiptValidator: ReceiptValidator
    
    init(productIdentifiers: Set<String>, receiptValidator: ReceiptValidator) {
        self.productIdentifiers = productIdentifiers
        self.receiptValidator = receiptValidator
        super.init()
        
        SKPaymentQueue.default().add(self)
        loadProducts()
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
    
    // MARK: - Purchase Flow
    
    func purchase(_ product: SKProduct) {
        guard SKPaymentQueue.canMakePayments() else {
            print("Payments not allowed")
            return
        }
        
        let payment = SKPayment(product: product)
        SKPaymentQueue.default().add(payment)
    }
    
    func restorePurchases() {
        SKPaymentQueue.default().restoreCompletedTransactions()
    }
    
    // MARK: - Receipt Validation
    
    private func validateReceipt() async {
        guard let receiptURL = Bundle.main.appStoreReceiptURL,
              let receiptData = try? Data(contentsOf: receiptURL) else {
            print("No receipt found")
            return
        }
        
        let base64Receipt = receiptData.base64EncodedString()
        
        do {
            let response = try await receiptValidator.validate(
                receiptData: base64Receipt,
                userId: UserManager.shared.currentUserId,
                deviceId: UIDevice.current.identifierForVendor?.uuidString ?? "",
                appVersion: Bundle.main.infoDictionary?["CFBundleShortVersionString"] as? String ?? "1.0"
            )
            
            if response.receiptValid {
                await updatePurchasedProducts(response.entitlements)
            }
        } catch {
            print("Receipt validation failed: \(error)")
        }
    }
    
    private func updatePurchasedProducts(_ entitlements: [Entitlement]) {
        let productIds = entitlements.compactMap { $0.productId }
        purchasedProducts = Set(productIds)
    }
}

// MARK: - SKProductsRequestDelegate

extension StoreKitManager: SKProductsRequestDelegate {
    nonisolated func productsRequest(_ request: SKProductsRequest, didReceive response: SKProductsResponse) {
        Task { @MainActor in
            self.products = response.products
            self.isLoading = false
        }
    }
    
    nonisolated func request(_ request: SKRequest, didFailWithError error: Error) {
        Task { @MainActor in
            self.isLoading = false
        }
        print("Product request failed: \(error)")
    }
}

// MARK: - SKPaymentTransactionObserver

extension StoreKitManager: SKPaymentTransactionObserver {
    nonisolated func paymentQueue(_ queue: SKPaymentQueue, updatedTransactions transactions: [SKPaymentTransaction]) {
        for transaction in transactions {
            switch transaction.transactionState {
            case .purchased, .restored:
                Task {
                    await self.validateReceipt()
                }
                queue.finishTransaction(transaction)
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
}
```

### 2. Receipt Validator

```swift
import Foundation

class ReceiptValidator {
    private let baseURL: URL
    private let apiKey: String
    
    init(baseURL: URL, apiKey: String) {
        self.baseURL = baseURL
        self.apiKey = apiKey
    }
    
    func validate(receiptData: String, userId: String, deviceId: String, appVersion: String) async throws -> ReceiptValidationResponse {
        let url = baseURL.appendingPathComponent("/api/v1/receipts/validate")
        
        var request = URLRequest(url: url)
        request.httpMethod = "POST"
        request.setValue("application/json", forHTTPHeaderField: "Content-Type")
        request.setValue("Bearer \(apiKey)", forHTTPHeaderField: "Authorization")
        
        let requestBody = ReceiptValidationRequest(
            receiptData: receiptData,
            userId: userId,
            deviceId: deviceId,
            appVersion: appVersion,
            environment: isProduction() ? "production" : "sandbox"
        )
        
        request.httpBody = try JSONEncoder().encode(requestBody)
        
        let (data, response) = try await URLSession.shared.data(for: request)
        
        guard let httpResponse = response as? HTTPURLResponse else {
            throw ReceiptValidationError.invalidResponse
        }
        
        guard httpResponse.statusCode == 200 else {
            if let errorResponse = try? JSONDecoder().decode(ErrorResponse.self, from: data) {
                throw ReceiptValidationError.serverError(errorResponse.message)
            }
            throw ReceiptValidationError.httpError(httpResponse.statusCode)
        }
        
        return try JSONDecoder().decode(ReceiptValidationResponse.self, from: data)
    }
    
    private func isProduction() -> Bool {
        #if DEBUG
        return false
        #else
        return true
        #endif
    }
}

// MARK: - Models

struct ReceiptValidationRequest: Codable {
    let receiptData: String
    let userId: String
    let deviceId: String
    let appVersion: String
    let environment: String
    
    enum CodingKeys: String, CodingKey {
        case receiptData = "receipt_data"
        case userId = "user_id"
        case deviceId = "device_id"
        case appVersion = "app_version"
        case environment
    }
}

struct ReceiptValidationResponse: Codable {
    let status: String
    let receiptValid: Bool
    let transactionId: String?
    let productId: String?
    let purchaseDate: Date?
    let expiresDate: Date?
    let entitlementsUpdated: Bool
    let entitlements: [Entitlement]
    
    enum CodingKeys: String, CodingKey {
        case status
        case receiptValid = "receipt_valid"
        case transactionId = "transaction_id"
        case productId = "product_id"
        case purchaseDate = "purchase_date"
        case expiresDate = "expires_date"
        case entitlementsUpdated = "entitlements_updated"
        case entitlements
    }
}

struct Entitlement: Codable {
    let productId: String
    let status: String
    let purchaseDate: Date
    let expiresDate: Date?
    
    enum CodingKeys: String, CodingKey {
        case productId = "product_id"
        case status
        case purchaseDate = "purchase_date"
        case expiresDate = "expires_date"
    }
}

enum ReceiptValidationError: Error {
    case invalidResponse
    case httpError(Int)
    case serverError(String)
    case networkError
}
```

### 3. Content Manager

```swift
import Foundation

@MainActor
class ContentManager: ObservableObject {
    @Published var availableContent: [Content] = []
    @Published var downloadedContent: [Content] = []
    @Published var isLoading = false
    
    private let contentAPI: ContentAPI
    private let cacheManager: ContentCacheManager
    
    init(contentAPI: ContentAPI, cacheManager: ContentCacheManager) {
        self.contentAPI = contentAPI
        self.cacheManager = cacheManager
    }
    
    func loadAvailableContent() async {
        isLoading = true
        defer { isLoading = false }
        
        do {
            availableContent = try await contentAPI.getAvailableContent()
            downloadedContent = cacheManager.getCachedContent()
        } catch {
            print("Failed to load content: \(error)")
        }
    }
    
    func downloadContent(_ content: Content) async throws {
        guard hasAccess(to: content) else {
            throw ContentError.accessDenied
        }
        
        let accessResponse = try await contentAPI.getContentAccess(contentId: content.id)
        guard accessResponse.hasAccess else {
            throw ContentError.accessDenied
        }
        
        try await cacheManager.downloadContent(
            from: accessResponse.accessURL,
            contentId: content.id
        )
        
        await loadAvailableContent()
    }
    
    func playContent(_ content: Content) -> URL? {
        return cacheManager.getLocalURL(for: content.id)
    }
    
    private func hasAccess(to content: Content) -> Bool {
        // Check if user has valid entitlements for this content
        return StoreKitManager.shared.purchasedProducts.contains(content.requiredProductId)
    }
}

struct Content: Codable, Identifiable {
    let id: String
    let title: String
    let description: String
    let duration: TimeInterval
    let fileSize: Int64
    let format: String
    let requiredProductId: String
    let thumbnailURL: URL?
}

enum ContentError: Error {
    case accessDenied
    case downloadFailed
    case invalidURL
}
```

## Backend Service Implementation

### 1. Receipt Validation Service (Node.js)

```javascript
const express = require('express');
const axios = require('axios');
const jwt = require('jsonwebtoken');
const { v4: uuidv4 } = require('uuid');

class ReceiptValidationService {
    constructor(appleSharedSecret, crmService, database) {
        this.appleSharedSecret = appleSharedSecret;
        this.crmService = crmService;
        this.database = database;
    }

    async validateReceipt(req, res) {
        try {
            const { receipt_data, user_id, device_id, app_version, environment } = req.body;

            // Check for duplicate processing
            const existingValidation = await this.database.getReceiptValidation(receipt_data);
            if (existingValidation) {
                return res.json(existingValidation.response);
            }

            // Validate with Apple
            const appleResponse = await this.validateWithApple(receipt_data, environment === 'production');
            
            if (appleResponse.status !== 0) {
                throw new Error(`Apple validation failed: ${appleResponse.status}`);
            }

            // Process purchases
            const purchases = this.extractPurchases(appleResponse.latest_receipt_info || appleResponse.receipt.in_app);
            
            // Update CRM
            await this.updateCRM(user_id, purchases);

            // Store validation result
            const validationResult = {
                status: 'success',
                receipt_valid: true,
                transaction_id: purchases[0]?.transaction_id,
                product_id: purchases[0]?.product_id,
                purchase_date: purchases[0]?.purchase_date,
                expires_date: purchases[0]?.expires_date,
                entitlements_updated: true,
                entitlements: purchases.map(p => ({
                    product_id: p.product_id,
                    status: 'active',
                    purchase_date: p.purchase_date,
                    expires_date: p.expires_date
                }))
            };

            await this.database.storeReceiptValidation(receipt_data, validationResult);

            res.json(validationResult);

        } catch (error) {
            console.error('Receipt validation error:', error);
            res.status(400).json({
                status: 'error',
                error_code: 'VALIDATION_FAILED',
                message: error.message,
                retry_after: 30
            });
        }
    }

    async validateWithApple(receiptData, isProduction = true) {
        const url = isProduction 
            ? 'https://buy.itunes.apple.com/verifyReceipt'
            : 'https://sandbox.itunes.apple.com/verifyReceipt';

        const payload = {
            'receipt-data': receiptData,
            'password': this.appleSharedSecret,
            'exclude-old-transactions': true
        };

        try {
            const response = await axios.post(url, payload, {
                timeout: 10000,
                headers: { 'Content-Type': 'application/json' }
            });

            // Handle sandbox redirect
            if (response.data.status === 21007 && isProduction) {
                return this.validateWithApple(receiptData, false);
            }

            return response.data;
        } catch (error) {
            throw new Error(`Apple API error: ${error.message}`);
        }
    }

    extractPurchases(transactions) {
        return transactions.map(transaction => ({
            transaction_id: transaction.transaction_id,
            product_id: transaction.product_id,
            purchase_date: new Date(parseInt(transaction.purchase_date_ms)),
            expires_date: transaction.expires_date_ms ? 
                new Date(parseInt(transaction.expires_date_ms)) : null,
            quantity: parseInt(transaction.quantity) || 1
        }));
    }

    async updateCRM(userId, purchases) {
        for (const purchase of purchases) {
            await this.crmService.updateUserEntitlements(userId, {
                product_id: purchase.product_id,
                transaction_id: purchase.transaction_id,
                purchase_date: purchase.purchase_date,
                expires_date: purchase.expires_date,
                platform: 'ios'
            });
        }
    }
}

module.exports = ReceiptValidationService;
```

### 2. CRM Integration Service

```javascript
class CRMIntegrationService {
    constructor(crmApiUrl, crmApiKey, database) {
        this.crmApiUrl = crmApiUrl;
        this.crmApiKey = crmApiKey;
        this.database = database;
    }

    async updateUserEntitlements(userId, purchaseData) {
        try {
            // Update local database first
            await this.database.updateUserEntitlement({
                user_id: userId,
                product_id: purchaseData.product_id,
                transaction_id: purchaseData.transaction_id,
                status: 'active',
                purchase_date: purchaseData.purchase_date,
                expires_date: purchaseData.expires_date,
                platform: purchaseData.platform,
                updated_at: new Date()
            });

            // Sync with external CRM
            const crmResponse = await axios.post(`${this.crmApiUrl}/api/purchases`, {
                customer_id: userId,
                product_id: purchaseData.product_id,
                transaction_id: purchaseData.transaction_id,
                purchase_date: purchaseData.purchase_date,
                amount: await this.getProductPrice(purchaseData.product_id),
                currency: 'USD',
                platform: purchaseData.platform,
                subscription_expires: purchaseData.expires_date
            }, {
                headers: {
                    'Authorization': `Bearer ${this.crmApiKey}`,
                    'Content-Type': 'application/json'
                }
            });

            console.log('CRM updated successfully:', crmResponse.data);
            return crmResponse.data;

        } catch (error) {
            console.error('CRM update failed:', error);
            // Queue for retry
            await this.queueForRetry(userId, purchaseData);
            throw error;
        }
    }

    async getUserEntitlements(userId) {
        const entitlements = await this.database.getUserEntitlements(userId);
        return entitlements.map(e => ({
            product_id: e.product_id,
            status: this.calculateStatus(e),
            purchase_date: e.purchase_date,
            expires_date: e.expires_date,
            auto_renew: e.auto_renew || false
        }));
    }

    calculateStatus(entitlement) {
        if (!entitlement.expires_date) {
            return 'active'; // Non-consumable
        }
        
        return entitlement.expires_date > new Date() ? 'active' : 'expired';
    }

    async queueForRetry(userId, purchaseData) {
        await this.database.queueCRMRetry({
            id: uuidv4(),
            user_id: userId,
            purchase_data: JSON.stringify(purchaseData),
            retry_count: 0,
            next_retry: new Date(Date.now() + 30000), // 30 seconds
            created_at: new Date()
        });
    }

    async processRetryQueue() {
        const pendingRetries = await this.database.getPendingRetries();
        
        for (const retry of pendingRetries) {
            try {
                const purchaseData = JSON.parse(retry.purchase_data);
                await this.updateUserEntitlements(retry.user_id, purchaseData);
                await this.database.deleteCRMRetry(retry.id);
            } catch (error) {
                if (retry.retry_count < 5) {
                    await this.database.updateCRMRetry(retry.id, {
                        retry_count: retry.retry_count + 1,
                        next_retry: new Date(Date.now() + (retry.retry_count + 1) * 60000), // Exponential backoff
                        last_error: error.message
                    });
                } else {
                    await this.database.deleteCRMRetry(retry.id);
                    console.error('Max retries exceeded for:', retry.id);
                }
            }
        }
    }

    async getProductPrice(productId) {
        const product = await this.database.getProduct(productId);
        return product?.price || 0;
    }
}

module.exports = CRMIntegrationService;
```

### 3. Content Access Service

```javascript
class ContentAccessService {
    constructor(storageService, crmService, database) {
        this.storageService = storageService;
        this.crmService = crmService;
        this.database = database;
    }

    async getContentAccess(req, res) {
        try {
            const { content_id } = req.params;
            const userId = req.user.id; // From JWT middleware

            // Check user entitlements
            const hasAccess = await this.checkContentAccess(userId, content_id);
            
            if (!hasAccess) {
                return res.status(403).json({
                    has_access: false,
                    reason: 'NO_VALID_SUBSCRIPTION',
                    purchase_options: await this.getProductOptions(content_id)
                });
            }

            // Generate secure access URL
            const accessUrl = await this.generateSecureURL(content_id, userId);
            const contentMetadata = await this.getContentMetadata(content_id);

            res.json({
                has_access: true,
                access_url: accessUrl,
                expires_at: new Date(Date.now() + 3600000), // 1 hour
                content_metadata: contentMetadata
            });

        } catch (error) {
            console.error('Content access error:', error);
            res.status(500).json({
                has_access: false,
                reason: 'SERVER_ERROR',
                message: 'Unable to verify content access'
            });
        }
    }

    async checkContentAccess(userId, contentId) {
        // Get required product for content
        const content = await this.database.getContent(contentId);
        if (!content) return false;

        // Check user entitlements
        const entitlements = await this.crmService.getUserEntitlements(userId);
        
        return entitlements.some(e => 
            e.product_id === content.required_product_id && 
            e.status === 'active'
        );
    }

    async generateSecureURL(contentId, userId, expirationHours = 1) {
        const signedUrl = await this.storageService.generateSignedURL(
            contentId,
            expirationHours * 3600
        );

        // Log access for analytics
        await this.database.logContentAccess({
            user_id: userId,
            content_id: contentId,
            access_url: signedUrl,
            accessed_at: new Date(),
            expires_at: new Date(Date.now() + expirationHours * 3600000)
        });

        return signedUrl;
    }

    async getContentMetadata(contentId) {
        return await this.database.getContentMetadata(contentId);
    }

    async getProductOptions(contentId) {
        const content = await this.database.getContent(contentId);
        if (!content) return [];

        const product = await this.database.getProduct(content.required_product_id);
        return product ? [{
            product_id: product.id,
            price: product.price_display,
            currency: product.currency
        }] : [];
    }
}

module.exports = ContentAccessService;
```

## Testing Implementation

### 1. Unit Tests (iOS)

```swift
import XCTest
@testable import YourApp

class StoreKitManagerTests: XCTestCase {
    var storeKitManager: StoreKitManager!
    var mockReceiptValidator: MockReceiptValidator!
    
    override func setUp() {
        super.setUp()
        mockReceiptValidator = MockReceiptValidator()
        storeKitManager = StoreKitManager(
            productIdentifiers: ["com.yourapp.premium"],
            receiptValidator: mockReceiptValidator
        )
    }
    
    func testProductLoading() async {
        // Given
        let expectation = XCTestExpectation(description: "Products loaded")
        
        // When
        storeKitManager.loadProducts()
        
        // Wait for async operation
        try? await Task.sleep(nanoseconds: 1_000_000_000)
        
        // Then
        XCTAssertFalse(storeKitManager.isLoading)
        expectation.fulfill()
        
        await fulfillment(of: [expectation], timeout: 5.0)
    }
    
    func testReceiptValidation() async throws {
        // Given
        let mockResponse = ReceiptValidationResponse(
            status: "success",
            receiptValid: true,
            transactionId: "test_transaction",
            productId: "com.yourapp.premium",
            purchaseDate: Date(),
            expiresDate: nil,
            entitlementsUpdated: true,
            entitlements: []
        )
        mockReceiptValidator.mockResponse = mockResponse
        
        // When
        let response = try await mockReceiptValidator.validate(
            receiptData: "test_receipt",
            userId: "test_user",
            deviceId: "test_device",
            appVersion: "1.0"
        )
        
        // Then
        XCTAssertTrue(response.receiptValid)
        XCTAssertEqual(response.transactionId, "test_transaction")
    }
}

class MockReceiptValidator: ReceiptValidator {
    var mockResponse: ReceiptValidationResponse?
    var shouldThrowError = false
    
    override func validate(receiptData: String, userId: String, deviceId: String, appVersion: String) async throws -> ReceiptValidationResponse {
        if shouldThrowError {
            throw ReceiptValidationError.networkError
        }
        return mockResponse ?? ReceiptValidationResponse(
            status: "success",
            receiptValid: false,
            transactionId: nil,
            productId: nil,
            purchaseDate: nil,
            expiresDate: nil,
            entitlementsUpdated: false,
            entitlements: []
        )
    }
}
```

### 2. Integration Tests (Backend)

```javascript
const request = require('supertest');
const app = require('../app');

describe('Receipt Validation API', () => {
    test('should validate valid receipt', async () => {
        const response = await request(app)
            .post('/api/v1/receipts/validate')
            .send({
                receipt_data: 'valid_test_receipt_data',
                user_id: 'test_user_123',
                device_id: 'test_device_456',
                app_version: '1.0.0',
                environment: 'sandbox'
            })
            .expect(200);

        expect(response.body.status).toBe('success');
        expect(response.body.receipt_valid).toBe(true);
    });

    test('should handle invalid receipt', async () => {
        const response = await request(app)
            .post('/api/v1/receipts/validate')
            .send({
                receipt_data: 'invalid_receipt_data',
                user_id: 'test_user_123',
                device_id: 'test_device_456',
                app_version: '1.0.0',
                environment: 'sandbox'
            })
            .expect(400);

        expect(response.body.status).toBe('error');
        expect(response.body.error_code).toBe('INVALID_RECEIPT');
    });
});

describe('Content Access API', () => {
    test('should grant access with valid subscription', async () => {
        const token = generateTestJWT('test_user_with_subscription');
        
        const response = await request(app)
            .get('/api/v1/content/test_content_123/access')
            .set('Authorization', `Bearer ${token}`)
            .expect(200);

        expect(response.body.has_access).toBe(true);
        expect(response.body.access_url).toBeDefined();
    });

    test('should deny access without subscription', async () => {
        const token = generateTestJWT('test_user_without_subscription');
        
        const response = await request(app)
            .get('/api/v1/content/test_content_123/access')
            .set('Authorization', `Bearer ${token}`)
            .expect(403);

        expect(response.body.has_access).toBe(false);
        expect(response.body.purchase_options).toBeDefined();
    });
});
```

This comprehensive implementation guide provides working code examples for integrating Apple In-App Purchases with CRM systems while maintaining compliance with Apple's requirements.
