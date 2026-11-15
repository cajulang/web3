# Changelog

All notable changes to `web3`, the EVM library for Caju. The format follows
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/) and the project uses
[Semantic Versioning](https://semver.org/).

## [Unreleased]

## [0.2.0] - 2026-11-15

### Added
- `web3/rpc` fallback: `ClientConfig.urls`; with neither `url` nor `urls` a client uses all of
  the chain's `rpc_urls`. On network errors, timeouts, 429, 5xx or other non-JSON-RPC HTTP
  failures a request moves on to the next URL at once (never on JSON-RPC errors such as
  reverts), sticks to the URL that answered (`Client.urls`, `Client.url`, `current_url()`), and
  repeats the round after a pause up to `retries` times. Batches fall back too.
- `Client.rank_urls(cb)`: orders the URLs by the latency of one `eth_blockNumber`, dead ones last.
- `rpc.fallback(transports)` (viem's fallback transport, sticky) and `rpc.http_transport(url, config)`.
- Replaced transactions: `wait_for_transaction_receipt` fetches the pending transaction (or takes
  `WaitOptions.transaction`), notices when its sender's nonce is used by another transaction
  (sped up, cancelled, repriced), goes on with the replacement and reports it through
  `WaitOptions.on_replaced` (`*rpc.Replacement`: `reason` "repriced" | "cancelled" | "replaced",
  `transaction_hash`, `receipt`, `transaction`, `replaced`); `rpc.replacement_reason`,
  `rpc.REPLACEMENT_SCAN_BLOCKS`.
- `multicall.balances(client, owner, tokens, cb fn(native, tokens, err))`: the native balance
  (Multicall3 `getEthBalance`) and ERC-20 balances in one `aggregate3`.
- ENS gateways: `ClientConfig.ccip_read` (`*rpc.CcipRead{disabled, allowed_hosts}`) disables
  CCIP-read or limits it to some gateway hosts (`*.` for subdomains); blocked lookups fail with
  `*ens.CcipBlockedError` (`ens.is_ccip_blocked`) without any request, for pages with a strict CSP.
  `rpc.url_host`, `CcipRead.allows`.
- Exported JSON-RPC readers in `web3/rpc`: `json_u256`, `json_int`, `json_str`, `json_bytes`,
  `json_address`, `json_strs`, `parse_transaction`, `parse_log`, `parse_logs`, `parse_receipt`,
  `parse_block`, `parse_rpc_error`.

### Changed
- A client without `url` uses all of the chain's `rpc_urls` with fallback (it used only the
  first). Set `url` alone to keep a single URL.
- `retries` counts rounds over the URLs (with one URL, the same as before).
- `web3/aa` reads bundler answers with the exported `rpc` JSON helpers instead of its own copies.

## [0.1.0] - 2026-11-15

### Added
- Core (`github.com/cajulang/web3`): hex helpers, Keccak-256, 256-bit integers (`U256`), units
  (`parse_units`, `format_units`, `parse_ether`, `format_ether`, gwei) and EIP-55 addresses.
- `web3/abi`: Solidity ABI encoding and decoding (functions, results, events, errors, packed),
  JSON and human-readable ABIs.
- `web3/rlp`, `web3/accounts`: secp256k1 keys, EIP-191 and EIP-712 signatures, legacy, EIP-2930,
  EIP-1559 and EIP-7702 transactions and authorizations; byte for byte as viem.
- `web3/hd`: BIP-39 mnemonics, BIP-32 keys, BIP-44 accounts.
- `web3/chains`: Ethereum, Base, Arbitrum, Optimism, Polygon, BNB, Avalanche, Gnosis, Linea,
  Scroll, zkSync and their testnets; any EVM chain with `define_chain`.
- `web3/rpc`: the public JSON-RPC client (balances, calls, gas and fee levels, blocks, receipts,
  logs, simulation, batches, waiting and watching), over fetch on the server and in the browser.
- `web3/wallet`: browser wallets through EIP-1193 and EIP-6963 (MetaMask and others), with
  chain switching, transactions, signatures and events (caju-ui 0.4.1's page bridge).
- `web3/contracts`, `web3/erc`, `web3/multicall`, `web3/ens`: contract reads, simulations and
  writes (with a local account or a wallet), decoded reverts and events; ERC-20 and ERC-721
  helpers; Multicall3 batches; ENS names, reverse lookups, text records and CCIP-read.
- `web3/aa`: account abstraction: ERC-4337 v0.7 and v0.8 user operations, bundler and paymaster
  (ERC-7677) clients, SimpleAccount and EIP-7702 smart accounts, `send_calls`.
