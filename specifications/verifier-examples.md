# OMAR Verifier Examples

This document provides examples of verifier contract implementations for the OMAR standard.

## Overview

Verifier contracts are responsible for handling ownership verification logic and calling the Core Registry to record collection data. Each verifier implements a specific verification method.

**Important**: All verifiers must be whitelisted by the registry owner before they can register collections. This ensures only trusted verification methods are used.

This document showcases various verifier implementation possibilities. These are examples of what can be implemented - not all verifiers shown here are currently deployed.

### Verifier Examples

1. **OwnerOf Verifier**: Verification using EIP-173 owner() function (same-chain only)
2. **Direct Registry**: Direct registration from contract deployment  (same-chain only)
3. **Manual Verifier**: Registry owner manual registration for special cases
4. **Signature Verifier**: EVM signature-based verification  (EVM only)
5. **Cross-chain Verifier**: Oracle-based cross-chain verification
6. **Mock Verifier**: Testing-only verifier
7. **Custom Verifier**: Any verifier using the interface

## OwnerOf Verifier

The OwnerOf Verifier uses the EIP-173 `owner()` function to verify ownership on the same chain as the registry. This is the simplest verification method.

### Features

- Uses EIP-173 standard for ownership verification
- Only works on the same chain as the registry
- Caller must be the actual owner of the contract
- Prevents duplicate registrations
- Provides both full interface and simplified registration methods

### Usage

1. Collection owner calls `registerCollection()` or `registerCollectionSimple()`
2. Verifier checks if caller is the owner using EIP-173 `owner()` function
3. If valid, collection is registered in the Core Registry
4. Duplicate registrations are prevented

### Example Implementation

```solidity
contract OwnerOfVerifier is IOMAVerifier {
    IOMARegistry public immutable registry;
    mapping(address => bool) private registeredCollections;

    constructor(address _registry) {
        registry = IOMARegistry(_registry);
    }

    function registerCollection(
        uint256 chainId,
        bytes calldata contractIdentifier,
        bytes calldata /* ownerIdentifier - unused */,
        address delegate,
        string[] calldata assetTypes,
        string[] calldata baseURIs,
        string[] calldata accessControls,
        bytes calldata /* proof - unused for ownerOf verifier */
    ) external override {
        require(chainId == block.chainid, "ChainId must match current chain");
        require(contractIdentifier.length == 20, "Must be valid address");
        
        address contractAddress = address(bytes20(contractIdentifier));
        require(!registeredCollections[contractAddress], "Already registered");
        
        // Verify ownership using EIP-173
        require(_isOwner(contractAddress, msg.sender), "Not contract owner");

        // Register with the registry
        registry.setCollectionRecord(
            chainId,
            contractIdentifier,
            abi.encodePacked(msg.sender),
            delegate,
            assetTypes,
            baseURIs,
            accessControls
        );

        registeredCollections[contractAddress] = true;
    }

    function _isOwner(address contractAddress, address potentialOwner) 
        internal view returns (bool) {
        (bool success, bytes memory data) = contractAddress.staticcall(
            abi.encodeWithSignature("owner()")
        );
        
        if (!success || data.length == 0) return false;
        
        address owner = abi.decode(data, (address));
        return owner != address(0) && owner == potentialOwner;
    }
}
```

## Direct Registry Verifier

The Direct Registry Verifier allows NFT contracts to register themselves through a verifier that validates the caller is the contract itself. This is the most efficient method for new contracts on the same chain.

### Features

- Self-registration through a verifier contract
- Verifier validates caller is the contract being registered
- Only works on the same chain as the registry
- Most efficient for new contracts
- Uses `msg.sender` to verify the calling contract

### Usage

1. NFT contract is deployed with verifier address
2. During constructor or initialization, contract calls the DirectRegistryVerifier
3. Verifier validates the caller is the contract itself
4. Verifier calls the registry to register the collection
5. Collection is immediately registered

### Example Implementation

```solidity
contract DirectRegistryVerifier is IOMAVerifier {
    IOMARegistry public immutable registry;
    mapping(address => bool) private registeredContracts;

    constructor(address _registry) {
        registry = IOMARegistry(_registry);
    }

    function registerCollection(
        uint256 chainId,
        bytes calldata contractIdentifier,
        bytes calldata ownerIdentifier,
        address delegate,
        string[] calldata assetTypes,
        string[] calldata baseURIs,
        string[] calldata accessControls,
        bytes calldata /* proof - unused for direct registry verifier */
    ) external override {
        require(chainId == block.chainid, "ChainId must match current chain");
        require(contractIdentifier.length == 20, "Must be valid address");
        
        address contractAddress = address(bytes20(contractIdentifier));
        require(contractAddress == msg.sender, "Only contract can register itself");
        require(!registeredContracts[contractAddress], "Contract already registered");

        // Register with the registry
        registry.setCollectionRecord(
            chainId,
            contractIdentifier,
            ownerIdentifier,
            delegate,
            assetTypes,
            baseURIs,
            accessControls
        );

        registeredContracts[contractAddress] = true;
    }
}

// Example NFT contract using DirectRegistryVerifier
contract MyNFT is ERC721, Ownable {
    DirectRegistryVerifier public immutable verifier;
    bool private registered;

    constructor(
        string memory name,
        string memory symbol,
        address _verifier,
        address delegate,
        string[] memory assetTypes,
        string[] memory baseURIs,
        string[] memory accessControls
    ) ERC721(name, symbol) {
        verifier = DirectRegistryVerifier(_verifier);
        
        // Register with OMAR during deployment
        _registerWithOMAR(delegate, assetTypes, baseURIs, accessControls);
    }

    function _registerWithOMAR(
        address delegate,
        string[] memory assetTypes,
        string[] memory baseURIs,
        string[] memory accessControls
    ) internal {
        require(!registered, "Already registered");
        require(assetTypes.length == baseURIs.length, "Array length mismatch");
        require(assetTypes.length == accessControls.length, "Array length mismatch");

        // Call the verifier to register this contract
        verifier.registerCollection(
            block.chainid,
            abi.encodePacked(address(this)),
            abi.encodePacked(owner()),
            delegate,
            assetTypes,
            baseURIs,
            accessControls,
            "" // No proof needed for direct registry
        );

        registered = true;
    }
}
```

## Manual Verifier

The Manual Verifier allows the registry owner to manually register collections without any verification. This is useful for administrative purposes and testing.

### Features

- Only registry owner can register collections
- No verification required - purely administrative
- Prevents duplicate manual registrations
- Useful for testing and emergency registrations

### Usage

1. Registry owner calls `registerCollection()` directly
2. All parameters are validated but no ownership verification is performed
3. Collection is immediately registered in the Core Registry
4. Duplicate registrations are prevented

### Example Implementation

```solidity
contract ManualVerifier is IOMAVerifier {
    IOMARegistry public immutable registry;
    mapping(uint256 => mapping(bytes => bool)) private manuallyRegistered;

    constructor(address _registry) {
        registry = IOMARegistry(_registry);
    }

    function registerCollection(
        uint256 chainId,
        bytes calldata contractIdentifier,
        bytes calldata ownerIdentifier,
        address delegate,
        string[] calldata assetTypes,
        string[] calldata baseURIs,
        string[] calldata accessControls,
        bytes calldata /* proof - unused for manual verifier */
    ) external override {
        // Only registry owner can manually register
        require(msg.sender == registry.owner(), "Only registry owner can manually register");
        
        // Prevent duplicate registrations
        require(!manuallyRegistered[chainId][contractIdentifier], 
                "Collection already registered manually");

        // Register with the registry
        registry.setCollectionRecord(
            chainId,
            contractIdentifier,
            ownerIdentifier,
            delegate,
            assetTypes,
            baseURIs,
            accessControls
        );

        // Mark as manually registered
        manuallyRegistered[chainId][contractIdentifier] = true;
    }
}
```

## Signature Verifier

The Signature Verifier uses cryptographic signatures to prove ownership of an NFT collection. This works for EVM-compatible chains.

### Features

- Recovers signer address from signature
- Verifies signer is the owner of the target NFT contract
- Calls Core Registry upon successful verification
- Works across different EVM chains

### Usage

1. Collection owner signs a message containing:
   - Chain ID
   - Contract Address
   - Metadata URI
   - Verifier identifier

2. The signature is submitted to the verifier contract
3. Verifier recovers the signer and verifies ownership
4. If valid, calls Core Registry to record the collection

### Example Implementation

```solidity
contract SignatureVerifier is IOMAVerifier, Ownable, ReentrancyGuard {
    function registerCollection(
        uint256 chainId,
        bytes calldata contractIdentifier,
        bytes calldata ownerIdentifier,
        address delegate,
        string[] calldata assetTypes,
        string[] calldata baseURIs,
        string[] calldata accessControls,
        bytes calldata proof
    ) external override nonReentrant {
        // Recover signer from signature
        address signer = _recoverSigner(chainId, contractIdentifier, ownerIdentifier, proof);
        
        // Verify ownership
        require(_verifyOwnership(contractIdentifier, signer), "Not contract owner");
        
        // Call registry
        registry.setCollectionRecord(
            chainId,
            contractIdentifier,
            ownerIdentifier,
            delegate,
            assetTypes,
            baseURIs,
            accessControls
        );
    }
}
```

## Cross-chain Verifier

The Cross-chain Verifier uses oracles to verify ownership across different networks.

### Features

- Initiates cross-chain verification requests
- Handles oracle callbacks
- Supports multiple oracle networks
- Enables cross-chain collection registration

### Usage

1. User submits registration request
2. Verifier initiates cross-chain oracle call
3. Oracle verifies ownership on source chain
4. Oracle returns result via callback
5. Verifier calls Core Registry if verification succeeds

### Example Implementation

```solidity
contract CrossChainVerifier is IOMAVerifier, Ownable, ReentrancyGuard {
    function registerCollection(
        uint256 chainId,
        bytes calldata contractIdentifier,
        bytes calldata ownerIdentifier,
        address delegate,
        string[] calldata assetTypes,
        string[] calldata baseURIs,
        string[] calldata accessControls,
        bytes calldata proof
    ) external override nonReentrant {
        // Generate request ID
        bytes32 requestId = keccak256(abi.encodePacked(chainId, contractIdentifier, ownerIdentifier));
        
        // Store request
        verificationRequests[requestId] = VerificationRequest(
            chainId,
            contractIdentifier,
            ownerIdentifier,
            delegate,
            assetTypes,
            baseURIs,
            accessControls
        );
        
        // Initiate oracle call
        oracle.requestVerification(requestId, chainId, contractIdentifier);
    }
    
    function completeVerification(
        bytes32 requestId,
        address verifiedOwner,
        bool success
    ) external onlyAuthorizedOracle {
        VerificationRequest memory request = verificationRequests[requestId];
        require(request.chainId != 0, "Request not found");
        
        if (success) {
            registry.setCollectionRecord(
                request.chainId,
                request.contractIdentifier,
                request.ownerIdentifier,
                request.delegate,
                request.assetTypes,
                request.baseURIs,
                request.accessControls
            );
        }
        
        delete verificationRequests[requestId];
    }
}
```



## Mock Verifier

The Mock Verifier is designed for testing purposes only and always succeeds without any verification.

### Features

- **Testing Only**: Should never be used in production
- Always succeeds - no verification performed
- Validates input parameters
- Useful for testing registry functionality

### Usage

1. Call `registerCollection()` with any valid parameters
2. Mock verifier performs basic validation
3. Collection is immediately registered
4. No ownership verification is performed

### Example Implementation

```solidity
contract MockVerifier is IOMAVerifier {
    IOMARegistry public immutable registry;

    constructor(address _registry) {
        registry = IOMARegistry(_registry);
    }

    function registerCollection(
        uint256 chainId,
        bytes calldata contractIdentifier,
        bytes calldata ownerIdentifier,
        address delegate,
        string[] calldata assetTypes,
        string[] calldata baseURIs,
        string[] calldata accessControls,
        bytes calldata /* proof - unused for mock verifier */
    ) external override {
        // Basic validation only
        require(chainId != 0, "ChainId cannot be zero");
        require(contractIdentifier.length > 0, "Contract identifier required");
        require(ownerIdentifier.length > 0, "Owner identifier required");
        require(delegate != address(0), "Delegate required");
        require(assetTypes.length == baseURIs.length, "Array length mismatch");

        // Mock verification - always succeeds
        // In production, you would verify ownership here

        // Register with the registry
        registry.setCollectionRecord(
            chainId,
            contractIdentifier,
            ownerIdentifier,
            delegate,
            assetTypes,
            baseURIs,
            accessControls
        );
    }
}
```

**⚠️ Warning**: The Mock Verifier should only be used for testing purposes and must never be whitelisted in production environments.

## Custom Verifier

You can create custom verifiers for specific use cases.

### Requirements

1. Implement `IOMAVerifier` interface
2. Handle ownership verification logic
3. Call `registry.setCollectionRecord()` upon success
4. Be whitelisted by Registry Owner

### Example Structure

```solidity
contract CustomVerifier is IOMAVerifier, Ownable, ReentrancyGuard {
    IOMARegistry public immutable registry;
    
    constructor(address _registry) {
        registry = IOMARegistry(_registry);
    }
    
    function registerCollection(
        uint256 chainId,
        address contractAddress,
        string calldata metadataURI,
        bytes calldata proof
    ) external override nonReentrant {
        // Custom verification logic here
        address verifiedOwner = _customVerification(chainId, contractAddress, proof);
        
        // Call registry
        registry.setCollectionRecord(chainId, contractAddress, verifiedOwner, metadataURI);
    }
    
    function _customVerification(
        uint256 chainId,
        address contractAddress,
        bytes calldata proof
    ) internal returns (address) {
        // Implement custom verification logic
    }
}
```

## Security Considerations

### Verifier Security

- All verifiers should be audited before whitelisting
- Implement proper access controls
- Use reentrancy guards
- Validate all inputs

### Registry Owner Security

- Use multi-signature wallets
- Implement proper key management
- Regular security reviews
- Emergency procedures for compromised keys

### Oracle Security

- Use reputable oracle networks
- Implement multiple oracle fallbacks
- Validate oracle responses
- Monitor for oracle failures