# Copilot Instructions

<!-- Use this file to provide workspace-specific custom instructions to Copilot. For more details, visit https://code.visualstudio.com/docs/copilot/copilot-customization#_use-a-githubcopilotinstructionsmd-file -->

## Project Context
This workspace contains architecture documentation and design specifications for implementing Apple In-App Purchases (IAP) in iOS applications with CRM integration for premium course content management using Apple's modern Server API v2.

## Key Guidelines
- Always follow Apple's App Store Review Guidelines for in-app purchases
- Ensure compliance with Apple's requirement that premium content must be sold through IAP
- Use Apple Server Notifications v2 with JWS signature verification (NO verifyReceipt API)
- Maintain separation between purchase flow (Apple) and content delivery (CRM)
- Consider privacy and data protection requirements
- Include proper error handling for network failures and edge cases

## Architecture Principles
- Use Apple's StoreKit framework for all purchase operations
- Implement real-time Apple Server Notifications v2 processing
- Design idempotent APIs for content unlocking
- Ensure graceful fallback when services are unavailable
- Implement proper logging and analytics
