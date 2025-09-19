# On-Chain Metaverse Asset Registry (OMAR) Standard - Modular Architecture

|     |     |
| --- | --- |
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

```solidity
interface IOMARegistry {
    struct AssetInfo {
        string baseURI;        // Base URI for this asset type
        string accessControl;  // JSON string defining access control for this asset type
    }
    
    struct CollectionRecord {
        bytes ownerIdentifier; // Verified owner (can be non-EVM)
        address delegate;      // EVM address for managing updates
        mapping(string => AssetInfo) assets; // Asset type to AssetInfo mapping
        address verifier;
        uint64 registrationTimestamp;
        bool isEnabled;
    }

    event CollectionRegistered(uint256 indexed chainId, bytes contractIdentifier, bytes ownerIdentifier, address delegate);
    event AssetUpdated(uint256 indexed chainId, bytes contractIdentifier, string assetType, string baseURI, address indexed delegate);
    event DelegateTransferred(uint256 indexed chainId, bytes contractIdentifier, address indexed oldDelegate, address indexed newDelegate);

    /**
     * @notice Called by a Verifier to register a collection and set the initial delegate.
     * @param ownerIdentifier The verified owner identifier (e.g., a non-EVM address).
     * @param delegate The EVM address that will manage this record.
     * @param assetTypes Array of asset types to register
     * @param baseURIs Array of base URIs for each asset type
     * @param accessControls Array of access control JSON strings for each asset type
     */
    function setCollectionRecord(
        uint256 chainId,
        bytes calldata contractIdentifier,
        bytes calldata ownerIdentifier,
        address delegate,
        string[] calldata assetTypes,
        string[] calldata baseURIs,
        string[] calldata accessControls
    ) external;

    /**
     * @notice Called by the current delegate to update an asset type's base URI and access control.
     */
    function updateAsset(
        uint256 chainId,
        bytes calldata contractIdentifier,
        string calldata assetType,
        string calldata baseURI,
        string calldata accessControl
    ) external;

    /**
     * @notice Called by the current delegate to transfer delegation to a new address.
     */
    function transferDelegate(
        uint256 chainId,
        bytes calldata contractIdentifier,
        address newDelegate
    ) external;

    /**
     * @notice Get the direct URL for a specific token and asset type.
     * @param chainId The chain ID of the NFT collection
     * @param contractIdentifier The contract identifier
     * @param tokenId The token ID
     * @param assetType The asset type (e.g., "gltf", "vrm", "mml")
     * @return baseURI The base URI for the asset type
     * @return accessControl The access control JSON for the asset type
     */
    function getTokenAsset(
        uint256 chainId,
        bytes calldata contractIdentifier,
        uint256 tokenId,
        string calldata assetType
    ) external view returns (string memory baseURI, string memory accessControl);

    /**
     * @notice Get all base URIs for a collection.
     * @param chainId The chain ID of the NFT collection
     * @param contractIdentifier The contract identifier
     * @return assetTypes Array of asset types
     * @return baseURIs Array of base URIs
     * @return accessControls Array of access control JSON strings
     */
    function getCollectionAssets(
        uint256 chainId,
        bytes calldata contractIdentifier
    ) external view returns (string[] memory assetTypes, string[] memory baseURIs, string[] memory accessControls);

    /**
     * @notice Get base URI for a specific asset type.
     * @param chainId The chain ID of the NFT collection
     * @param contractIdentifier The contract identifier
     * @param assetType The asset type
     * @return baseURI The base URI for the asset type
     * @return accessControl The access control JSON for the asset type
     */
    function getAssetType(
        uint256 chainId,
        bytes calldata contractIdentifier,
        string calldata assetType
    ) external view returns (string memory baseURI, string memory accessControl);

    // Governance and view functions remain, adapted for `bytes` identifier.
}
```

**Verifier Contract Interface (IOMAVerifier.sol)**

The Verifier's role is to perform the initial, robust ownership check and then call setCollectionRecord with the verified owner and the creator-specified delegate.

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
| verifier | Address | The verifier contract that processed registration |
| registrationTimestamp | Uint64 | When the collection was registered |
| isEnabled | Bool | Whether the collection is active |

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

#### 3.3.5. Token-Level Metadata

Token-specific metadata files should follow standard ERC-721 metadata format and be stored at the URI resolved from the `metadata` asset type's baseURI + tokenId. These files contain individual token attributes and traits.

#### 3.3.6. Access Control

Access control is defined per asset type using JSON strings stored in the `accessControl` field of each `AssetInfo` struct. This allows for granular control over different asset types within the same collection. The access control JSON should follow the same specification as defined in previous versions, allowing for token-gating and other access control mechanisms.

### 3.4. Asset File Storage

While collection metadata and asset pointers are stored on-chain, the actual asset files should be stored using decentralized storage strategies:

- **Asset Files (GLB, glTF, VRM, etc.):** For public assets, storage on a content-addressed network like **IPFS** is recommended. For token-gated assets, files may be hosted on private servers accessible via the accessControl verification endpoint.
- **Token Metadata Files:** Individual token metadata JSON files should be stored using the same strategy as asset files for consistency and reliability.
- **Base URIs:** Should point to content-addressed storage (like IPFS) to ensure immutability and prevent "link rot."

## 4. Rationale

- **Universal Compatibility:** Using bytes for contractIdentifier and ownerIdentifier makes the registry blockchain-agnostic, capable of storing identifiers from any EVM or non-EVM chain.
- **Delegate Pattern for Efficiency:** The separation of a permanent ownerIdentifier from a manageable delegate is a critical innovation. It allows for a single, high-security, cross-chain verification at setup. All subsequent updates are handled by the delegate via standard, low-cost EVM transactions, making the system practical and affordable for creators on any chain.
- **On-Chain Storage Benefits:** Storing asset retrieval information directly on-chain eliminates dependencies on external storage systems, ensures data consistency, and provides immediate availability without additional HTTP requests.
- **Flexible Query Patterns:** The three query patterns (token-specific, collection-wide, and asset-type-specific) provide efficient access to asset information for different use cases and platforms.
- **Granular Access Control:** Per-asset-type access control allows creators to implement different access policies for different asset formats within the same collection.
- **Scalability:** The on-chain mapping structure efficiently handles collections with multiple asset types while maintaining the proven baseURI + tokenId pattern for individual assets.
- **Modularity and Governance:** The Verifier pattern provides security and extensibility, allowing the governing body to safely add new verification methods for future chains or identity systems without altering the core registry.

## 5. Security Considerations

- **Delegate Key Security:** The security of a collection's metadata is critically dependent on the security of the delegate's private key. If this key is compromised, an attacker can maliciously alter the collection's metadata pointers. Standard EVM wallet security practices are paramount for delegates.

- **Initial Verification:** The integrity of the entire system relies on the Verifier contracts performing accurate and secure ownership checks during the initial registration and delegate appointment.

- **Registry Owner Security:** The security of the system's governance depends on the operational security of the Registry Owner's multi-signature wallet.