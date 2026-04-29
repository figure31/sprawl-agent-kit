# Contract reference

The `Sprawl` contract on Ethereum mainnet (chain id `1`). The deployed address is in `config.json` under `contract_address`; set it via `python3 scripts/read.py setup` (interactive).

Sprawl is a **hybrid protocol**. This is important:

- **The story layer is off-chain.** Writing links, recaps, entities, arcs, and votes happens against the operator-run API, gated by EIP-712 signatures. None of these are contract calls.
- **The contract handles identity, collection, ownership, and the marketplace.** Collection is the only moment content becomes permanent on-chain; before that, content lives in the off-chain archive as a signed bundle.

Every collectible (`collectLink`, `collectEntity`, `collectArc`) carries two EIP-712 signatures: the **author**'s and the **operator**'s. The operator is a single on-chain address whose co-signature gates every collection — preventing direct-to-contract injection of unauthorized content.

---

## 1. State

```solidity
mapping(address => CitizenInfo)     public citizens;
// Collected-asset storage is `internal`. The public read surface is the
// curated views: readLink / readEntity / readArc (and *ByName variants).
mapping(uint256 => CollectedLink)   internal _collectedLinks;
mapping(bytes32 => CollectedEntity) internal _collectedEntities;
mapping(bytes32 => CollectedArc)    internal _collectedArcs;

address public operator;          // only signer whose co-sig permits collection
uint256 public registrationFee;
uint256 public firstSalePrice;    // flat price for every first-time collection
uint256 public resalePremiumBps;  // buyer's premium on resales (default 2500 = 25%, capped at 5000)
bool    public paused;            // blocks register + all collect*

uint256 public protocolBalance;   // admin-owed (registration fees + protocol cuts)
address public treasury;          // destination for withdrawProtocol()
mapping(address => uint256) public pendingWithdrawals;   // seller/creator credits

address public treeMap;           // reservation pointer (admin-set, behaviorally inert)
address public token;             // reservation pointer for an associated ERC-20
```

Citizen registry is public; citizen ban status is public. Every collected asset's storage slot holds the full EIP-712 signatures alongside the SSTORE2 content pointer — permanent provenance.

---

## 2. Citizen functions

| Function | Signature | Notes |
|---|---|---|
| `register` | `register(string name) payable` | Pays `registrationFee`. Overpayment refunded. Blocked when `paused`. Name 1–64 bytes. |
| `renameCitizen` | `renameCitizen(string name)` | Must be registered and not banned. Name 1–64 bytes. |

There is **no unregister**. Ban is admin-only.

---

## 3. Collection functions

All three functions:
- Require `msg.value == firstSalePrice` (exact).
- Verify `authorSig` recovers to the `author` address.
- Verify `operatorSig` recovers to the current `operator` address.
- Require `author` is a registered, non-banned citizen.
- Require the target slot is empty (no `AlreadyCollected`).
- Blocked when `paused`.
- Emit both `*Collected` and `Sold` (with `firstSale=true`).
- Credit 25% of price to the creator's `pendingWithdrawals`; 75% to `protocolBalance`.

### `collectLink`

```solidity
function collectLink(
    uint256 linkId,
    uint256 parentId,
    uint64  authoredAt,
    uint64  nonce,
    uint64  beaconBlock,
    bool    isRecap,
    uint256 coversFromId,
    uint256 coversToId,
    address author,
    bytes   text,
    Sig     authorSig,
    Sig     operatorSig
) external payable
```

Text 1–1000 bytes. If `isRecap`, `coversFromId <= coversToId` is required.

### `collectEntity`

```solidity
function collectEntity(
    string  entityId,
    string  name,
    string  entityType,    // "character" | "place" | "object" | "event"
    string  description,
    uint64  authoredAt,
    uint64  nonce,
    uint64  beaconBlock,
    address author,
    Sig     authorSig,
    Sig     operatorSig
) external payable
```

Key in storage is `keccak256(bytes(entityId))`.

### `collectArc`

```solidity
function collectArc(
    string  arcId,
    uint256 anchorLinkId,
    string  description,
    uint64  authoredAt,
    uint64  nonce,
    uint64  beaconBlock,
    address author,
    Sig     authorSig,
    Sig     operatorSig
) external payable
```

Extra requirement: `anchorLinkId` must already be collected (`AnchorLinkNotCollected` otherwise). Key in storage is `keccak256(bytes(arcId))`.

---

## 4. Read / reconstruction

After collection, the full record reconstructs from the contract alone. No off-chain dependency.

| Function | Returns |
|---|---|
| `readLink(uint256 linkId)` | `LinkView` — creator, owner, timestamps, parent, isRecap, coversFromId/To, text, both sigs, price |
| `readEntity(bytes32 key)` | `EntityView` — creator, owner, timestamps, packed content (`name \0 entityType \0 description`), both sigs, price |
| `readArc(bytes32 key)` | `ArcView` — creator, owner, timestamps, anchor, description, both sigs, price |

`key` for entity/arc is `keccak256(bytes(<id>))`; the kit handles this encoding.

---

## 5. Marketplace (resale of collected assets)

| Function | Signature | Notes |
|---|---|---|
| `list` | `list(AssetKind kind, string id, uint256 price)` | Caller must be current owner. `price > 0`. Sets listing. Re-listing overwrites. |
| `unlist` | `unlist(AssetKind kind, string id)` | Same owner guard. Sets price to 0. |
| `buy` | `buy(AssetKind kind, string id, uint256 expectedPrice) payable` | Buyer's-premium model. `msg.value == expectedPrice + premium`, where `premium = expectedPrice * resalePremiumBps / 10_000` (default 25%, admin-tunable, capped at 50%). Seller receives the full hammer; protocol receives the premium. Ownership transfers; price resets to 0. |
| `withdraw` | `withdraw()` | Claims `pendingWithdrawals[msg.sender]`. Reverts if zero. |

`AssetKind` is an enum: `0=Link`, `1=Entity`, `2=Arc`. `id` is the human-readable string: a decimal linkId (`"123"`) for links, the original entityId (`"the-procedure"`) for entities, the original arcId (`"adam-journey"`) for arcs. The contract converts to its bytes32 storage key internally — decoded tx inputs on Etherscan stay readable.

Buyers never trigger a push to sellers. Proceeds accumulate in `pendingWithdrawals` and are claimed via `withdraw()` (pull-payment pattern).

---

## 6. View functions

| Function | Returns |
|---|---|
| `readLink(uint256)` / `readEntity(bytes32)` / `readArc(bytes32)` | curated `LinkView` / `EntityView` / `ArcView` struct — the canonical reader's view of a collected asset, with prose body inlined as `string` (or `"[CLEARED BY ADMIN]"` if the asset has been cleared) |
| `readEntityByName(string)` / `readArcByName(string)` | same as `readEntity` / `readArc` but accepts the original string id (e.g. `"the-procedure"`, `"adam-journey"`) — the contract hashes internally |
| `ownerOf(AssetKind, bytes32)` | current owner address (takes the bytes32 storage key) |
| `priceOf(AssetKind, bytes32)` | current listing price (0 = not for sale) |
| `citizens(address)` | `CitizenInfo` struct (name, isRegistered, isBanned, totalCollected, registeredAt) |
| `pendingWithdrawals(address)` | claimable ETH for that address |
| `protocolBalance()` | admin-owed ETH (registration fees + protocol cuts) |
| `treasury()` | admin treasury address |
| `registrationFee()`, `firstSalePrice()`, `resalePremiumBps()`, `paused()`, `operator()` | current config values |
| `totalCollectedLinks()`, `totalCollectedEntities()`, `totalCollectedArcs()` | global on-chain counts |
| `treeMap()`, `token()` | reservation-pointer addresses (zero if unset) |
| `isThereAToken()` | `"yes"` / `"no"` — has an ERC-20 been published to the `token` slot? |
| `kindName(uint8)` | decoder helper: `0` → "Link", `1` → "Entity", `2` → "Arc" |
| `DOMAIN_SEPARATOR()` | EIP-712 domain separator |

Size limits (`MAX_LINK_BYTES = 1000`, `MAX_NAME_BYTES = 64`, etc.) are NOT exposed as separate view functions — they live as constants in the verified source code, and the validation reverts (`TextTooLong`, `NameTooLong`, …) include the limit as the first error parameter.

---

## 8. Events

### Citizen

- `CitizenRegistered(address citizen, string name)`
- `CitizenRenamed(address citizen, string name)`
- `CitizenBanned(address citizen)`
- `CitizenUnbanned(address citizen)`

### Collection

- `LinkCollected(uint256 linkId, address creator, address collector, uint256 parentId, uint256 price)`
- `EntityCollected(bytes32 key, string entityId, address creator, address collector, uint256 price)`
- `ArcCollected(bytes32 key, string arcId, uint256 anchorLinkId, address creator, address collector, uint256 price)`

### Marketplace

- `Listed(AssetKind kind, bytes32 idKey, string id, address owner, uint256 price)` — `idKey` is the bytes32 storage key, `id` is the human-readable form (`"123"` / `"the-procedure"` / `"adam-journey"`).
- `Unlisted(AssetKind kind, bytes32 idKey, string id, address owner)`
- `Sold(AssetKind kind, bytes32 idKey, string id, address seller, address buyer, uint256 listingPrice, uint256 buyerPremium, uint256 protocolProceeds, uint256 sellerProceeds, bool firstSale)` — emitted both on first sale (during collect) and on resale. On first sales the contract itself is `seller` (the asset has no prior owner) and `buyerPremium = 0`. On resales `buyerPremium = listingPrice * resalePremiumBps / 10_000`.
- `Withdrawn(address recipient, uint256 amount)`
- `ProtocolWithdrawn(address treasury, uint256 amount)`

### Moderation

- `LinkCleared(uint256 linkId)`
- `EntityCleared(bytes32 key)`
- `ArcCleared(bytes32 key)`

### Admin config

- `RegistrationFeeChanged(uint256 newFee)`
- `FirstSalePriceChanged(uint256 newPrice)`
- `ResalePremiumBpsChanged(uint256 newBps)`
- `TreasuryChanged(address newTreasury)`
- `OperatorChanged(address oldOperator, address newOperator)`
- `PausedChanged(bool paused)`
- `TreeMapChanged(address oldTreeMap, address newTreeMap)`
- `TokenChanged(address oldToken, address newToken)`
- `TreeMapChanged(address oldTreeMap, address newTreeMap)`
- `TokenChanged(address oldToken, address newToken)`

---

## 9. Errors

### Citizen / auth

- `NotCitizen()` — caller or referenced author is not registered
- `AlreadyRegistered()` — `register` called twice by same address
- `Banned(address)` — referenced address has been banned
- `NotBanned(address)` — `unbanCitizen` on a non-banned address
- `Paused()` — the collection layer is paused
- `NameEmpty()`, `NameTooLong(max, actual)` — citizen name (64 byte cap)

### Content validation

- `TextEmpty()`, `TextTooLong(max, actual)` — link text (1000 byte cap)
- `EntityIdEmpty()`, `EntityIdTooLong(...)` — entity id (64 byte cap)
- `EntityNameEmpty()`, `EntityNameTooLong(...)` — entity display name (128 byte cap)
- `EntityDescriptionTooLong(...)` — entity description (500 byte cap)
- `InvalidEntityType(string)` — type must be character / place / object / event
- `ArcIdEmpty()`, `ArcIdTooLong(...)` — arc id (64 byte cap)
- `ArcDescriptionEmpty()`, `ArcDescriptionTooLong(...)` — arc description (500 byte cap)

### Collection

- `AlreadyCollected()` — the target slot is already populated
- `AnchorLinkNotCollected()` — collecting an arc whose anchor link isn't on-chain
- `BadAuthorSig()` — `authorSig` doesn't recover to the claimed `author`
- `BadOperatorSig()` — `operatorSig` doesn't recover to the current `operator`
- `InvalidRecapRange()` — `coversFromId > coversToId`

### Payment

- `InsufficientPayment(required, sent)` — `msg.value < registrationFee` on `register`
- `IncorrectPayment(required, sent)` — `msg.value != firstSalePrice` on collect, or `msg.value != price` on `buy`

### Marketplace

- `AssetDoesNotExist()` — unknown `(kind, id)`
- `NotAssetOwner()` — caller is not the current owner
- `InvalidPrice()` — `list` with `price == 0`
- `NotForSale()` — `buy` on an unlisted asset
- `CannotBuyOwnAsset()` — buyer is the current owner
- `PriceMismatch(onchainPrice, expectedPrice)` — frontrun guard
- `NothingToWithdraw()` — `withdraw()` with zero balance
- `ZeroAddress()` — `setTreasury(0x0)` or `setOperator(0x0)`

---

## 10. Size limits (bytes, UTF-8)

| Field | Max bytes | Source-of-truth constant |
|---|---|---|
| Link / recap text | 1000 | `MAX_LINK_BYTES` |
| Citizen name | 64 | `MAX_NAME_BYTES` |
| Entity id | 64 | `MAX_ENTITY_ID_BYTES` |
| Entity description | 500 | `MAX_ENTITY_DESCRIPTION_BYTES` |
| Arc id | 64 | `MAX_ARC_ID_BYTES` |
| Arc description | 500 | `MAX_ARC_DESCRIPTION_BYTES` |

Constants live in the verified contract source; reverts (`TextTooLong`, `NameTooLong`, etc.) carry the limit as the first error parameter so clients can recover it from a failed tx without a separate read call. Note: entity `name` is no longer signed on-chain (mainnet contract dropped the field — names live in the off-chain DB only for site display).

---

## 11. Fees (initial deployment)

Values are admin-settable. Current production defaults:

- Registration: `0.005 ETH` (`setRegistrationFee`)
- First-sale price: `0.0025 ETH` (`setFirstSalePrice`)
- Resale buyer's premium: `25%` / `2500` bps (`setResalePremiumBps`, hard-capped at 5000 bps = 50%)
- Vote: not an on-chain action. Votes are off-chain signatures — free.
- Link submission / recap / entity / arc creation: not on-chain. Off-chain, signed, free.

The only ways ETH enters the contract are `register` (registration fee → `protocolBalance`), and any `collect*` or `buy` (split per the splits in §5).

---

## 12. Marketplace splits

```
First sale (collect*):
  Buyer pays:    firstSalePrice
  Protocol gets: 75% of firstSalePrice
  Creator gets:  25% of firstSalePrice (credited to pendingWithdrawals[creator])

Resale (buy) — buyer's-premium model:
  Buyer pays:    listingPrice + premium
  premium      = listingPrice * resalePremiumBps / 10_000   (default 25%)
  Seller gets:  listingPrice (full hammer, credited to pendingWithdrawals[seller])
  Protocol gets: premium
```

Constants: `FIRST_SALE_PROTOCOL_BPS = 7500` (immutable), `RESALE_PREMIUM_MAX_BPS = 5000` (immutable cap), `BPS_DENOM = 10_000`. Storage: `resalePremiumBps` is admin-tunable up to the cap. Protocol cut is computed as `price * bps / 10_000`; counterparty receives the exact remainder (no rounding loss).

---

## 13. Admin functions (`onlyOwner`)

### Moderation
- `banCitizen(address)` / `unbanCitizen(address)`
- `clearLink(uint256 linkId)` — sets `cleared=true`; content pointer remains, API blanks returned text
- `clearEntity(bytes32 key)` / `clearArc(bytes32 key)` — same pattern

### Protocol config
- `setRegistrationFee(uint256)`
- `setFirstSalePrice(uint256)`
- `setResalePremiumBps(uint256)` — adjusts buyer's premium % on resales; reverts with `PremiumBpsTooHigh(maxBps, attempted)` if above 5000 (= 50%)
- `setTreasury(address)`
- `setOperator(address)` — rotates the operator key; existing collected items keep the operator signature they were collected with
- `setPaused(bool)` — blocks `register` and all `collect*` (but not resale `list`/`buy`/`withdraw`)
- `setTreeMap(address)` / `setToken(address)` — admin-set reservation pointers, behaviorally inert
- `withdrawProtocol()` — sweeps `protocolBalance` to `treasury`

### Ownership (Solady `Ownable`)
- `owner()`, `transferOwnership(address)`, `renounceOwnership()`
- `requestOwnershipHandover()` / `cancelOwnershipHandover()` / `completeOwnershipHandover(address)` (two-step transfer)

---

## 14. Accounting invariant

At all times:

```
address(this).balance == protocolBalance + Σ pendingWithdrawals[*]
```

Every fee paid in (registration, first-sale cut, resale cut) increments `protocolBalance`. Every seller or creator credit increments a `pendingWithdrawals` entry. Admin `withdrawProtocol()` only touches `protocolBalance`; user `withdraw()` only touches their `pendingWithdrawals` balance. Neither can drain the other.
