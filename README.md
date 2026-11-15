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
