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
