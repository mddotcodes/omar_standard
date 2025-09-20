# OMAR Access Control Specification

This document defines the access control system for the On-Chain Metaverse Asset Registry (OMAR) standard. Access control is stored on-chain as JSON strings in the `accessControl` field of each `AssetInfo` struct.

## Overview

The OMAR access control system allows creators to implement access control for different asset types within the same collection. Access control is defined per asset type and stored directly on-chain, enabling immediate verification without external dependencies.

**Note**: This is an early proposal for access control. The system is designed to be simple and extensible for future enhancements.

## Access Control Structure

Access control is defined using JSON objects stored as strings in the `accessControl` field. The JSON structure follows a standardized format that supports three core access types.

### Core Access Control Object

```json
{
  "type": "public" | "access_token" | "signature",
  "verification": {
    "method": "header" | "query" | "signature",
    "endpoint": "string",
    "description": "string"
  }
}
```

## Access Control Types

### 1. Public Access

Assets with public access are available to everyone without verification.

```json
{
  "type": "public"
}
```

### 2. Access Token

Assets with access token authentication require a valid token to access the content. This allows metaverses to have a single token per collection.

#### Header-based Access Token
```json
{
  "type": "access_token",
  "verification": {
    "method": "header",
    "description": "Pass access token in Authorization header"
  }
}
```

#### Query-based Access Token
```json
{
  "type": "access_token",
  "verification": {
    "method": "query",
    "description": "Pass access token as query parameter"
  }
}
```

**How it works:**
1. User requests the asset file directly
2. Access token is passed as specified in the method (header or query)
3. Server validates the token and serves the content
4. No separate verification endpoint needed


### 3. Signature Verification

Assets with signature verification require a cryptographic signature to access the content.

```json
{
  "type": "signature",
  "verification": {
    "method": "signature",
    "endpoint": "https://api.example.com/verify-signature",
    "description": "Verify signature to access 3D model"
  }
}
```

**How it works:**
1. User generates a signature for the access request
2. Platform sends the signature and request details to the verification endpoint
3. Endpoint validates the signature and returns the asset URL
4. User can access the content

## Verification Requirements

### Access Token Authentication

For access token authentication, the token should be passed as specified in the method:

- **Header Method**: `Authorization: Bearer <access_token>`
- **Query Method**: `?access_token=<access_token>`

### Signature Verification Endpoint

Signature verification requires a verification endpoint that accepts requests in the following format:

```json
{
  "signature": "0xabcd...",
  "message": "Access request for asset",
  "userAddress": "0x1234...",
  "assetType": "gltf",
  "timestamp": 1640995200
}
```

**Response Format:**
```json
{
  "success": true | false,
  "accessUrl": "string",           // URL to access the asset (if successful)
  "error": "string"                // Error message (if unsuccessful)
}
```

## Implementation Examples

### Public Asset

```json
{
  "type": "public"
}
```

### Access Token Asset (Header Method)

```json
{
  "type": "access_token",
  "verification": {
    "method": "header",
    "description": "Pass access token in Authorization header"
  }
}
```

### Access Token Asset (Query Method)

```json
{
  "type": "access_token",
  "verification": {
    "method": "query",
    "description": "Pass access token as query parameter"
  }
}
```

### Signature Verification Asset

```json
{
  "type": "signature",
  "verification": {
    "method": "signature",
    "endpoint": "https://api.example.com/verify-signature",
    "description": "Verify signature to access 3D model"
  }
}
```
