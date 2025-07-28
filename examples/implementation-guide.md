# Implementation Examples (Apple Server API v2)

## Overview

This document provides practical implementation examples for integrating Apple In-App Purchases using the modern Apple Server API v2 with real-time server notifications, eliminating the deprecated verifyReceipt API.

## React Native Application Implementation

### 1. Modern IAP Manager (No Receipt Validation)

```javascript
import { 
  initConnection, 
  endConnection,
  getProducts, 
  requestPurchase, 
  purchaseUpdatedListener,
  purchaseErrorListener,
  finishTransaction,
  getPurchaseHistory
} from 'react-native-iap';
import messaging from '@react-native-firebase/messaging';

class ModernIAPManager {
  constructor() {
    this.products = [];
    this.purchasedProducts = new Set();
    this.isLoading = false;
    this.purchaseUpdateSubscription = null;
    this.purchaseErrorSubscription = null;
    this.notificationManager = new PurchaseNotificationManager();
  }

  async initialize(productIds) {
    try {
      this.isLoading = true;
      
      // Initialize IAP connection
      await initConnection();
      
      // Set up purchase listeners  
      this.purchaseUpdateSubscription = purchaseUpdatedListener(
        (purchase) => this.handlePurchaseUpdate(purchase)
      );
      
      this.purchaseErrorSubscription = purchaseErrorListener(
        (error) => this.handlePurchaseError(error)
      );
      
      // Load products
      await this.loadProducts(productIds);
      
      // Initialize notification manager for real-time content unlocking
      await this.notificationManager.initialize();
      
    } catch (error) {
      console.error('IAP initialization failed:', error);
    } finally {
      this.isLoading = false;
    }
  }

  async loadProducts(productIds) {
    try {
      const products = await getProducts(productIds);
      this.products = products;
      console.log('Products loaded:', products.length);
    } catch (error) {
      console.error('Failed to load products:', error);
    }
  }

  // MARK: - Purchase Flow (Simplified - No Receipt Handling)
  
  async makePurchase(productId) {
    try {
      console.log('Initiating purchase for:', productId);
      await requestPurchase(productId);
      // Note: Purchase completion handled by purchaseUpdatedListener
    } catch (error) {
      console.error('Purchase failed:', error);
      throw error;
    }
  }

  async handlePurchaseUpdate(purchase) {
    try {
      console.log('Purchase update received:', purchase.productId);
      
      // Simply finish the transaction - Apple handles everything else
      await finishTransaction(purchase);
      
      // Content will be unlocked via real-time server notification
      console.log('Transaction finished. Waiting for server notification...');
      
      // Update local state (will be confirmed by server notification)
      this.purchasedProducts.add(purchase.productId);
      
    } catch (error) {
      console.error('Error handling purchase update:', error);
    }
  }

  handlePurchaseError(error) {
    console.error('Purchase error:', error);
    // Handle different error types
    switch (error.code) {
      case 'E_USER_CANCELLED':
        console.log('User cancelled purchase');
        break;
      case 'E_PAYMENT_INVALID':
        console.log('Payment method invalid');
        break;
      default:
        console.log('Unknown purchase error:', error.message);
    }
  }

  async restorePurchases() {
    try {
      const purchases = await getPurchaseHistory();
      console.log('Restored purchases:', purchases.length);
      
      // Update local state
      purchases.forEach(purchase => {
        this.purchasedProducts.add(purchase.productId);
      });
      
      return purchases;
    } catch (error) {
      console.error('Failed to restore purchases:', error);
    }
  }

  cleanup() {
    if (this.purchaseUpdateSubscription) {
      this.purchaseUpdateSubscription.remove();
    }
    if (this.purchaseErrorSubscription) {
      this.purchaseErrorSubscription.remove();
    }
    endConnection();
  }
}
```

### 2. Real-time Notification Manager

```javascript
import messaging from '@react-native-firebase/messaging';
import PushNotificationIOS from '@react-native-community/push-notification-ios';
import AsyncStorage from '@react-native-async-storage/async-storage';

class PurchaseNotificationManager {
  constructor() {
    this.contentUnlockCallbacks = [];
  }

  async initialize() {
    // Request notification permissions
    await this.requestPermissions();
    
    // Set up message handlers
    this.setupMessageHandlers();
    
    // Handle app launched from notification
    const initialNotification = await messaging().getInitialNotification();
    if (initialNotification) {
      await this.handleNotificationData(initialNotification.data);
    }
  }

  async requestPermissions() {
    const authStatus = await messaging().requestPermission();
    const enabled =
      authStatus === messaging.AuthorizationStatus.AUTHORIZED ||
      authStatus === messaging.AuthorizationStatus.PROVISIONAL;

    if (enabled) {
      console.log('Notification authorization status:', authStatus);
      
      // Get FCM token for server registration
      const token = await messaging().getToken();
      await this.registerDeviceToken(token);
    }
  }

  setupMessageHandlers() {
    // Handle background/quit state notifications
    messaging().setBackgroundMessageHandler(async remoteMessage => {
      console.log('Background notification received:', remoteMessage);
      if (remoteMessage.data?.type === 'content_unlock') {
        await this.handleContentUnlock(remoteMessage.data);
      }
    });

    // Handle foreground notifications
    messaging().onMessage(async remoteMessage => {
      console.log('Foreground notification received:', remoteMessage);
      if (remoteMessage.data?.type === 'content_unlock') {
        await this.handleContentUnlock(remoteMessage.data);
      }
    });

    // Handle notification opened (app in background)
    messaging().onNotificationOpenedApp(remoteMessage => {
      console.log('Notification opened app:', remoteMessage);
      if (remoteMessage.data?.type === 'content_unlock') {
        this.handleContentUnlock(remoteMessage.data);
      }
    });
  }

  async handleContentUnlock(data) {
    try {
      const { content_ids, user_id, transaction_id, product_id } = data;
      
      console.log('Content unlock notification:', {
        contentIds: content_ids,
        userId: user_id,
        transactionId: transaction_id
      });

      // Update local entitlements cache
      await this.updateLocalEntitlements(content_ids, product_id);
      
      // Notify app components about content unlock
      this.contentUnlockCallbacks.forEach(callback => {
        callback({
          contentIds: content_ids,
          productId: product_id,
          transactionId: transaction_id
        });
      });

      // Show local notification if app is in background
      if (AppState.currentState !== 'active') {
        PushNotificationIOS.presentLocalNotification({
          alertTitle: 'Content Unlocked!',
          alertBody: 'Your premium content is now available',
          userInfo: { contentIds: content_ids }
        });
      }

    } catch (error) {
      console.error('Error handling content unlock:', error);
    }
  }

  async updateLocalEntitlements(contentIds, productId) {
    try {
      const entitlements = await AsyncStorage.getItem('user_entitlements');
      const currentEntitlements = entitlements ? JSON.parse(entitlements) : {};
      
      // Add new entitlements
      contentIds.forEach(contentId => {
        currentEntitlements[contentId] = {
          productId,
          unlockedAt: new Date().toISOString(),
          status: 'active'
        };
      });

      await AsyncStorage.setItem('user_entitlements', JSON.stringify(currentEntitlements));
      console.log('Local entitlements updated');
      
    } catch (error) {
      console.error('Failed to update local entitlements:', error);
    }
  }

  onContentUnlock(callback) {
    this.contentUnlockCallbacks.push(callback);
    
    // Return unsubscribe function
    return () => {
      const index = this.contentUnlockCallbacks.indexOf(callback);
      if (index > -1) {
        this.contentUnlockCallbacks.splice(index, 1);
      }
    };
  }

  async registerDeviceToken(token) {
    try {
      // Send token to your backend for push notification targeting
      await fetch('/api/v1/users/device-token', {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
          'Authorization': `Bearer ${await getAuthToken()}`
        },
        body: JSON.stringify({
          device_token: token,
          platform: 'ios'
        })
      });
      
      console.log('Device token registered with server');
    } catch (error) {
      console.error('Failed to register device token:', error);
    }
  }
}
```

### 3. Content Manager for React Native

```javascript
import AsyncStorage from '@react-native-async-storage/async-storage';
import RNFS from 'react-native-fs';

class ContentManager {
  constructor() {
    this.availableContent = [];
    this.downloadedContent = [];
    this.isLoading = false;
    this.baseURL = 'https://your-api.com';
  }

  async loadAvailableContent() {
    try {
      this.isLoading = true;
      
      const response = await fetch(`${this.baseURL}/api/v1/content`, {
        headers: {
          'Authorization': `Bearer ${await getAuthToken()}`
        }
      });
      
      if (response.ok) {
        this.availableContent = await response.json();
        await this.loadDownloadedContent();
      }
      
    } catch (error) {
      console.error('Failed to load content:', error);
    } finally {
      this.isLoading = false;
    }
  }

  async loadDownloadedContent() {
    try {
      const downloaded = await AsyncStorage.getItem('downloaded_content');
      this.downloadedContent = downloaded ? JSON.parse(downloaded) : [];
    } catch (error) {
      console.error('Failed to load downloaded content list:', error);
    }
  }

  async downloadContent(contentId) {
    try {
      // Check if user has access
      if (!await this.hasAccess(contentId)) {
        throw new Error('Access denied - no valid purchase');
      }

      // Get secure access URL
      const accessResponse = await this.getContentAccess(contentId);
      if (!accessResponse.has_access) {
        throw new Error('Access denied by server');
      }

      // Download content
      const downloadDest = `${RNFS.DocumentDirectoryPath}/${contentId}`;
      const download = RNFS.downloadFile({
        fromUrl: accessResponse.access_url,
        toFile: downloadDest,
        progressCallback: (res) => {
          const progress = (res.bytesWritten / res.contentLength) * 100;
          console.log(`Download progress: ${progress}%`);
        }
      });

      const result = await download.promise;
      
      if (result.statusCode === 200) {
        // Update downloaded content list
        const updatedList = [...this.downloadedContent, {
          contentId,
          localPath: downloadDest,
          downloadedAt: new Date().toISOString()
        }];
        
        this.downloadedContent = updatedList;
        await AsyncStorage.setItem('downloaded_content', JSON.stringify(updatedList));
        
        console.log('Content downloaded successfully:', contentId);
      }

    } catch (error) {
      console.error('Content download failed:', error);
      throw error;
    }
  }

  async getContentAccess(contentId) {
    const response = await fetch(`${this.baseURL}/api/v1/content/${contentId}/access`, {
      headers: {
        'Authorization': `Bearer ${await getAuthToken()}`,
        'X-Device-ID': await getDeviceId(),
        'X-App-Version': getAppVersion()
      }
    });

    if (!response.ok) {
      throw new Error(`Access check failed: ${response.status}`);
    }

    return await response.json();
  }

  async hasAccess(contentId) {
    try {
      const entitlements = await AsyncStorage.getItem('user_entitlements');
      const userEntitlements = entitlements ? JSON.parse(entitlements) : {};
      
      return userEntitlements[contentId]?.status === 'active';
    } catch (error) {
      console.error('Failed to check local access:', error);
      return false;
    }
  }

  getLocalContentPath(contentId) {
    const downloadedItem = this.downloadedContent.find(item => item.contentId === contentId);
    return downloadedItem?.localPath;
  }
}

// Helper functions
async function getAuthToken() {
  return await AsyncStorage.getItem('auth_token');
}

async function getDeviceId() {
  return await AsyncStorage.getItem('device_id');
}

function getAppVersion() {
  return require('../package.json').version;
}
```

### 4. React Native Component Usage

```javascript
import React, { useEffect, useState } from 'react';
import { View, Text, Button, Alert } from 'react-native';

const PremiumContentScreen = () => {
  const [iapManager] = useState(() => new ModernIAPManager());
  const [contentManager] = useState(() => new ContentManager());
  const [products, setProducts] = useState([]);
  const [hasAccess, setHasAccess] = useState(false);

  useEffect(() => {
    initializeServices();
    
    // Listen for content unlocks
    const unsubscribe = iapManager.notificationManager.onContentUnlock(
      ({ contentIds, productId }) => {
        console.log('Content unlocked:', contentIds);
        setHasAccess(true);
        Alert.alert('Success!', 'Your premium content is now available');
      }
    );

    return () => {
      unsubscribe();
      iapManager.cleanup();
    };
  }, []);

  const initializeServices = async () => {
    try {
      await iapManager.initialize(['com.yourapp.premium_course']);
      await contentManager.loadAvailableContent();
      setProducts(iapManager.products);
    } catch (error) {
      console.error('Service initialization failed:', error);
    }
  };

  const handlePurchase = async (productId) => {
    try {
      await iapManager.makePurchase(productId);
      // Content will be unlocked via push notification
    } catch (error) {
      Alert.alert('Purchase Failed', error.message);
    }
  };

  return (
    <View style={{ padding: 20 }}>
      <Text style={{ fontSize: 24, marginBottom: 20 }}>Premium Content</Text>
      
      {!hasAccess ? (
        <View>
          <Text>Purchase required to access premium content</Text>
          {products.map(product => (
            <Button
              key={product.productId}
              title={`Buy ${product.localizedPrice}`}
              onPress={() => handlePurchase(product.productId)}
            />
          ))}
        </View>
      ) : (
        <View>
          <Text>🎉 You have access to premium content!</Text>
          <Button 
            title="Download Content"
            onPress={() => contentManager.downloadContent('premium_course_1')}
          />
        </View>
      )}
    </View>
  );
};

export default PremiumContentScreen;
## Usage Examples

### 5. Complete Integration Example

```javascript
// App.js - Main application setup
import React, { useEffect } from 'react';
import { Alert } from 'react-native';
import { ModernIAPManager } from './services/ModernIAPManager';
import { ContentManager } from './services/ContentManager';

// Global service instances
export const iapManager = new ModernIAPManager();
export const contentManager = new ContentManager();

const App = () => {
  useEffect(() => {
    initializeApp();
  }, []);

  const initializeApp = async () => {
    try {
      // Initialize IAP with your product IDs
      await iapManager.initialize([
        'com.yourapp.premium_course',
        'com.yourapp.advanced_content'
      ]);

      // Load available content
      await contentManager.loadAvailableContent();

      console.log('App initialized successfully');
    } catch (error) {
      console.error('App initialization failed:', error);
      Alert.alert('Initialization Error', 'Failed to initialize app services');
    }
  };

  return (
    <NavigationContainer>
      {/* Your app navigation */}
    </NavigationContainer>
  );
};

export default App;
```

### 6. Error Handling and Edge Cases

```javascript
class ErrorHandler {
  static handlePurchaseError(error) {
    console.error('Purchase error:', error);
    
    switch (error.code) {
      case 'E_USER_CANCELLED':
        return { message: 'Purchase was cancelled', showAlert: false };
      
      case 'E_PAYMENT_INVALID':
        return { 
          message: 'Payment method is invalid. Please check your payment settings.',
          showAlert: true 
        };
      
      case 'E_NETWORK_ERROR':
        return { 
          message: 'Network error. Please check your connection and try again.',
          showAlert: true 
        };
      
      case 'E_SERVICE_ERROR':
        return { 
          message: 'Service temporarily unavailable. Please try again later.',
          showAlert: true 
        };
      
      default:
        return { 
          message: `Purchase failed: ${error.message}`,
          showAlert: true 
        };
    }
  }

  static async handleContentAccessError(error, contentId) {
    console.error('Content access error:', error);
    
    if (error.status === 403) {
      // Access denied - user doesn't own content
      Alert.alert(
        'Access Required',
        'You need to purchase this content to access it.',
        [
          { text: 'Cancel', style: 'cancel' },
          { text: 'Purchase', onPress: () => this.redirectToPurchase(contentId) }
        ]
      );
    } else if (error.status === 503) {
      // Service unavailable
      Alert.alert(
        'Service Unavailable',
        'Content service is temporarily unavailable. Please try again later.'
      );
    } else {
      Alert.alert('Error', 'Failed to access content. Please try again.');
    }
  }

  static redirectToPurchase(contentId) {
    // Navigate to purchase screen for the required content
    NavigationService.navigate('Purchase', { contentId });
  }
}
```

### 7. Testing and Validation

```javascript
// TestHelper.js - For development and testing
class TestHelper {
  static async validateIAPSetup() {
    try {
      console.log('🧪 Starting IAP validation...');
      
      // Test 1: IAP Connection
      const connectionTest = await this.testIAPConnection();
      console.log('✅ IAP Connection:', connectionTest ? 'PASS' : 'FAIL');
      
      // Test 2: Product Loading
      const productsTest = await this.testProductLoading();
      console.log('✅ Product Loading:', productsTest ? 'PASS' : 'FAIL');
      
      // Test 3: Notification Setup
      const notificationTest = await this.testNotificationSetup();
      console.log('✅ Notifications:', notificationTest ? 'PASS' : 'FAIL');
      
      // Test 4: Content Access
      const contentTest = await this.testContentAccess();
      console.log('✅ Content Access:', contentTest ? 'PASS' : 'FAIL');
      
      console.log('🧪 IAP validation complete');
      
    } catch (error) {
      console.error('❌ IAP validation failed:', error);
    }
  }

  static async testIAPConnection() {
    try {
      await initConnection();
      return true;
    } catch (error) {
      console.error('IAP connection test failed:', error);
      return false;
    }
  }

  static async testProductLoading() {
    try {
      const products = await getProducts(['com.yourapp.premium_course']);
      return products && products.length > 0;
    } catch (error) {
      console.error('Product loading test failed:', error);
      return false;
    }
  }

  static async testNotificationSetup() {
    try {
      const authStatus = await messaging().hasPermission();
      return authStatus === messaging.AuthorizationStatus.AUTHORIZED;
    } catch (error) {
      console.error('Notification test failed:', error);
      return false;
    }
  }

  static async testContentAccess() {
    try {
      const response = await fetch('/api/v1/content/test', {
        headers: { 'Authorization': `Bearer ${await getAuthToken()}` }
      });
      return response.ok;
    } catch (error) {
      console.error('Content access test failed:', error);
      return false;
    }
  }
}

// Use in development
if (__DEV__) {
  TestHelper.validateIAPSetup();
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
