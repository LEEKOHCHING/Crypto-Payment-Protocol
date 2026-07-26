# Crypto Payment Protocol (URI)

> A lightweight, chain-agnostic URI scheme for crypto payments — designed for QR-code POS terminals, deep links, and wallet interoperability.

[![Status](https://img.shields.io/badge/status-draft-yellow)](https://github.com/)
[![Version](https://img.shields.io/badge/version-1.0.0--draft-green)](https://github.com/)
[![License](https://img.shields.io/badge/license-MIT-blue)](LICENSE)

---

## Abstract

This document proposes the **CryptoPay URI Protocol** — a simple, consistent URI scheme for initiating cryptocurrency payments across multiple blockchains. The format encodes the destination chain, recipient address, token contract, and payment amount in a single string that can be embedded in a QR code, NFC tag, or clickable deep link.

When a wallet scans or receives a CryptoPay URI, it should automatically select the correct network, identify the token, pre-fill all payment fields, and present the user with a single confirmation step — eliminating manual data entry and reducing payment errors.

---

## Motivation

Real-world crypto payments are unnecessarily complex. A customer scanning a merchant QR code today must manually:

1. Select the correct network
2. Choose the right token
3. Type in the recipient address
4. Enter the payment amount

Four steps that introduce friction and human error.

Bitcoin solved this for BTC with **BIP-21** in 2012. Ethereum introduced **EIP-681**, but its ABI-encoded format is difficult to generate and parse, especially across different token standards and chains. Neither handles multi-chain ERC-20 payments in a developer-friendly way.

The CryptoPay URI scheme fills this gap: a format that is **easy to generate**, **easy to parse**, **human-readable**, and **consistent across BSC, Ethereum, Polygon, Solana, and any future chain**.

> **Goal:** A user scanning a CryptoPay QR should see a pre-filled confirmation screen in their wallet with zero manual input required. One tap to pay.

---

## Specification

### URI Format

```
chain:recipient_address?token=contract_address&amount=decimal_amount[&label=text][&desc=text]
```

### Anatomy

```
bsc:0xded849dedb95bb59dee408650cc8f4e18bd458f4?token=0x319558c8aD708dc42f45ab70eADA4750d6c942d7&amount=49.95&label=TabbyPOS
│    │                                          │      │                                          │        │
│    └─ Recipient address (required)            │      └─ ERC-20 contract address (required)     │        └─ Merchant name (optional)
│                                               └─ Parameter: token                              └─ Parameter: amount (human-readable)
└─ Chain identifier (required)
```

### ABNF Grammar

```abnf
cryptopay-uri   = chain ":" address "?" required-params *( "&" optional-param )

chain           = 1*( ALPHA / DIGIT / "-" )   ; e.g. "bsc", "eth", "polygon"
address         = 1*pchar                      ; chain-native address format

required-params = token-param "&" amount-param
                / amount-param                 ; native coin: omit token

token-param     = "token=" address             ; ERC-20 / SPL contract address
amount-param    = "amount=" decimal            ; human-readable, NOT in wei
decimal         = 1*DIGIT [ "." 1*DIGIT ]

optional-param  = label-param / desc-param / decimals-param
label-param     = "label=" 1*pchar             ; merchant display name
desc-param      = "desc=" 1*pchar              ; order reference / note
decimals-param  = "decimals=" 1*DIGIT          ; fallback hint if on-chain query fails
```

> **Important:** `amount` is always expressed in **human-readable decimal units**, never in the token's smallest unit (wei, lamports, etc.). Wallets must multiply by the token's `decimals()` value internally. This prevents off-by-18 errors and makes URIs readable by humans.

---

## Parameters

| Parameter | Required | Type | Description |
|-----------|----------|------|-------------|
| `chain` (prefix) | ✅ Yes | string | Chain identifier before `:`. See Chain Registry below. |
| `address` (path) | ✅ Yes | string | Recipient wallet address in the native format of the chain. |
| `token` | ✅ Yes* | address | ERC-20 / SPL token contract address. *Omit for native coin transfers (ETH, BNB, SOL).* |
| `amount` | ✅ Yes | decimal | Payment amount in human-readable units. Must be a positive decimal. e.g. `49.95` |
| `label` | ❌ No | string | Merchant or payee display name shown in wallet UI. URL-encoded. |
| `desc` | ❌ No | string | Payment description or order reference. URL-encoded. |
| `decimals` | ❌ No | integer | Token decimal hint (0–18). Fallback only if `decimals()` on-chain query fails. |

---

## Chain Registry

The following chain identifiers are defined in v1.0:

| Identifier | Network | Chain ID | Address Format |
|------------|---------|----------|----------------|
| `sol` | Solana Mainnet | — | Base58 (32–44 chars) |
| `icp` | Internet Computer | — | Account Identifier / Principal |
| `bsc` | BNB Smart Chain | 56 | EVM (`0x…`, 42 hex chars) |
| `aptos` | Aptos Mainnet | 1 | Move 32-byte hex (0x + up to 64 hex chars) |
| `eth` | Ethereum Mainnet | 1 | EVM (`0x…`, 42 hex chars) |
| `polygon` | Polygon PoS | 137 | EVM (`0x…`, 42 hex chars) |
| `arb` | Arbitrum One | 42161 | EVM (`0x…`, 42 hex chars) |
| `op` | Optimism | 10 | EVM (`0x…`, 42 hex chars) |
| `base` | Base | 8453 | EVM (`0x…`, 42 hex chars) |
| `avax` | Avalanche C-Chain | 43114 | EVM (`0x…`, 42 hex chars) |


> Additional chains may be registered by opening a PR with: chain name, EVM chain ID (if applicable), address format specification, and a reference implementation.

> **Unrecognized chain:** If a wallet encounters an unknown chain prefix, it must display a human-readable error: `"Unsupported network: [chain]"` and must not attempt to process the transaction.

---

## Examples

### ERC-20 token payment (BSC)
```
bsc:0xded849dedb95bb59dee408650cc8f4e18bd458f4?token=0x319558c8aD708dc42f45ab70eADA4750d6c942d7&amount=49.95&label=TabbyPOS
```

### Native coin payment (ETH)
```
eth:0xded849dedb95bb59dee408650cc8f4e18bd458f4?amount=0.05&label=Coffee+Shop
```

### USDT on Polygon with order reference
```
polygon:0xded849dedb95bb59dee408650cc8f4e18bd458f4?token=0xc2132D05D31c914a87C6611C10748AEb04B58e8F&amount=12.50&desc=Order%23A1042
```

### SPL token on Solana
```
sol:7xKXtg2CW87d97TXJSDpbD5jBkheTqA83TZRuJosgAsU?token=EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v&amount=10.00
```

---

## Wallet Implementation Guide

Wallets that support CryptoPay URI should handle the format via both QR scanning and URI deep links. When a CryptoPay URI is detected:

1. **Parse the URI** — Extract `chain` from the prefix, `recipient_address` from the path, and all query parameters. Reject malformed URIs with a clear error message.

2. **Validate the chain** — Check the chain identifier against the supported chain registry. If unknown, show `"Unsupported network: [chain]"` and stop.

3. **Switch network** — If the wallet is on a different network, prompt the user to switch to the required chain before proceeding.

4. **Validate the recipient address** — Verify the address format matches the target chain's specification (EVM checksum, Base58, etc.). Reject invalid addresses.

5. **Resolve token decimals** — If `token` is present, call `decimals()` on the contract. Fall back to the `decimals` URL parameter if the RPC call fails. Use 18 as last resort for EVM tokens.

6. **Show the confirmation screen** — Pre-fill all fields and display: recipient address (with ENS/domain if resolvable), token name and logo, the `label` and `desc` values, and the exact amount. **Do not auto-submit — always require explicit user confirmation.**

7. **Execute and confirm** — After user approval, broadcast the transaction and show confirmation with a block explorer link.

### Minimal Reference Parser (JavaScript)

```javascript
/**
 * Parse a CryptoPay URI string.
 * Returns null if the URI is invalid or unsupported.
 */
function parseCryptoPayUri(uri) {
  // Match:  chain:address?params
  const match = uri.match(/^([a-z][a-z0-9-]*):([^?]+)\?(.+)$/i);
  if (!match) return null;

  const [, chain, address, queryString] = match;
  const params = Object.fromEntries(new URLSearchParams(queryString));

  if (!params.amount || isNaN(parseFloat(params.amount))) return null;

  return {
    chain:    chain.toLowerCase(),
    to:       address,
    token:    params.token    ?? null,   // null = native coin transfer
    amount:   parseFloat(params.amount),
    label:    params.label    ?? null,
    desc:     params.desc     ?? null,
    decimals: params.decimals != null ? parseInt(params.decimals) : null,
  };
}
```

---

## Security Considerations

### Address Validation
Wallets must strictly validate the recipient address format for the target chain. For EVM chains, validate the `0x`-prefix and 40-hex-character format. Apply EIP-55 checksum verification where supported. Reject visually similar addresses (homograph attacks).

### Amount Sanity Check
Wallets should verify the user has sufficient balance before showing the confirmation screen. Display a clear warning if the requested amount exceeds 50% of the user's holdings in that token.

### Unknown Token Contracts

> ⚠️ **Warning:** If the `token` contract address is not in the wallet's verified token list, display a prominent warning: *"Unverified token contract — verify before proceeding."* Show the raw contract address. Do not attempt to display a logo or name from an unverified source.

### No Automatic Execution
A CryptoPay URI must **never** trigger an automatic transaction. The URI is a *payment request*, not an authorization. The user must always explicitly confirm before any funds are moved.

### QR Code Tampering
Users should be educated that physical QR codes can be tampered with (sticker-over-sticker attacks). Wallets should display the full recipient address on the confirmation screen so users can visually verify it.

---

## Prior Art

CryptoPay URI is inspired by existing standards and designed to complement, not replace, them.

| Standard | Chain | Amount Format | Multi-chain | Notes |
|----------|-------|--------------|-------------|-------|
| **BIP-21** (2012) | Bitcoin only | Decimal (BTC) | ❌ | Simple and elegant, but single-chain |
| **EIP-681** (2017) | Ethereum only | Wei (uint256) | ❌ | ABI-encoded, hard to generate manually |
| **CryptoPay URI** (2025) | Any chain | Human-readable decimal | ✅ | Simple URL string, no SDK required |

**Key differences from EIP-681:**
- Amount is human-readable (`49.95`) — not ABI-encoded in the smallest unit (`49950000000000000000`)
- Chain is identified by a readable prefix (`bsc:`, `eth:`, `sol:`) — not an Ethereum-only `chainId` parameter
- Works natively for non-EVM chains (Solana, ICP)
- Easy to generate with a standard URL builder — no ABI encoding library needed

---

## Adoption

This protocol is most valuable when multiple wallets support it.

| Role | How to participate |
|------|--------------------|
| **Wallet developers** | Implement the parser and confirmation UI described in §Implementation Guide. Open a PR to add your wallet to the compatibility list. |
| **POS / payment builders** | Generate CryptoPay URIs when creating payment QR codes. Use URL encoding for `label` and `desc` values. |
| **Chain teams** | Open a PR to register your chain identifier and address validation rules in the chain registry. |
| **Reviewers** | Open issues with feedback on the spec, edge cases, or security concerns. All constructive input is welcome. |

---

## Compatible Wallets

> *Be the first — open a PR to add your wallet here.*

| Wallet | Chains | Status |
|--------|--------|--------|
| Sophia Wallet | BSC | ✅ Reference implementation |

---

## Contributing

1. Fork this repository
2. For **spec changes**: open an issue first to discuss before submitting a PR
3. For **chain registry additions**: include chain name, chain ID (EVM), address format, and a reference parser
4. For **wallet compatibility**: add a row to the Compatible Wallets table with a link to your implementation

---

## License

MIT — free to implement, fork, and extend.

---

*CryptoPay URI Protocol · v1.0.0-draft · 2025*  
*Proposed by [Lee](https://github.com/lkching7) — Founder, TabbyPOS*
