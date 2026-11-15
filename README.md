# web3

<!-- Each sub-package adds its own section below. -->

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
