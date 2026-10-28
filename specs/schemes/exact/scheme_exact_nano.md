# Scheme: `exact` on `Nano`

## x402 Versions Supported

- ✅ v2

## Supported Networks

- `nano:mainnet` — Nano mainnet (Main network)
- `nano:*` — Any other Nano network

## Summary

The `exact` scheme on Nano uses send blocks for fixed-amount payments. The **`Client`** creates a [send block] based on the **`Resource Server`**'s payment requirement. The **`Facilitator`** verifies and settles this send block on the network. No fee calculation is required as Nano is a feeless network. Settlement typically takes about half a second.

## Protocol Flow

The high-level flow for the `exact` scheme is as follows:

1. **`Client`** sends an HTTP request to the **`Resource Server`** for a resource.
2. **`Resource Server`** responds with a `402 Payment Required` status, including `PaymentRequirements` inside the `accepts` array of a `PaymentRequired` object carried in the `PAYMENT-REQUIRED` header, specifying the `exact` scheme, network, asset `XNO`, account to `payTo`, and `amount` (in [raw units]).
3. **`Client`** constructs and signs a [send block] for the exact amount, and places it inside a `PaymentPayload`. Block construction requires communication with the Nano network to retrieve up-to-date account information.
4. **`Client`** resends the request to **`Resource Server`** with the `PaymentPayload` attached in the `PAYMENT-SIGNATURE` header.
5. **`Resource Server`** forwards the `PaymentPayload` to the **`Facilitator`** for verification (`/verify`).
6. **`Facilitator`** verifies the `PaymentPayload` by querying the Nano network to retrieve up-to-date account and block information.
7. Upon verification, the **`Resource Server`** forwards the `PaymentPayload` to the **`Facilitator`** for settlement (`/settle`).
8. **`Facilitator`** processes the send block contained in the `PaymentPayload` and broadcasts it to the Nano network for confirmation.
9. **`Facilitator`** polls the network for block confirmation.
10. Upon block confirmation (Settlement), **`Facilitator`** communicates the send block's hash back to the **`Resource Server`** in the `PAYMENT-RESPONSE` header.
11. **`Resource Server`** delivers the paid resource to the **`Client`** along with `PAYMENT-RESPONSE` header.

```mermaid
sequenceDiagram
    participant Client;
    participant Server;
    participant Facilitator;
    participant Nano Network;

    Client->>Server: GET /api
    Server-->>Client: 402 PAYMENT-REQUIRED
    Client->>Nano Network: Nano RPC action<br>"account_info"
    Nano Network-->>Client: Account info
    Client->>Client: Create Payment Payload<br>(generate send block)
    Client->>Server: GET /api<br>PAYMENT-SIGNATURE
    Server->>Facilitator: POST /verify<br>PAYMENT-REQUIRED<br>PAYMENT-SIGNATURE
    Facilitator->>Nano Network: Nano RPC actions<br>"account_info"<br>"block_info"
    Nano Network-->>Facilitator: Account and block info
    Facilitator-->>Server: 200 Verified
    Server->>Facilitator: POST /settle<br>PAYMENT-REQUIRED<br>PAYMENT-SIGNATURE
    Facilitator->>Nano Network: Nano RPC action "process"
    Nano Network->>Facilitator: Hash of the send block
    Nano Network->>Nano Network: Confirm send block
    Facilitator->>Nano Network: Nano RPC action "block_info"
    Nano Network-->>Facilitator: confirmed: "true"
    Facilitator-->>Server: 200 Settled<br>PAYMENT-RESPONSE
    Server-->>Client: 200 OK<br>PAYMENT-RESPONSE<br>+ Content

```

## `PaymentRequirements` for `exact`

`PaymentRequirements` for `exact` appears inside the `accepts` array of a `PaymentRequired` object carried in the `PAYMENT-REQUIRED` header.

```json
{
  "scheme": "exact",
  "network": "nano:mainnet",
  "asset": "XNO",
  "amount": "10000000000000000000000000000",
  "payTo": "nano_3ah94brzdm3e5nzzpstzcyxgkddt3ydgzi4478jshawrh7gtof5sb8twxqxt",
  "maxTimeoutSeconds": 120
}
```

- `scheme`: Always `exact` for this scheme.
- `network`: CAIP-2 network identifier, e.g. `nano:mainnet` (Nano mainnet)
- `asset`: This value MUST always be `XNO` as the payment is in native Nano currency. XNO is the ticker for Nano currency.
- `amount`: The amount MUST be expressed in [raw units] (1 nano unit = 10^30 raw units). For example 10000000000000000000000000000 raw units = 0.01 nano units.
- `payTo`: The Nano account that will be receiving the payment. Nano accounts are prefixed with `nano_`.
- `maxTimeoutSeconds`: The maximum number of seconds the **`Facilitator`** SHOULD check for confirmation of the send block on the Nano network.

The following `extra` fields are optional.

- `extra.workOptional`: Signal to **`Client`** that **`Facilitator`** can generate the work that goes in `PaymentPayload.payload.block.work`. See **Who should generate work for the send block?** in **Appendix**.

## `PaymentPayload` for `exact` (`PAYMENT-SIGNATURE` Header)

**`Client`** includes `PaymentPayload` inside a base64-encoded `PAYMENT-SIGNATURE` header:

```json
{
  "x402Version": 2,
  "resource": {
    "url": "https://example.com/api/data",
    "description": "Access to protected API endpoint",
    "mimeType": "application/json"
  },
  "accepted": {
    "scheme": "exact",
    "network": "nano:mainnet",
    "asset": "XNO",
    "amount": "10000000000000000000000000000",
    "payTo": "nano_3ah94brzdm3e5nzzpstzcyxgkddt3ydgzi4478jshawrh7gtof5sb8twxqxt",
    "maxTimeoutSeconds": 120
  },
  "payload": {
    "block": {
      "type": "state",
      "account": "nano_3qgmh14nwztqw4wmcdzy4xpqeejey68chx6nciczwn9abji7ihhum9qtpmdr",
      "previous": "F47B23107E5F34B2CE06F562B5C435DF72A533251CB414C51B2B62A8F63A00E4",
      "representative": "nano_1hza3f7wiiqa7ig3jczyxj5yo86yegcmqk3criaz838j91sxcckpfhbhhra1",
      "balance": "20000000000000000000000000000",
      "link": "19D3D919475DEED4696B5D13018151D1AF88B2BD3BCFF048B45031C1F36D1858",
      "link_as_account": "nano_3ah94brzdm3e5nzzpstzcyxgkddt3ydgzi4478jshawrh7gtof5sb8twxqxt",
      "signature": "3BFBA64A775550E6D49DF1EB8EEC2136DCD74F....77FFF15FD11E6E2162A1714731B743D1E941FA4560A",
      "work": "ffffffd2e1234567"
    }
  },
  "extensions": {}
}
```

`PaymentPayload.payload` in the `PAYMENT-SIGNATURE` header must follow the following schema:

#### **`block`** field

`PaymentPayload.payload.block` contains the full contents of the [send block] which authorizes the transfer of [raw units] on the Nano network.

During creation of the send block, the **`Client`** MUST use Nano RPC action [account_info] (call with parameter `include_confirmed` set to `true`) to request the response fields it needs to construct the block. Construct the block using response fields `frontier` and `balance`; if response field `frontier` differs from response field `confirmed_frontier`, wait for confirmation before building, because a block constructed using response field `confirmed_frontier` forks the awaiting block (see **Fork protection** in **Appendix**).

`PaymentPayload.payload.block.work`, the required Proof-of-Work for anti-spam purposes on the Nano network, MUST be generated by **`Client`** unless the `extra.workOptional` in `PaymentRequirements` is set to `true`. See **Who should generate work for the send block?** in **Appendix** for further information about work generation.

The `representative` field returned by Nano RPC action [account_info] (call with parameter `representative` parameter set to `true`) SHOULD be used during creation of `PaymentPayload.payload.block` to avoid re-delegation of the voting weight for the **`Client`**'s Nano account.

## Verification

For the **`Facilitator`** to verify a payment in the `exact` scheme, these checks MUST be carried out in the following order:

1. **x402 Version Check** - `PaymentPayload.x402Version` MUST be equal to any of the **`Facilitator`**'s supported x402 versions.
2. **Scheme Check** - `PaymentPayload.accepted.scheme` and `PaymentRequirements.scheme` MUST have the value `exact`.
3. **Network Supported Check** - `PaymentPayload.accepted.network` and `PaymentRequirements.network` MUST be prefixed with `nano:`.
4. **Network Mismatch Check** - `PaymentPayload.accepted.network` and `PaymentRequirements.network` MUST be equal.
5. **Asset Check** - `PaymentPayload.accepted.asset` and `PaymentRequirements.asset` MUST have the value `XNO`.
6. **Block Structure Check** - `PaymentPayload.payload.block`'s structure MUST match that of a Nano [send block].
7. **Valid Block Check** - `PaymentPayload.payload.block` MUST be validated by decoding `PaymentPayload.payload.block.account` into its 32-byte public key, [computing the hash] of the block and verifying `PaymentPayload.payload.block.signature` against that hash and public key.
8. **payTo Check** - `PaymentPayload.payload.block.link_as_account` MUST exactly match `PaymentPayload.accepted.payTo`. `PaymentPayload.payload.block.link_as_account` MUST also be the correct Nano account encoding of the 32-byte public key in `PaymentPayload.payload.block.link`.
9. **Previous Confirmation Check** - **`Facilitator`** MUST verify that `PaymentPayload.payload.block` does not already exist on the Nano network. Use Nano RPC action [block_info], passing the calculated hash of `PaymentPayload.payload.block` as a parameter, to check for existence of this block. If [block_info] finds the block, **`Facilitator`** MUST check that the RPC response's `subtype` field is `send` and its `amount` field equals `PaymentPayload.accepted.amount`, returning `error_amount_mismatch` otherwise. If that check passes then **`Facilitator`** MUST return `error_block_already_confirmed` if RPC response field `confirmed` is `true` and `error_block_awaiting_confirmation` if RPC response field `confirmed` is `false`.
10. **Account Check** - `PaymentPayload.payload.block.account` MUST exist on the Nano network. Use Nano RPC action [account_info] to query information for `PaymentPayload.payload.block.account` and ensure that the RPC response has a `frontier` field.
11. **Frontier Hash Check** - `PaymentPayload.payload.block.previous` MUST exactly match the `frontier` field returned by Nano RPC action [account_info] (call with parameter `include_confirmed` set to `true`) for `PaymentPayload.payload.block.account`; if it does not, **`Facilitator`** MUST return `error_previous_not_frontier`. `frontier` field MUST also equal the `confirmed_frontier` field returned in the same RPC response; if it does not, `PaymentPayload.payload.block.account`'s latest block is still awaiting confirmation and **`Facilitator`** MUST return `error_frontier_not_confirmed`. **`Client`** then re-presents the same `PaymentPayload` unchanged after a short wait; it MUST NOT build a new block on `confirmed_frontier`.
12. **Balance Check** - `PaymentPayload.payload.block.account` MUST have balance to cover payment. Use Nano RPC action [account_info] to query information for `PaymentPayload.payload.block.account` and ensure that the delta between the `confirmed_balance` field (_previous_balance_) returned in the RPC response and `PaymentPayload.payload.block.balance` (_new_balance_) is equal to `PaymentPayload.accepted.amount` (_payment_amount_). Calculation is _previous_balance_ - _new_balance_ = _payment_amount_. If _previous_balance_ < _payment_amount_, return `error_not_enough_balance`; otherwise, if _previous_balance_ - _new_balance_ differs from _payment_amount_, return `error_amount_mismatch`.
13. **Work Difficulty Check** - `PaymentPayload.payload.block.work` MUST be validated to ensure it meets the Nano network's current minimum difficulty threshold for a send transaction. If **`Facilitator`** has signalled work generation capabilities and `PaymentPayload.payload.block.work` is `"0000000000000000"`, this check is deferred to Settlement, after work generation has taken place (see **Who should generate work for the send block?** in **Appendix**).

### Valid Verification Response

A valid Verification Response returned by **`Facilitator`** has the following structure:

```json
{
  "isValid": true,
  "payer": "nano_3qgmh14nwztqw4wmcdzy4xpqeejey68chx6nciczwn9abji7ihhum9qtpmdr"
}
```

### Invalid Verification Response

An invalid Verification Response returned by **`Facilitator`** has the following structure:

```json
{
  "isValid": false,
  "invalidReason": "error_not_enough_balance",
  "payer": "nano_3qgmh14nwztqw4wmcdzy4xpqeejey68chx6nciczwn9abji7ihhum9qtpmdr"
}
```

See **Error Vocabulary** for the full list of `invalidReason` values.

**`Facilitator`** MAY also include an `invalidMessage` field in the invalid Verification Response as free text for humans. **`Client`** and **`Resource Server`** MUST NOT branch on it, `invalidReason` is the contract.

```json
{
  "isValid": false,
  "invalidReason": "error_not_enough_balance",
  "invalidMessage": "Balance for payer's account is not enough to cover payment amount",
  "payer": "nano_3qgmh14nwztqw4wmcdzy4xpqeejey68chx6nciczwn9abji7ihhum9qtpmdr"
}
```

## Settlement

The **`Facilitator`** determines Settlement using a two-step approach:

1. **Broadcast Step** - Before broadcasting, **`Facilitator`** MUST re-verify against the full set of Verification checks. Any check that fails MUST produce an unsuccessful Settlement Response containing the corresponding `errorReason`. If re-verification is successful then **`Facilitator`** broadcasts `PaymentPayload.payload.block` to the Nano network with the Nano RPC action [process] (the `subtype` parameter MUST be set to `send`) and notes the returned hash of the broadcasted block.

2. **Confirmed Step** - **`Facilitator`** polls the network for confirmation of the broadcasted block's hash using Nano node RPC action [block_info]. Check that the field `confirmed` is `true` in the RPC response to determine if the block is confirmed. Polling SHOULD occur at one second intervals, up to a maximum of `PaymentRequirements.maxTimeoutSeconds`.

`Broadcasted` means that **`Facilitator`** has checked the validity of the block and broadcasted the block to the Nano network. The block is not yet confirmed.

`Confirmed` means that the block has successfully gathered enough votes from Nano network representatives to reach quorum and has achieved absolute, irreversible finality.

It is recommended that the **`Facilitator`** SHOULD wait half a second before initial polling for block confirmation using Nano node RPC action [block_info]. A typical Nano block confirms in about half a second so this small wait should reduce the need for additional polling and provides an improved payment experience for the **`Client`**.

It is possible that the **`Client`** adds a block to their Nano account (like an auto-receive block or another concurrent payment) between the Verification and Settlement stages of an ongoing payment attempt. This will result in an unsuccessful Settlement because the **`Client`**'s Nano account now has a different frontier hash to the one that MUST exactly match `PaymentPayload.payload.block.previous` (see **Frontier Hash Check** in Verification). If this happens the **`Facilitator`** MUST return an `errorReason` of `error_previous_not_frontier` (see **Error Vocabulary**) so that the **`Client`** understands that it needs to construct a new [send block] and not attempt to resubmit the now outdated send block.

### Successful Settlement Response

A successful Settlement Response returned by **`Facilitator`** has the following structure:

```json
{
  "success": true,
  "payer": "nano_3qgmh14nwztqw4wmcdzy4xpqeejey68chx6nciczwn9abji7ihhum9qtpmdr",
  "transaction": "0DF7EC8E0955E2A242E3C68B36CBDD639FBB56914AF63AE87295F17D30D9C9D4",
  "network": "nano:mainnet"
}
```

Field Descriptions:

- `payer`: The **`Client`**'s Nano account that signed the send block.
- `transaction`: Hash of the send block.

The **`Resource Server`** base64-encodes the Settlement Response and sends it to **`Client`** in a `PAYMENT-RESPONSE` header:

```json
HTTP/1.1 200 OK
Content-Type: application/json
PAYMENT-RESPONSE: eyJzdWNjZXNzI.....I2NiJ9

{
  "data": "paid resource response data",
  "timestamp": "2026-08-26T10:30:00Z"
}
```

### Unsuccessful Settlement Response

An unsuccessful Settlement Response returned by **`Facilitator`** has the following structure:

```json
{
  "success": false,
  "errorReason": "error_nano_rpc",
  "payer": "nano_3qgmh14nwztqw4wmcdzy4xpqeejey68chx6nciczwn9abji7ihhum9qtpmdr",
  "transaction": "",
  "network": "nano:mainnet"
}
```

If `errorReason` for the unsuccessful Settlement is `error_block_awaiting_confirmation` or `error_block_already_confirmed`, the **`Facilitator`** MUST fill `transaction` in the unsuccessful Settlement Response with the block hash of `PaymentPayload.payload.block` (see **When should Resource Server deliver the resource to Client?** in **Appendix**).

```json
{
  "success": false,
  "errorReason": "error_block_awaiting_confirmation",
  "payer": "nano_3qgmh14nwztqw4wmcdzy4xpqeejey68chx6nciczwn9abji7ihhum9qtpmdr",
  "transaction": "0DF7EC8E0955E2A242E3C68B36CBDD639FBB56914AF63AE87295F17D30D9C9D4",
  "network": "nano:mainnet"
}
```

```json
{
  "success": false,
  "errorReason": "error_block_already_confirmed",
  "payer": "nano_3qgmh14nwztqw4wmcdzy4xpqeejey68chx6nciczwn9abji7ihhum9qtpmdr",
  "transaction": "0DF7EC8E0955E2A242E3C68B36CBDD639FBB56914AF63AE87295F17D30D9C9D4",
  "network": "nano:mainnet"
}
```

See **Error Vocabulary** in the next section for the full list of `errorReason` values.

**`Facilitator`** MAY also include an `errorMessage` field in the unsuccessful Settlement Response as free text for humans. **`Client`** and **`Resource Server`** MUST NOT branch on it, `errorReason` is the contract.

```json
{
  "success": false,
  "errorReason": "error_invalid_block",
  "errorMessage": "Payer's send block is invalid",
  "payer": "nano_3qgmh14nwztqw4wmcdzy4xpqeejey68chx6nciczwn9abji7ihhum9qtpmdr",
  "transaction": "",
  "network": "nano:mainnet"
}
```

## Error Vocabulary

**`Facilitator`** MUST use the following error codes across `invalidReason` (Verification failure), `errorReason` (Settlement failure) and `PaymentRequired.error` (**`Resource Server`** error) to describe errors encountered.

| Error Code                          | Meaning                                                                                                                                                                                                                                                                                | Used In                                                                                           |
| ----------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------- |
| `error_x402_version_not_supported`  | `PaymentPayload.x402Version` is not supported                                                                                                                                                                                                                                          | x402 Version Check, Unsuccessful Settlement Response                                              |
| `error_unsupported_scheme`          | `PaymentPayload.accepted.scheme` and/or `PaymentRequirements.scheme` do not have the value `exact`                                                                                                                                                                                     | Scheme Check, Unsuccessful Settlement Response                                                    |
| `error_unsupported_network`         | `PaymentPayload.accepted.network` and/or `PaymentRequirements.network` are unsupported                                                                                                                                                                                                 | Network Supported Check, Unsuccessful Settlement Response                                         |
| `error_network_mismatch`            | `PaymentPayload.accepted.network` differs from `PaymentRequirements.network`                                                                                                                                                                                                           | Network Mismatch Check, Unsuccessful Settlement Response                                          |
| `error_invalid_asset`               | `PaymentPayload.accepted.asset` and/or `PaymentRequirements.asset` do not have the value `XNO`                                                                                                                                                                                         | Asset Check, Unsuccessful Settlement Response                                                     |
| `error_invalid_block`               | `PaymentPayload.payload.block` is not a valid Nano send block                                                                                                                                                                                                                          | Block Structure Check, Valid Block Check, Unsuccessful Settlement Response                        |
| `error_payto_link_mismatch`         | `PaymentPayload.payload.block.link_as_account` differs from `PaymentPayload.accepted.payTo`                                                                                                                                                                                            | payTo Check, Unsuccessful Settlement Response                                                     |
| `error_account_not_found`           | `PaymentPayload.payload.block.account` not found on the Nano network                                                                                                                                                                                                                   | Account Check, Unsuccessful Settlement Response                                                   |
| `error_previous_not_frontier`       | `PaymentPayload.payload.block.previous` does not match `frontier` for `PaymentPayload.payload.block.account`                                                                                                                                                                           | Frontier Hash Check, Unsuccessful Settlement Response                                             |
| `error_frontier_not_confirmed`      | `frontier` for `PaymentPayload.payload.block.account` does not match `confirmed_frontier` for `PaymentPayload.payload.block.account`                                                                                                                                                   | Frontier Hash Check, Unsuccessful Settlement Response                                             |
| `error_not_enough_balance`          | Balance for `PaymentPayload.payload.block.account` is not enough to cover `PaymentPayload.accepted.amount`                                                                                                                                                                             | Balance Check, Unsuccessful Settlement Response                                                   |
| `error_amount_mismatch`             | Delta between the balance for `PaymentPayload.payload.block.account` and `PaymentPayload.payload.block.balance` is not equal to `PaymentPayload.accepted.amount`; or, for a block that [block_info] already finds, its `amount` field is not equal to `PaymentPayload.accepted.amount` | Balance Check, Previous Confirmation Check, Unsuccessful Settlement Response                      |
| `error_invalid_work`                | `PaymentPayload.payload.block.work` is invalid                                                                                                                                                                                                                                         | Work Difficulty Check, Unsuccessful Settlement Response                                           |
| `error_timeout_expired`             | `PaymentRequirements.maxTimeoutSeconds` has expired                                                                                                                                                                                                                                    | Unsuccessful Settlement Response                                                                  |
| `error_block_already_confirmed`     | `PaymentPayload.payload.block` is already confirmed on the Nano network                                                                                                                                                                                                                | Previous Confirmation Check, Unsuccessful Settlement Response                                     |
| `error_block_awaiting_confirmation` | `PaymentPayload.payload.block` has been broadcast to the Nano network and is awaiting confirmation                                                                                                                                                                                     | Previous Confirmation Check, Unsuccessful Settlement Response                                     |
| `error_block_not_broadcast`         | `PaymentPayload.payload.block` unable to be broadcast to the Nano network                                                                                                                                                                                                              | Unsuccessful Settlement Response                                                                  |
| `error_work_unavailable`            | **`Facilitator`** unable to perform work generation to set value for `PaymentPayload.payload.block.work`                                                                                                                                                                               | PaymentRequired.error, Unsuccessful Settlement Response                                           |
| `error_nano_rpc`                    | Nano RPC failure                                                                                                                                                                                                                                                                       | Previous Confirmation Check, Account Check, Frontier Hash Check, Unsuccessful Settlement Response |

## Nano RPC examples

### [account_info] action

Returns information for an account on the Nano network. Specify the Nano account with `account`. Make sure both `include_confirmed` and `representative` are set to `true`.

```
{
  "action": "account_info",
  "account": "nano_3qgmh14nwztqw4wmcdzy4xpqeejey68chx6nciczwn9abji7ihhum9qtpmdr",
  "include_confirmed": "true",
  "representative": "true"
}
```

Used by: **`Client`**, **`Facilitator`**

### [block_info] action

Returns information for a block on the Nano network. Specify the block by passing its block hash into `hash`.

```
{
  "action": "block_info",
  "hash": "1D562FA09DD3D6E99AE41B065861A36930456BE907002BD8F93220DB0FE3995F",
  "json_block": "true"
}
```

Used by: **`Facilitator`**

### [process] action (for a send block)

Broadcasts a [send block] to the Nano network. Make sure that `subtype` is set to `send`, and that `block` has the same structure as `PaymentPayload.payload.block`.

```
{
  "action": "process",
  "block": {
    "type": "state",
    "account": "nano_3qgmh14nwztqw4wmcdzy4xpqeejey68chx6nciczwn9abji7ihhum9qtpmdr",
    "previous": "F47B23107E5F34B2CE06F562B5C435DF72A533251CB414C51B2B62A8F63A00E4",
    "representative": "nano_1hza3f7wiiqa7ig3jczyxj5yo86yegcmqk3criaz838j91sxcckpfhbhhra1",
    "balance": "20000000000000000000000000000",
    "link": "19D3D919475DEED4696B5D13018151D1AF88B2BD3BCFF048B45031C1F36D1858",
    "link_as_account": "nano_3ah94brzdm3e5nzzpstzcyxgkddt3ydgzi4478jshawrh7gtof5sb8twxqxt",
    "signature": "3BFBA64A775550E6D49DF1EB8EEC2136DCD74F....77FFF15FD11E6E2162A1714731B743D1E941FA4560A",
    "work": "ffffffd2e1234567"
  },
  "subtype": "send",
  "json_block": "true"
}
```

Used by: **`Facilitator`**

## Test Vectors

Generated on 2026-10-01 against a live facilitator (`facilitator.pursekeeper.dev`, Node implementation) and the Nano main network. Every hash below can be checked on any Nano node or block explorer. V1 and V13 are real settled payments; V2 to V12 are `/verify` requests that broadcast nothing. V1 to V12 were generated on 2026-10-01, V13 on 2026-10-14 against the same facilitator after it took up the amount clause of check 9 (Previous Confirmation Check), V14 on 2026-10-27, and V15 on 2026-10-28 (two real 1-raw sends, the second refused while the first was awaiting confirmation and settled unchanged once it was). The payer is an ordinary account funded for this purpose; `payTo` is a seller account. The capture of every request and response as sent and received is at [pursekeeper/api, examples/research/x402-vectors-2026-10-01.json](https://github.com/pursekeeper/api/blob/main/examples/research/x402-vectors-2026-10-01.json) and, for V13, [examples/research/x402-vector-v13-2026-10-14.json](https://github.com/pursekeeper/api/blob/main/examples/research/x402-vector-v13-2026-10-14.json); for V15, [examples/research/x402-vector-v15-2026-10-28.json](https://github.com/pursekeeper/api/blob/main/examples/research/x402-vector-v15-2026-10-28.json).

The `invalidReason` and `errorReason` values are the ones this facilitator returned when the vectors were cut, before this document's Error Vocabulary existed; the vocabulary code each corresponds to is given beside it. Check numbers follow the Verification section of this document.

### Common inputs

```json
{
  "paymentRequirements": {
    "scheme": "exact",
    "network": "nano:mainnet",
    "asset": "XNO",
    "amount": "1000000000000000000000000000",
    "payTo": "nano_1xug1q5t7nxoj3ywwzokiea9jz8fq8qfgzp8pbyfr3co3e5xgj755uofu8ue",
    "maxTimeoutSeconds": 60
  }
}
```

`payTo` decodes to the public key `776E05C7A2D3B5887DCE7EB2831078FCCDB9AED77EC6B27CDC05550B07D744A3`.

Payer `nano_1i3y944esngqw6wb6ia68dotj4yuqctch9kx8ct65twt8ewi4rdcfgax7ggf`. Nano RPC `account_info` for the payer immediately before V1 (fields not used by the scheme omitted):

```json
{
  "frontier": "DE891E21F480F772E68B61B1C7D8BF6525E5D9FAE64AD5B7BC71F9486348B943",
  "balance": "10787318189447476890320950701",
  "representative": "nano_3arg3asgtigae3xckabaaewkx3bzsh7nwz7jkmjos79ihyaxwphhm6qgjps4",
  "confirmed_frontier": "DE891E21F480F772E68B61B1C7D8BF6525E5D9FAE64AD5B7BC71F9486348B943",
  "confirmed_height": "98",
  "block_count": "98"
}
```

Work was generated against `previous` at the send threshold `fffffff800000000`.

### V0: the seller's 402

`GET https://pursekeeper.dev/v1/echo?msg=vector` without payment answered `402` with a `PAYMENT-REQUIRED` header decoding to a `PaymentRequired` whose `accepts` entry has exactly the six fields above (this seller adds `extra.work: "optional"`, an extension of its own that the vectors do not use).

### V1: valid payment, verified, settled, confirmed

`PaymentPayload` (the decoded `PAYMENT-SIGNATURE` header and the `paymentPayload` field of the `/verify` and `/settle` bodies):

```json
{
  "x402Version": 2,
  "resource": {
    "url": "https://pursekeeper.dev/v1/echo?msg=vector",
    "description": "returns what you sent (test your payment client)",
    "mimeType": "application/json"
  },
  "accepted": {
    "scheme": "exact",
    "network": "nano:mainnet",
    "asset": "XNO",
    "amount": "1000000000000000000000000000",
    "payTo": "nano_1xug1q5t7nxoj3ywwzokiea9jz8fq8qfgzp8pbyfr3co3e5xgj755uofu8ue",
    "maxTimeoutSeconds": 60
  },
  "payload": {
    "block": {
      "type": "state",
      "account": "nano_1i3y944esngqw6wb6ia68dotj4yuqctch9kx8ct65twt8ewi4rdcfgax7ggf",
      "previous": "DE891E21F480F772E68B61B1C7D8BF6525E5D9FAE64AD5B7BC71F9486348B943",
      "representative": "nano_3arg3asgtigae3xckabaaewkx3bzsh7nwz7jkmjos79ihyaxwphhm6qgjps4",
      "balance": "9787318189447476890320950701",
      "link": "776E05C7A2D3B5887DCE7EB2831078FCCDB9AED77EC6B27CDC05550B07D744A3",
      "link_as_account": "nano_1xug1q5t7nxoj3ywwzokiea9jz8fq8qfgzp8pbyfr3co3e5xgj755uofu8ue",
      "signature": "65AFBF7622955A1D6E5E0703B479C33BA652CB4C29477968D2E8864891124875C3EA3F658F597CDB689086F2DE1638BAC0847CFC05CF682682967927D465FB03",
      "work": "3f6ff9e7b7ba27a7"
    }
  }
}
```

Block hash (BLAKE2b-256 over the six state-block fields, the hash the signature covers): `1D60903EFD920FA78746E18F08C021EE7B8155205709DF5A4125BBDCF96D0BEB`.

`POST /verify` with `{ "x402Version": 2, "paymentPayload": <above>, "paymentRequirements": <common inputs> }` answered `200`:

```json
{
  "isValid": true,
  "payer": "nano_1i3y944esngqw6wb6ia68dotj4yuqctch9kx8ct65twt8ewi4rdcfgax7ggf"
}
```

`POST /settle` with the same body answered `200` after 584 ms, which includes the `process` call and the confirmation poll:

```json
{
  "success": true,
  "transaction": "1D60903EFD920FA78746E18F08C021EE7B8155205709DF5A4125BBDCF96D0BEB",
  "network": "nano:mainnet",
  "payer": "nano_1i3y944esngqw6wb6ia68dotj4yuqctch9kx8ct65twt8ewi4rdcfgax7ggf"
}
```

Nano RPC `block_info` for `transaction` afterwards:

```json
{
  "confirmed": "true",
  "height": "99",
  "amount": "1000000000000000000000000000",
  "subtype": "send",
  "block_account": "nano_1i3y944esngqw6wb6ia68dotj4yuqctch9kx8ct65twt8ewi4rdcfgax7ggf"
}
```

### V2: replay of the settled block

The exact V1 body presented again after settlement. `/verify`:

```json
{
  "isValid": false,
  "invalidReason": "block_already_exists",
  "payer": "nano_1i3y944esngqw6wb6ia68dotj4yuqctch9kx8ct65twt8ewi4rdcfgax7ggf"
}
```

`/settle`:

```json
{
  "success": false,
  "errorReason": "block_already_exists",
  "transaction": "1D60903EFD920FA78746E18F08C021EE7B8155205709DF5A4125BBDCF96D0BEB",
  "network": "nano:mainnet",
  "payer": "nano_1i3y944esngqw6wb6ia68dotj4yuqctch9kx8ct65twt8ewi4rdcfgax7ggf"
}
```

The settle refusal still names the original `transaction`, so a resource server whose first success reply was lost can learn that the payment went through without being served twice. This is the behaviour the **Unsuccessful Settlement Response** and **Blocks have no expiry** sections ask for; in the Error Vocabulary the code is `error_block_already_confirmed`, from check 9 (Previous Confirmation Check).

### V3 to V12: refused at verification, nothing broadcast

All are built on the payer's frontier after V1 (`1D60903EFD920FA78746E18F08C021EE7B8155205709DF5A4125BBDCF96D0BEB`, balance `9787318189447476890320950701`) with valid work `feec391ed40f8b85` for that frontier, except V7 (built on the pre-V1 frontier with V1's work) and V11 (a different, never-opened account). Each block is correctly signed by its account unless the row says otherwise, so checks 6 to 8 can be re-run offline from the JSON; checks 9 to 13 depend on the ledger state recorded above and will answer `error_previous_not_frontier` (`frontier_moved` here) once the payer's account moves on.

| Vector | Mutation from a valid block | Verification check that fails | `invalidReason` | Error Vocabulary |
|---|---|---|---|---|
| V3 | signature: last hex character changed | 7 (Valid Block Check) | `invalid_signature` | `error_invalid_block` |
| V4 | balance = previous balance − amount + 1 (1 raw short) | 12 (Balance Check) | `amount_mismatch` | `error_amount_mismatch` |
| V5 | balance = previous balance − 2 × amount (twice the amount) | 12 (Balance Check) | `amount_mismatch` | `error_amount_mismatch` |
| V6 | link = public key of another account; link_as_account agrees with link | 8 (payTo Check) | `invalid_payto` | `error_payto_link_mismatch` |
| V7 | previous = the frontier before V1 landed (a fork of V1 with a different balance) | 11 (Frontier Hash Check) | `frontier_moved` | `error_previous_not_frontier` |
| V8 | work = 0000000000000001 | 13 (Work Difficulty Check) | `invalid_work` | `error_invalid_work` |
| V9 | accepted.network = nano:testnet while paymentRequirements.network = nano:mainnet | 4 (Network Mismatch Check) | `requirements_mismatch` | `error_network_mismatch` |
| V10 | x402Version = 1 | 1 (x402 Version Check) | `unsupported_x402_version` | `error_x402_version_not_supported` |
| V11 | payer is an account with no blocks; previous = 64 zeros, balance 0 | 10 (Account Check) | `account_not_found` | `error_account_not_found` |
| V12 | link_as_account names another account while link is payTo's key | 8 (payTo Check) | `invalid_payto` | `error_payto_link_mismatch` |

<details>
<summary>Blocks and responses for V3 to V12</summary>

**V3**: Signature does not verify.

```json
{
  "block": {
    "type": "state",
    "account": "nano_1i3y944esngqw6wb6ia68dotj4yuqctch9kx8ct65twt8ewi4rdcfgax7ggf",
    "previous": "1D60903EFD920FA78746E18F08C021EE7B8155205709DF5A4125BBDCF96D0BEB",
    "representative": "nano_3arg3asgtigae3xckabaaewkx3bzsh7nwz7jkmjos79ihyaxwphhm6qgjps4",
    "balance": "8787318189447476890320950701",
    "link": "776E05C7A2D3B5887DCE7EB2831078FCCDB9AED77EC6B27CDC05550B07D744A3",
    "link_as_account": "nano_1xug1q5t7nxoj3ywwzokiea9jz8fq8qfgzp8pbyfr3co3e5xgj755uofu8ue",
    "signature": "37B0AFA22ACE7F4B7BFBEB49C3DAF84FF6EB32766DD9791027D948E0C422AF98358B923608EA11360C2C8548FCBE94E5D58F6CAF94186E6C39FF36A24B513A00",
    "work": "feec391ed40f8b85"
  },
  "verify_response": {
    "isValid": false,
    "invalidReason": "invalid_signature",
    "payer": "nano_1i3y944esngqw6wb6ia68dotj4yuqctch9kx8ct65twt8ewi4rdcfgax7ggf"
  }
}
```

**V4**: Amount mismatch: sends 1 raw less than required.

```json
{
  "block": {
    "type": "state",
    "account": "nano_1i3y944esngqw6wb6ia68dotj4yuqctch9kx8ct65twt8ewi4rdcfgax7ggf",
    "previous": "1D60903EFD920FA78746E18F08C021EE7B8155205709DF5A4125BBDCF96D0BEB",
    "representative": "nano_3arg3asgtigae3xckabaaewkx3bzsh7nwz7jkmjos79ihyaxwphhm6qgjps4",
    "balance": "8787318189447476890320950702",
    "link": "776E05C7A2D3B5887DCE7EB2831078FCCDB9AED77EC6B27CDC05550B07D744A3",
    "link_as_account": "nano_1xug1q5t7nxoj3ywwzokiea9jz8fq8qfgzp8pbyfr3co3e5xgj755uofu8ue",
    "signature": "EACDFCA51178D880E5E0E134C169821C4A1533298C007291C2F67D7A0DB83489415279698572088C0D4A6F4C6E2B51ED87282DA43788B25E8CA64C60D871BE03",
    "work": "feec391ed40f8b85"
  },
  "verify_response": {
    "isValid": false,
    "invalidReason": "amount_mismatch",
    "payer": "nano_1i3y944esngqw6wb6ia68dotj4yuqctch9kx8ct65twt8ewi4rdcfgax7ggf"
  }
}
```

**V5**: Amount mismatch: sends twice the required amount.

```json
{
  "block": {
    "type": "state",
    "account": "nano_1i3y944esngqw6wb6ia68dotj4yuqctch9kx8ct65twt8ewi4rdcfgax7ggf",
    "previous": "1D60903EFD920FA78746E18F08C021EE7B8155205709DF5A4125BBDCF96D0BEB",
    "representative": "nano_3arg3asgtigae3xckabaaewkx3bzsh7nwz7jkmjos79ihyaxwphhm6qgjps4",
    "balance": "7787318189447476890320950701",
    "link": "776E05C7A2D3B5887DCE7EB2831078FCCDB9AED77EC6B27CDC05550B07D744A3",
    "link_as_account": "nano_1xug1q5t7nxoj3ywwzokiea9jz8fq8qfgzp8pbyfr3co3e5xgj755uofu8ue",
    "signature": "4765D27CFE1D7B407C80D12BBD2F18EFECD3062BED67727B3C9F687C3F626E338A1017150AF8777975F00EF4815E695972C42626E58B237D77BE64E43E13AB01",
    "work": "feec391ed40f8b85"
  },
  "verify_response": {
    "isValid": false,
    "invalidReason": "amount_mismatch",
    "payer": "nano_1i3y944esngqw6wb6ia68dotj4yuqctch9kx8ct65twt8ewi4rdcfgax7ggf"
  }
}
```

**V6**: payTo mismatch: link is not the public key of payTo.

```json
{
  "block": {
    "type": "state",
    "account": "nano_1i3y944esngqw6wb6ia68dotj4yuqctch9kx8ct65twt8ewi4rdcfgax7ggf",
    "previous": "1D60903EFD920FA78746E18F08C021EE7B8155205709DF5A4125BBDCF96D0BEB",
    "representative": "nano_3arg3asgtigae3xckabaaewkx3bzsh7nwz7jkmjos79ihyaxwphhm6qgjps4",
    "balance": "8787318189447476890320950701",
    "link": "A30E0A32ED41C8607AA9212843392E853FCBCB4E7CB194E35C94F07F91DE59EF",
    "link_as_account": "nano_3arg3asgtigae3xckabaaewkx3bzsh7nwz7jkmjos79ihyaxwphhm6qgjps4",
    "signature": "71ADC33C1B0BA2EF6C56909F3DC9AE6EF61EE4612EF93B118F1B2125D1704503A1F0ED79023C7983B274AFE5AC8B50659C653AC5C023A02038E4734A4F7C6E0E",
    "work": "feec391ed40f8b85"
  },
  "verify_response": {
    "isValid": false,
    "invalidReason": "invalid_payto",
    "payer": "nano_1i3y944esngqw6wb6ia68dotj4yuqctch9kx8ct65twt8ewi4rdcfgax7ggf"
  }
}
```

**V7**: Frontier moved: previous is no longer the account frontier.

```json
{
  "block": {
    "type": "state",
    "account": "nano_1i3y944esngqw6wb6ia68dotj4yuqctch9kx8ct65twt8ewi4rdcfgax7ggf",
    "previous": "DE891E21F480F772E68B61B1C7D8BF6525E5D9FAE64AD5B7BC71F9486348B943",
    "representative": "nano_3arg3asgtigae3xckabaaewkx3bzsh7nwz7jkmjos79ihyaxwphhm6qgjps4",
    "balance": "9787318189447476890320950700",
    "link": "776E05C7A2D3B5887DCE7EB2831078FCCDB9AED77EC6B27CDC05550B07D744A3",
    "link_as_account": "nano_1xug1q5t7nxoj3ywwzokiea9jz8fq8qfgzp8pbyfr3co3e5xgj755uofu8ue",
    "signature": "13811539AFC7A1918572F8672A936EE0E095F4634B0EDBB0227ACF839C0C47D2191FE04B0E7C25ABABE1884A3237E60AF06C6783C25270D3116EB12E9E5F2302",
    "work": "3f6ff9e7b7ba27a7"
  },
  "verify_response": {
    "isValid": false,
    "invalidReason": "frontier_moved",
    "payer": "nano_1i3y944esngqw6wb6ia68dotj4yuqctch9kx8ct65twt8ewi4rdcfgax7ggf"
  }
}
```

**V8**: Work below the send threshold.

```json
{
  "block": {
    "type": "state",
    "account": "nano_1i3y944esngqw6wb6ia68dotj4yuqctch9kx8ct65twt8ewi4rdcfgax7ggf",
    "previous": "1D60903EFD920FA78746E18F08C021EE7B8155205709DF5A4125BBDCF96D0BEB",
    "representative": "nano_3arg3asgtigae3xckabaaewkx3bzsh7nwz7jkmjos79ihyaxwphhm6qgjps4",
    "balance": "8787318189447476890320950701",
    "link": "776E05C7A2D3B5887DCE7EB2831078FCCDB9AED77EC6B27CDC05550B07D744A3",
    "link_as_account": "nano_1xug1q5t7nxoj3ywwzokiea9jz8fq8qfgzp8pbyfr3co3e5xgj755uofu8ue",
    "signature": "37B0AFA22ACE7F4B7BFBEB49C3DAF84FF6EB32766DD9791027D948E0C422AF98358B923608EA11360C2C8548FCBE94E5D58F6CAF94186E6C39FF36A24B513A0F",
    "work": "0000000000000001"
  },
  "verify_response": {
    "isValid": false,
    "invalidReason": "invalid_work",
    "payer": "nano_1i3y944esngqw6wb6ia68dotj4yuqctch9kx8ct65twt8ewi4rdcfgax7ggf"
  }
}
```

**V9**: accepted.network differs from the requirements. `accepted` in the payload: `{"scheme": "exact", "network": "nano:testnet", "asset": "XNO", "amount": "1000000000000000000000000000", "payTo": "nano_1xug1q5t7nxoj3ywwzokiea9jz8fq8qfgzp8pbyfr3co3e5xgj755uofu8ue", "maxTimeoutSeconds": 60}`.

```json
{
  "block": {
    "type": "state",
    "account": "nano_1i3y944esngqw6wb6ia68dotj4yuqctch9kx8ct65twt8ewi4rdcfgax7ggf",
    "previous": "1D60903EFD920FA78746E18F08C021EE7B8155205709DF5A4125BBDCF96D0BEB",
    "representative": "nano_3arg3asgtigae3xckabaaewkx3bzsh7nwz7jkmjos79ihyaxwphhm6qgjps4",
    "balance": "8787318189447476890320950701",
    "link": "776E05C7A2D3B5887DCE7EB2831078FCCDB9AED77EC6B27CDC05550B07D744A3",
    "link_as_account": "nano_1xug1q5t7nxoj3ywwzokiea9jz8fq8qfgzp8pbyfr3co3e5xgj755uofu8ue",
    "signature": "37B0AFA22ACE7F4B7BFBEB49C3DAF84FF6EB32766DD9791027D948E0C422AF98358B923608EA11360C2C8548FCBE94E5D58F6CAF94186E6C39FF36A24B513A0F",
    "work": "feec391ed40f8b85"
  },
  "verify_response": {
    "isValid": false,
    "invalidReason": "requirements_mismatch",
    "payer": ""
  }
}
```

**V10**: Unsupported x402Version. `x402Version`: 1.

```json
{
  "block": {
    "type": "state",
    "account": "nano_1i3y944esngqw6wb6ia68dotj4yuqctch9kx8ct65twt8ewi4rdcfgax7ggf",
    "previous": "1D60903EFD920FA78746E18F08C021EE7B8155205709DF5A4125BBDCF96D0BEB",
    "representative": "nano_3arg3asgtigae3xckabaaewkx3bzsh7nwz7jkmjos79ihyaxwphhm6qgjps4",
    "balance": "8787318189447476890320950701",
    "link": "776E05C7A2D3B5887DCE7EB2831078FCCDB9AED77EC6B27CDC05550B07D744A3",
    "link_as_account": "nano_1xug1q5t7nxoj3ywwzokiea9jz8fq8qfgzp8pbyfr3co3e5xgj755uofu8ue",
    "signature": "37B0AFA22ACE7F4B7BFBEB49C3DAF84FF6EB32766DD9791027D948E0C422AF98358B923608EA11360C2C8548FCBE94E5D58F6CAF94186E6C39FF36A24B513A0F",
    "work": "feec391ed40f8b85"
  },
  "verify_response": {
    "isValid": false,
    "invalidReason": "unsupported_x402_version",
    "payer": ""
  }
}
```

**V11**: Payer account not found on the ledger.

```json
{
  "block": {
    "type": "state",
    "account": "nano_1nwj688ksukwpae96tszqi4xkd66y59d89xp6kaq66jawjcz7oo5xkowcyyr",
    "previous": "0000000000000000000000000000000000000000000000000000000000000000",
    "representative": "nano_3arg3asgtigae3xckabaaewkx3bzsh7nwz7jkmjos79ihyaxwphhm6qgjps4",
    "balance": "0",
    "link": "776E05C7A2D3B5887DCE7EB2831078FCCDB9AED77EC6B27CDC05550B07D744A3",
    "link_as_account": "nano_1xug1q5t7nxoj3ywwzokiea9jz8fq8qfgzp8pbyfr3co3e5xgj755uofu8ue",
    "work": "feec391ed40f8b85",
    "signature": "BE065BC7071EE8F73DDED66E0FECABF3FCCC3F3D2F26BD75E562BCCFF44BDCDEE2EDD5FB4B20BF5C4B43D797CAE08EC39DB212DD3AB332CE12F41740304F0702"
  },
  "verify_response": {
    "isValid": false,
    "invalidReason": "account_not_found",
    "payer": "nano_1nwj688ksukwpae96tszqi4xkd66y59d89xp6kaq66jawjcz7oo5xkowcyyr"
  }
}
```

**V12**: link_as_account disagrees with link.

```json
{
  "block": {
    "type": "state",
    "account": "nano_1i3y944esngqw6wb6ia68dotj4yuqctch9kx8ct65twt8ewi4rdcfgax7ggf",
    "previous": "1D60903EFD920FA78746E18F08C021EE7B8155205709DF5A4125BBDCF96D0BEB",
    "representative": "nano_3arg3asgtigae3xckabaaewkx3bzsh7nwz7jkmjos79ihyaxwphhm6qgjps4",
    "balance": "8787318189447476890320950701",
    "link": "776E05C7A2D3B5887DCE7EB2831078FCCDB9AED77EC6B27CDC05550B07D744A3",
    "link_as_account": "nano_3arg3asgtigae3xckabaaewkx3bzsh7nwz7jkmjos79ihyaxwphhm6qgjps4",
    "signature": "37B0AFA22ACE7F4B7BFBEB49C3DAF84FF6EB32766DD9791027D948E0C422AF98358B923608EA11360C2C8548FCBE94E5D58F6CAF94186E6C39FF36A24B513A0F",
    "work": "feec391ed40f8b85"
  },
  "verify_response": {
    "isValid": false,
    "invalidReason": "invalid_payto",
    "payer": "nano_1i3y944esngqw6wb6ia68dotj4yuqctch9kx8ct65twt8ewi4rdcfgax7ggf"
  }
}
```

</details>

### V13: a confirmed send of 1 raw to `payTo`, presented at the full price (check 9)

Generated on 2026-10-14. The payer sent 1 raw to `payTo` in a real block (`34845D9CCF5F574A3CF01A77ACB3B02AC970FB5745B3A317E35524BAF62E2472`, height 118 of the payer), verified and settled through the facilitator with `accepted.amount` = `1` (success in 549 ms; `block_info` afterwards: `subtype` `send`, `amount` `1`, `confirmed` `true`). The same confirmed block was then presented with `accepted.amount` and `PaymentRequirements.amount` at the full price, `1000000000000000000000000000` raw. Check 9 finds the block with [block_info] and refuses on its `amount` before the confirmation state is considered. The Balance Check cannot run for a block already on the chain, since `account_info` now holds the post-send balance; without the amount clause this block would read as already confirmed, and a **`Resource Server`** with no record for it would deliver a full-price resource for 1 raw. `transaction` stays empty on this refusal: the fault is the amount, not the prior broadcast. Presented once more at `accepted.amount` = `1`, the block is refused as already confirmed and the refusal carries the hash.

| Step | Presented as | Verification check that fails | `invalidReason` / `errorReason` | `transaction` | Error Vocabulary |
|---|---|---|---|---|---|
| V13 | the confirmed 1-raw block with `accepted.amount` = `1000000000000000000000000000` | 9 (Previous Confirmation Check, amount clause) | `amount_mismatch` | `""` | `error_amount_mismatch` |
| V13, again at 1 raw | the same block with `accepted.amount` = `1` | 9 (Previous Confirmation Check) | `block_already_exists` | the block hash | `error_block_already_confirmed` |

<details>
<summary>Block, requests and responses for V13</summary>

The block (on the chain since the step at `accepted.amount` = `1`):

```json
{
  "type": "state",
  "account": "nano_1i3y944esngqw6wb6ia68dotj4yuqctch9kx8ct65twt8ewi4rdcfgax7ggf",
  "previous": "30CCB03D899B3197100E5322791E38371ED48ADF837819EA6CB1F63B34A7A1F2",
  "representative": "nano_3arg3asgtigae3xckabaaewkx3bzsh7nwz7jkmjos79ihyaxwphhm6qgjps4",
  "balance": "48738598785207476890320950700",
  "link": "776E05C7A2D3B5887DCE7EB2831078FCCDB9AED77EC6B27CDC05550B07D744A3",
  "link_as_account": "nano_1xug1q5t7nxoj3ywwzokiea9jz8fq8qfgzp8pbyfr3co3e5xgj755uofu8ue",
  "signature": "BCF4E3F690E94C5CD7A266C6091EFCA591040C9F49E9A65F744808B579AB9130F018D9CF910AA67C12ECAF78FE15D69FACB5818AA5538D53FB305AD2D6C4A401",
  "work": "9f3a100ca25db498"
}
```

`block_info` after settlement at 1 raw:

```json
{
  "confirmed": "true",
  "height": "118",
  "amount": "1",
  "subtype": "send",
  "block_account": "nano_1i3y944esngqw6wb6ia68dotj4yuqctch9kx8ct65twt8ewi4rdcfgax7ggf",
  "local_timestamp": "1791953315"
}
```

`/verify` request at the full price (the `PaymentPayload` carries the same block; `accepted.amount` and `paymentRequirements.amount` are `1000000000000000000000000000`):

```json
{
  "x402Version": 2,
  "paymentPayload": {
    "x402Version": 2,
    "resource": {
      "url": "https://pursekeeper.dev/v1/echo?msg=vector",
      "description": "returns what you sent (test your payment client)",
      "mimeType": "application/json"
    },
    "accepted": {
      "scheme": "exact",
      "network": "nano:mainnet",
      "asset": "XNO",
      "amount": "1000000000000000000000000000",
      "payTo": "nano_1xug1q5t7nxoj3ywwzokiea9jz8fq8qfgzp8pbyfr3co3e5xgj755uofu8ue",
      "maxTimeoutSeconds": 60
    },
    "payload": {
      "block": {
        "type": "state",
        "account": "nano_1i3y944esngqw6wb6ia68dotj4yuqctch9kx8ct65twt8ewi4rdcfgax7ggf",
        "previous": "30CCB03D899B3197100E5322791E38371ED48ADF837819EA6CB1F63B34A7A1F2",
        "representative": "nano_3arg3asgtigae3xckabaaewkx3bzsh7nwz7jkmjos79ihyaxwphhm6qgjps4",
        "balance": "48738598785207476890320950700",
        "link": "776E05C7A2D3B5887DCE7EB2831078FCCDB9AED77EC6B27CDC05550B07D744A3",
        "link_as_account": "nano_1xug1q5t7nxoj3ywwzokiea9jz8fq8qfgzp8pbyfr3co3e5xgj755uofu8ue",
        "signature": "BCF4E3F690E94C5CD7A266C6091EFCA591040C9F49E9A65F744808B579AB9130F018D9CF910AA67C12ECAF78FE15D69FACB5818AA5538D53FB305AD2D6C4A401",
        "work": "9f3a100ca25db498"
      }
    }
  },
  "paymentRequirements": {
    "scheme": "exact",
    "network": "nano:mainnet",
    "asset": "XNO",
    "amount": "1000000000000000000000000000",
    "payTo": "nano_1xug1q5t7nxoj3ywwzokiea9jz8fq8qfgzp8pbyfr3co3e5xgj755uofu8ue",
    "maxTimeoutSeconds": 60
  }
}
```

`/verify` response:

```json
{
  "isValid": false,
  "invalidReason": "amount_mismatch",
  "detail": "the block on the chain sends 1 raw; exactly 1000000000000000000000000000 raw required",
  "payer": "nano_1i3y944esngqw6wb6ia68dotj4yuqctch9kx8ct65twt8ewi4rdcfgax7ggf"
}
```

`/settle` response for the same body:

```json
{
  "success": false,
  "errorReason": "amount_mismatch",
  "detail": "the block on the chain sends 1 raw; exactly 1000000000000000000000000000 raw required",
  "transaction": "",
  "network": "nano:mainnet",
  "payer": "nano_1i3y944esngqw6wb6ia68dotj4yuqctch9kx8ct65twt8ewi4rdcfgax7ggf"
}
```

The same block presented again at `accepted.amount` = `1`. `/verify`:

```json
{
  "isValid": false,
  "invalidReason": "block_already_exists",
  "detail": "this block was already used to pay",
  "payer": "nano_1i3y944esngqw6wb6ia68dotj4yuqctch9kx8ct65twt8ewi4rdcfgax7ggf"
}
```

`/settle`:

```json
{
  "success": false,
  "errorReason": "block_already_exists",
  "detail": "block 34845D9CCF5F574A3CF01A77ACB3B02AC970FB5745B3A317E35524BAF62E2472 was settled through this facilitator for the same payTo and amount and answered success at 2026-10-14T04:48:35.557Z; a second success is not given for the same block",
  "transaction": "34845D9CCF5F574A3CF01A77ACB3B02AC970FB5745B3A317E35524BAF62E2472",
  "network": "nano:mainnet",
  "payer": "nano_1i3y944esngqw6wb6ia68dotj4yuqctch9kx8ct65twt8ewi4rdcfgax7ggf"
}
```

</details>

### V14: `workOptional`, the V1 block with the sixteen-zero work placeholder

The V1 block again, with `"work": "0000000000000000"` in the `PaymentPayload` (the placeholder from **Who should generate work for the send block?** in **Appendix**) and `extra.workOptional: true` in `PaymentRequirements`. Nothing else differs from V1. Generated 2026-10-27; broadcasts nothing, since the V1 block is already confirmed.

`PaymentPayload.payload.block`:

```json
{
  "type": "state",
  "account": "nano_1i3y944esngqw6wb6ia68dotj4yuqctch9kx8ct65twt8ewi4rdcfgax7ggf",
  "previous": "DE891E21F480F772E68B61B1C7D8BF6525E5D9FAE64AD5B7BC71F9486348B943",
  "representative": "nano_3arg3asgtigae3xckabaaewkx3bzsh7nwz7jkmjos79ihyaxwphhm6qgjps4",
  "balance": "9787318189447476890320950701",
  "link": "776E05C7A2D3B5887DCE7EB2831078FCCDB9AED77EC6B27CDC05550B07D744A3",
  "link_as_account": "nano_1xug1q5t7nxoj3ywwzokiea9jz8fq8qfgzp8pbyfr3co3e5xgj755uofu8ue",
  "signature": "65AFBF7622955A1D6E5E0703B479C33BA652CB4C29477968D2E8864891124875C3EA3F658F597CDB689086F2DE1638BAC0847CFC05CF682682967927D465FB03",
  "work": "0000000000000000"
}
```

What a verifier can check offline, computed with the same library as V1:

- The block hash is `1D60903EFD920FA78746E18F08C021EE7B8155205709DF5A4125BBDCF96D0BEB`, identical to V1's: `work` is not an input to the hash.
- V1's `signature` verifies against that hash and the `account` public key unchanged: the Client signed nothing about the work.
- `"0000000000000000"` does not meet the send threshold `fffffff800000000` over `previous`; V1's `"3f6ff9e7b7ba27a7"` does. The placeholder is never valid work, so a broadcaster that forwards it unchanged gets the node's refusal, and the draft's "MUST replace before `process`" is what makes the flag usable.

The `process` request the broadcaster sends, with the work it generated over `previous` in place of the placeholder. The `PaymentPayload` the Client sent is not touched:

```json
{
  "action": "process",
  "subtype": "send",
  "json_block": "true",
  "block": {
    "type": "state",
    "account": "nano_1i3y944esngqw6wb6ia68dotj4yuqctch9kx8ct65twt8ewi4rdcfgax7ggf",
    "previous": "DE891E21F480F772E68B61B1C7D8BF6525E5D9FAE64AD5B7BC71F9486348B943",
    "representative": "nano_3arg3asgtigae3xckabaaewkx3bzsh7nwz7jkmjos79ihyaxwphhm6qgjps4",
    "balance": "9787318189447476890320950701",
    "link": "776E05C7A2D3B5887DCE7EB2831078FCCDB9AED77EC6B27CDC05550B07D744A3",
    "link_as_account": "nano_1xug1q5t7nxoj3ywwzokiea9jz8fq8qfgzp8pbyfr3co3e5xgj755uofu8ue",
    "signature": "65AFBF7622955A1D6E5E0703B479C33BA652CB4C29477968D2E8864891124875C3EA3F658F597CDB689086F2DE1638BAC0847CFC05CF682682967927D465FB03",
    "work": "3f6ff9e7b7ba27a7"
  }
}
```

Presented live to a Facilitator that advertises `workOptional: false`, the `/verify` answer is from check 9, not check 13: the Facilitator computes the hash from the placeholder payload, finds the confirmed V1 block, and names it, `transaction` included on `/settle`. The work field is read only after the chain has been asked.

`/verify`:

```json
{
  "isValid": false,
  "invalidReason": "block_already_exists",
  "detail": "block already on the chain and confirmed",
  "payer": "nano_1i3y944esngqw6wb6ia68dotj4yuqctch9kx8ct65twt8ewi4rdcfgax7ggf",
  "invalidMessage": "block already on the chain and confirmed"
}
```

`/settle`:

```json
{
  "success": false,
  "errorReason": "block_already_exists",
  "detail": "block already on the chain and confirmed (hash 1D60903EFD920FA78746E18F08C021EE7B8155205709DF5A4125BBDCF96D0BEB, not broadcast through this facilitator)",
  "transaction": "1D60903EFD920FA78746E18F08C021EE7B8155205709DF5A4125BBDCF96D0BEB",
  "network": "nano:mainnet",
  "payer": "nano_1i3y944esngqw6wb6ia68dotj4yuqctch9kx8ct65twt8ewi4rdcfgax7ggf",
  "errorMessage": "block already on the chain and confirmed (hash 1D60903EFD920FA78746E18F08C021EE7B8155205709DF5A4125BBDCF96D0BEB, not broadcast through this facilitator)"
}
```

In the Error Vocabulary the code is `error_block_already_confirmed`, as in V2. A Facilitator that advertises `workOptional: true` and holds no such block would instead defer check 13, run checks 1 to 12, generate work over `previous` at Settlement and `process` the block above with that work.

### V15: a block built on an unconfirmed frontier, refused and then accepted unchanged (check 11)

Generated on 2026-10-28. Two 1-raw sends from the payer to `payTo`, both signed and worked before anything was broadcast: block A on the payer's confirmed frontier (`D7A5CC4E527C53A127067A8D336BDA6DBDFC9C79E53D104ED8A3A12EAAE2A1F8`, height 131), block B on A's hash. A was broadcast straight to the node with [process] (not through the facilitator), and from the moment the node answered (34 ms) B's `PaymentPayload` was presented to `/verify` every 40 ms, with `account_info` (`include_confirmed`) read beside each probe. Four probes, at 34, 205, 272 and 352 ms after broadcast, were refused: `frontier` was already A while `confirmed_frontier` was still height 131. The fifth probe, at 410 ms, was accepted with the same bytes, `confirmed_frontier` now A, and B was settled through `/settle` unchanged (success in 548 ms). Both blocks are confirmed at heights 132 and 133. The window is the node's confirmation time, under half a second here; a **`Client`** that had rebuilt B on `confirmed_frontier` during those four probes would have signed a fork of A.

| Vector | Input | Check | `invalidReason` (this facilitator) | `transaction` | Error Vocabulary |
|---|---|---|---|---|---|
| V15, 34 to 352 ms | B (`previous` = A's hash) while A is the unconfirmed frontier | 11 (Frontier Hash Check, second condition) | `frontier_unconfirmed` | `""` | `error_frontier_not_confirmed` |
| V15, 410 ms | the same `PaymentPayload` unchanged, A confirmed | passes; settled | | B's hash | |

<details>
<summary>Block B, the refusal, the acceptance and the settlement</summary>

`PaymentPayload.payload.block` (B):

```json
{
  "type": "state",
  "account": "nano_1i3y944esngqw6wb6ia68dotj4yuqctch9kx8ct65twt8ewi4rdcfgax7ggf",
  "previous": "C1DB4769A2879E49F7311484BD5E80117DB516C46EF8CD16F1A9499C8792E196",
  "representative": "nano_3arg3asgtigae3xckabaaewkx3bzsh7nwz7jkmjos79ihyaxwphhm6qgjps4",
  "balance": "43317291780736537993152545307",
  "link": "776E05C7A2D3B5887DCE7EB2831078FCCDB9AED77EC6B27CDC05550B07D744A3",
  "link_as_account": "nano_1xug1q5t7nxoj3ywwzokiea9jz8fq8qfgzp8pbyfr3co3e5xgj755uofu8ue",
  "signature": "E1850FC859D5078D5D913B33FCE891A5B346B3655FFB442BEDDFE969F30ACF45D7B6095CA939658065F4C80E1F7456ACB89C4B14AC8E6E7D7F1A9D99B3B8BD0C",
  "work": "885456feea5b2c1d"
}
```

`account_info` for the payer at the first probe, 34 ms after A was broadcast (fields not used by the scheme omitted):

```json
{
  "frontier": "C1DB4769A2879E49F7311484BD5E80117DB516C46EF8CD16F1A9499C8792E196",
  "confirmed_frontier": "D7A5CC4E527C53A127067A8D336BDA6DBDFC9C79E53D104ED8A3A12EAAE2A1F8",
  "balance": "43317291780736537993152545308",
  "confirmed_balance": "43317291780736537993152545309",
  "confirmed_height": "131",
  "block_count": "132"
}
```

`/verify` at 34 ms (and at 205, 272 and 352 ms, byte-identical):

```json
{
  "isValid": false,
  "invalidReason": "frontier_unconfirmed",
  "payer": "nano_1i3y944esngqw6wb6ia68dotj4yuqctch9kx8ct65twt8ewi4rdcfgax7ggf"
}
```

`account_info` at the fifth probe, 410 ms:

```json
{
  "frontier": "C1DB4769A2879E49F7311484BD5E80117DB516C46EF8CD16F1A9499C8792E196",
  "confirmed_frontier": "C1DB4769A2879E49F7311484BD5E80117DB516C46EF8CD16F1A9499C8792E196",
  "balance": "43317291780736537993152545308",
  "confirmed_balance": "43317291780736537993152545308",
  "confirmed_height": "132",
  "block_count": "132"
}
```

`/verify` at 410 ms, same request:

```json
{
  "isValid": true,
  "payer": "nano_1i3y944esngqw6wb6ia68dotj4yuqctch9kx8ct65twt8ewi4rdcfgax7ggf"
}
```

`/settle`, same request:

```json
{
  "success": true,
  "transaction": "CE21A65EA5DEA3ED96882E731218A5B3F90E35E4C6F7C54EA8711A14369B623E",
  "network": "nano:mainnet",
  "payer": "nano_1i3y944esngqw6wb6ia68dotj4yuqctch9kx8ct65twt8ewi4rdcfgax7ggf"
}
```

[block_info] afterwards: A `C1DB4769A2879E49F7311484BD5E80117DB516C46EF8CD16F1A9499C8792E196` height 132, `subtype` `send`, `amount` `1`, `confirmed` `true`; B `CE21A65EA5DEA3ED96882E731218A5B3F90E35E4C6F7C54EA8711A14369B623E` height 133, the same.

</details>

In the Error Vocabulary the refusal is `error_frontier_not_confirmed`; this facilitator's `frontier_unconfirmed` predates the vocabulary. The Balance Check (12) did not run during the window: check 11 refused first, so `confirmed_balance` was never compared with B's `balance`. The `/verify` answers above are shown with the `detail` and `invalidMessage` fields omitted; the capture file has them.

## Appendix

### Fork protection (recommended practice for **`Resource Server`** and **`Facilitator`**)

**`Resource Server`** SHOULD temporarily store the `PaymentPayload.payload.block.previous` value of any ongoing Verification / Settlement attempt. If any further payment attempt reaches the **`Resource Server`** with the same `PaymentPayload.payload.block.previous` field, the **`Resource Server`** should reject it. Once the ongoing payment attempt is verified and settled, the **`Resource Server`** no longer needs to store the `PaymentPayload.payload.block.previous` value.

This precaution prevents accidental or malicious fork submission. Forks are blocks with the same `previous` value.

If two blocks with the same `previous` value are broadcast to the Nano network, the first block to reach quorum among Nano network representatives will be confirmed onto the Nano network. The other block gets rejected forever.

### Blocks have no expiry

Valid blocks that have been broadcast by the **`Facilitator`** but not yet confirmed do not expire. They exist on the network until confirmed, which typically takes about half a second. `PaymentRequirements.maxTimeoutSeconds` bounds the **`Facilitator`**'s polling period for confirmation of the send block, not expiration.

If `PaymentRequirements.maxTimeoutSeconds` expires, the Unsuccessful Settlement Response returned by **`Facilitator`** MUST have `error_timeout_expired` in `errorReason` (see **Error Vocabulary**) and the send block's hash in `transaction`.

```json
{
  "success": false,
  "errorReason": "error_timeout_expired",
  "transaction": "0DF7EC8E0955E2A242E3C68B36CBDD639FBB56914AF63AE87295F17D30D9C9D4",
  "payer": "nano_3qgmh14nwztqw4wmcdzy4xpqeejey68chx6nciczwn9abji7ihhum9qtpmdr",
  "network": "nano:mainnet"
}
```

If `PaymentRequirements.maxTimeoutSeconds` expires or if the Settlement status for a [send block] needs to be re-queried, make another Settlement attempt (resettlement) of the `PaymentPayload` containing the send block and read the `errorReason` and `transaction` fields of the Settlement Response.

### When should **`Resource Server`** deliver the resource to **`Client`**?

When a **`Resource Server`** wants to determine whether a resource should be delivered to the **`Client`**, there are three individual **Settlement Response** scenarios of note:

#### Scenario 1

```json
{
  "success": true,
  "transaction": "0DF7EC8E0955E2A242E3C68B36CBDD639FBB56914AF63AE87295F17D30D9C9D4",
  "payer": "nano_3qgmh14nwztqw4wmcdzy4xpqeejey68chx6nciczwn9abji7ihhum9qtpmdr",
  "network": "nano:mainnet"
}
```

- `success` equals `true`

Send block has been broadcast and confirmed on the Nano network. `transaction` contains the send block's hash.

Resource MUST be delivered to **`Client`** and **`Resource Server`** MUST record the send block's hash as served at the moment of delivery because _Scenario 2_ reads that record. If multiple **`Resource Servers`** share the same `payTo` destination, they SHOULD record in a unified database keyed by the send block's hash to ensure delivery tracking is shared across all processes.

_Scenario 1_ can only happen once per send block.

#### Scenario 2

```json
{
  "success": false,
  "errorReason": "error_block_already_confirmed",
  "transaction": "0DF7EC8E0955E2A242E3C68B36CBDD639FBB56914AF63AE87295F17D30D9C9D4",
  "payer": "nano_3qgmh14nwztqw4wmcdzy4xpqeejey68chx6nciczwn9abji7ihhum9qtpmdr",
  "network": "nano:mainnet"
}
```

- `success` equals `false`
- `errorReason` equals `error_block_already_confirmed`

Send block has previously been broadcast and confirmed on the Nano network. `transaction` contains the send block's hash.

This response does not tell the **`Resource Server`** whether the resource was already delivered to the **`Client`**. **`Resource Server`** MUST deliver to **`Client`** only if its own records show that the resource has not yet been delivered to **`Client`**; otherwise this is a replay and the resource MUST NOT be delivered again.

However if resource had not been previously delivered, **`Resource Server`** SHOULD now deliver the resource and record the send block's hash as served at the moment of delivery. If multiple **`Resource Servers`** share the same `payTo` destination, they SHOULD record in a unified database keyed by the send block's hash to ensure delivery tracking is shared across all processes.

#### Scenario 3

```json
{
  "success": false,
  "errorReason": "error_block_awaiting_confirmation",
  "transaction": "0DF7EC8E0955E2A242E3C68B36CBDD639FBB56914AF63AE87295F17D30D9C9D4",
  "payer": "nano_3qgmh14nwztqw4wmcdzy4xpqeejey68chx6nciczwn9abji7ihhum9qtpmdr",
  "network": "nano:mainnet"
}
```

- `success` equals `false`
- `errorReason` equals `error_block_awaiting_confirmation`

Send block has been broadcast but not yet confirmed on the Nano network. `transaction` contains the send block's hash (see **Blocks have no expiry** in **Appendix**).

Resource MUST NOT be delivered to **`Client`**, until a resettlement encounters _Scenario 2_ and the **`Resource Server`**'s own records show that the resource has not yet been delivered to **`Client`**.

If the block is still awaiting confirmation after a bound of the **`Resource Server`**'s choosing, it SHOULD stop resettling and answer 402 again; the **`Client`**'s remedy is a new send block. If the earlier block confirms later, it is a payment to `payTo` and the **`Client`** may present it under _Scenario 2_.

### Who should generate work for the send block?

Normally it is expected that **`Client`** will generate the work that goes into `PaymentPayload.payload.block.work`. However to help in situations where **`Client`** does not have access to work generation, **`Facilitator`** can signal that it can generate the work that goes into `PaymentPayload.payload.block.work`.

Any work generated MUST use `PaymentPayload.payload.block.previous` as input and MUST be generated at the send difficulty threshold documented by the Nano node ([difficulty thresholds]); which at the time of writing is `fffffff800000000` on `nano:mainnet`.

To be eligible for work generated by **`Facilitator`**, **`Client`** MUST have `PaymentPayload.payload.block.work` set to a placeholder value of `"0000000000000000"` (a string with sixteen zeroes). The hash for `PaymentPayload.payload.block` does not cover work, so **`Facilitator`** replacing the placeholder work value later changes nothing the Client signed.

**`Facilitator`** signals work generation capabilities by setting `extra.workOptional: true` in a `SupportedKind` object in the kinds array of the `SupportedResponse`.

```json
{
  "kinds": [
    {
      "x402Version": 2,
      "scheme": "exact",
      "network": "nano:mainnet",
      "extra": {
        "workOptional": true
      }
    }
  ]
}
```

If `extra.workOptional: true` in `SupportedKind`, **`Facilitator`** MUST generate work for `PaymentPayload.payload.block.work` only during Settlement, and if `PaymentPayload.payload.block.work` is set to `"0000000000000000"`.

If **`Facilitator`** fails to generate work, it MUST set `errorReason` to `error_work_unavailable` in the Unsuccessful Settlement Response.

A **`Resource Server`**, whose **`Facilitator`** has `extra.workOptional: true` in `SupportedKind` MUST set `extra.workOptional: true` in `PaymentRequirements`. To the **`Client`** the `extra.workOptional: true` flag in `PaymentRequirements` means that someone downstream will generate the work; it need not know who.

If **`Facilitator`** returns `error_work_unavailable` in `errorReason` of an Unsuccessful Settlement Response, **`Resource Server`** MUST communicate this to **`Client`** by returning a fresh `402 Payment Required` status and setting `PaymentRequired.error` to `error_work_unavailable` (see **Error Vocabulary**).

**`Client`** MAY then generate work, set it to `PaymentPayload.payload.block.work` and re-present the same signed send block (along with original `PaymentRequirements`) to **`Resource Server`**.

**`Facilitator`** MUST be aware of the possibility for incoming payments to exhaust their work generation capabilities and MUST plan for this event. This is implementation-specific and out of the scope of this document, but some ideas for reducing the chance of work generation capability exhaustion are:

- Limiting the number of concurrent work generations
- Caching generated work keyed by `PaymentPayload.payload.block.previous` values for a period of time, or until the associated payment is complete
- Rate limiting by `PaymentPayload.payload.block.account`
- Rate limiting by IP address

[send block]: https://docs.nano.org/protocol-design/blocks/#state-blocks
[raw units]: https://docs.nano.org/integration-guides/the-basics/#units
[work]: https://docs.nano.org/protocol-design/spam-work-and-prioritization/#work-algorithm-details
[account_info]: https://docs.nano.org/commands/rpc-protocol/#account_info
[block_info]: https://docs.nano.org/commands/rpc-protocol/#block_info
[process]: https://docs.nano.org/commands/rpc-protocol/#process
[difficulty thresholds]: https://docs.nano.org/integration-guides/work-generation/#difficulty-thresholds
[computing the hash]: https://docs.nano.org/integration-guides/the-basics/#self-signed-blocks
