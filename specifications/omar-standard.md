# On-Chain Metaverse Asset Registry (OMAR) Standard - Modular Architecture

|     |     |
| **Status** | Draft |
| **Type** | Standards Track |
| **Category** | Application |
| **Created** | 2025-09-19 |
| **Requires** | EIP-173 (owner()) |

## 1. Abstract

This standard defines a modular and extensible on-chain registry for providing direct access to 3D models and other assets associated with non-fungible token (NFT) collections. The registry is designed for universal compatibility, supporting both EVM and non-EVM chains by using a generic bytes type for contract identifiers.

The architecture separates the core ledger from ownership verification. A central Registry Contract stores asset retrieval information, while whitelisted Verifier Contracts handle initial ownership proofs. A key feature is the establishment of an EVM-based Delegate address upon registration. This allows for a single, robust verification of the true collection owner (who may be on a non-EVM chain), while subsequent asset information updates can be performed efficiently and affordably by the designated delegate on the EVM chain where the registry resides.

## 2. Motivation

The metaverse requires interoperability across a multi-chain landscape. This standard addresses two critical challenges:

**Cross-Chain Identification:** Most registries are EVM-centric. By using generic bytes for identifiers, this standard can support any blockchain's address format (e.g., Solana's 32-byte keys, TON's composite addresses), making it truly universal.

**Update Efficiency:** Requiring cross-chain verification for every metadata update is impractical. This standard introduces a delegate pattern: ownership is proven once during a secure registration process, which appoints an EVM delegate address. This delegate is then empowered to manage the collection's metadata with simple, low-cost transactions, removing the need for repeated, complex verifications.

## 3. Specification

### 3.1. Definitions

- **Core Registry:** The central smart contract storing asset retrieval information for NFT collections.
- **Verifier Contract:** A whitelisted contract that handles a specific ownership verification method.
- **Registry Owner:** A designated address (e.g., multi-sig) that manages the Verifier whitelist.
- **Collection:** An NFT smart contract, uniquely identified by its chainId and contractIdentifier.
- **Collection Owner:** The verified owner of the collection, identified by a bytes string. This entity performs the initial registration and sets the initial delegate.
- **Delegate:** An EVM address authorized to manage a collection's asset information after the initial registration.
- **Asset Type:** A standardized string identifier for different asset formats (e.g., "gltf", "vrm", "mml", "ubf").
- **Base URI:** A URI that, when combined with a tokenId, resolves to a specific asset file for a given asset type.
- **Asset Access Control:** A JSON string defining access control rules for a specific asset type within a collection.

### 3.2. Core Registry & Verifier Interfaces

**Core Registry Interface (IOMARegistry.sol)**

The Core Registry interface defines all functions available for collection and asset management:

```solidity
interface IOMARegistry {
    // Collection Registration
    function setCollectionRecord(
        uint256 chainId,
        bytes calldata contractIdentifier,
        bytes calldata ownerIdentifier,
        address delegate,
        string[] calldata assetTypes,
        string[] calldata baseURIs,
        string[] calldata accessControls
    ) external;
    
    // Asset Management
    function updateAsset(
        uint256 chainId,
        bytes calldata contractIdentifier,
        string calldata assetType,
        string calldata baseURI,
        string calldata accessControl
    ) external;
    
    function transferDelegate(
        uint256 chainId,
        bytes calldata contractIdentifier,
        address newDelegate
    ) external;
    
    // Asset Queries
    function getTokenAsset(
        uint256 chainId,
        bytes calldata contractIdentifier,
        uint256 tokenId,
        string calldata assetType
    ) external view returns (string memory baseURI, string memory accessControl);
    
    function getCollectionAssets(
        uint256 chainId,
        bytes calldata contractIdentifier
    ) external view returns (string[] memory assetTypes, string[] memory baseURIs, string[] memory accessControls);
    
    function getAssetType(
        uint256 chainId,
        bytes calldata contractIdentifier,
        string calldata assetType
    ) external view returns (string memory baseURI, string memory accessControl);
    
    // Asset Type Management
    function allowAssetType(string calldata assetType) external;
    function disallowAssetType(string calldata assetType) external;
    function isAssetTypeAllowed(string calldata assetType) external view returns (bool);
    function getAllowedAssetTypeCount() external view returns (uint256);
    function getAllowedAssetTypeByIndex(uint256 index) external view returns (string memory);
    function getAllAllowedAssetTypes() external view returns (string[] memory);
    
    // Collection Enumeration
    function getCollectionCount() external view returns (uint256);
    function getCollectionByIndex(uint256 index) external view returns (uint256 chainId, bytes memory contractIdentifier);
    function collectionExists(uint256 chainId, bytes calldata contractIdentifier) external view returns (bool);
    function getCollectionDelegate(uint256 chainId, bytes calldata contractIdentifier) external view returns (address);
    function getCollectionOwner(uint256 chainId, bytes calldata contractIdentifier) external view returns (bytes memory);
    
    // Asset Type Queries
    function assetTypeExists(uint256 chainId, bytes calldata contractIdentifier, string calldata assetType) external view returns (bool);
    function getAssetTypeCount(uint256 chainId, bytes calldata contractIdentifier) external view returns (uint256);
    
    // Verifier Management
    function addVerifier(address verifier) external;
    function removeVerifier(address verifier) external;
    function isWhitelistedVerifier(address verifier) external view returns (bool);
    function getVerifierCount() external view returns (uint256);
    function getVerifierByIndex(uint256 index) external view returns (address);
    function getAllVerifiers() external view returns (address[] memory);
    
    // Registry Management
    function owner() external view returns (address);
    function transferOwnership(address newOwner) external;
}
```

**Verifier Contract Interface (IOMAVerifier.sol)**
```solidity
interface IOMAVerifier {
    function registerCollection(
        uint256 chainId,
        bytes calldata contractIdentifier,
        bytes calldata ownerIdentifier,
        address delegate,
        string[] calldata assetTypes,
        string[] calldata baseURIs,
        string[] calldata accessControls,
        bytes calldata proof
    ) external;
    
    function getVerificationMethod() external pure returns (string memory);
    function getVersion() external pure returns (string memory);
}
```

### 3.3. Events

The OMAR standard defines several events that are emitted by the Core Registry and Verifier contracts:

**Collection Events:**
- `CollectionRegistered(uint256 indexed chainId, bytes contractIdentifier, bytes ownerIdentifier, address delegate, address indexed verifier)` - Emitted when a collection is registered
- `AssetUpdated(uint256 indexed chainId, bytes contractIdentifier, string assetType, string baseURI, address indexed delegate)` - Emitted when an asset type is updated
- `DelegateTransferred(uint256 indexed chainId, bytes contractIdentifier, address indexed oldDelegate, address indexed newDelegate)` - Emitted when delegation is transferred

**Verifier Management Events:**
- `VerifierAdded(address indexed verifier)` - Emitted when a verifier is added to the whitelist
- `VerifierRemoved(address indexed verifier)` - Emitted when a verifier is removed from the whitelist

**Asset Type Management Events:**
- `AssetTypeAllowed(string indexed assetType)` - Emitted when an asset type is allowed
- `AssetTypeDisallowed(string indexed assetType)` - Emitted when an asset type is disallowed

**Registry Management Events:**
- `OwnershipTransferred(address indexed previousOwner, address indexed newOwner)` - Emitted when registry ownership is transferred

**Verifier Events:**
- `VerificationSuccess(uint256 indexed chainId, bytes contractIdentifier, bytes ownerIdentifier, address delegate)` - Emitted when verification succeeds
- `VerificationFailed(uint256 indexed chainId, bytes contractIdentifier, string reason)` - Emitted when verification fails

The Verifier's role is to perform the initial, robust ownership check and then call setCollectionRecord with the verified owner and the creator-specified delegate.

**Verifier Implementation Examples:** Sample verifiers are available at [https://github.com/mddotcodes/omar_standard/blob/master/specifications/verifier-examples.md](./verifier-examples.md)

### 3.3. On-Chain Asset Storage

The registry stores asset retrieval information directly on-chain, focusing purely on providing access to 3D models and other assets. This approach provides several benefits:

- **Immediate availability:** No dependency on external storage systems
- **Consistency:** Data cannot be modified without proper authorization
- **Efficiency:** Direct queries without additional HTTP requests
- **Transparency:** All asset information is publicly verifiable on-chain

#### 3.3.1. Collection Data Structure

Each collection record stores only the essential information for asset retrieval:

| Property | Type | Description |
| --- | --- | --- |
| ownerIdentifier | Bytes | Verified owner identifier (can be non-EVM) |
| delegate | Address | EVM address for managing asset updates |
| assets | Mapping | Asset type → AssetInfo mapping |

#### 3.3.2. Asset Information Structure

Each asset type within a collection is defined by an `AssetInfo` struct:

| Property | Type | Description |
| --- | --- | --- |
| baseURI | String | Base URI for this asset type (e.g., "ipfs://<folder_cid>/") |
| accessControl | String | JSON string defining access control for this asset type |

#### 3.3.3. Supported Asset Types

The registry supports standardized asset type identifiers:

- **gltf**: glTF/GLB 3D models
- **vrm**: VRM avatar models  
- **mml**: Metaverse Markup Language scripts/objects
- **ubf**: Universal Binary Format files
- **metadata**: Token-specific metadata JSON files

Additional asset types can be added as new standards emerge.

#### 3.3.4. Query Patterns

The registry supports three main query patterns to retrieve asset information:

1. **Token-Specific Asset Query**: Get the direct URL for a specific token and asset type
   - Query: `getTokenAsset(chainId, contractIdentifier, tokenId, assetType)`
   - Returns: `baseURI` and `accessControl` for the asset type
   - Usage: Construct final URL as `baseURI + tokenId`

2. **Collection Assets Query**: Get all asset types and their base URIs for a collection
   - Query: `getCollectionAssets(chainId, contractIdentifier)`
   - Returns: Arrays of `assetTypes`, `baseURIs`, and `accessControls`

3. **Asset Type Query**: Get base URI for a specific asset type
   - Query: `getAssetType(chainId, contractIdentifier, assetType)`
   - Returns: `baseURI` and `accessControl` for the specified asset type


#### 3.3.5. Asset Type Management

The registry implements a comprehensive asset type management system that allows the registry owner to control which asset types can be registered by collections.

**Asset Type Management Features:**
- **Allowed Asset Types**: Registry owner can add/remove allowed asset types
- **Asset Type Validation**: Only whitelisted asset types can be registered
- **Asset Type Enumeration**: Query all allowed asset types and their count
- **Asset Type Events**: Events emitted when asset types are allowed/disallowed

**Asset Type Management Functions:**
- `allowAssetType(string calldata assetType)` - Add new allowed asset type
- `disallowAssetType(string calldata assetType)` - Remove allowed asset type
- `isAssetTypeAllowed(string calldata assetType)` - Check if asset type is allowed
- `getAllowedAssetTypeCount()` - Get total number of allowed asset types
- `getAllowedAssetTypeByIndex(uint256 index)` - Get asset type by index
- `getAllAllowedAssetTypes()` - Get all allowed asset types

#### 3.3.6. Collection Enumeration

The registry provides comprehensive collection enumeration capabilities for querying all registered collections.

**Collection Enumeration Features:**
- **Collection Counting**: Track total number of registered collections
- **Collection Indexing**: Access collections by index for enumeration
- **Collection Information**: Retrieve chain ID and contract identifier for each collection
- **Collection Existence**: Check if a collection exists in the registry

**Collection Enumeration Functions:**
- `getCollectionCount()` - Get total number of registered collections
- `getCollectionByIndex(uint256 index)` - Get collection by index
- `collectionExists(uint256 chainId, bytes calldata contractIdentifier)` - Check if collection exists
- `getCollectionDelegate(uint256 chainId, bytes calldata contractIdentifier)` - Get collection delegate
- `getCollectionOwner(uint256 chainId, bytes calldata contractIdentifier)` - Get collection owner

#### 3.3.7. Verifier Management

The registry implements a whitelist system for verifier contracts that can register collections.

**Verifier Management Features:**
- **Verifier Whitelist**: Only whitelisted verifiers can register collections
- **Verifier Enumeration**: Query all whitelisted verifiers
- **Verifier Validation**: Check if an address is a whitelisted verifier
- **Verifier Events**: Events emitted when verifiers are added/removed

**Verifier Management Functions:**
- `addVerifier(address verifier)` - Add verifier to whitelist
- `removeVerifier(address verifier)` - Remove verifier from whitelist
- `isWhitelistedVerifier(address verifier)` - Check if verifier is whitelisted
- `getVerifierCount()` - Get total number of whitelisted verifiers
- `getVerifierByIndex(uint256 index)` - Get verifier by index
- `getAllVerifiers()` - Get all whitelisted verifiers

#### 3.3.8. Asset Type Query Functions

The registry provides additional query functions for asset type information within collections.

**Asset Type Query Functions:**
- `assetTypeExists(uint256 chainId, bytes calldata contractIdentifier, string calldata assetType)` - Check if asset type exists for collection
- `getAssetTypeCount(uint256 chainId, bytes calldata contractIdentifier)` - Get number of asset types for collection

#### 3.3.9. Access Control

Access control is defined per asset type using JSON strings stored in the `accessControl` field of each `AssetInfo` struct. This allows for granular control over different asset types within the same collection. The access control JSON should follow the same specification as defined in previous versions, allowing for token-gating and other access control mechanisms.

**Access control implementation examples:** Sample specification available at [https://github.com/mddotcodes/omar_standard/blob/master/specifications/access-control.md](https://github.com/mddotcodes/omar_standard/blob/master/specifications/access-control.md)

### 3.4. Asset File Storage

While collection metadata and asset pointers are stored on-chain, the actual asset files should be stored using decentralized storage strategies:

- **Asset Files (GLB, glTF, VRM, etc.):** For public assets, storage on a content-addressed network like **IPFS** is recommended. For token-gated assets, files may be hosted on private servers accessible via the accessControl verification endpoint.
- **Base URIs:** Should point to content-addressed storage (like IPFS) to ensure immutability and prevent "link rot."
- **Token Metadata:** The aim of this standard is to centralize asset management for collections. Token metadata is not replicated in the registry and should be accessed directly via the collection contract via standard methods.

## 4. Rationale

- **Universal Compatibility:** Using bytes for contractIdentifier and ownerIdentifier makes the registry blockchain-agnostic, capable of storing identifiers from any EVM or non-EVM chain.
- **Delegate Pattern for Efficiency:** The separation of a permanent ownerIdentifier from a manageable delegate is a critical innovation. It allows for a single, high-security, cross-chain verification at setup. All subsequent updates are handled by the delegate via standard, low-cost EVM transactions, making the system practical and affordable for creators on any chain.
- **On-Chain Storage Benefits:** Storing asset retrieval information directly on-chain eliminates dependencies on external storage systems, ensures data consistency, and provides immediate availability without additional HTTP requests.
- **Flexible Query Patterns:** The three query patterns (token-specific, collection-wide, and asset-type-specific) provide efficient access to asset information for different use cases and platforms.
- **Granular Access Control:** Per-asset-type access control allows creators to implement different access policies for different asset formats within the same collection.
- **Scalability:** The on-chain mapping structure efficiently handles collections with multiple asset types while maintaining the proven baseURI + tokenId pattern for individual assets.
- **Modularity and Governance:** The Verifier pattern provides security and extensibility, allowing the governing body to safely add new verification methods for future chains or identity systems without altering the core registry.
- **Asset Type Management:** Centralized control over allowed asset types ensures standardization and prevents registration of unsupported formats, maintaining registry integrity and compatibility.
- **Collection Enumeration:** Comprehensive enumeration capabilities enable discovery and indexing of all registered collections, supporting ecosystem-wide queries and analytics.
- **Verifier Management:** Whitelist-based verifier system ensures only trusted verification methods can register collections, maintaining security while allowing extensibility.

## 5. Security Considerations

- **Delegate Key Security:** The security of a collection's metadata is critically dependent on the security of the delegate's private key. If this key is compromised, an attacker can maliciously alter the collection's metadata pointers. Standard EVM wallet security practices are paramount for delegates.

- **Initial Verification:** The integrity of the entire system relies on the Verifier contracts performing accurate and secure ownership checks during the initial registration and delegate appointment.

- **Registry Owner Security:** The security of the system's governance depends on the operational security of the Registry Owner's multi-signature wallet.