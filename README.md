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
