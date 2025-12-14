# CHAINLINK_INTEGRATION.md - Oracle Setup & Smart Contract Architecture

**Chainlink Cross-Chain Interoperability Protocol (CCIP) Integration**

---

## Overview

ArweaveStamp uses **Chainlink's Cross-Chain Interoperability Protocol (CCIP)** to deliver cryptographic attestations across multiple blockchain networks (Ethereum, Polygon, Avalanche) simultaneously.

Key advantages of Chainlink over alternatives:
- **$38B+ total value secured** across 1,300+ projects
- **Market leader in oracle infrastructure** with 7+ year track record
- **CCIP enables native cross-chain messaging**, not just single-chain relays
- **Institutional adoption** by Aave, Compound, and enterprise partners
- **Proven security** with multiple witness nodes and randomized node selection

---

## Part 1: Chainlink CCIP Architecture

### High-Level Flow

```
ArweaveStamp CLI
  ↓
[Arweave Upload] Document uploaded to immutable storage
  ↓ (txId, metadata hash)
[Claude Analysis] Semantic classification and entity extraction
  ↓ (analysis JSON, entities, summary)
[Hash Computation] SHA-256(content) + SHA-256(metadata)
  ↓ (documentId = combined hash)
[Chainlink CCIP Request] Send attestation to Ethereum (primary)
  ↓ Chainlink relayer picks up message
  ↑
[Automatic Cross-Chain Replication] CCIP relays to Polygon + Avalanche
  ↓ (all three chains in parallel)
Final State: 3-chain attestation with Merkle root anchors
```

### Smart Contract Architecture

```typescript
// Contract: AttestationOracle.sol
interface AttestationOracle {
  // Receive attestation from ArweaveStamp
  function attestDocument(
    string calldata arweaveUrl,      // https://arweave.net/{txId}
    bytes32 documentHash,             // SHA-256(content + metadata)
    string calldata documentType,     // e.g., "contract", "invoice"
    address payable callback          // ArweaveStamp oracle service
  ) external payable returns (bytes32 attestationId);

  // Verify attestation without re-downloading
  function verifyAttestation(
    bytes32 attestationId,
    bytes32 documentHash
  ) external view returns (bool verified, uint256 timestamp);
}

// Contract: CrossChainRelay.sol
interface CrossChainRelay {
  // Called by Chainlink CCIP when relaying from Ethereum to Polygon/Avax
  function ccipReceive(
    Client.Any2EVMMessage calldata message
  ) external;
  
  // Emit event when attestation relayed
  event AttestationRelayed(
    bytes32 indexed attestationId,
    string sourceChain,
    string destinationChain,
    uint256 blockNumber
  );
}
```

### Network Addresses (Mainnet + Testnet)

**Mainnet Chainlink CCIP Routers:**

| Network | Router Address | LINK Token | Supported? |
|---------||-|-|
| Ethereum | `0x80226fc0Ee2b05F1f63473b78bEd9d5C0092d58d` | Native | ✅ |
| Polygon | `0x70499c328e1e2a3c41ce5d8e3fef69911bc8248e` | `0xb0897686c545045afc77cf20ec7a532e3120e0f1` | ✅ |
| Avalanche | `0xF694E193200268f9666cD1DFbDea7D7B29a1ccd1` | `0x5947BB275c521040541C922d6414FA868B4c5f78` | ✅ |

**Testnet (Sepolia/Mumbai/Fuji):**

| Network | Router Address | Status |
|---------|-|-|
| Sepolia | `0xD0daae2231E9CB06b24bD713aAFf85034E286364a` | Testing |
| Mumbai | `0x9C32fCB86BF0924CCC8921d375742ad67C8E6CEb` | Testing |
| Fuji | `0xF694E193200268f9666cD1DFbDea7D7B29a1ccd1` | Testing |

**Chainlink LINK Token Addresses:**

```typescript
const LINK_TOKENS = {
  ethereum: '0x514910771AF9Ca656af840dff83E8264EcF986CA',  // Mainnet
  polygon: '0xb0897686c545045afc77cf20ec7a532e3120e0f1',   // Mainnet
  avalanche: '0x5947BB275c521040541C922d6414FA868B4c5f78',  // Mainnet
  sepolia: '0x779877A7B0D9C06BeEFEAA6CfAFFF0b847F84F76',    // Testnet
  mumbai: '0x326C977E6b7C9c93616C26B9407CAF3eA3d7A633',     // Testnet
  fuji: '0x0b9d5D9d882a6346875621166D7281e8333715C6'        // Testnet
};
```

---

## Part 2: Integration Points

### Module: `src/oracle/chainlink-attestor.ts`

Responsible for interfacing with Chainlink smart contracts.

```typescript
import { ethers } from 'ethers';
import { CCIP_ROUTER_ADDRESSES } from './chainlink-config';

class ChainlinkAttestor {
  private providers: Map<string, ethers.providers.Provider>;
  private routers: Map<string, ChainlinkRouter>;
  
  constructor(privateKey: string) {
    this.providers = new Map([
      ['ethereum', new ethers.providers.JsonRpcProvider(process.env.ETH_RPC_URL)],
      ['polygon', new ethers.providers.JsonRpcProvider(process.env.POLYGON_RPC_URL)],
      ['avalanche', new ethers.providers.JsonRpcProvider(process.env.AVAX_RPC_URL)]
    ]);
    
    this.routers = new Map([
      ['ethereum', this.initRouter('ethereum')],
      ['polygon', this.initRouter('polygon')],
      ['avalanche', this.initRouter('avalanche')]
    ]);
  }
  
  // Main attestation flow
  async attestDocument(
    arweaveUrl: string,
    documentHash: string,
    documentType: string,
    metadata: DocumentMetadata
  ): Promise<AttestationProof> {
    // 1. Submit attestation to Ethereum (primary chain)
    const ethereumTx = await this.submitToChain(
      'ethereum',
      arweaveUrl,
      documentHash,
      documentType
    );
    
    // 2. Chainlink CCIP automatically relays to Polygon + Avalanche
    // No need to explicitly call secondary chains
    
    // 3. Monitor cross-chain confirmations
    const confirmations = await this.monitorCrossChain(
      ethereumTx.attestationId
    );
    
    return {
      attestationId: ethereumTx.attestationId,
      chains: {
        ethereum: { txHash: ethereumTx.hash, confirmed: confirmations.ethereum },
        polygon: { txHash: ethereumTx.ccipMessageId, confirmed: confirmations.polygon },
        avalanche: { txHash: ethereumTx.ccipMessageId, confirmed: confirmations.avalanche }
      },
      timestamp: ethereumTx.blockTimestamp,
      gasUsed: ethereumTx.gasUsed
    };
  }
  
  private async submitToChain(
    chain: string,
    arweaveUrl: string,
    documentHash: string,
    documentType: string
  ): Promise<any> {
    const router = this.routers.get(chain)!;
    const provider = this.providers.get(chain)!;
    
    // Estimate gas
    const estimatedGas = await router.attestDocument.estimateGas(
      arweaveUrl,
      documentHash,
      documentType
    );
    
    // Submit transaction
    const tx = await router.attestDocument(
      arweaveUrl,
      documentHash,
      documentType,
      { gasLimit: estimatedGas.mul(110).div(100) } // 10% buffer
    );
    
    // Wait for confirmation (12 blocks for Ethereum)
    const receipt = await tx.wait(12);
    
    return {
      attestationId: this.extractAttestationId(receipt),
      hash: tx.hash,
      blockNumber: receipt.blockNumber,
      blockTimestamp: receipt.timestamp,
      gasUsed: receipt.gasUsed,
      ccipMessageId: this.extractCCIPMessageId(receipt)
    };
  }
  
  // Monitor CCIP relay status
  private async monitorCrossChain(
    attestationId: string
  ): Promise<Map<string, boolean>> {
    const confirmations = new Map<string, boolean>();
    
    // Check each chain for attestation
    for (const [chain, router] of this.routers) {
      try {
        const verified = await router.verifyAttestation(
          attestationId,
          ethers.constants.HashZero // We'll add specific hash later
        );
        confirmations.set(chain, verified);
      } catch (error) {
        confirmations.set(chain, false);
      }
    }
    
    return confirmations;
  }
}

export { ChainlinkAttestor };
```

### Module: `src/oracle/chainlink-verifier.ts`

Responsible for verifying attestations in real-time.

```typescript
class ChainlinkVerifier {
  private routers: Map<string, ChainlinkRouter>;
  
  // Verify on single chain
  async verifyOnChain(
    chain: string,
    attestationId: string,
    documentHash: string
  ): Promise<VerificationResult> {
    const router = this.routers.get(chain)!;
    
    try {
      const result = await router.verifyAttestation(attestationId, documentHash);
      return {
        verified: result.verified,
        timestamp: result.timestamp,
        chain: chain,
        blockNumber: await router.provider.getBlockNumber()
      };
    } catch (error) {
      throw new Error(`Verification failed on ${chain}: ${error.message}`);
    }
  }
  
  // Verify across all chains (consensus)
  async verifyMultichain(
    attestationId: string,
    documentHash: string,
    requiredConfirmations: number = 2
  ): Promise<MultiChainVerification> {
    const results: VerificationResult[] = [];
    
    for (const [chain] of this.routers) {
      const result = await this.verifyOnChain(chain, attestationId, documentHash);
      results.push(result);
    }
    
    const confirmedChains = results.filter(r => r.verified).length;
    
    return {
      verified: confirmedChains >= requiredConfirmations,
      confirmedChains: confirmedChains,
      totalChains: results.length,
      results: results
    };
  }
}

export { ChainlinkVerifier };
```

---

## Part 3: Cost Optimization

### Gas Cost Breakdown

**Ethereum (Primary Attestation)**:
- `attestDocument()` call: ~150,000 gas
- At 50 gwei: ~0.0075 ETH (~$25 at current prices)
- *This triggers CCIP cross-chain relay automatically*

**Polygon (Via CCIP Relay)**:
- Automatic relay from Ethereum
- ~0.02-0.05 MATIC (~$0.01-0.03)

**Avalanche (Via CCIP Relay)**:
- Automatic relay from Ethereum
- ~0.1-0.2 AVAX (~$0.05-0.15)

**Total per Attestation**: ~$25-26 (includes 3-chain relay)

### Cost Reduction Strategies

#### 1. Batched Attestations
```typescript
// Instead of individual attestations, batch multiple documents
async batchAttest(documents: Document[]): Promise<AttestationProof[]> {
  // Submit all documents in single transaction
  const merkleRoot = calculateMerkleRoot(documents);
  const batchTx = await router.attestBatch(merkleRoot, documents);
  
  // Cost: ~200,000 gas for 10 documents = ~$0.02 per document
}
```

#### 2. Use Layer 2s for Verification
```typescript
// Submit primary attestation to Polygon (cheaper) instead of Ethereum
const attestationChain = isProd ? 'ethereum' : 'polygon';
```

#### 3. Chainlink Automation
```typescript
// Use Chainlink Automation to batch check-ins instead of manual polling
registerAutomationTask({
  target: router.address,
  interval: 3600, // 1 hour
  maxGasPrice: ethers.utils.parseUnits('100', 'gwei')
});
```

---

## Part 4: Environment Setup

### 1. Install Dependencies

```bash
npm install ethers dotenv @chainlink/contracts
```

### 2. Create `.env` File

```bash
# RPC Endpoints
ETH_RPC_URL=https://eth.llamarpc.com
POLYGON_RPC_URL=https://polygon-rpc.com
AVAX_RPC_URL=https://avalanche-c-chain-rpc.publicnode.com

# Private Key (for signing transactions)
PRIVATE_KEY=0x...

# Chainlink Router Addresses
CHAINLINK_ETH_ROUTER=0x80226fc0Ee2b05F1f63473b78bEd9d5C0092d58d
CHAINLINK_POLYGON_ROUTER=0x70499c328e1e2a3c41ce5d8e3fef69911bc8248e
CHAINLINK_AVAX_ROUTER=0xF694E193200268f9666cD1DFbDea7D7B29a1ccd1

# LINK Token for paying Chainlink
CHAINLINK_LINK_TOKEN=0x514910771AF9Ca656af840dff83E8264EcF986CA

# Gas Configuration
GAS_PRICE_MULTIPLIER=1.2  # Pay 20% above current for faster confirmation
MAX_GAS_LIMIT=500000       # Cap on single transaction
```

### 3. Initialize Chainlink Attestor

```typescript
import { ChainlinkAttestor } from './oracle/chainlink-attestor';

const attestor = new ChainlinkAttestor(process.env.PRIVATE_KEY!);

// Later in CLI
const proof = await attestor.attestDocument(
  arweaveUrl,
  documentHash,
  documentType,
  metadata
);

console.log(`Attestation ID: ${proof.attestationId}`);
console.log(`Ethereum: ${proof.chains.ethereum.txHash}`);
console.log(`Polygon: ${proof.chains.polygon.txHash}`);
console.log(`Avalanche: ${proof.chains.avalanche.txHash}`);
```

---

## Part 5: Testing

### Unit Tests

```bash
# Test Chainlink integration
npm run test -- oracle/chainlink-attestor.test.ts
```

### Integration Tests (Testnet)

```bash
# Deploy to Sepolia/Mumbai/Fuji and test
npm run test:integration -- --testnet sepolia
```

### Local Hardhat Testing

```typescript
// test/oracle/chainlink-attestor.test.ts
import { expect } from 'chai';
import { ethers } from 'hardhat';

describe('ChainlinkAttestor', () => {
  let attestor: ChainlinkAttestor;
  let router: ChainlinkRouter;
  
  beforeEach(async () => {
    // Deploy mock Chainlink router
    router = await ethers.deployContract('MockChainlinkRouter');
    attestor = new ChainlinkAttestor(ethers.provider, router.address);
  });
  
  it('should attest document on primary chain', async () => {
    const arweaveUrl = 'https://arweave.net/abc123';
    const documentHash = ethers.utils.keccak256('0x1234');
    
    const proof = await attestor.attestDocument(
      arweaveUrl,
      documentHash.toString(),
      'contract',
      {}
    );
    
    expect(proof.attestationId).to.not.be.empty;
    expect(proof.chains.ethereum.txHash).to.match(/^0x/);
  });
  
  it('should verify cross-chain attestation', async () => {
    const arweaveUrl = 'https://arweave.net/xyz789';
    const documentHash = ethers.utils.keccak256('0x5678');
    
    const proof = await attestor.attestDocument(arweaveUrl, documentHash.toString(), 'invoice', {});
    
    const verifier = new ChainlinkVerifier(ethers.provider, router.address);
    const verification = await verifier.verifyMultichain(
      proof.attestationId,
      documentHash.toString(),
      2 // Require 2/3 chains
    );
    
    expect(verification.verified).to.be.true;
    expect(verification.confirmedChains).to.be.greaterThanOrEqual(2);
  });
});
```

---

## Part 6: Monitoring & Debugging

### Track Attestation Status

```bash
# Query attestation status on all chains
stamp attest status <attestationId>

# Example output:
# Attestation ID: 0x1234...
# Ethereum: ✅ Confirmed (block 19842123)
# Polygon: ✅ Confirmed (block 52145987)
# Avalanche: ⏳ Pending (estimated 10 mins)
```

### Monitor Gas Prices

```typescript
// src/oracle/gas-monitor.ts
class GasMonitor {
  async monitorGasPrices(): Promise<GasReport> {
    return {
      ethereum: await this.fetchGasPrice('ethereum'),
      polygon: await this.fetchGasPrice('polygon'),
      avalanche: await this.fetchGasPrice('avalanche')
    };
  }
  
  // Alert if gas exceeds threshold
  async checkGasPrices(): Promise<void> {
    const prices = await this.monitorGasPrices();
    const MAX_GAS_PRICE = ethers.utils.parseUnits('150', 'gwei');
    
    if (prices.ethereum > MAX_GAS_PRICE) {
      console.warn('Ethereum gas price too high, delaying attestation');
      // Queue for later batch submission
    }
  }
}
```

---

## Part 7: Troubleshooting

### Common Issues

#### Issue: "CCIP message not relayed to destination"
**Solution**: 
- Check Chainlink DON (Decentralized Oracle Network) status at https://chain.link/
- Verify LINK token balance is sufficient
- Ensure router address is correct for network

#### Issue: "Attestation not found on Polygon/Avalanche"
**Solution**:
- CCIP relay takes 15-30 mins (not instant)
- Use `attest status` command to check cross-chain confirmation
- Manually trigger relay if stuck: `stamp attest relay <attestationId>`

#### Issue: "Gas estimation failed"
**Solution**:
- Increase `GAS_PRICE_MULTIPLIER` in .env
- Check if network is congested (look at etherscan/polygonscan)
- Use `stamp attest config` to adjust gas parameters

---

## Part 8: Migration from Witnet

### Data Structure Changes

**Old Witnet Structure**:
```typescript
witnet: {
  requestId: string;
  taskId: string;
  report: { reportId, result, timestamp, finality };
}
```

**New Chainlink Structure**:
```typescript
chainlink: {
  attestationId: string;           // Unique identifier
  sourceChain: 'ethereum';         // Primary chain
  ccipMessageId: string;           // CCIP message identifier
  chains: {                        // Multi-chain confirmations
    ethereum: { txHash, confirmed, blockNumber };
    polygon: { txHash, confirmed, blockNumber };
    avalanche: { txHash, confirmed, blockNumber };
  };
}
```

### Migration Script

```bash
# Migrate existing Witnet proofs to Chainlink format
npm run migrate:witnet-to-chainlink

# This:
# 1. Reads all existing proofs from database
# 2. Re-attests on Chainlink for compatibility
# 3. Updates proof records with new structure
# 4. Verifies cross-chain confirmation
```

---

## References

- [Chainlink CCIP Docs](https://docs.chain.link/ccip)
- [Chainlink VRF](https://docs.chain.link/vrf) (if randomization needed)
- [Chainlink Functions](https://docs.chain.link/functions) (custom computation)
- [GitHub: Chainlink Contracts](https://github.com/smartcontractkit/chainlink)

---

**Last Updated**: December 2025  
**Status**: Ready for Phase 1 Implementation  
**Next**: Deploy AttestationOracle contracts to Ethereum, Polygon, Avalanche
