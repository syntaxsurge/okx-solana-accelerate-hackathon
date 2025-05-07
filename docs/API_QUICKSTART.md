# OKX DEX API Overview

This document provides an overview of the OKX DEX API for participants in the Solana Accelerate Hackathon.

## Introduction

The OKX DEX API enables developers to integrate decentralized exchange functionality into their applications.

## Building Swap Applications

For the Solana Accelerate Hackathon, we recommend the **API-First Approach**: Directly interact with OKX DEX API endpoints for complete control over the implementation.

## Key Features

### Price Aggregation

The API aggregates prices across multiple liquidity sources to:
- Ensure best execution prices
- Minimize price impact for large trades
- Access deep liquidity pools

## Implementation Process

### 1. Set Up Environment

```javascript
// Required libraries
import base58 from "bs58";
import BN from "bn.js";
import * as solanaWeb3 from "@solana/web3.js";
import { Connection } from "@solana/web3.js";
import cryptoJS from "crypto-js";
import axios from "axios";
import dotenv from 'dotenv';
```

### 2. Authentication

To use the OKX DEX API, you'll need to:
1. Register for an API key at the [OKX Developer Portal](https://www.okx.com/developers)
2. Include the API key in your requests using the proper headers:

```javascript
function getHeaders(timestamp, method, requestPath, queryString = "", body = "") {
  const stringToSign = timestamp + method + requestPath + (queryString || body);
  return {
    "Content-Type": "application/json",
    "OK-ACCESS-KEY": apiKey,
    "OK-ACCESS-SIGN": cryptoJS.enc.Base64.stringify(
      cryptoJS.HmacSHA256(stringToSign, secretKey)
    ),
    "OK-ACCESS-TIMESTAMP": timestamp,
    "OK-ACCESS-PASSPHRASE": apiPassphrase,
    "OK-ACCESS-PROJECT": projectId,
  };
}
```

### 3. Get Swap Data

```javascript
async function getSwapData(
  fromTokenAddress, 
  toTokenAddress, 
  amount, 
  slippage = '0.5'
) {
  const timestamp = new Date().toISOString();
  const requestPath = "/api/v5/dex/aggregator/swap";

  const params = {
    amount: amount,
    chainId: SOLANA_CHAIN_ID,
    fromTokenAddress: fromTokenAddress,
    toTokenAddress: toTokenAddress,
    userWalletAddress: userAddress,
    slippage: slippage
  };

  const queryString = "?" + new URLSearchParams(params).toString();
  const headers = getHeaders(timestamp, "GET", requestPath, queryString);

  const response = await axios.get(
    `https://web3.okx.com${requestPath}${queryString}`,
    { headers }
  );
  
  return response.data.data[0];
}
```

### 4. Prepare Transaction

```javascript
async function prepareTransaction(callData) {
  // Decode the base58 encoded transaction data
  const decodedTransaction = base58.decode(callData);
  
  // Get the latest blockhash
  const recentBlockHash = await connection.getLatestBlockhash();
  
  let tx;
  
  // Try to deserialize as a versioned transaction first
  try {
    tx = solanaWeb3.VersionedTransaction.deserialize(decodedTransaction);
    tx.message.recentBlockhash = recentBlockHash.blockhash;
  } catch (e) {
    // Fall back to legacy transaction if versioned fails
    tx = solanaWeb3.Transaction.from(decodedTransaction);
    tx.recentBlockhash = recentBlockHash.blockhash;
  }
  
  return { transaction: tx, recentBlockHash };
}
```

### 5. Sign Transaction

```javascript
async function signTransaction(tx) {
  const feePayer = solanaWeb3.Keypair.fromSecretKey(
    base58.decode(userPrivateKey)
  );
  
  if (tx instanceof solanaWeb3.VersionedTransaction) {
    tx.sign([feePayer]);
  } else {
    tx.partialSign(feePayer);
  }
  
  return tx;
}
```

### 6. Broadcast Transaction

There are two ways to broadcast your transaction:

#### Using Solana RPC

```javascript
async function broadcastWithRPC(signedTx) {
  const txId = await connection.sendRawTransaction(signedTx.serialize());
  await connection.confirmTransaction(txId);
  return txId;
}
```

#### Using Onchain Gateway API

```javascript
async function broadcastTransaction(signedTx) {
  const serializedTx = signedTx.serialize();
  const encodedTx = base58.encode(serializedTx);
  
  const path = "dex/pre-transaction/broadcast-transaction";
  const url = `https://web3.okx.com/api/v5/${path}`;
  
  const broadcastData = {
    signedTx: encodedTx,
    chainIndex: SOLANA_CHAIN_ID,
    address: userAddress
  };
  
  const bodyString = JSON.stringify(broadcastData);
  const timestamp = new Date().toISOString();
  const requestPath = `/api/v5/${path}`;
  const headers = getHeaders(timestamp, 'POST', requestPath, "", bodyString);
  
  const response = await axios.post(url, broadcastData, { headers });
  return response.data.data[0].orderId;
}
```

### 7. Track Transaction

```javascript
async function trackTransaction(orderId, intervalMs = 5000, timeoutMs = 300000) {
  const startTime = Date.now();
  let lastStatus = '';

  while (Date.now() - startTime < timeoutMs) {
    const path = 'dex/post-transaction/orders';
    const url = `https://web3.okx.com/api/v5/${path}`;

    const params = {
      orderId: orderId,
      chainIndex: SOLANA_CHAIN_ID,
      address: userAddress,
      limit: '1'
    };

    const timestamp = new Date().toISOString();
    const requestPath = `/api/v5/${path}`;
    const queryString = "?" + new URLSearchParams(params).toString();
    const headers = getHeaders(timestamp, 'GET', requestPath, queryString);

    const response = await axios.get(url, { params, headers });
    
    if (response.data.code === '0' && response.data.data?.[0]?.orders?.[0]) {
      const txData = response.data.data[0].orders[0];
      const status = txData.txStatus;

      if (status !== lastStatus) {
        lastStatus = status;

        if (status === '1') {
          console.log(`Transaction pending`);
        } else if (status === '2') {
          console.log(`Transaction successful`);
          return txData;
        } else if (status === '3') {
          throw new Error(`Transaction failed: ${txData.failReason || 'Unknown reason'}`);
        }
      }
    }

    await new Promise(resolve => setTimeout(resolve, intervalMs));
  }

  throw new Error('Transaction tracking timed out');
}
```


The API will handle the gas fee payment on behalf of the user, and the transaction will be submitted to the network by OKX's relayer.

## Complete Implementation

Check the full implementation in the examples directory for:
- Error handling with specific error types and actions
- Additional tracking using the SWAP API
- Best practices for transaction broadcasting
- Command-line interface for testing

## Best Practices

1. **Error Handling**: Implement robust error handling for failed transactions
2. **Rate Limiting**: Respect API rate limits to avoid throttling
3. **Slippage Protection**: Always include slippage tolerance for swaps
4. **User Authorization**: Implement proper authorization flows for user transactions
5. **Testing**: Test thoroughly on testnet before moving to mainnet

## Resources

- [OKX DEX SDK on GitHub](https://github.com/okx/okx-dex-sdk)
- [ElizaOS Agent Plugin](https://github.com/Julian-dev28/plugin-okx)
- [Solana Agent Kit OKX DEX Starter](https://github.com/sendaifun/solana-agent-kit/tree/main/examples/okx-dex-starter)

## Support

For technical support during the hackathon:
- Join the [Discord server](https://discord.gg/hW4EvbVgem)
- Attend scheduled office hours
- Check the [FAQ document](./faq.md)