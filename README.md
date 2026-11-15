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

| Package | What |
|---|---|
| `github.com/cajulang/web3` | hex, Keccak-256, `U256`, units, EIP-55 addresses |
| `web3/abi` | ABI encoding and decoding |
| `web3/rlp`, `web3/accounts`, `web3/hd` | keys, signatures (EIP-191, EIP-712), transactions (legacy to EIP-7702), mnemonics |
| `web3/chains`, `web3/rpc` | chains and the public JSON-RPC client |
| `web3/wallet` | browser wallets (EIP-1193, EIP-6963), in caju-ui pages |
| `web3/contracts`, `web3/erc`, `web3/multicall`, `web3/ens` | contracts, ERC-20/721, Multicall3, ENS |
| `web3/aa` | account abstraction (ERC-4337, EIP-7702 smart accounts) |

On the server, a program that uses `web3/rpc` imports `github.com/cajulang/fetch/native` once
(the browser's program does not); `web3/wallet` works only inside a page. Requires caju 0.9.0.

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

**Configuration** (`ClientConfig`): `chain`, `url` and `urls` (see **Fallback**; by default all
of `chain.rpc_urls`), `transport` (`fn(method, params_json, cb fn(result_json, err)) -> fn()`, for
tests, mocks or an EIP-1193 provider; nil is JSON-RPC over HTTP with fetch), `timeout_ms` (10000,
per HTTP request), `retries` (2; negative for none: only network errors, timeouts, 429 and 5xx
are retried, after 200, 400, 800... ms), `headers`, `poll_ms` (default from the chain's block
time, 1 to 4 s), `http_client` (a `*fetch.Client` of your own, for interceptors), `ccip_read`
(`*rpc.CcipRead`: which CCIP-read gateways ENS may reach, see `web3/ens`).

**Fallback.** A client over HTTP has a list of URLs, `client.urls`: `url` first, then `urls`
(duplicates dropped); with neither, all of the chain's `rpc_urls`; `url` alone is that URL only.
A request goes to the URL that answered last (`client.url`, `client.current_url()`). On a network
error, a timeout, 429, 5xx or another HTTP failure without a JSON-RPC error in it (403, an HTML
page) it moves on to the next URL at once, wrapping around; a JSON-RPC error (a revert, invalid
params) is an answer and is returned as it is. When every URL has failed with a retryable error,
the round starts over after the pause, up to `retries` more rounds; the callback gets the last
error. The URL that answers becomes the one used next (sticky). Batches fall back the same way.

```caju
client := rpc.create_public_client(&rpc.ClientConfig{chain: chains.base})   # mainnet.base.org, then base-rpc.publicnode.com
mine := rpc.create_public_client(&rpc.ClientConfig{chain: chains.ethereum, url: my_provider, urls: chains.ethereum.rpc_urls})

client.rank_urls(fn(order: []str, err: error) { ... })   # one eth_blockNumber per URL: fastest first, dead ones last

# transports: the browser wallet's node first, then HTTP (viem's fallback)
t := rpc.fallback([]fn(str, str, fn(str, error)) -> fn(){wallet.eip1193_transport(w), rpc.http_transport(chains.base.rpc_urls[0], nil)})
via_wallet := rpc.create_public_client(&rpc.ClientConfig{chain: chains.base, transport: t})
```

`rank_urls(cb)` measures each URL once (no retries, one after the other) and reorders
`client.urls` by latency, the ones that failed last; it changes nothing if none answered.
`rpc.fallback(transports)` is the same rotation over transports (sticky; the client's `retries`
repeat the whole round); `rpc.http_transport(url, config)` is one URL as a transport.

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
| `wait_for_transaction_receipt(hash, *WaitOptions)` | `*Receipt` (the replacement's, if it was replaced) |
| `rank_urls` | the URLs, fastest first |
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

**Replaced transactions.** A wallet can speed up or cancel a pending transaction by sending
another one with the same nonce. `wait_for_transaction_receipt` notices: while there is no
receipt it fetches the transaction (`get_transaction`, or give it as `WaitOptions.transaction`)
and checks the sender's nonce (`eth_getTransactionCount`); once the nonce is used, it searches the
latest blocks (up to `rpc.REPLACEMENT_SCAN_BLOCKS`, 64) for the transaction with that sender and
nonce. If it has another hash, the wait goes on with that transaction, and when its receipt has
the confirmations `WaitOptions.on_replaced` gets a `*rpc.Replacement` just before the callback
gets that receipt:

```caju
client.wait_for_transaction_receipt(hash, &rpc.WaitOptions{on_replaced: fn(r: *rpc.Replacement) {
  print(r.reason + ": " + r.transaction_hash)    # "repriced" | "cancelled" | "replaced"
}}, fn(rc: *rpc.Receipt, err: error) { ... })     # rc: the replacement's receipt
```

`Replacement` has `reason` (`"repriced"`: the same call with other fees; `"cancelled"`: 0 to the
sender itself; `"replaced"`: anything else, as viem's `onReplaced`), `transaction_hash`,
`receipt`, `transaction` (the replacement) and `replaced` (the original). If the nonce is used but
the replacement is not found in three polls, the wait fails with an error saying so.

`watch_blocks` reports the latest block number at each poll (blocks that come between two polls
are not reported one by one); `watch_event` asks `eth_getLogs` for the blocks after the last
range it saw (starting after the current block, or at `filter.from_block`).

## web3/rlp

Recursive Length Prefix, the encoding of Ethereum transactions.

```caju
import "github.com/cajulang/web3/rlp"

raw := rlp.encode([]any{"cat", "dog"})       # "\xc8\x83cat\x83dog"
rlp.encode(1024)                              # integers: int (>= 0) or *web3.U256
item, err := rlp.decode(raw)                  # a str (bytes) or []any (list)
list, err := rlp.as_list(item)
n, err := rlp.as_int(list[0])                 # also as_uint (U256) and as_bytes
```

| Function | |
|---|---|
| `encode(v: any) -> str` | `str` bytes, `[]any` / `[]str` lists, `int` and `*web3.U256` integers |
| `encode_bytes(b)`, `encode_uint(u)`, `encode_int(n)`, `encode_list_raw(payload)` | pieces |
| `decode(data) -> (any, error)` | strict: canonical sizes, no leading zeros in lengths, no wrapped single bytes, no trailing bytes |
| `as_bytes`, `as_list`, `as_uint`, `as_int` | read decoded items (integers: no leading zeros) |

## web3/accounts

Local accounts on secp256k1: keys, ECDSA signatures, EIP-191 messages, EIP-712 typed data,
transactions (legacy/EIP-155, EIP-2930, EIP-1559, EIP-7702) and EIP-7702 authorizations.

```caju
import "github.com/cajulang/web3"
import "github.com/cajulang/web3/accounts"

a, err := accounts.private_key_to_account("0xac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80")
a.address                                   # "0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266"

sig := a.sign_message("hello world")       # EIP-191: "0x" + 65 bytes (r || s || v)
addr, err := accounts.recover_message_address("hello world", sig)

tx := &accounts.Transaction{type_: 2, chain_id: 1, nonce: 0, gas: 21000,
  max_fee_per_gas: web3.must_u256("30000000000"), max_priority_fee_per_gas: web3.must_u256("1000000000"),
  to_: "0x70997970C51812dc3A010C7d01b50e0d17dc79C8", value: web3.parse_ether("0.1")}
raw, err := a.sign_transaction(tx)        # raw bytes; web3.to_hex(raw) for eth_sendRawTransaction
accounts.transaction_hash(raw)              # "0x..."
```

- **Keys**: `private_key_to_account(key)` (32 raw bytes or hex), `generate_private_key()` ("0x"
  hex), `public_key_to_address(pub)` (65, 64 or 33 bytes, raw or hex; "" when invalid),
  `private_key_to_public_key(key, compressed)`, `compress_public_key`, `decompress_public_key`,
  `curve_order()`. `Account { address, public_key (65 bytes), private_key (32 bytes) }`.
- **ECDSA**: `a.sign_hash(h) -> (*Signature, error)` (h: 32 bytes, raw or hex; nothing is
  prefixed), `recover_address(h, sig)`, `recover_public_key(h, sig)`, `ecrecover(h, r, s, v)`
  (v 0/1, 27/28 or EIP-155), `verify_hash(address, h, sig)`. `Signature { r, s: *web3.U256,
  y_parity, v }`; `serialize_signature(sig)` (65 bytes, v 27/28), `parse_signature(b)` (65 bytes
  or 64-byte EIP-2098 compact, raw or hex), `compact_signature(sig)`.
- **EIP-191**: `hash_message(msg)`, `a.sign_message(msg)`, `recover_message_address(msg, sig)`,
  `verify_message(address, msg, sig)`.
- **EIP-712**: `TypedData { domain: map[str]any, types: map[str][]*TypedField, primary_type,
  message: map[str]any }`, `hash_typed_data(td)`, `domain_separator(td)`,
  `a.sign_typed_data(td)`, `recover_typed_data_address(td, sig)`, `verify_typed_data`. The
  `EIP712Domain` type is derived from the domain keys present (unless `types` has one). Values:
  `uintN`/`intN` as `*web3.U256`, `int` or a decimal/hex `str`; `address` as a hex `str`; `bool`;
  `string`; `bytes` as raw bytes; `bytesN` raw (N bytes) or hex; arrays `[]any`; structs
  `map[str]any`.
- **Transactions**: `Transaction { type_, chain_id, nonce, gas, gas_price, max_fee_per_gas,
  max_priority_fee_per_gas, to_, value, data, access_list, authorization_list }` (`to_` because
  `to` is a Caju keyword; `""` creates a contract; nil amounts are zero; `data` is raw bytes).
  `serialize_transaction(tx, sig) -> (str, error)` (sig nil: the payload that is signed),
  `transaction_signing_hash(tx)`, `a.sign_transaction(tx)`, `parse_transaction(raw) ->
  (*Transaction, *Signature, error)` (the signature is nil for unsigned payloads),
  `recover_transaction_address(raw)`, `transaction_hash(raw)`. Legacy transactions with a
  chain id use EIP-155 (`v = chain_id * 2 + 35 + y`). Access list storage keys are "0x" hex
  (raw 32 bytes accepted).
- **EIP-7702**: `Authorization { chain_id, address, nonce, y_parity, r, s }`,
  `a.sign_authorization(chain_id, contract_address, nonce) -> (*Authorization, error)`,
  `hash_authorization(...)`, `recover_authorization_address(auth)`.

Implementation: field elements in ten 26-bit limbs with an unrolled multiplication and the
special reduction of p = 2^256 - 2^32 - 977; complete projective formulas (Renes-Costello-Batina,
no special cases); k*G through a precomputed table of 64 4-bit windows read entirely with masks
(the same operations for every secret scalar); RFC 6979 nonces (the same signatures as viem and
ethers); low s (EIP-2). On a typical x86-64 machine: about 1 ms to sign, 2 ms to recover, 0.5 ms
from key to address, plus 5 ms once to build the table.

## web3/hd

BIP-39 mnemonics (English), BIP-32 hierarchical keys and BIP-44 Ethereum accounts.

```caju
import "github.com/cajulang/web3/hd"

m, err := hd.generate_mnemonic(128)                  # 12 words (128..256 bits: 12..24 words)
err = hd.validate_mnemonic(m)                         # nil, or why it is invalid (word, checksum)
acct, err := hd.mnemonic_to_account(m, "", 0)         # m/44'/60'/0'/0/0, like MetaMask/Hardhat/viem

seed := hd.mnemonic_to_seed(m, "passphrase")          # 64 bytes (PBKDF2-HMAC-SHA512, 2048 rounds)
k, err := hd.master_from_seed(seed).derive("m/44'/60'/0'/0/3")
k.xprv()                                              # "xprv..."; k.xpub() "xpub..."
a3, err := k.account()
```

| Function | |
|---|---|
| `generate_mnemonic(bits)`, `entropy_to_mnemonic(e)`, `mnemonic_to_entropy(m)`, `validate_mnemonic(m)` | BIP-39 with the English list (`wordlist()`) |
| `mnemonic_to_seed(m, passphrase)`, `pbkdf2_sha512(...)` | the seed |
| `master_from_seed(seed) -> *HDKey` | `HDKey { private_key, chain_code, public_key (33 bytes, compressed), depth, index, parent_fingerprint }` |
| `k.derive("m/44'/60'/0'/0/0")`, `k.child(index)` | hardened with `'` or `h` (or `index >= hd.HARDENED`); paths are relative to `k` |
| `k.xprv()`, `k.xpub()`, `k.fingerprint()`, `k.account()` | serialization (Base58Check) and the Ethereum account |
| `mnemonic_to_account(m, passphrase, index)` | BIP-44 `m/44'/60'/0'/0/index` |
| `base58check(payload)`, `ripemd160(data)` | helpers |

The mnemonic is used with its whitespace trimmed and collapsed; BIP-39 also asks for Unicode
NFKD, which leaves ASCII unchanged: normalize non-ASCII passphrases before calling. Deriving a
seed takes about 35 ms. Not included yet: parsing xprv/xpub strings and public-only derivation.

## web3/aa

Account abstraction: ERC-4337 user operations (EntryPoint v0.7 and v0.8), bundler and paymaster
(ERC-7677) clients, and smart accounts: eth-infinitism's SimpleAccount and EIP-7702 "smart EOAs"
delegated to Simple7702Account.

```caju
import "github.com/cajulang/web3"
import "github.com/cajulang/web3/aa"
import "github.com/cajulang/web3/accounts"
import "github.com/cajulang/web3/chains"
import "github.com/cajulang/web3/rpc"

owner, err := accounts.private_key_to_account(key)
public := rpc.create_public_client(&rpc.ClientConfig{chain: chains.sepolia})
bundler := aa.create_bundler_client(&rpc.ClientConfig{url: "https://api.pimlico.io/v2/sepolia/rpc?apikey=..."})

acct := aa.simple_account(owner, public, web3.zero())   # SimpleAccount v0.7, salt 0
calls := []*aa.Call{&aa.Call{to_: "0xd8dA6BF26964aF9D7eEd9e03E53415D37aA96045", value: web3.parse_ether("0.001")}}
opts := &aa.SendOptions{paymaster: bundler.paymaster()}  # nil: the account pays its own gas
aa.send_calls(acct, bundler, public, calls, opts, fn(hash: str, err: error) {
  if err != nil { print(err.Error()); return }
  bundler.wait_for_user_operation_receipt(hash, nil, fn(r: *aa.UserOperationReceipt, err: error) {
    if err == nil { print(r.success, r.receipt.transaction_hash) }
  })
})
```

`send_calls` fills the user operation: the sender's address, the factory when it has no code yet
(or, for an EIP-7702 account, the authorization when the EOA is not delegated yet), the nonce
(`EntryPoint.getNonce`), the fees (`public.estimate_fees_per_gas`), the call data, paymaster stub
data, the gas (the bundler's `eth_estimateUserOperationGas`, with a dummy signature), the final
paymaster data; then it signs and sends. `prepare_user_operation(...)` stops before signing
(`cb(op, chain_id, err)`); `sign_user_operation(acct, op, chain_id)` signs.

- **User operations**: `UserOperation { sender, nonce, factory, factory_data, call_data,
  call_gas_limit, verification_gas_limit, pre_verification_gas, max_fee_per_gas,
  max_priority_fee_per_gas, paymaster, paymaster_verification_gas_limit,
  paymaster_post_op_gas_limit, paymaster_data, signature, eip7702_auth }` (byte fields raw,
  amounts `*web3.U256`, nil is zero). `pack(op) -> *PackedUserOperation` (initCode,
  accountGasLimits, gasFees, paymasterAndData as the EntryPoint takes them) and `unpack(p)`.
  `user_operation_hash(op, entry_point, chain_id)` ("0x" hex): the EntryPoint v0.7 formula, or
  `user_operation_hash_v08` (EIP-712, with the EIP-7702 initCode rule) when `entry_point` is
  `ENTRY_POINT_V08`. `get_user_operation_hash(public, entry_point, op, cb)` asks the EntryPoint
  (eth_call), `get_nonce(public, entry_point, sender, key, cb)`.
- **JSON-RPC form** (ERC-7769, v0.7 fields, hex quantities): `user_operation_json(op) ->
  map[str]any` (factory and paymaster fields only when set; `eip7702Auth` when set),
  `parse_user_operation(v *json.Value)`.
- **Bundler**: `create_bundler_client(&rpc.ClientConfig{url: ...})` (or `transport:` for mocks).
  `b.send_user_operation(op, entry_point, cb(hash, err))`,
  `b.estimate_user_operation_gas(op, entry_point, cb(*GasEstimate, err))`,
  `b.get_user_operation_receipt(hash, cb(*UserOperationReceipt, err))` (nil while pending),
  `b.get_user_operation_by_hash(hash, cb(*UserOperationInfo, err))`,
  `b.supported_entry_points(cb)`, `b.chain_id(cb)`,
  `b.wait_for_user_operation_receipt(hash, opts *rpc.WaitOptions, cb)` (polls like
  `rpc.wait_for_transaction_receipt`; 180 s timeout by default). `entry_point` "" is v0.7.
- **Paymasters** (ERC-7677): `create_paymaster_client(cfg)` or `b.paymaster()`;
  `p.get_paymaster_stub_data(op, entry_point, chain_id, context, cb(*PaymasterResult, err))`,
  `p.get_paymaster_data(...)`, `apply_paymaster(op, r)`. `context` is the service's options
  (`map[str]any{"sponsorshipPolicyId": ...}`; nil is `{}`).
- **Smart accounts**: the `SmartAccount` interface (`entry_point()`, `address(cb)`,
  `factory_args() -> (factory, data, err)`, `encode_calls(calls)`,
  `sign_user_operation_hash(h)`, `dummy_signature()`, `delegation()`,
  `sign_authorization(chain_id, nonce)`); implement it to add Safe, Kernel, Coinbase Smart
  Wallet and others.
  - `simple_account(owner, public, salt) -> *SimpleAccount`: eth-infinitism SimpleAccount v0.7
    (factory `SIMPLE_ACCOUNT_FACTORY_V07`, EntryPoint v0.7). The address is computed offline with
    CREATE2 (`simple_account_address(owner, salt)`, the same as the factory's `getAddress`); with
    another `factory`, it is read with `getAddress` by eth_call. One call is `execute`, several
    `executeBatch(address[], uint256[], bytes[])`; signatures are EIP-191 of the hash.
  - `simple_7702_account(owner, public) -> *Simple7702Account`: the owner's EOA delegated with
    EIP-7702 to `SIMPLE_7702_ACCOUNT_V08` (eth-infinitism Simple7702Account, EntryPoint v0.8;
    `implementation` can name another contract with the same interface). Through a bundler,
    `send_calls` adds `eip7702Auth` and the `0x7702` factory marker while the EOA is not
    delegated. Without a bundler, `a.send_transaction(calls, cb(tx_hash, err))` sends a type-4
    transaction from the EOA to itself (it delegates and runs `executeBatch` at once; type 2 once
    delegated), and `a.build_transaction(...)` builds it offline. `encode_7702_calls(calls)`,
    `delegation_of(code)` (the delegate of a `0xef0100 || address` code).
- **Addresses** (checked on Ethereum, Base, Sepolia and Base Sepolia): `ENTRY_POINT_V07`
  `0x0000000071727De22E5E9d8BAf0edAc6f37da032`, `ENTRY_POINT_V08`
  `0x4337084D9E255Ff0702461CF8895CE9E3b5Ff108`, `SIMPLE_ACCOUNT_FACTORY_V07`
  `0x91E60e0613810449d098b0b5Ec8b51A0FE8c8985` (implementation
  `0x68641DE71cfEa5a5d0D29712449Ee254bb1400C2`), `SIMPLE_7702_ACCOUNT_V08`
  `0x4Cd241E8d1510e30b2076397afc7508Ae59C66c9`.

Not included yet: EntryPoint v0.6, other accounts (Safe, Kernel, Light, Coinbase Smart Wallet),
bundler-specific gas price methods (`pimlico_getUserOperationGasPrice`), state overrides in gas
estimation, ERC-1271/ERC-6492 signature verification and ERC-5792 `wallet_sendCalls`.

## web3/contracts

A contract through its ABI: reads (`eth_call`), simulations, gas estimates, writes from a local
account or from the browser wallet, and events. Reverts come back as a `*contracts.ContractError`
decoded with the contract's ABI.

```caju
import "github.com/cajulang/web3"
import "github.com/cajulang/web3/contracts"
import "github.com/cajulang/web3/erc"

usdc := contracts.contract("0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48", erc.ERC20, client)

usdc.read("balanceOf", []any{owner}, fn(vals: []any, err: error) {
  print(web3.format_units(vals[0].(*web3.U256), 6))
})

# would it work? (eth_call from an account)
usdc.simulate("transfer", []any{to, amount}, &contracts.WriteOptions{from_: me}, fn(vals: []any, err: error) {
  if ce, ok := err.(*contracts.ContractError); ok {
    print(ce.message)          # ERC20InsufficientBalance(0xf39F..., 100, 250)
    return
  }
})

# send it: a local account (server) signs it; the browser wallet signs it in the page
contracts.write_with_account(usdc, acct, "transfer", []any{to, amount}, nil, fn(hash: str, err: error) { ... })
contracts.write_with_wallet(usdc, w, "transfer", []any{to, amount}, nil, fn(hash: str, err: error) { ... })

# native coin
contracts.send_value_with_account(client, acct, to, web3.parse_ether("0.1"), fn(hash: str, err: error) { ... })
w.send_transaction(&wallet.TxRequest{to_: to, value: web3.parse_ether("0.1")}, fn(hash: str, err: error) { ... })   # browser

# events
contracts.get_events(usdc, "Transfer", map[str]any{"from": owner}, rpc.quantity_int(19000000), "latest", fn(logs: []*contracts.DecodedLog, err: error) {
  for _, l := range logs { print(l.args["to"], l.args["value"].(*web3.U256).dec(), l.log.transaction_hash) }
})
stop := contracts.watch_contract_event(usdc, "Transfer", nil, &rpc.WatchOptions{}, fn(logs: []*contracts.DecodedLog) { ... }, nil)
```

| | |
|---|---|
| `contract(address, a *abi.Abi, client) -> *Contract` | `Contract { address, abi, client }` |
| `c.read(fn_name, args, cb fn([]any, error))` | `eth_call` at `"latest"` and the decoded return values |
| `c.simulate(fn_name, args, opts *WriteOptions, cb fn([]any, error))` | `eth_call` with `from_`, `value`, `gas` (and `opts.block`) |
| `c.estimate_gas(fn_name, args, opts, cb fn(*web3.U256, error))` | `eth_estimateGas` |
| `c.encode_write(fn_name, args) -> (str, error)` | the calldata (raw bytes) |
| `c.function(name, nargs) -> (*abi.Item, error)` | overloads: a signature (`"get(uint256,bool)"`), or the first with that many inputs |
| `write_with_account(c, acct, fn_name, args, opts, cb fn(hash, error))` | fills, signs, `eth_sendRawTransaction` |
| `write_with_wallet(c, w, fn_name, args, opts, cb fn(hash, error))` | `wallet.send_transaction` (the wallet fills the rest) |
| `send_transaction_with_account(client, acct, to, data, opts, cb)` | any transaction from a local account (calldata from `erc.erc20_transfer_data`...) |
| `send_value_with_account(client, acct, to, value, cb)` | a native transfer |
| `get_events(c, event_name, args, from_block, to_block, cb fn([]*DecodedLog, error))` | `eth_getLogs` filtered by indexed args by name (missing: any); `""` event: all of the contract's |
| `watch_contract_event(c, event_name, args, opts *rpc.WatchOptions, on_logs, on_error)` | polling, as `rpc.watch_event` |
| `decode_revert(a, data) -> *ContractError`, `contract_error(a, fn_name, address, err) -> error` | revert decoding for your own calls |

`WriteOptions { from_, value, gas, max_fee_per_gas, max_priority_fee_per_gas, gas_price, nonce,
block }`, all optional. A write from a local account fills in what is missing: the chain id (the
client's chain, or `eth_chainId`), the nonce (`eth_getTransactionCount(..., "pending")`), the gas
(`eth_estimateGas` + 20%) and EIP-1559 fees (`rpc.estimate_fees_per_gas`: twice the base fee plus
the node's tip); on a chain without a base fee, or with `gas_price` set, it sends a legacy
transaction. A revert while estimating gas is a `ContractError` too, and nothing is sent.

`ContractError { name, args, data, message, function, address, cause }`: `name` is the custom
error (`"ERC20InsufficientBalance"`), `"Error"` (a `require` message: `Error("not owner")`) or
`"Panic"` (`Panic(0x11: arithmetic underflow or overflow)`), `""` when the data is missing or
unknown; `data` the revert data (raw bytes); `cause` the node's `*rpc.RpcError`. Errors that are
not reverts (network, node) are passed through as they are. A read that returns nothing (no
contract at that address on this chain) is an error saying so.

## web3/erc

Ready ABIs and helpers for ERC-20 tokens and ERC-721 collections. `erc.ERC20` and `erc.ERC721` are
parsed `*abi.Abi` (`ERC20_ABI` / `ERC721_ABI` are the human-readable lines), with OpenZeppelin 5's
custom errors (`ERC20InsufficientBalance`, `ERC20InsufficientAllowance`, `ERC721NonexistentToken`,
`ERC721IncorrectOwner`...).

```caju
import "github.com/cajulang/web3/erc"

erc.erc20_metadata(client, usdc, fn(m: *erc.TokenMeta, err: error) {
  print(m.name, m.symbol, m.decimals)      # USD Coin USDC 6 (one eth_call, through Multicall3)
})
erc.erc20_balance_of(client, usdc, owner, fn(v: *web3.U256, err: error) { ... })

data, err := erc.erc20_transfer_data(to, web3.must_u256("2500000"))       # 2.5 USDC
w.send_transaction(&wallet.TxRequest{to_: usdc, data: data}, cb)           # browser wallet
contracts.send_transaction_with_account(client, acct, usdc, data, nil, cb) # local account
```

| ERC-20 | |
|---|---|
| `erc20_balance_of(client, token, owner, cb fn(*web3.U256, error))` | |
| `erc20_allowance(client, token, owner, spender, cb)`, `erc20_total_supply(client, token, cb)` | `*web3.U256` |
| `erc20_decimals(client, token, cb fn(int, error))` | |
| `erc20_name`, `erc20_symbol(client, token, cb fn(str, error))` | `string`, or `bytes32` for old tokens (MKR, SAI) |
| `erc20_metadata(client, token, cb fn(*TokenMeta, error))` | `TokenMeta { address, name, symbol, decimals }` in one multicall (three calls without Multicall3); an error if `decimals()` fails |
| `erc20_transfer_data(to, amount)`, `erc20_approve_data(spender, amount)`, `erc20_transfer_from_data(from_, to, amount)` | `(calldata, error)` |
| `erc20(client, token) -> *contracts.Contract` | for anything else (`get_events(..., "Transfer", ...)`) |

| ERC-721 | |
|---|---|
| `erc721_owner_of(client, token, id, cb fn(str, error))` | a nonexistent token reverts with `ERC721NonexistentToken(id)` |
| `erc721_balance_of(client, token, owner, cb fn(*web3.U256, error))` | |
| `erc721_token_uri(client, token, id, cb fn(str, error))` | as the contract answers it (`ipfs://`, `https://`, `data:`) |
| `erc721_transfer_from_data(from_, to, id)`, `erc721_safe_transfer_from_data(from_, to, id, data)` | `data` `""`: the 3-argument overload |
| `erc721_approve_data(to, id)`, `erc721_set_approval_for_all_data(operator, approved)` | |
| `erc721(client, token) -> *contracts.Contract` | |

Reads are `contracts` reads: a revert is a `*contracts.ContractError`.

## web3/multicall

Many reads in one `eth_call` through [Multicall3](https://www.multicall3.com) (`aggregate3`, with
`allowFailure` per call), at the chain's `multicall3` address (an error on a chain without one).
Long lists go in chunks of `multicall.CHUNK_SIZE` (500) calls, one `eth_call` after the other.

```caju
import "github.com/cajulang/web3/multicall"

multicall.erc20_balances(client, owner, []str{usdc, usdt, dai}, fn(bs: []*web3.U256, err: error) {
  # one per token, nil where the call failed (not a token, no contract)
})
multicall.get_eth_balances(client, []str{a, b}, fn(bs: []*web3.U256, err: error) { ... })
# native + ERC-20 balances in one aggregate3 (the same block for all)
multicall.balances(client, owner, []str{usdc, dai}, fn(native: *web3.U256, tokens: []*web3.U256, err: error) { ... })

c1, _ := multicall.call_of(usdc, erc.ERC20.function("symbol"), []any{}, true)
c2 := &multicall.Call{target: dai, data: calldata, allow_failure: false}
multicall.aggregate(client, []*multicall.Call{c1, c2}, "latest", fn(rs: []*multicall.Result, err: error) {
  vals, err := multicall.decode_result(erc.ERC20.function("symbol"), rs[0])
})
```

`Call { target, data, allow_failure }`, `Result { success, data }` (raw bytes; the revert data
when it failed). A call with `allow_failure` false that fails makes the whole `eth_call` revert.
Also: `aggregate_at(client, address, calls, block, chunk, cb)` (another Multicall3, another chunk
size), `address_of(client)`, `encode_aggregate3(calls)`, `decode_aggregate3(data)` and the ABI
(`MULTICALL3_ABI`, `MULTICALL3`).

## web3/ens

ENS names on Ethereum and Sepolia, through the ENS Universal Resolver (the chain's
`ens_universal_resolver`): forward and reverse resolution, text records, wildcards and offchain
names. The CCIP-read round trip (EIP-3668) is done here, including the local batch gateway
(`x-batch-gateway:true`, ENSIP-21), so names like `jesse.base.eth` resolve too. Gateways are
reached with fetch, through the client's `http_client` transport but without its headers.

**Gateways and CSP.** A page with a strict Content-Security-Policy can only `connect-src` to the
hosts it lists; an offchain name's gateway is somewhere else. `ClientConfig.ccip_read` decides
which gateways may be fetched: `&rpc.CcipRead{disabled: true}` fetches none, and
`&rpc.CcipRead{allowed_hosts: []str{"ccip.ens.xyz", "*.example.com"}}` only those hosts
(`*.` for subdomains). Gateway URLs that are not allowed are skipped without a request; when no
allowed one is left the lookup fails with an `*ens.CcipBlockedError` (`ens.is_ccip_blocked(err)`)
naming the hosts, instead of a CSP violation in the console. Onchain names are not affected.

```caju
client := rpc.create_public_client(&rpc.ClientConfig{chain: chains.ethereum, ccip_read: &rpc.CcipRead{disabled: true}})
ens.resolve_address(client, "jesse.base.eth", fn(a: str, err: error) {
  if ens.is_ccip_blocked(err) { print("offchain names are not supported here") }
})
```

```caju
import "github.com/cajulang/web3/ens"

ens.resolve_address(client, "vitalik.eth", fn(addr: str, err: error) { ... })     # "0xd8dA..."; "" if none
ens.lookup_name(client, "0xd8dA6BF26964aF9D7eEd9e03E53415D37aA96045", fn(name: str, err: error) { ... })   # "vitalik.eth"
ens.get_text(client, "vitalik.eth", "url", fn(v: str, err: error) { ... })
ens.get_avatar_url(client, "vitalik.eth", fn(v: str, err: error) { ... })          # the record as it is

web3.to_hex(ens.namehash("foo.eth"))   # "0xde9b09fd7c5f901e23a3f19fecc54828e9c848539801e86591bd9801b019f84f"
```

| | |
|---|---|
| `normalize(name) -> (str, error)` | ASCII names only (see below) |
| `namehash(name)`, `labelhash(label)` | raw 32 bytes; a `[64 hex digits]` label is its hash |
| `dns_encode(name)` | DNS wire format (labels over 255 bytes as `[labelhash]`) |
| `resolve_address(client, name, cb fn(str, error))` | `addr(bytes32)`; `""` without a resolver or an address |
| `lookup_name(client, address, cb fn(str, error))` | `name()` of `<addr>.addr.reverse`, kept only if it resolves back to the address |
| `get_text(client, name, key, cb fn(str, error))`, `get_avatar_url(client, name, cb)` | `text(bytes32,string)`; `""` when unset |

"Not found" is not an error: a name without a resolver, a resolver without that record, or an
unset record gives `""` with a nil error; a resolver that reverts is an error with its decoded
reason. **Normalization:** `normalize` lower-cases ASCII names and accepts labels of `a-z`, `0-9`,
`-` and leading `_` (rejecting empty labels and `--` at positions 3-4). Full UTS-46 / ENSIP-15
normalization (Unicode, emoji, confusables) is not implemented: names with non-ASCII characters
are refused with an error instead of being hashed wrongly. `namehash`, `labelhash` and
`dns_encode` take the name as given.
