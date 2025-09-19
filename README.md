# OMAR - On-Chain Metaverse Asset Registry

A comprehensive standard for registering and managing metaverse assets on-chain.

## Repository Structure

- `contracts/` - Smart contracts implementing the OMAR standard
- `demo/` - Next.js demo application showcasing OMAR functionality
- `specifications/` - Documentation and technical specifications
- `docs/` - Additional documentation and guides

## Key Features

- **Modular Architecture**: Separates core registry from verification logic
- **Token-Gated Assets**: Standardized access control for protected 3D content
- **Cross-Platform Support**: Works across different metaverse platforms
- **Extensible Verification**: Support for multiple ownership verification methods
- **Web Standards**: JSON-LD and Schema.org compatible metadata

## Getting Started

### Prerequisites
- Node.js 18+
- Hardhat or Foundry for smart contract development
- Vercel CLI for deployment

### Development

1. Install dependencies:
   ```bash
   npm install
   ```

2. Set up smart contracts:
   ```bash
   cd contracts
   npm install
   ```

3. Run the demo application:
   ```bash
   cd demo
   npm install
   npm run dev
   ```

## License

MIT License - see LICENSE file for details
