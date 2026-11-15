# web3

Ethereum and other EVM chains for [Caju](https://github.com/cajulang/cajulang) programs, on the
server and in the browser ([caju-ui](https://github.com/cajulang/ui) pages), in the spirit of
[viem](https://viem.sh).

```sh
caju pkg add github.com/cajulang/web3
```

```caju
import "github.com/cajulang/web3"

v := web3.parse_ether("1.5")                         # a U256: 1500000000000000000 wei
print(web3.format_ether(v) + " ETH")
print(web3.checksum_address("0xd8da6bf26964af9d7eed9e03e53415d37aa96045"))
```

The core (`github.com/cajulang/web3`) has hex helpers, Keccak-256, 256-bit integers (`U256`),
units (`parse_units`, `format_units`, `parse_ether`...) and addresses (EIP-55). Bytes are raw
Caju strings; hex strings start with `0x`. Operations that go to the network or to a wallet take a
callback last, `fn(result, err: error)`, and return a function that cancels them; on the server
the callback runs before the call returns, in the browser it runs later.

## abi

`import "github.com/cajulang/web3/abi"`: Solidity ABI encoding and decoding: function calls and
results, events and logs, revert errors, `abi.encodePacked` and deploy data. Everything here
works on raw bytes (Caju `str`); use `web3.to_hex` / `web3.must_hex` at the edges.

```caju
import "github.com/cajulang/web3"
import "github.com/cajulang/web3/abi"

erc20 := abi.must_human([]str{
  "function balanceOf(address owner) view returns (uint256)",
  "function transfer(address to, uint256 amount) returns (bool)",
  "event Transfer(address indexed from, address indexed to, uint256 value)",
  "error ERC20InsufficientBalance(address sender, uint256 balance, uint256 needed)",
})

# calldata for transfer(to, 1 USDC)
data, err := abi.encode_function_data(erc20.function("transfer"), []any{"0xd8dA6BF26964aF9D7eEd9e03E53415D37aA96045", 1000000})
print(web3.to_hex(data))          # 0xa9059cbb...

# the uint256 a call returned
out, err := abi.decode_function_result(erc20.function("balanceOf"), result)
balance := out[0].(*web3.U256)

# a log from eth_getLogs (topics and data as raw bytes)
ev, fields, err := abi.decode_event_log(erc20, &abi.Log{address: addr, topics: topics, data: log_data})
print(ev.name, fields["from"], fields["to"], fields["value"].(*web3.U256).dec())

# filter topics: topic0 + indexed args ("" is a wildcard)
topics, err := abi.encode_event_topics(erc20.event("Transfer"), map[str]any{"to": me})

# why a call reverted
name, args, err := abi.decode_error_result(erc20, revert_data)   # "Error" [msg], "Panic" [code], or a custom error
if name == "Panic" { code, _ := args[0].(*web3.U256).to_int(); print(abi.panic_reason(code)) }
```

ABIs come from JSON (`abi.parse_json(text)`: solc/Etherscan arrays, or an artifact object with an
`"abi"` field) or from human-readable lines (`abi.parse_human(lines)`, viem/ethers style, with
`tuple(...)`/`(...)` components, `indexed`, mutability, `returns (...)` and `struct Name { ...; }`
lines used as types). Look items up with `a.function(name)` (or by signature,
`a.function("safeTransferFrom(address,address,uint256)")` for overloads), `a.event(name)`,
`a.error_(name)` and `a.constructor_()`.

Values in Caju:

| Solidity | Decoded as | Encoded from |
|---|---|---|
| `uint<N>`, `int<N>` | `*web3.U256` (`int` as two's complement: `.signed_dec()`) | `*web3.U256`, `int`, or a decimal/hex `str`; checked against N bits |
| `address` | EIP-55 `str` | any-case hex `str` (validated) |
| `bool` | `bool` | `bool` |
| `bytes`, `bytes<N>`, `string` | `str` (raw bytes) | `str` (raw bytes; `bytes<N>` must have exactly N) |
| `T[]`, `T[k]` | `[]any` | `[]any` (or `[]str`, `[]int`, `[]bool`, `[]*web3.U256`) |
| tuple | `[]any` (positional) | `[]any`, or `map[str]any` by component name |

Other functions: `abi.encode(params, values)` / `abi.decode(params, data)`, `abi.encode_types` /
`abi.decode_types` with type strings (`"uint256"`, `"(address,uint256)[]"`), `abi.signature(item)`,
`abi.selector(item)` (4 bytes), `abi.event_topic(item)` (32 bytes), `abi.decode_function_data(a,
data)` (finds the function by selector), `abi.encode_packed(types, values)`,
`abi.encode_deploy_data(a, bytecode, args)`.

Decoding is strict and never panics on hostile data: offsets and lengths must stay inside the
data, `bool` must be 0 or 1, addresses and `bytes<N>` must be zero-padded, integers must fit their
size (and `int<N>` be sign-extended), and the work spent on offsets that share data is bounded.
Indexed `string`, `bytes`, arrays and tuples in events come back as their 32-byte Keccak hash.

## wallet

`import "github.com/cajulang/web3/wallet"`: the browser's Ethereum wallets (MetaMask and any
EIP-1193 wallet) for [caju-ui](https://github.com/cajulang/ui) programs. The program runs as
WebAssembly in a Web Worker and the wallets live in the page, so everything goes through caju-ui's
page runtime (ui 0.4.0 or newer, which has the `ui.eth.*` bridge): wallets are found with
EIP-6963, requests go to the wallet's `provider.request`, and its events come back as page events
(`ui.on_page_event`).

```caju
import "github.com/cajulang/web3"
import "github.com/cajulang/web3/wallet"

wallet.find_metamask(fn(p: *wallet.ProviderInfo, err: error) {
  if p == nil {
    status.set("install MetaMask")
    return
  }
  w := wallet.connect(p)
  w.request_addresses(fn(addrs: []str, err: error) {
    if wallet.is_user_rejected(err) {
      status.set("canceled")
      return
    }
    base := &wallet.AddChainParams{chain_id: 8453, name: "Base", symbol: "ETH", rpc_urls: []str{"https://mainnet.base.org"}}
    w.switch_chain(8453, base, fn(err: error) {
      tx := &wallet.TxRequest{to_: shop, value: web3.parse_ether("0.01")}
      w.send_transaction(tx, fn(hash: str, err: error) { ... })
    })
  })
  w.on_accounts_changed(fn(addrs: []str) { ... })
  w.on_chain_changed(fn(chain_id: int) { ... })
})
```

Asynchronous calls take a callback last, `fn(result, err: error)`, called once, and return a
`fn()` that cancels the wait (the wallet may still show its prompt; the callback is then not
called). Errors from the wallet are `*wallet.WalletError` with the EIP-1193 code.

| | |
|---|---|
| `wallet.providers(fn([]*ProviderInfo, error))` | the wallets in the page: each EIP-6963 announcement (`id` is its uuid, `name`, `icon` as a data URI, `rdns`) and `window.ethereum` as `id: "injected"` when no announced wallet is it |
| `wallet.find_metamask(fn(*ProviderInfo, error))` | the wallet with `rdns` `"io.metamask"`, or an injected one with `is_metamask`; `nil` when there is none |
| `wallet.connect(p) -> *WalletClient` | a client for that wallet (no prompt yet) |
| `w.request_addresses(cb fn([]str, error))` | `eth_requestAccounts`: asks the user to connect; EIP-55 addresses (kept in `w.addresses`, `w.address()` is the first) |
| `w.get_addresses(cb)` | `eth_accounts`: the addresses already connected, without a prompt |
| `w.get_chain_id(cb fn(int, error))` | `eth_chainId` (kept in `w.chain_id`) |
| `w.switch_chain(chain_id, add, cb fn(error))` | `wallet_switchEthereumChain`; when the wallet does not know the chain (4902) and `add` is not `nil`, `wallet_addEthereumChain` with it and switches again |
| `w.add_chain(params, cb fn(error))` | `wallet_addEthereumChain` (EIP-3085) from `AddChainParams{chain_id, name, symbol, decimals (0: 18), rpc_urls, explorer_urls}` |
| `w.send_transaction(tx, cb fn(hash: str, error))` | `eth_sendTransaction`: `TxRequest{from_, to_, value, data, gas, max_fee_per_gas, max_priority_fee_per_gas, nonce}` (quantities are `*web3.U256`, `data` raw bytes; the wallet gets them hex-encoded; empty fields are left to the wallet, `from_` defaults to `w.address()`). `wallet.tx_params(tx, from)` gives the JSON-RPC object |
| `w.sign_message(address, message, cb fn(sig: str, error))` | `personal_sign` (EIP-191) with the message hex-encoded |
| `w.sign_typed_data_json(address, typed_data_json, cb)` | `eth_signTypedData_v4` (EIP-712) |
| `w.watch_asset(token, symbol, decimals, image, cb fn(bool, error))` | `wallet_watchAsset` (EIP-747, ERC-20): whether the user added it |
| `w.request(method, params_json, cb fn(result_json: str, error))` | any EIP-1193 request, with JSON params (`""` for none) and the result as JSON |
| `w.on_accounts_changed(fn([]str))`, `w.on_chain_changed(fn(int))`, `w.on_disconnect(fn())` | the wallet's events (`accountsChanged`, `chainChanged`, `disconnect`); each returns a function that stops listening |
| `wallet.eip1193_transport(w)` | the wallet as a JSON-RPC transport, `fn(method, params_json, cb fn(result_json, error)) -> fn()`, for the public client's `ClientConfig.transport`: reads go through the wallet's own node |
| `wallet.is_user_rejected(err)`, `wallet.is_unknown_chain(err)`, `wallet.error_code(err)` | code 4001, code 4902, the code of a `*WalletError` (0 otherwise); constants `USER_REJECTED`, `UNAUTHORIZED`, `UNSUPPORTED_METHOD`, `DISCONNECTED`, `UNKNOWN_CHAIN`, `INVALID_PARAMS`, `INTERNAL_ERROR` |

`from` and `to` are keywords in Caju, hence the fields `from_` and `to_`. Outside a page (a native
program, a server render) every call fails with a `WalletError` (code 4900) instead of reaching a
wallet.

How events reach the program: `on_*` asks the page once per client to subscribe
(`ui.eth.subscribe`); the page runtime then sends each of the wallet's events as a page event
(`ui.eth.accountsChanged` and so on, with `{id, data}`), which the client filters by its wallet's
id. The tests (`tests/wallet_test.caju`) run against caju-ui's fake DOM, answering the page calls
as the runtime would.

# web3

## Chains (`web3/chains`)

```caju
import "github.com/cajulang/web3/chains"

c := chains.base
c.id                                  # 8453
c.rpc_urls[0]                         # "https://mainnet.base.org"
c.native_currency.symbol              # "ETH"
c.explorer_tx("0x...")                # "https://basescan.org/tx/0x..."
c.explorer_address("0xd8dA...")       # "https://basescan.org/address/0xd8dA..."
chains.by_id(42161).name              # "Arbitrum One" (nil for an unknown id)
chains.all()                          # every chain below

local := chains.define_chain(31337, "Anvil", "ETH", "http://127.0.0.1:8545")
```

Ready to use: `ethereum`, `sepolia`, `holesky` (retired in 2025, kept for its id), `hoodi`,
`base`, `base_sepolia`, `arbitrum`, `arbitrum_sepolia`, `optimism`, `optimism_sepolia`,
`polygon`, `polygon_amoy`, `bsc`, `bsc_testnet`, `avalanche`, `avalanche_fuji`, `gnosis`,
`linea`, `scroll`, `zksync`.

A `Chain` has `id`, `name`, `network` (a slug), `native_currency` (`name`, `symbol`,
`decimals`), `rpc_urls` (public, keyless, https), `explorer_url`, `explorer_api` (Etherscan's V2
endpoint for the chain), `testnet`, `multicall3`, `ens_registry` and `ens_universal_resolver`
(Ethereum and Sepolia), `entry_point_v07` (ERC-4337) and `block_time_ms`. The public RPC URLs are
rate limited: in production put your provider's URL in `ClientConfig.url`.

## Public client (`web3/rpc`)

```caju
import "github.com/cajulang/web3"
import "github.com/cajulang/web3/chains"
import "github.com/cajulang/web3/rpc"
import "github.com/cajulang/fetch/native"     # in a server / native program only

client := rpc.create_public_client(&rpc.ClientConfig{chain: chains.ethereum})

client.get_balance("0xd8dA6BF26964aF9D7eEd9e03E53415D37aA96045", "latest", fn(v: *web3.U256, err: error) {
  if err != nil { print(err.Error()); return }
  print(web3.format_ether(v) + " ETH")
})

# USDC.decimals()
client.call(&rpc.CallRequest{to_: "0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48", data: web3.must_hex("0x313ce567")}, "latest", fn(out: str, err: error) {
  if data, ok := rpc.revert_data(err); ok { ...decode the revert...; return }
  print(web3.from_bytes(out).dec())             # 6
})
```

**Callbacks.** Every network operation takes `fn(result, err: error)` as its last parameter and
returns a cancel function `fn()` (after a cancel the callback is not called). On the server
(`fetch/native` is synchronous) the callback runs before the call returns; in a caju-ui page it
runs later. The library does not import `fetch/native` (the browser build carries no sockets):
a native program imports it itself.

**Values.** Quantities are `*web3.U256`; chain ids, block numbers, nonces and gas are `int`; hashes
are `"0x"` hex strings; addresses are EIP-55 strings; bytes (call data, return data, code, log
data, topics) are raw bytes. A block is `"latest"`, `"pending"`, `"safe"`, `"finalized"`,
`"earliest"` or `rpc.quantity_int(n)`; `""` means `"latest"`. `from` and `to` are reserved words
in Caju, so the fields are `from_` and `to_`.

**Configuration** (`ClientConfig`): `chain`, `url` (default `chain.rpc_urls[0]`), `transport`
(`fn(method, params_json, cb fn(result_json, err)) -> fn()`, for tests, mocks or an EIP-1193
provider; nil is JSON-RPC over HTTP with fetch), `timeout_ms` (10000), `retries` (2; negative for
none: only network errors, timeouts, 429 and 5xx are retried, after 200, 400, 800... ms),
`headers`, `poll_ms` (default from the chain's block time, 1 to 4 s), `http_client` (a
`*fetch.Client` of your own, for interceptors).

**Errors.** A JSON-RPC error is an `*rpc.RpcError` (`code`, `message`, `data`); a revert carries
its revert data, which `rpc.revert_data(err)` returns as bytes (`rpc.is_revert(err)` tells a
revert). A status other than 2xx is an `*rpc.HttpError` (`status`, `url`, `body`); no response at
all is fetch's `*fetch.Error` (`"network"`, `"timeout"`). "Not found" is not an error: the
lookups below give `nil, nil`.

| method | result |
|---|---|
| `request(method, params []any, cb)` | the result as JSON text (`*web3.U256` params become hex quantities) |
| `batch(calls []*BatchCall, cb)` | one HTTP request; each `BatchCall` gets its `result` (JSON) or `error` |
| `chain_id`, `block_number` | `int` |
| `get_balance(addr, block)` | `*web3.U256` |
| `get_transaction_count(addr, block)` | `int` |
| `get_code(addr, block)`, `get_storage_at(addr, slot, block)` | raw bytes |
| `call(req *CallRequest, block)` | raw bytes; a revert is an `*RpcError` with its data |
| `estimate_gas(req)`, `gas_price`, `max_priority_fee_per_gas` | `*web3.U256` |
| `fee_history(blocks, newest, percentiles)` | `*FeeHistory` |
| `estimate_fees_per_gas` | `*Fees` (`base_fee`, `max_fee_per_gas`, `max_priority_fee_per_gas`) |
| `estimate_fee_levels` | `slow, average, fast *Fees` |
| `get_block(block or hash, full_txs)` | `*Block` (nil, nil if missing) |
| `get_transaction(hash)` | `*TransactionInfo` (nil, nil if unknown) |
| `get_transaction_receipt(hash)` | `*Receipt` (nil, nil while pending) |
| `wait_for_transaction_receipt(hash, *WaitOptions)` | `*Receipt` |
| `send_raw_transaction(raw)` | the hash |
| `get_logs(*LogFilter)` | `[]*Log` |
| `simulate_calls(calls, block)` | `[]*SimulateResult` (eth_simulateV1: data, status, gas, logs, error per call) |
| `create_access_list(req, block)` | `*AccessListResult` |
| `get_proof(addr, keys, block)` | `*Proof` |

`CallRequest` has `from_`, `to_`, `data` (bytes), `value`, `gas`, `gas_price`,
`max_fee_per_gas`, `max_priority_fee_per_gas`. `LogFilter` has `address []str`, `topics [][]str`
(raw 32-byte topics; a nil position matches anything, several values at one position match any
of them), `from_block`, `to_block` and `block_hash`.

**Fees.** `max_fee_per_gas = 2 * base_fee + priority`, room for the base fee to double.
`estimate_fees_per_gas` uses the latest block's base fee and `eth_maxPriorityFeePerGas` (gas price
minus base fee where that method is missing); on a chain without EIP-1559 it gives the gas price
with `base_fee` nil. `estimate_fee_levels` asks `eth_feeHistory` for the last 20 blocks at the
10th, 50th and 90th percentiles and uses the mean tip of each (empty blocks' zero tips left out)
on top of the next block's base fee; `rpc.fee_levels(history)` does the same math on a history
you have.

**Waiting and watching** (polling):

```caju
client.wait_for_transaction_receipt(hash, &rpc.WaitOptions{confirmations: 2}, fn(r: *rpc.Receipt, err: error) { ... })

opts := &rpc.WatchOptions{poll_ms: 2000}
stop := client.watch_blocks(opts, fn(n: int) { print(n) }, fn(err: error) { print(err.Error()) })
client.watch_event(&rpc.LogFilter{address: []str{token}, topics: [][]str{[]str{transfer_topic}}}, opts, fn(logs: []*rpc.Log) { ... }, nil)
```

- In a caju-ui page they poll on page timers (`ui.set_interval`) and return at once; the returned
  function (or `opts.stop()`) stops them.
- On the server, inside a `std/http` server, pass `every: http.every` in `WatchOptions`: they poll
  in the server's event loop, between requests, and return at once.
- On the server without `every` they **block**, polling in a loop with `sleep`, until
  `opts.stop()` is called from a callback or `max_polls` is reached.
  `wait_for_transaction_receipt` blocks too until the receipt comes (or `timeout_ms`, 180 s by
  default, counted in polls).

`watch_blocks` reports the latest block number at each poll (blocks that come between two polls
are not reported one by one); `watch_event` asks `eth_getLogs` for the blocks after the last
range it saw (starting after the current block, or at `filter.from_block`).
