# Changelog

All notable changes to `web3`, the EVM library for Caju. The format follows
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/) and the project uses
[Semantic Versioning](https://semver.org/).

## [Unreleased]

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
