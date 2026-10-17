# AskSwarmRegistry

Onchain memory of what the IMD swarm has attested. Anyone pays 0.5 IMD, the question goes to the
swarm through Intake v2 with this contract as callback, and the signed answer is verified and
stored under its question hash. Every answer anyone ever bought is then readable, free, by any
contract or client, with proof the swarm said it.

Built for the Identity MD hackathon by the AskSwarm team (Chromia, Eval Engine). The Telegram bot
[@askswarm_bot](https://t.me/askswarm_bot) and the Pay-with-IMD page route paid questions through
`ask`; the bot's free tier reads prior answers before charging anyone. Bot source:
[ChromiaAgents/askswarm](https://github.com/ChromiaAgents/askswarm).

## How it works

```
caller ──ask(body)──▶ AskSwarmRegistry ──requestWithFailure──▶ Intake v2 ──▶ swarm panel
                             ▲                                      │
                             └────── onOracleResult(id, a, sig) ◀───┘  (writer calls complete)
                                     verify EIP-712 ▸ store ▸ index by questionHash
```

1. `ask(bytes body)` pulls exactly `Intake.priceOf(action, IMD)` from the caller and forwards it in
   the same call. `body` is the oracle.request JSON the HTTP door takes. The contract never holds IMD
   between calls.
2. Intake registers this contract as callback (`onOracleResult`, selector `0x510379c7`) and failure
   hook (`onFailure`), and emits `Requested` with the body.
3. When the panel ends, the plane's writer calls `Intake.complete`, which calls back here with
   `(bytes32 requestId, OracleAttestation a, bytes signature)` under a fixed 200k gas stipend.
4. `onOracleResult` recomputes the EIP-712 digest under domain
   `{"IdentityMD Oracle", "2", chainId, address(this)}`, recovers the signer, and sets `verified` only
   when the attester matches, `agreed >= quorum`, and the attestation has not expired. The answer is
   stored in three slots (measured ~105k gas) and, if verified, becomes `latestFor[questionHash]`.
5. Refusals (status 1) and split panels (status 2) land in `onFailure` and are recorded by status, so
   a reader can tell "no consensus" from "never asked".

## Read it

| call | returns |
|---|---|
| `answerOf(bytes32 questionHash)` | latest verified answer for that question, with agreement and expiry |
| `get(bytes32 requestId)` | the ask (who, when, status) and its answer if any |
| `latestFor(bytes32 questionHash)` | the request id behind `answerOf` |
| `digest(OracleAttestation)` | the exact EIP-712 digest this contract verifies, for offline cross-checks |

Answers up to 32 bytes (bool, address, bytes32, uint256) are stored inline. Longer answers
(`address[]`, `bytes32[]`) are stored by keccak256 and emitted in full in the `Answered` event.

## Trust

- Only Intake can deliver, and only once per request. Intake only accepts `complete` from its writer.
- The signature check is belt and braces on top of that: a stored answer is `verified` only if the
  plane's attester key signed this exact attestation for this contract.
- No owner, no admin, no upgrade path, no ETH, no loops, no delegatecall. Intake and IMD addresses
  are immutable; Intake is deployed by CREATE2 at the same address on every chain.
- The contract pins nothing about which questions it accepts. Anyone can ask anything the swarm will
  take; readers should check `panelSize`, `agreed` and `expiresAt` for what they trust.

## Addresses

| chain | address |
|---|---|
| Sepolia (self-deployed, test) | `0x1d57A9aa2479fd115E3a1F545Ed5B8660548b900` |
| Ethereum mainnet | pending swarm launch |

Constructor `(intake, imd, signer)` = Intake v2 `0xa43e6f75ee006411f79ac1c84120606c2330de82`, IMD
`0xd34a99bc0f67ae1bbd63c660e6d0b0dd03e263b7` (IMDtest `0x44a1cd38474fb1748400e7deb5f8d786cce3f89a` on
Sepolia), attester `0x5598aa9146215bc13eb26f2c692ad1461fd32982`.

## Tests

Mainnet fork tests against the real Intake and IMD, impersonating Intake's writer to deliver results:

```
forge test --fork-url $ETH_RPC_URL -vv
```

Covers: price pull and registration, delivery inside the stipend with `delivered == true`, signature
tamper, wrong domain, quorum and expiry gating, failure hook, access control, 352-byte answers, and a
direct gas measurement of the callback. One test recovers the plane's real attester from a published
attestation (request `a9a6d603`, CHF, 11/15) to prove the hashing matches.

Note for Sepolia: contract creation there costs ~7x mainnet gas (repriced testnet fork); pass an
explicit gas limit of ~13M.

## Layout

```
src/AskSwarmRegistry.sol   the contract
src/IIntake.sol            the slice of Intake v2 it calls, and the OracleAttestation struct
test/                      fork tests
script/Deploy.s.sol        forge script, picks IMD by chain id
LAUNCH.md                  the brief given to the swarm for the evm_contracts launch
lib/forge-std              vendored, tests only
```

MIT.
