# Frontend Integration Guide - Intent Parser SSE

This document provides comprehensive guidance for integrating with the Intent Parser Server-Sent Events (SSE) endpoint. The SSE stream delivers real-time updates about the parsing process, user interactions, and execution states.

## Table of Contents

- [Overview](#overview)
- [API Endpoint](#api-endpoint)
- [Request Format](#request-format)
- [Event Types](#event-types)
- [Implementation Examples](#implementation-examples)
- [Error Handling](#error-handling)
- [Best Practices](#best-practices)

## Overview

The Intent Parser uses Server-Sent Events to provide real-time updates during the parsing and execution process. This allows for dynamic UI updates, user interactions, and progress tracking without polling.

## API Endpoint

```
POST https://igris-engine.hyperbola.network/api/intent-parser/sse
Content-Type: application/json
Authorization: Bearer <YOUR_API_KEY>
```

**Base URL**: `https://igris-engine.hyperbola.network`

**Authorization**: Contact the Effortless Labs team to obtain your authorization key. The API requires a valid Bearer token in the Authorization header for all requests.

**Service Status**: You can check the service health at `https://igris-engine.hyperbola.network/` which returns service information including version and available features.

## Request Format

```typescript
interface IntentParserRequest {
  input: string;                    // Required: User input text
  chatHistory?: ChatMessage[];      // Optional: Previous conversation context
  wallets?: string[];               // Optional: Available wallet addresses
}

interface ChatMessage {
  content: string;                  // Message content
  type: 'human' | 'ai'              // Message type
  role?: string;                    // Optional role identifier
  timestamp?: string;               // Optional timestamp
  metadata?: Record<string, any>;   // Optional additional data
}
```

### Example Request

```typescript
const response = await fetch('https://igris-engine.hyperbola.network/api/intent-parser/sse', {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json',
    'Authorization': 'Bearer YOUR_API_KEY_HERE', // Replace with your actual API key
  },
  body: JSON.stringify({
    input: "Swap 100 USDC to ETH",
    chatHistory: [
      {
        content: "I want to swap some tokens",
        type: "human",
        timestamp: "2024-01-15T10:30:00Z"
      }
    ],
    wallets: ["0x1234...", "0x5678..."]
  })
});
```

## Event Types

The SSE stream delivers several types of events that your frontend should handle. **Important**: Token selection events are intermediate steps that require user input and a new API call to continue the flow.

### 🔄 **Complete Token Selection Flow**

**How the System Works After Token Selection:**

1. **Initial Request**: User makes request (e.g., "I want to get 100 USDC")
2. **Token Selection Event**: System sends `SOURCE_HAVE_MULTIPLE_TOKENS` or `DESTINATION_HAVE_MULTIPLE_TOKENS`
3. **Frontend UI**: Shows token selection modal with available options
4. **User Selection**: User selects specific token (e.g., "ETH on ethereum")
5. **New API Call**: Frontend makes NEW API call with selected token in input
6. **Flow Continuation**: System continues with complete information
7. **Final Result**: System sends `EXECUTE_ROUTE` with complete swap data

**Key Points for Frontend Developers:**
- Token selection events are **NOT final responses**
- **Always make a new API call** after user selects a token
- Include the selection in `chatHistory` for context
- Specify the network when making selections (e.g., "ETH on ethereum", "USDC on base")
- The flow continues automatically after token selection
- Final result is always `EXECUTE_ROUTE` with complete swap information

**Example Flow:**
```
User: "I want to get 100 USDC"
→ API: SOURCE_HAVE_MULTIPLE_TOKENS (shows ETH, MATIC options)
→ User selects: "ETH on ethereum"
→ Frontend: New API call with "I want to get 100 USDC using ETH on ethereum"
→ API: EXECUTE_ROUTE (complete swap data)
```

### 1. Append Events

Events that add new UI elements or states.

#### DEFAULT_LOADER
```typescript
{
  type: 'append',
  value: 'DEFAULT_LOADER',
  message: 'Loading...'
}
```

**Usage**: Display a loading indicator while processing the request.

**Note**: The message is hardcoded as "Loading..." in the backend, so you can either use this value or display your own custom loading message.

### 2. Update Events

Events that modify existing UI elements or provide new information.

**Event Value Mapping Table:**

| Response Value | Description |
|---------------|-------------|
| `EXECUTE_ROUTE` | Execute swap route with route data and tokens |
| `SOURCE_HAVE_MULTIPLE_TOKENS` | Multiple source tokens available for selection |
| `DESTINATION_HAVE_MULTIPLE_TOKENS` | Multiple destination tokens available for selection |
| `AI_MESSAGE_ELEMENT` | AI-generated message content |
| `SOMETHING_WENT_WRONG` | Error message display |

#### EXECUTE_ROUTE
```typescript
{
  type: 'update',
  value: 'EXECUTE_ROUTE',
  message: string,
  route: Route,                                    // LiFi SDK route data for the swap
  sourceToken: TokenWithBalancePriceNetworkWallet, // Selected source token with balance, price, and wallet info
  destinationToken: TokenWithBalancePriceNetworkWallet // Selected destination token with price and wallet info
}
```

**Usage**: Execute a swap route with the provided parameters. This response contains the complete route information and selected tokens for the swap execution.

**Additional Token Types:**
```typescript
interface TokenWithBalancePriceNetworkWallet {
  // Basic token information
  name: string                    // Token name (e.g., "USD Coin")
  slug: string                    // Token identifier (e.g., "usd-coin")
  address: string                 // Token contract address
  symbol: string                  // Token symbol (e.g., "USDC")
  decimals: number                // Token decimal places
  network: Network                // Blockchain network (ethereum, arbitrum, bsc, base, polygon, optimism)
  logoURI: string                // Token logo URL
  
  // Balance and price information
  balance: string                 // Token balance in wei/smallest unit
  price: string                   // Token price in USD
  balanceInUsd: number           // Balance value in USD
  
  // Wallet and network details
  wallet: string                  // Wallet address holding the token
  network: Network                // Network where token exists
}

// Route type from LiFi SDK
interface Route {
  // LiFi SDK route data structure
  // This contains all the necessary information to execute the swap
  // including DEX paths, gas estimates, and transaction details
}
```

#### SOURCE_HAVE_MULTIPLE_TOKENS
```typescript
{
  type: 'update',
  value: 'SOURCE_HAVE_MULTIPLE_TOKENS',
  message: string,
  sourceTokenSuggestion: TokenWithBalancePriceNetworkWallet[],    // Array of available source tokens with balance, price, and wallet info
  destinationToken: TokenWithBalancePriceNetworkWallet | null    // Selected destination token with balance, price, and wallet info (can be null)
}
```

**Token Types:**
```typescript
interface TokenWithBalancePriceNetworkWallet {
  // Basic token information
  name: string                    // Token name (e.g., "USD Coin")
  slug: string                    // Token identifier (e.g., "usd-coin")
  address: string                 // Token contract address
  symbol: string                  // Token symbol (e.g., "USDC")
  decimals: number                // Token decimal places
  network: Network                // Blockchain network (ethereum, arbitrum, bsc, base, polygon, optimism)
  logoURI: string                // Token logo URL
  
  // Balance and price information
  balance: string                 // Token balance in wei/smallest unit
  price: string                   // Token price in USD
  balanceInUsd: number           // Balance value in USD
  
  // Wallet and network details
  wallet: string                  // Wallet address holding the token
  network: Network                // Network where token exists
}

enum Network {
  ethereum = 'ethereum',
  arbitrum = 'arbitrum',
  bsc = 'bsc',
  base = 'base',
  polygon = 'polygon',
  optimism = 'optimism'
}
```

**Usage**: Display token selection UI when multiple source tokens are available. The user needs to choose which source token to use for the swap.

**Important Flow Information**: 
- **This is NOT the final step** - it's an intermediate step requiring user input
- **Frontend must make a NEW API call** after user selects a token
- **Include the selection in chatHistory** for context
- **Specify the network** when making the selection (e.g., "ETH on ethereum")

**Complete Flow After Token Selection:**
1. User selects a token from suggestions
2. Frontend makes new API call with selected token in input
3. System continues flow with complete token information
4. System calculates source amount needed and finds swap route
5. System sends `EXECUTE_ROUTE` event with complete swap data

**Example Frontend Implementation:**
```typescript
// Handle token selection event
case 'SOURCE_HAVE_MULTIPLE_TOKENS':
  showTokenSelectionModal(event.sourceTokenSuggestion);
  break;

// User selects token
onTokenSelected(selectedToken) {
  // Make new API call with selection
  const newRequest = {
    input: `I want to get 100 USDC using ${selectedToken.symbol} on ${selectedToken.network}`,
    chatHistory: [...existingHistory, userSelection],
    wallets: wallets
  };
  
  // Start new parsing session
  startParsing(newRequest);
}
```

**Note**: This response value and the response from multiple source tokens in destination flow share the same structure, so your frontend can handle both with the same UI component.

**Important**: The backend implementation shows that both `SOURCE_HAVE_MULTIPLE_TOKENS` and multiple source tokens in destination flow events are handled in the same case statement and send identical response structures. This means your frontend can use the same handler for both events.

#### DESTINATION_HAVE_MULTIPLE_TOKENS
```typescript
{
  type: 'update',
  value: 'DESTINATION_HAVE_MULTIPLE_TOKENS',
  message: string,
  destinationTokenSuggestion: TokenWithBalancePriceNetworkWallet[]    // Array of available destination tokens with balance, price, and wallet info
}
```

**Usage**: Display token selection UI when multiple destination tokens are available. The user needs to choose which destination token they want to receive.

**Important Flow Information**: 
- **This is NOT the final step** - it's an intermediate step requiring user input
- **Frontend must make a NEW API call** after user selects a token
- **Include the selection in chatHistory** for context
- **Specify the network** when making the selection (e.g., "USDC on base")

**Complete Flow After Token Selection:**
1. User selects a destination token from suggestions
2. Frontend makes new API call with selected token in input
3. System continues flow with complete token information
4. System finds suitable source tokens and calculates amounts
5. System sends `EXECUTE_ROUTE` event with complete swap data

**Example Frontend Implementation:**
```typescript
// Handle token selection event
case 'DESTINATION_HAVE_MULTIPLE_TOKENS':
  showDestinationTokenSelectionModal(event.destinationTokenSuggestion);
  break;

// User selects destination token
onDestinationTokenSelected(selectedToken) {
  // Make new API call with selection
  const newRequest = {
    input: `${selectedToken.symbol} on ${selectedToken.network}`,
    chatHistory: [...existingHistory],
    wallets: wallets
  };
  
  // Start new parsing session
  startParsing(newRequest);
}
```

#### Multiple Source Tokens in Destination Flow
```typescript
{
  type: 'update',
  value: 'SOURCE_HAVE_MULTIPLE_TOKENS',  // Note: Always returns SOURCE_HAVE_MULTIPLE_TOKENS
  message: string,
  sourceTokenSuggestion: TokenWithBalancePriceNetworkWallet[],    // Array of available source tokens with balance, price, and wallet info
  destinationToken: TokenWithBalancePriceNetworkWallet            // Selected destination token with balance, price, and wallet info
}
```

**Usage**: This response is triggered in the destination flow when:
1. The user has already selected a destination token and amount
2. The system is trying to find suitable source tokens to fulfill the swap
3. Multiple source tokens are found that could potentially complete the swap
4. The user needs to choose which source token to use

This typically happens when the user says something like "I want to get 100 USDC" without specifying which token they want to swap from, and the system finds multiple tokens in their wallet that could potentially complete the swap.

**Important**: Even though this response is triggered for multiple source tokens in destination flow, the response always contains `value: 'SOURCE_HAVE_MULTIPLE_TOKENS'`. Your frontend should handle both responses with the same logic since they produce identical response structures.

The frontend should display a token selection UI showing the available source tokens with their balances and prices, allowing the user to choose the most suitable one for their swap.

#### AI_MESSAGE_ELEMENT
```typescript
{
  type: 'update',
  value: 'AI_MESSAGE_ELEMENT',
  message: string    // AI-generated message content
}
```

**Usage**: Display AI-generated messages or responses in the chat interface.

#### SOMETHING_WENT_WRONG
```typescript
{
  type: 'update',
  value: 'SOMETHING_WENT_WRONG',
  message: string    // Error message to display to the user
}
```

**Usage**: Display error messages or fallback UI states when something goes wrong during the parsing or execution process.

#### Default Case
```typescript
{
  type: 'update',
  value: string    // The original input value from the event
}
```

**Usage**: This is the default case that handles any other event values not explicitly defined above. The `value` field will contain whatever was originally passed in the event.

### 3. Chat Chunk Events

Real-time streaming of AI responses.

```typescript
{
  type: 'chat_chunk',
  text: string,
  runId: string
}
```

**Usage**: Stream AI responses character by character for a natural typing effect.

### 4. Error Events

Error handling and connection issues.

```typescript
{
  type: 'error',
  message: string,
  error: string
}
```

**Usage**: Display error messages and provide retry options.

## Implementation Examples

### Basic SSE Client Implementation

```typescript
class IntentParserClient {
  private eventSource: EventSource | null = null;

  async startParsing(request: IntentParserRequest): Promise<void> {
    const response = await fetch('https://igris-engine.hyperbola.network/api/intent-parser/sse', {
      method: 'POST',
      headers: { 
        'Content-Type': 'application/json',
        'Authorization': 'Bearer YOUR_API_KEY_HERE' // Replace with your actual API key
      },
      body: JSON.stringify(request)
    });

    if (!response.ok) {
      throw new Error(`HTTP error! status: ${response.status}`);
    }

    const reader = response.body?.getReader();
    if (!reader) throw new Error('No response body');

    const decoder = new TextDecoder();
    
    while (true) {
      const { done, value } = await reader.read();
      if (done) break;

      const chunk = decoder.decode(value);
      const lines = chunk.split('\n');

      for (const line of lines) {
        if (line.startsWith('data: ')) {
          try {
            const data = JSON.parse(line.slice(6));
            this.handleEvent(data);
          } catch (e) {
            console.error('Failed to parse SSE data:', e);
          }
        }
      }
    }
  }

  private handleEvent(event: any): void {
    switch (event.type) {
      case 'append':
        this.handleAppendEvent(event);
        break;
      case 'update':
        this.handleUpdateEvent(event);
        break;
      case 'chat_chunk':
        this.handleChatChunk(event);
        break;
      case 'error':
        this.handleError(event);
        break;
    }
  }

  private handleAppendEvent(event: any): void {
    switch (event.value) {
      case 'DEFAULT_LOADER':
        this.showLoader(event.message);
        break;
    }
  }

  private handleUpdateEvent(event: any): void {
    switch (event.value) {
      case 'EXECUTE_ROUTE':
        this.executeRoute(event.route, event.sourceToken, event.destinationToken);
        break;
      case 'SOURCE_HAVE_MULTIPLE_TOKENS':
        this.showSourceTokenSelection(event.sourceTokenSuggestion, event.destinationToken);
        break;
      case 'DESTINATION_HAVE_MULTIPLE_TOKENS':
        this.showDestinationTokenSelection(event.destinationTokenSuggestion);
        break;
      case 'AI_MESSAGE_ELEMENT':
        this.displayAIMessage(event.message);
        break;
      case 'SOMETHING_WENT_WRONG':
        this.showErrorMessage(event.message);
        break;
    }
  }

  private handleChatChunk(event: any): void {
    this.appendChatText(event.text, event.runId);
  }

  private handleError(event: any): void {
    this.showError(event.message, event.error);
  }

  // UI methods - implement based on your framework
  private showLoader(message: string): void {
    // Show loading indicator
  }

  private executeRoute(route: any, sourceToken: any, destinationToken: any): void {
    // Execute the swap route
  }

  private showSourceTokenSelection(suggestions: any[], destinationToken: any): void {
    // Show token selection modal
  }

  private showDestinationTokenSelection(suggestions: any[]): void {
    // Show destination token selection modal
  }

  private displayAIMessage(message: string): void {
    // Display AI message in chat
  }

  private showErrorMessage(message: string): void {
    // Show error message
  }

  private appendChatText(text: string, runId: string): void {
    // Append text to chat stream
  }

  private showError(message: string, error: string): void {
    // Show error with details
  }
}
```

### React Hook Implementation

```typescript
import { useState, useEffect, useCallback } from 'react';

interface UseIntentParserOptions {
  onLoader?: (message: string) => void;
  onExecuteRoute?: (route: any, sourceToken: any, destinationToken: any) => void;
  onTokenSelection?: (type: 'source' | 'destination', suggestions: any[]) => void;
  onAIMessage?: (message: string) => void;
  onError?: (message: string, error?: string) => void;
  onChatChunk?: (text: string, runId: string) => void;
}

export function useIntentParser(options: UseIntentParserOptions) {
  const [isLoading, setIsLoading] = useState(false);
  const [error, setError] = useState<string | null>(null);

  const startParsing = useCallback(async (request: IntentParserRequest) => {
    setIsLoading(true);
    setError(null);

    try {
      const response = await fetch('/api/intent-parser/sse', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(request)
      });

      if (!response.ok) {
        throw new Error(`HTTP error! status: ${response.status}`);
      }

      const reader = response.body?.getReader();
      if (!reader) throw new Error('No response body');

      const decoder = new TextDecoder();
      
      while (true) {
        const { done, value } = await reader.read();
        if (done) break;

        const chunk = decoder.decode(value);
        const lines = chunk.split('\n');

        for (const line of lines) {
          if (line.startsWith('data: ')) {
            try {
              const data = JSON.parse(line.slice(6));
              handleEvent(data);
            } catch (e) {
              console.error('Failed to parse SSE data:', e);
            }
          }
        }
      }
    } catch (err) {
      const errorMessage = err instanceof Error ? err.message : 'Unknown error';
      setError(errorMessage);
      options.onError?.(errorMessage, errorMessage);
    } finally {
      setIsLoading(false);
    }
  }, [options]);

  const handleEvent = useCallback((event: any) => {
    switch (event.type) {
      case 'append':
        if (event.value === 'DEFAULT_LOADER') {
          options.onLoader?.(event.message);
        }
        break;
      case 'update':
        switch (event.value) {
          case 'EXECUTE_ROUTE':
            options.onExecuteRoute?.(event.route, event.sourceToken, event.destinationToken);
            break;
          case 'SOURCE_HAVE_MULTIPLE_TOKENS':
            options.onTokenSelection?.('source', event.sourceTokenSuggestion);
            break;
          case 'DESTINATION_HAVE_MULTIPLE_TOKENS':
            options.onTokenSelection?.('destination', event.destinationTokenSuggestion);
            break;
          case 'AI_MESSAGE_ELEMENT':
            options.onAIMessage?.(event.message);
            break;
          case 'SOMETHING_WENT_WRONG':
            options.onError?.(event.message);
            break;
        }
        break;
      case 'chat_chunk':
        options.onChatChunk?.(event.text, event.runId);
        break;
      case 'error':
        options.onError?.(event.message, event.error);
        break;
    }
  }, [options]);

  return {
    startParsing,
    isLoading,
    error
  };
}
```

### Vue.js Composition API

```typescript
import { ref, reactive } from 'vue';

export function useIntentParser() {
  const isLoading = ref(false);
  const error = ref<string | null>(null);

  const startParsing = async (request: IntentParserRequest) => {
    isLoading.value = true;
    error.value = null;

    try {
      const response = await fetch('/api/intent-parser/sse', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(request)
      });

      if (!response.ok) {
        throw new Error(`HTTP error! status: ${response.status}`);
      }

      const reader = response.body?.getReader();
      if (!reader) throw new Error('No response body');

      const decoder = new TextDecoder();
      
      while (true) {
        const { done, value } = await reader.read();
        if (done) break;

        const chunk = decoder.decode(value);
        const lines = chunk.split('\n');

        for (const line of lines) {
          if (line.startsWith('data: ')) {
            try {
              const data = JSON.parse(line.slice(6));
              handleEvent(data);
            } catch (e) {
              console.error('Failed to parse SSE data:', e);
            }
          }
        }
      }
    } catch (err) {
      const errorMessage = err instanceof Error ? err.message : 'Unknown error';
      error.value = errorMessage;
    } finally {
      isLoading.value = false;
    }
  };

  const handleEvent = (event: any) => {
    // Implement event handling logic
  };

  return {
    isLoading,
    error,
    startParsing
  };
}
```

## Error Handling

### Connection Errors
- Implement automatic reconnection logic
- Show user-friendly error messages
- Provide retry mechanisms

### Parsing Errors
- Handle malformed SSE data gracefully
- Log errors for debugging
- Fallback to error states

### User Feedback
- Always inform users about errors
- Provide actionable error messages
- Maintain UI consistency during error states

## Best Practices

### 1. Connection Management
- Implement proper cleanup on component unmount
- Handle connection timeouts gracefully
- Provide connection status indicators

### 2. Event Handling
- Use event delegation for better performance
- Implement proper error boundaries
- Handle all event types, even unknown ones

### 3. UI/UX Considerations
- Show loading states for better user experience
- Implement progressive disclosure for complex flows
- Provide clear feedback for all user actions

### 4. Performance
- Debounce rapid events when appropriate
- Implement proper memory management
- Use efficient data structures for state management

### 5. Accessibility
- Provide alternative text for loading states
- Ensure error messages are screen reader friendly
- Implement proper ARIA labels for dynamic content

## Testing

### Unit Testing
- Mock SSE responses for component testing
- Test error handling scenarios
- Verify event parsing logic

### Integration Testing
- Test with real SSE endpoints
- Verify error recovery mechanisms
- Test connection stability

### User Testing
- Validate user experience flows
- Test error scenarios with real users
- Gather feedback on loading states and error messages

## Troubleshooting

### Common Issues

1. **Connection Drops**: Implement exponential backoff for reconnections
2. **Memory Leaks**: Ensure proper cleanup of event listeners
3. **Race Conditions**: Handle concurrent requests appropriately
4. **Browser Compatibility**: Test across different browsers and versions

### Debug Tools

- Use browser DevTools to monitor SSE connections
- Implement comprehensive logging
- Use network tab to debug connection issues

## Support

For technical support or questions about the integration:

- Check the API documentation
- Review error logs and network requests
- Contact the development team with specific error details

---

*This documentation is maintained by the Intent Parser development team. Please ensure you're using the latest version for the most up-to-date information.* 