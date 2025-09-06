# Integration Documentation

## API Endpoints

### GET /v1/get-intent-quote

Gets a quote for cross-chain token swaps across multiple bridges.

#### Input Parameters (Query String)

```typescript
interface GetIntentQuoteInput {
  senderAddress: string;                    // Required: Address of the sender
  recipientAddress?: string;               // Optional: Address of the recipient (defaults to senderAddress)
  tokenInAddress: string;                  // Required: Contract address of input token
  tokenOutAddress: string;                 // Required: Contract address of output token
  tokenInNetwork: Network;                 // Required: Network of input token
  tokenOutNetwork: Network;                // Required: Network of output token
  tokenAmount: string;                     // Required: Amount of input 
}

enum Network {
  ethereum = 'ethereum',
  arbitrum = 'arbitrum',
  bsc = 'bsc',
  base = 'base',
  polygon = 'polygon',
  optimism = 'optimism',
}
```

#### Output Response

```typescript
interface GetIntentQuoteResponse {
  data: Route | null;  // The quote data or null if no quote available
}

interface Route {
  intentId: string;                        // Unique identifier for this quote/intent
  senderAddress: string;                   // Address of the sender
  recipientAddress: string;                // Address of the recipient
  tokenIn: TokenInOut;                     // Input token details
  tokenOut: TokenInOut;                    // Output token details
  exchangeRate: string;                    // Exchange rate between tokens
  timeEstimation: string;                  // Estimated time for completion
  totalPriceImpact: PriceImpact;          // Price impact information
  bridge: Bridge;                          // Bridge used for the swap
  steps: Step[];                           // Transaction steps required
  metadata: Meta;                          // Additional metadata
  fees?: Record<string, Fee>;              // Optional fee breakdown
}

interface TokenInOut {
  tokenAddress: string;                    // Contract address of the token
  amountInUsd: string;                     // USD value of the amount
  rawAmountFormatted: string;              // Human-readable formatted amount
  rawAmount: string;                       // Raw amount in smallest unit
}

interface PriceImpact {
  inPercentage: string;                    // Price impact as percentage
  inUsd: string;                           // Price impact in USD
}

interface Bridge {
  name: string;                            // Human-readable bridge name
  slug: string;                            // Bridge identifier slug
  icon: string;                            // Bridge icon URL
  url: string;                             // Bridge website URL
}

interface Step {
  requestId: string;                       // Unique request identifier
  transactionType: StepTransactionType;    // Type of transaction
  action: string;                          // Human-readable action description
  description: string;                     // Detailed description
  kind: 'transaction' | 'signature';       // Type of step (transaction or signature)
  data: TransactionData;                   // Transaction data
}

type StepTransactionType = 'deposit' | 'approve' | 'authorize' | 'authorize1' | 'authorize2' | 'swap' | 'send';

interface TransactionData {
  from: string;                            // From address
  to: string;                              // To address
  data: string;                            // Transaction data (hex)
  value: string;                           // ETH value (in wei)
  chainId: number;                         // Chain ID
  maxFeePerGas: string;                    // Maximum fee per gas
  maxPriorityFeePerGas: string;            // Maximum priority fee per gas
}

interface Meta {
  tokenIn: Token;                          // Input token metadata
  tokenOut: Token;                         // Output token metadata
}

interface Token {
  readonly address: string;                // Token contract address
  readonly decimals: number;               // Token decimals
  readonly symbol: string;                 // Token symbol
  readonly name: string;                   // Token name
  readonly logoURI: string;                // Token logo URL
  readonly network: Network;               // Token network
}

interface Fee {
  token: Token;                            // Fee token details
  rawAmount: string;                       // Raw fee amount
  rawAmountFormatted: string;              // Formatted fee amount
  amountInUsd: string;                     // Fee amount in USD
}
```

#### Example Request

```bash
GET /v1/get-intent-quote?senderAddress=0x123...&tokenInAddress=0xA0b86a33E6441b8C4C8C0C4C0C4C0C4C0C4C0C4C&tokenOutAddress=0xB0b86a33E6441b8C4C8C0C4C0C4C0C4C0C4C0C4C&tokenInNetwork=ethereum&tokenOutNetwork=arbitrum&tokenAmount=1000000000000000000
```

#### Example Response

```json
{
  "data": {
    "intentId": "intent_123456789",
    "senderAddress": "0x123...",
    "recipientAddress": "0x123...",
    "tokenIn": {
      "tokenAddress": "0xA0b86a33E6441b8C4C8C0C4C0C4C0C4C0C4C0C4C",
      "amountInUsd": "1000.00",
      "rawAmountFormatted": "1.0",
      "rawAmount": "1000000000000000000"
    },
    "tokenOut": {
      "tokenAddress": "0xB0b86a33E6441b8C4C8C0C4C0C4C0C4C0C4C0C4C",
      "amountInUsd": "995.00",
      "rawAmountFormatted": "995.0",
      "rawAmount": "995000000000000000000"
    },
    "exchangeRate": "995.0",
    "timeEstimation": "5-10 minutes",
    "totalPriceImpact": {
      "inPercentage": "0.5",
      "inUsd": "5.00"
    },
    "bridge": {
      "name": "Relay Bridge",
      "slug": "relay",
      "icon": "https://example.com/relay-icon.png",
      "url": "https://relay.com"
    },
    "steps": [
      {
        "requestId": "req_123",
        "transactionType": "approve",
        "action": "Approve USDC",
        "description": "Approve USDC spending for bridge contract",
        "kind": "transaction",
        "data": {
          "from": "0x123...",
          "to": "0xA0b86a33E6441b8C4C8C0C4C0C4C0C4C0C4C0C4C",
          "data": "0x095ea7b3000000000000000000000000...",
          "value": "0",
          "chainId": 1,
          "maxFeePerGas": "20000000000",
          "maxPriorityFeePerGas": "2000000000"
        }
      }
    ],
    "metadata": {
      "tokenIn": {
        "address": "0xA0b86a33E6441b8C4C8C0C4C0C4C0C4C0C4C0C4C",
        "decimals": 18,
        "symbol": "USDC",
        "name": "USD Coin",
        "logoURI": "https://example.com/usdc.png",
        "network": "ethereum"
      },
      "tokenOut": {
        "address": "0xB0b86a33E6441b8C4C8C0C4C0C4C0C4C0C4C0C4C",
        "decimals": 18,
        "symbol": "USDC",
        "name": "USD Coin",
        "logoURI": "https://example.com/usdc.png",
        "network": "arbitrum"
      }
    },
    "fees": {
      "bridge": {
        "token": {
          "address": "0xA0b86a33E6441b8C4C8C0C4C0C4C0C4C0C4C0C4C",
          "decimals": 18,
          "symbol": "USDC",
          "name": "USD Coin",
          "logoURI": "https://example.com/usdc.png",
          "network": "ethereum"
        },
        "rawAmount": "10000000000000000",
        "rawAmountFormatted": "0.01",
        "amountInUsd": "0.01"
      }
    }
  }
}
```

#### Error Responses

```typescript
interface ErrorResponse {
  error: string;                           // Error message
  details?: string;                        // Additional error details
}
```

#### Notes

- All amounts are returned as strings to avoid precision loss
- The `intentId` can be used to track the status of the swap
- The `steps` array contains all transactions that need to be executed
- The `fees` object provides a breakdown of all applicable fees
- If no quote is available, `data` will be `null`
