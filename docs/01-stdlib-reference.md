# Tolk Standard Library Reference

Complete reference for Tolk v1.2 standard library functions. This document covers all 6 stdlib modules with detailed function signatures, parameters, return values, and practical examples from real contracts.

## Table of Contents

1. [common.tolk - Core Functions](#commonfolk---core-functions)
2. [tvm-dicts.tolk - Dictionary Operations](#tvm-dictstolk---dictionary-operations)
3. [exotic-cells.tolk - Exotic Cell Handling](#exotic-cellstolk---exotic-cell-handling)
4. [gas-payments.tolk - Gas and Fee Calculations](#gas-paymentstolk---gas-and-fee-calculations)
5. [lisp-lists.tolk - Lisp-Style Lists](#lisp-liststolk---lisp-style-lists)
6. [tvm-lowlevel.tolk - Low-Level TVM Operations](#tvm-lowleveltolk---low-level-tvm-operations)

---

## common.tolk - Core Functions

The `common.tolk` module is automatically available and contains the most frequently used primitives for smart contract development.

### Built-in Types

#### Primitive Types

| Type           | Description                          | TVM Representation     |
| -------------- | ------------------------------------ | ---------------------- |
| `int`          | 257-bit signed integer               | Native integer         |
| `bool`         | Boolean: `true` or `false`           | -1 (true) or 0 (false) |
| `cell`         | Cell with up to 1023 bits and 4 refs | Native cell            |
| `slice`        | Cell opened for reading              | Cell slice             |
| `builder`      | Cell under construction              | Cell builder           |
| `continuation` | Executable TVM bytecode              | Continuation           |
| `tuple`        | Collection of 0-255 elements         | Tuple                  |
| `address`      | Internal address (workchain + hash)  | 267-bit slice          |
| `any_address`  | Internal/external/none address       | Slice                  |
| `void`         | Unit type (no value)                 | Empty                  |
| `never`        | Type for functions that never return | Special                |

#### Fixed-Width Integer Types

```tolk
int8, int16, int32, int64, int128, int256, int257  // Signed
uint8, uint16, uint32, uint64, uint128, uint256     // Unsigned
```

**Important**: Fixed-width types are still `int` at runtime. Overflow checking happens at serialization, not at assignment.

#### Special Types

```tolk
coins                    // Nanotoncoins (10^9 nanotons = 1 TON)
varint16, varint32      // Variadic signed integers
varuint16, varuint32    // Variadic unsigned integers
bits256, bits512        // Fixed-width slices with N bits and 0 refs
bytes8, bytes32         // Alias for bits(N*8)
```

#### Collection Types

```tolk
map<K, V>               // Typed dictionary (K must be fixed-width)
dict                    // Low-level TVM dictionary (cell?)
Cell<T>                 // Typed cell reference
```

**Example from jetton storage:**

```tolk
struct WalletStorage {
    jettonBalance: coins
    ownerAddress: address
    minterAddress: address
}
```

### Mathematical Functions

#### min / max

```tolk
@pure
fun min(x: int, y: int): int
```

Computes the minimum of two integers.

**Example:**

```tolk
var storageReserve = min(currentBalance, MIN_TONS_FOR_STORAGE);
```

---

```tolk
@pure
fun max(x: int, y: int): int
```

Computes the maximum of two integers.

---

#### minMax

```tolk
@pure
fun minMax(x: int, y: int): (int, int)
```

Sorts two integers, always returning `(smaller, larger)`.

**Example:**

```tolk
var (low, high) = minMax(price1, price2);
```

---

#### abs / sign

```tolk
@pure
fun abs(x: int): int
```

Returns the absolute value of an integer.

---

```tolk
@pure
fun sign(x: int): int
```

Returns `-1` if x < 0, `0` if x == 0, `1` if x > 0.

---

#### divMod / modDiv

```tolk
@pure
fun divMod(x: int, y: int): (int, int)
```

Returns `(quotient, remainder)` of x / y.

**Example:**

```tolk
var (days, hours) = divMod(totalHours, 24);
```

---

```tolk
@pure
fun modDiv(x: int, y: int): (int, int)
```

Returns `(remainder, quotient)` - same as divMod but swapped order.

---

#### High-Precision Multiplication

```tolk
@pure
fun mulDivFloor(x: int, y: int, z: int): int
```

Computes `floor(x * y / z)` using 513-bit intermediate result to prevent overflow.

**Example from fee calculation:**

```tolk
var proportionalFee = mulDivFloor(amount, feeNumerator, feeDenominator);
```

---

```tolk
@pure
fun mulDivRound(x: int, y: int, z: int): int
```

Computes `round(x * y / z)`.

---

```tolk
@pure
fun mulDivCeil(x: int, y: int, z: int): int
```

Computes `ceil(x * y / z)`.

---

```tolk
@pure
fun mulDivMod(x: int, y: int, z: int): (int, int)
```

Returns `(quotient, remainder)` of `(x * y / z)`.

---

#### ton()

```tolk
@pure
fun ton(floatString: slice): coins
```

Converts a constant floating-point string to nanotoncoins at compile-time.

**Examples:**

```tolk
const MIN_TONS_FOR_STORAGE = ton("0.01");      // 10000000 nanotons
const GAS_CONSUMPTION = ton("0.015");           // 15000000 nanotons
const FORWARD_AMOUNT = ton("0.05");             // 50000000 nanotons
```

**Note**: Only works with constant strings, not variables.

---

### Contract State Management

#### contract.getAddress

```tolk
@pure
fun contract.getAddress(): address
```

Returns the internal address of the current smart contract.

**Example from wallet:**

```tolk
var myWorkchain = contract.getAddress().getWorkchain();
```

---

#### contract.getOriginalBalance

```tolk
@pure
fun contract.getOriginalBalance(): coins
```

Returns the balance in nanotoncoins at the start of Computation Phase.

**Example from jetton wallet:**

```tolk
var tonBalanceBeforeMsg = contract.getOriginalBalance() - msgValue;
var storageFee = MIN_TONS_FOR_STORAGE - min(tonBalanceBeforeMsg, MIN_TONS_FOR_STORAGE);
```

---

#### contract.getOriginalBalanceWithExtraCurrencies

```tolk
@pure
fun contract.getOriginalBalanceWithExtraCurrencies(): [coins, ExtraCurrenciesMap]
```

Returns balance in TON and extra currencies as a tuple.

---

#### contract.getData / contract.setData

```tolk
@pure
fun contract.getData(): cell
```

Returns the persistent contract storage cell.

**Example pattern from jetton:**

```tolk
fun WalletStorage.load() {
    return WalletStorage.fromCell(contract.getData())
}
```

---

```tolk
fun contract.setData(c: cell): void
```

Sets the persistent contract storage.

**Example pattern:**

```tolk
fun WalletStorage.save(self) {
    contract.setData(self.toCell())
}
```

---

#### contract.getCode / contract.setCodePostponed

```tolk
@pure
fun contract.getCode(): cell
```

Retrieves the code of the smart contract from c7.

---

```tolk
fun contract.setCodePostponed(newCode: cell): void
```

Creates an output action to change contract code after successful termination.

---

### Blockchain State Getters

#### blockchain.now

```tolk
@pure
fun blockchain.now(): int
```

Returns current Unix timestamp in seconds.

**Example from wallet-v5:**

```tolk
assert (msg.validUntil > blockchain.now()) throw ERROR_EXPIRED;
```

---

#### blockchain.logicalTime

```tolk
@pure
fun blockchain.logicalTime(): int
```

Returns the logical time of the current transaction.

---

#### blockchain.currentBlockLogicalTime

```tolk
@pure
fun blockchain.currentBlockLogicalTime(): int
```

Returns the starting logical time of the current block.

---

#### blockchain.configParam

```tolk
@pure
fun blockchain.configParam(x: int): cell?
```

Returns the value of global configuration parameter with index `x`, or `null`.

**Example:**

```tolk
var gasConfig = blockchain.configParam(20);  // Gas limits configuration
```

---

#### commitContractDataAndActions

```tolk
fun commitContractDataAndActions(): void
```

Commits current state of c4 (persistent data) and c5 (actions) so that execution is considered successful even if an exception is thrown later.

**Example from wallet-v5:**

```tolk
// Store and commit the seqno increment to prevent replays
storage.seqno += 1;
storage.save();
commitContractDataAndActions();
processActions(msg.outActions, msg.hasExtraActions, msg.extraActions, true, false);
```

---

### Serialization Functions

#### T.toCell

```tolk
@pure
fun T.toCell(self, options: PackOptions = {}): Cell<T>
```

Converts any object to a typed cell (most commonly used for structures).

**Example from jetton:**

```tolk
var st: WalletStorage = { jettonBalance: balance, ownerAddress: owner, minterAddress: minter };
contract.setData(st.toCell());
```

**PackOptions:**

- `skipBitsNValidation: bool = false` - Skip runtime validation of `bitsN` types

---

#### T.fromCell

```tolk
@pure
fun T.fromCell(packedCell: cell, options: UnpackOptions = {}): T
```

Parses an object from a cell.

**Example:**

```tolk
var storage = WalletStorage.fromCell(contract.getData());
```

**UnpackOptions:**

- `assertEndAfterReading: bool = true` - Ensure no remaining data after reading
- `throwIfOpcodeDoesNotMatch: int = 63` - Exception code if prefix doesn't match

---

#### T.fromSlice

```tolk
@pure
fun T.fromSlice(rawSlice: slice, options: UnpackOptions = {}): T
```

Parses an object from a slice without mutating it.

**Example from jetton wallet:**

```tolk
val msg = lazy AllowedMessageToWallet.fromSlice(in.body);
match (msg) {
    InternalTransferStep => { ... }
    AskToTransfer => { ... }
}
```

**Note**: The slice is NOT mutated, its pointer is NOT shifted.

---

#### slice.loadAny / slice.skipAny

```tolk
@pure
fun slice.loadAny<T>(mutate self, options: UnpackOptions = {}): T
```

Loads an object from slice, shifting its internal pointer (like `loadUint`).

**Example:**

```tolk
var header: MessageHeader = cs.loadAny();
var body: MessageBody = cs.loadAny();
```

---

```tolk
@pure
fun slice.skipAny<T>(mutate self, options: UnpackOptions = {}): self
```

Skips an object in a slice without reading it.

---

#### builder.storeAny

```tolk
@pure
fun builder.storeAny<T>(mutate self, v: T, options: PackOptions = {}): self
```

Stores any object to a builder.

**Example:**

```tolk
var b = beginCell()
    .storeUint(0x12345678, 32)
    .storeAny(payload)
    .endCell();
```

---

#### T.getDeclaredPackPrefix / T.getDeclaredPackPrefixLen

```tolk
@pure
fun T.getDeclaredPackPrefix(): int
```

Returns the serialization prefix of a struct at compile-time.

**Example:**

```tolk
struct (0xF0) AssetRegular { ... }
AssetRegular.getDeclaredPackPrefix()     // Returns 240
```

---

```tolk
@pure
fun T.getDeclaredPackPrefixLen(): int
```

Returns the prefix length in bits.

**Example:**

```tolk
AssetRegular.getDeclaredPackPrefixLen()  // Returns 16
```

---

#### Cell<T>.load

```tolk
@pure
fun Cell<T>.load(self, options: UnpackOptions = {}): T
```

Unpacks data from a typed cell reference.

**Example:**

```tolk
struct MyStorage {
    extra: Cell<ExtraData>
}

var st = MyStorage.fromCell(contract.getData());
var extraData = st.extra.load();  // Unpacks ExtraData from ref
```

---

### Cryptography and Hashing

#### Compile-Time Hash Functions

```tolk
@pure
fun stringCrc32(constString: slice): int
```

Calculates CRC32 of a constant string at compile-time.

**Example:**

```tolk
const OP_TRANSFER = stringCrc32("op::transfer");  // 0xF8A7EA5
```

---

```tolk
@pure
fun stringCrc16(constString: slice): int
```

Calculates CRC16 (XMODEM) of a constant string.

---

```tolk
@pure
fun stringSha256(constString: slice): int
```

Calculates SHA256 of a constant string, returns 256-bit integer.

---

```tolk
@pure
fun stringSha256_32(constString: slice): int
```

Calculates SHA256 and takes first 32 bits.

---

```tolk
@pure
fun stringToBase256(constString: slice): int
```

Interprets N-char ASCII string as base-256 number.

**Example:**

```tolk
const value = stringToBase256("AB");  // = 16706 (65*256 + 66)
```

---

#### Runtime Hash Functions

```tolk
@pure
fun cell.hash(self): uint256
```

Computes the representation hash of a cell.

**Example from exotic cells:**

```tolk
return beginCell()
    .storeInt(ExoticCellType.LibraryReference as int, 8)
    .storeUint(self.hash(), 256)
    .endCellSpecial();
```

---

```tolk
@pure
fun slice.hash(self): uint256
```

Computes the hash of a slice (same as creating cell and hashing it).

**Example from wallet-v5:**

```tolk
assert (isSignatureValid(signedSlice.hash(), signature, storage.publicKey)) throw ERROR_INVALID_SIGNATURE;
```

---

```tolk
@pure
fun builder.hash(self): uint256
```

Computes hash without creating a cell.

---

```tolk
@pure
fun slice.bitsHash(self): uint256
```

Computes SHA256 of data bits only (without refs). Throws if bit length not divisible by 8.

---

#### Signature Verification

```tolk
@pure
fun isSignatureValid(hash: int, signature: slice, publicKey: int): bool
```

Checks Ed25519 signature of a hash using public key.

**Parameters:**

- `hash`: 256-bit unsigned integer (usually from `slice.hash()`)
- `signature`: Must contain at least 512 bits
- `publicKey`: 256-bit unsigned integer

**Note**: Data is hashed twice - once by you, once inside `CHKSIGNS`.

**Example from wallet-v5:**

```tolk
var signedSlice = inMsgBody.removeLastBits(SIZE_SIGNATURE);
var signature = inMsgBody.getLastBits(SIZE_SIGNATURE);
assert (isSignatureValid(signedSlice.hash(), signature, storage.publicKey)) throw ERROR_INVALID_SIGNATURE;
```

---

```tolk
@pure
fun isSliceSignatureValid(data: slice, signature: slice, publicKey: int): bool
```

Verifies Ed25519 signature of the data portion of a slice. Throws if bit length not divisible by 8.

---

### Random Number Generation

```tolk
fun random.uint256(): uint256
```

Generates pseudo-random unsigned 256-bit integer.

**Important**: Call `random.initialize()` first to make it unpredictable!

---

```tolk
fun random.range(limit: int): int
```

Generates pseudo-random integer in range `0..limit-1` (or `limit..-1` if limit < 0).

---

```tolk
@pure
fun random.getSeed(): uint256
```

Returns the current random seed.

---

```tolk
fun random.setSeed(seed: uint256): void
```

Sets the random seed.

---

```tolk
fun random.initializeBy(mixSeedWith: uint256): void
```

Mixes random seed with provided value.

---

```tolk
fun random.initialize(): void
```

Initializes random seed with current time (makes random generation unpredictable).

**Example:**

```tolk
random.initialize();
var randomValue = random.range(100);  // 0..99
```

---

### Cell/Slice/Builder Primitives

#### Creating Cells and Slices

```tolk
@pure
fun createEmptyCell(): cell
```

Creates a cell with zero bits and references. Equivalent to `beginCell().endCell()`.

---

```tolk
@pure
fun createEmptySlice(): slice
```

Creates a slice with zero remaining bits and references.

---

```tolk
@pure
fun beginCell(): builder
```

Creates a new empty builder.

**Example:**

```tolk
var cell = beginCell()
    .storeUint(0x12345678, 32)
    .storeAddress(addr)
    .endCell();
```

---

```tolk
@pure
fun builder.endCell(self): cell
```

Converts a builder into a cell.

---

```tolk
@pure
fun cell.beginParse(self): slice
```

Converts a cell into a slice for reading.

---

```tolk
@pure
fun builder.toSlice(self): slice
```

Converts builder to slice (same as `b.endCell().beginParse()`).

---

#### Slice Reading Functions

```tolk
@pure
fun slice.loadInt(mutate self, len: int): int
```

Loads a signed `len`-bit integer from slice.

**Example:**

```tolk
var opcode = cs.loadInt(32);
```

---

```tolk
@pure
fun slice.loadUint(mutate self, len: int): int
```

Loads an unsigned `len`-bit integer.

**Example:**

```tolk
var amount = cs.loadUint(64);
```

---

```tolk
@pure
fun slice.loadBits(mutate self, len: int): slice
```

Loads first `len` bits (0 ≤ len ≤ 1023).

---

```tolk
@pure
fun slice.loadRef(mutate self): cell
```

Loads the next reference from slice.

---

```tolk
@pure
fun slice.loadCoins(mutate self): coins
```

Loads serialized nanotoncoins (unsigned integer up to 2^120 - 1).

**Example from jetton:**

```tolk
var jettonAmount = cs.loadCoins();
```

---

```tolk
@pure
fun slice.loadBool(mutate self): bool
```

Loads a boolean (1 bit: -1 for true, 0 for false).

---

```tolk
@pure
fun slice.loadDict(mutate self): dict
```

Loads a dictionary (TVM cell?).

---

```tolk
@pure
fun slice.loadMaybeRef(mutate self): cell?
```

Loads (Maybe ^Cell): reads 1 bit, if true loads ref, otherwise returns `null`.

---

```tolk
@pure
fun slice.loadAddress(mutate self): address
```

Loads a standard internal address (MsgAddressInt). Throws if not internal.

---

```tolk
@pure
fun slice.loadAddressOpt(mutate self): address?
```

Loads internal address or returns `null` if "none address" (00).

---

```tolk
@pure
fun slice.loadAddressAny(mutate self): any_address
```

Loads any valid address (internal/external/none).

---

#### Slice Preloading (Non-Mutating)

```tolk
@pure
fun slice.preloadInt(self, len: int): int
@pure
fun slice.preloadUint(self, len: int): int
@pure
fun slice.preloadBits(self, len: int): slice
@pure
fun slice.preloadRef(self): cell
@pure
fun slice.preloadDict(self): dict
@pure
fun slice.preloadMaybeRef(self): cell?
```

All `preload` functions read data without mutating the slice.

---

#### Slice Skipping

```tolk
@pure
fun slice.skipBits(mutate self, len: int): self
```

Shifts slice pointer forward by `len` bits.

---

```tolk
@pure
fun slice.skipDict(mutate self): self
```

Skips a dictionary.

---

```tolk
@pure
fun slice.skipMaybeRef(mutate self): self
```

Skips (Maybe ^Cell).

---

#### Slice Manipulation

```tolk
@pure
fun slice.getFirstBits(self, len: int): slice
```

Returns first `len` bits of a slice.

---

```tolk
@pure
fun slice.removeLastBits(mutate self, len: int): self
```

Removes last `len` bits.

**Example from wallet-v5:**

```tolk
var signature = inMsgBody.getLastBits(SIZE_SIGNATURE);
var signedSlice = inMsgBody.removeLastBits(SIZE_SIGNATURE);
```

---

```tolk
@pure
fun slice.getLastBits(self, len: int): slice
```

Returns last `len` bits.

---

```tolk
@pure
fun slice.getMiddleBits(self, offset: int, len: int): slice
```

Extracts `len` bits starting from `offset`.

---

#### Slice Checks

```tolk
fun slice.assertEnd(self): void
```

Throws exception 9 if slice is not empty.

**Example:**

```tolk
cs.assertEnd();  // Ensure all data consumed
```

---

```tolk
@pure
fun slice.isEmpty(self): bool
```

Checks if slice has no bits and no refs.

---

```tolk
@pure
fun slice.isEndOfBits(self): bool
```

Checks if slice has no bits of data.

---

```tolk
@pure
fun slice.isEndOfRefs(self): bool
```

Checks if slice has no references.

---

```tolk
@pure
fun slice.bitsEqual(self, b: slice): bool
```

Checks if data parts of two slices are equal.

---

```tolk
@pure
fun slice.remainingBitsCount(self): int
```

Returns the number of data bits remaining.

**Example from jetton:**

```tolk
assert (msg.forwardPayload.remainingBitsCount()) throw ERR_INVALID_PAYLOAD;
```

---

```tolk
@pure
fun slice.remainingRefsCount(self): int
```

Returns the number of references remaining.

---

```tolk
@pure
fun slice.remainingBitsAndRefsCount(self): (int, int)
```

Returns both bits and refs counts.

---

#### Builder Storing Functions

```tolk
@pure
fun builder.storeInt(mutate self, x: int, len: int): self
```

Stores a signed `len`-bit integer (0 ≤ len ≤ 257).

---

```tolk
@pure
fun builder.storeUint(mutate self, x: int, len: int): self
```

Stores an unsigned `len`-bit integer (0 ≤ len ≤ 256).

---

```tolk
@pure
fun builder.storeSlice(mutate self, s: slice): self
```

Stores a slice.

---

```tolk
@pure
fun builder.storeRef(mutate self, c: cell): self
```

Stores a reference to a cell.

---

```tolk
@pure
fun builder.storeAddress(mutate self, addr: address): self
```

Stores an internal address.

---

```tolk
@pure
fun builder.storeAddressOpt(mutate self, addrOrNull: address?): self
```

Stores internal address or '00' if `null`.

---

```tolk
@pure
fun builder.storeAddressAny(mutate self, addrAny: any_address): self
```

Stores any address (internal/external/none).

---

```tolk
@pure
fun builder.storeAddressNone(mutate self): self
```

Stores "none address": '00' (two zero bits).

---

```tolk
@pure
fun builder.storeCoins(mutate self, x: coins): self
```

Stores nanotoncoins amount.

---

```tolk
@pure
fun builder.storeBool(mutate self, x: bool): self
```

Stores a boolean (1 bit).

---

```tolk
@pure
fun builder.storeDict(mutate self, c: dict): self
```

Stores a dictionary (stores 0 if null, 1 + ref if not null).

---

```tolk
@pure
fun builder.storeMaybeRef(mutate self, c: cell?): self
```

Stores (Maybe ^Cell).

---

```tolk
@pure
fun builder.storeBuilder(mutate self, from: builder): self
```

Concatenates two builders.

---

#### Builder Checks

```tolk
@pure
fun builder.bitsCount(self): int
```

Returns number of data bits already stored.

---

```tolk
@pure
fun builder.refsCount(self): int
```

Returns number of references already stored.

---

### Address Functions

#### address()

```tolk
@pure
fun address(stdAddress: slice): address
```

Parses a valid contract address at compile-time.

**Examples:**

```tolk
address("EQCRDM9h4k3UJdOePPuyX40mCgA4vxge5Dc5vjBR8djbEKC5")
address("0:527964d55cfa6eb731f4bfc07e9d025098097ef8505519e853986279bd8400d8")
```

---

#### address.getWorkchainAndHash

```tolk
@pure
fun address.getWorkchainAndHash(self): (int8, uint256)
```

Extracts workchain and hash from an internal address.

**Example from wallet-v5:**

```tolk
var (senderWorkchain, senderAddrHash) = in.senderAddress.getWorkchainAndHash();
```

---

```tolk
@pure
fun address.getWorkchain(self): int8
```

Extracts only workchain.

**Example from jetton:**

```tolk
assert (msg.transferRecipient.getWorkchain() == BASECHAIN) throw ERR_WRONG_WORKCHAIN;
```

---

#### createAddressNone

```tolk
@pure
fun createAddressNone(): any_address
```

Creates "none external address" (addr_none$00 - two zero bits).

**Note**: For internal addresses, use `address?` nullable type with `null` value.

---

#### any_address.isNone / isInternal / isExternal

```tolk
@pure
fun any_address.isNone(self): bool
fun any_address.isInternal(self): bool
fun any_address.isExternal(self): bool
```

Check address type.

---

#### address.fromWorkchainAndHash

```tolk
fun address.fromWorkchainAndHash(workchain: int8, hash: uint256): address
```

Constructs an internal address from workchain and hash.

**Example:**

```tolk
var addr = address.fromWorkchainAndHash(0, computedHash);
```

---

### Message Creation and Sending

#### Constants for Send Modes

```tolk
const SEND_MODE_REGULAR = 0
const SEND_MODE_PAY_FEES_SEPARATELY = 1
const SEND_MODE_IGNORE_ERRORS = 2
const SEND_MODE_BOUNCE_ON_ACTION_FAIL = 16
const SEND_MODE_DESTROY = 32
const SEND_MODE_CARRY_ALL_REMAINING_MESSAGE_VALUE = 64
const SEND_MODE_CARRY_ALL_BALANCE = 128
const SEND_MODE_ESTIMATE_FEE_ONLY = 1024
```

---

#### BounceMode Enum

```tolk
enum BounceMode {
    NoBounce                // Message disappears on error
    Only256BitsOfBody       // Bounced body: 0xFFFFFFFF + first 256 bits (most cheap)
    RichBounce              // Full bounce information (most expensive)
    RichBounceOnlyRootCell  // Rich bounce without refs in originalBody
}
```

**Example from jetton wallet:**

```tolk
val deployMsg = createMessage({
    bounce: BounceMode.Only256BitsOfBody,
    dest: recipientWallet,
    value: 0,
    body: InternalTransferStep { ... }
});
deployMsg.send(SEND_MODE_CARRY_ALL_REMAINING_MESSAGE_VALUE);
```

---

#### createMessage

```tolk
@pure
fun createMessage<TBody>(options: CreateMessageOptions<TBody>): OutMessage
```

Creates an outgoing internal message.

**CreateMessageOptions:**

- `bounce: BounceMode | OldBounceMode` - Bounce behavior
- `value: coins | (coins, ExtraCurrenciesMap)` - Attached value
- `dest: address | builder | (int8, uint256) | AutoDeployAddress` - Destination
- `body: TBody` - Message body (any serializable object)

**Examples from jetton wallet:**

```tolk
// Send notification to owner
val notifyOwnerMsg = createMessage({
    bounce: BounceMode.NoBounce,
    dest: storage.ownerAddress,
    value: msg.forwardTonAmount,
    body: TransferNotificationForRecipient {
        queryId: msg.queryId,
        jettonAmount: msg.jettonAmount,
        transferInitiator: msg.transferInitiator,
        forwardPayload: msg.forwardPayload
    }
});
notifyOwnerMsg.send(SEND_MODE_PAY_FEES_SEPARATELY);

// Return excesses
val excessesMsg = createMessage({
    bounce: BounceMode.NoBounce,
    dest: msg.sendExcessesTo!,
    value: msgValue,
    body: ReturnExcessesBack {
        queryId: msg.queryId
    }
});
excessesMsg.send(SEND_MODE_IGNORE_ERRORS);
```

**Hint**: Don't call `body.toCell()`, pass body directly! The compiler optimizes:

- Small bodies are inlined without cell creation
- Large bodies are automatically wrapped in refs

---

#### AutoDeployAddress

```tolk
struct AutoDeployAddress {
    workchain: int8 = BASECHAIN
    stateInit: ContractState | cell
    toShard: AddressShardingOptions? = null
}
```

Used to deploy a contract by sending message to auto-calculated address.

**Example:**

```tolk
val deployMsg = createMessage({
    dest: {
        workchain: 0,
        stateInit: {
            code: jettonWalletCode,
            data: jettonWalletData.toCell()
        }
    },
    value: ton("0.05"),
    body: InitialMessage { ... }
});
```

---

#### AutoDeployAddress.calculateAddress

```tolk
@pure
fun AutoDeployAddress.calculateAddress(self): address
```

Calculates the address a deployed contract will have.

**Example from jetton:**

```tolk
val jwDeployed = {
    workchain: 0,
    stateInit: {
        code: contract.getCode(),
        data: WalletStorage {
            jettonBalance: 0,
            ownerAddress: owner,
            minterAddress: contract.getAddress()
        }.toCell()
    }
};
var jwAddress = jwDeployed.calculateAddress();
```

---

#### OutMessage.send

```tolk
fun OutMessage.send(self, sendMode: int): void
```

Sends a ready message cell.

**Example:**

```tolk
createMessage({ ... }).send(SEND_MODE_REGULAR);
```

---

#### OutMessage.sendAndEstimateFee

```tolk
fun OutMessage.sendAndEstimateFee(self, sendMode: int): coins
```

Sends message and returns estimated fee.

---

#### OutMessage.estimateFeeWithoutSending

```tolk
@pure
fun OutMessage.estimateFeeWithoutSending(self, sendMode: int): coins
```

Estimates fee without actually sending.

---

#### createExternalLogMessage

```tolk
@pure
fun createExternalLogMessage<TBody>(options: CreateExternalLogMessageOptions<TBody>): OutMessage
```

Creates an external outgoing message (for logging/events).

**CreateExternalLogMessageOptions:**

- `dest: any_address | builder | ExtOutLogBucket` - External destination
- `body: TBody` - Message body

**Example:**

```tolk
val emitMsg = createExternalLogMessage({
    dest: createAddressNone(),
    body: DepositEvent {
        user: userAddress,
        amount: depositAmount
    }
});
emitMsg.send(SEND_MODE_REGULAR);
```

---

### Reserve Functions

#### Constants for Reserve Modes

```tolk
const RESERVE_MODE_EXACT_AMOUNT = 0
const RESERVE_MODE_ALL_BUT_AMOUNT = 1
const RESERVE_MODE_AT_MOST = 2
const RESERVE_MODE_INCREASE_BY_ORIGINAL_BALANCE = 4
const RESERVE_MODE_NEGATE_AMOUNT = 8
const RESERVE_MODE_BOUNCE_ON_ACTION_FAIL = 16
```

---

#### reserveToncoinsOnBalance

```tolk
fun reserveToncoinsOnBalance(nanoTonCoins: coins, reserveMode: int): void
```

Creates an output action to reserve Toncoins on balance.

**Example:**

```tolk
reserveToncoinsOnBalance(MIN_TONS_FOR_STORAGE, RESERVE_MODE_EXACT_AMOUNT);
```

---

#### reserveExtraCurrenciesOnBalance

```tolk
fun reserveExtraCurrenciesOnBalance(nanoTonCoins: coins, extraAmount: dict, reserveMode: int): void
```

Reserves TON and extra currencies.

---

### Map Operations

#### createEmptyMap

```tolk
@pure
fun createEmptyMap<K, V>(): map<K, V>
```

Returns an empty typed map (essentially PUSHNULL).

**Example:**

```tolk
var extensions: map<uint256, bool> = createEmptyMap();
```

---

#### map.isEmpty

```tolk
@pure
fun map<K, V>.isEmpty(self): bool
```

Checks if map is empty (cell is null).

**Example from wallet-v5:**

```tolk
assert (!storage.extensions.isEmpty()) throw ERROR_DISABLE_SIGNATURE_WHEN_EXTENSIONS_IS_EMPTY;
```

**Note**: Use `m.isEmpty()`, not `m == null`.

---

#### map.exists

```tolk
@pure
fun map<K, V>.exists(self, key: K): bool
```

Checks if key exists in map.

**Example from wallet-v5:**

```tolk
if (!storage.extensions.exists(senderAddrHash)) {
    return;
}
```

---

#### map.get

```tolk
@pure
fun map<K, V>.get(self, key: K): MapLookupResult<V>
```

Gets an element by key. Returns `MapLookupResult` with `isFound` flag.

**Example:**

```tolk
val result = balances.get(userAddress);
if (result.isFound) {
    var balance = result.loadValue();
}
```

---

#### map.mustGet

```tolk
@pure
fun map<K, V>.mustGet(self, key: K, throwIfNotFound: int = 9): V
```

Gets element or throws exception if not found.

---

#### map.set

```tolk
@pure
fun map<K, V>.set(mutate self, key: K, value: V): self
```

Sets an element by key.

**Example:**

```tolk
balances.set(userAddress, newBalance);
```

Returns self, so calls can be chained.

---

#### map.setAndGetPrevious

```tolk
@pure
fun map<K, V>.setAndGetPrevious(mutate self, key: K, value: V): MapLookupResult<V>
```

Sets element and returns previous value (or `isFound = false`).

---

#### map.replaceIfExists

```tolk
@pure
fun map<K, V>.replaceIfExists(mutate self, key: K, value: V): bool
```

Only sets if key already exists. Returns whether replaced.

---

#### map.addIfNotExists

```tolk
@pure
fun map<K, V>.addIfNotExists(mutate self, key: K, value: V): bool
```

Only sets if key doesn't exist. Returns whether added.

**Example from wallet-v5:**

```tolk
val inserted = storage.extensions.addIfNotExists(extensionAddrHash, true);
assert (inserted) throw ERROR_ADD_EXTENSION;
```

---

#### map.delete

```tolk
@pure
fun map<K, V>.delete(mutate self, key: K): bool
```

Deletes element. Returns whether deleted.

**Example from wallet-v5:**

```tolk
val removed = storage.extensions.delete(extensionAddrHash);
assert (removed) throw ERROR_REMOVE_EXTENSION;
```

---

#### map.deleteAndGetDeleted

```tolk
@pure
fun map<K, V>.deleteAndGetDeleted(mutate self, key: K): MapLookupResult<V>
```

Deletes and returns deleted element.

---

#### Map Iteration

```tolk
@pure
fun map<K, V>.findFirst(self): MapEntry<K, V>
fun map<K, V>.findLast(self): MapEntry<K, V>
fun map<K, V>.findKeyGreater(self, pivotKey: K): MapEntry<K, V>
fun map<K, V>.findKeyGreaterOrEqual(self, pivotKey: K): MapEntry<K, V>
fun map<K, V>.findKeyLess(self, pivotKey: K): MapEntry<K, V>
fun map<K, V>.findKeyLessOrEqual(self, pivotKey: K): MapEntry<K, V>
fun map<K, V>.iterateNext(self, current: MapEntry<K, V>): MapEntry<K, V>
fun map<K, V>.iteratePrev(self, current: MapEntry<K, V>): MapEntry<K, V>
```

**Example - iterate over all elements:**

```tolk
var entry = balances.findFirst();
while (entry.isFound) {
    var key = entry.getKey();
    var value = entry.loadValue();
    // ... process key and value
    entry = balances.iterateNext(entry);
}
```

---

### Tuple Operations

```tolk
@pure
fun createEmptyTuple(): tuple
fun tuple.push<T>(mutate self, value: T): void
fun tuple.first<T>(self): T
fun tuple.get<T>(self, index: int): T
fun tuple.set<T>(mutate self, value: T, index: int): void
fun tuple.size(self): int
fun tuple.last<T>(self): T
fun tuple.pop<T>(mutate self): T
```

**Example:**

```tolk
var t = createEmptyTuple();
t.push(42);
t.push(100);
var first = t.first<int>();    // 42
var second = t.get<int>(1);    // 100
```

---

#### T.toTuple / T.fromTuple

```tolk
@pure
fun T.toTuple(self): tuple
fun T.fromTuple(packedObject: tuple): T
```

Packs/unpacks object to/from tuple (useful for logging or low-level operations).

**Example:**

```tolk
struct Point { x: int, y: int }

var p: Point = { x: 1, y: 2 };
var t = p.toTuple();         // [1, 2]
p = Point.fromTuple(t);      // back
var x = t.get<int>(0);       // 1
```

---

### Size Calculation

```tolk
@pure
fun cell.calculateSize(self, maxCells: int): (int, int, int, bool)
fun slice.calculateSize(self, maxCells: int): (int, int, int, bool)
fun cell.calculateSizeStrict(self, maxCells: int): (int, int, int)
fun slice.calculateSizeStrict(self, maxCells: int): (int, int, int)
```

Calculates `(distinctCells, totalBits, totalRefs, success)` for storage fee estimation.

**Example:**

```tolk
var (cells, bits, refs, ok) = userDataCell.calculateSize(1000);
if (ok) {
    var storageFee = calculateStorageFee(0, 86400, bits, cells);
}
```

---

```tolk
@pure
fun cell?.depth(self): int
fun slice.depth(self): int
fun builder.depth(self): int
```

Returns depth: 0 if no refs, otherwise 1 + max depth of all refs.

---

```tolk
@pure
fun sizeof<T>(anyVariable: T): int
```

Returns stack slots an object occupies (compile-time).

**Example:**

```tolk
struct Point { x: int, y: int }
sizeof(somePoint)     // 2 (two fields, one slot each)
sizeof(nullableInt)   // 1 (int? is one TVM slot)
```

---

### Debug Functions

```tolk
fun debug.print<T>(x: T): void
fun debug.printString<T>(x: T): void
fun debug.dumpStack(): void
```

Only work in local TVM execution with debug verbosity.

**Example:**

```tolk
debug.print(currentBalance);
debug.printString("Processing transfer");
```

---

### Utility Functions

```tolk
@pure
fun T.typeName(): slice
fun T.typeNameOfObject(self): slice
```

Returns human-readable type name for logging.

**Example:**

```tolk
debug.printString(int.typeName());           // "int"
debug.printString(42.typeNameOfObject());    // "int"
```

---

```tolk
@pure
fun stringHexToSlice(constStringBytesHex: slice): slice
```

Converts constant hex string to slice at compile-time.

**Example:**

```tolk
const header = stringHexToSlice("abcdef");  // slice with bytes 0xAB,0xCD,0xEF
```

---

### Bounced Message Handling

```tolk
@pure
fun slice.skipBouncedPrefix(mutate self): self
```

Skips 0xFFFFFFFF prefix from bounced message body.

**Example from jetton wallet:**

```tolk
fun onBouncedMessage(in: InMessageBounced) {
    in.bouncedBody.skipBouncedPrefix();
    val msg = lazy BounceOpToHandle.fromSlice(in.bouncedBody);
    match (msg) {
        InternalTransferStep => { ... },
        BurnNotificationForMinter => { ... },
    };
}
```

---

### Deprecated Functions

```tolk
@deprecated("use `senderAddress == ownerAddress`, not bitsEqual")
fun address.bitsEqual(self, b: address): bool

@deprecated("use modern `onInternalMessage` handler")
fun slice.loadMessageFlags(mutate self): int
fun slice.loadMessageOp(mutate self): int
fun slice.loadMessageQueryId(mutate self): int
```

These are kept for FunC compatibility but should not be used in new code.

---

## tvm-dicts.tolk - Dictionary Operations

Low-level API for working with TVM dictionaries. **Not recommended** - use `map<K, V>` instead, which is easier and more efficient.

**Note**: In this module, `dict` is an alias for `cell?`.

### Creating Dictionaries

```tolk
@pure
fun createEmptyDict(): dict
```

Creates empty dictionary (returns null).

---

```tolk
@pure
fun dict.dictIsEmpty(self): bool
```

Checks if dictionary is empty.

---

### Get Operations

Three variants for different key types:

- `iDict*` - Signed integer keys
- `uDict*` - Unsigned integer keys
- `sDict*` - Arbitrary slice keys

```tolk
@pure
fun dict.iDictGet(self, keyLen: int, key: int): (slice?, bool)
fun dict.uDictGet(self, keyLen: int, key: int): (slice?, bool)
fun dict.sDictGet(self, keyLen: int, key: slice): (slice?, bool)
```

**Parameters:**

- `keyLen`: Key length in bits
- `key`: The key to lookup

**Returns**: `(value, found)` tuple

**Example:**

```tolk
var (value, found) = myDict.uDictGet(256, userHash);
if (found) {
    var balance = value!.loadCoins();
}
```

---

### Set Operations

```tolk
@pure
fun dict.iDictSet(mutate self, keyLen: int, key: int, value: slice): void
fun dict.uDictSet(mutate self, keyLen: int, key: int, value: slice): void
fun dict.sDictSet(mutate self, keyLen: int, key: slice, value: slice): void
```

Sets a value in dictionary.

---

```tolk
@pure
fun dict.iDictSetRef(mutate self, keyLen: int, key: int, value: cell): void
fun dict.uDictSetRef(mutate self, keyLen: int, key: int, value: cell): void
fun dict.sDictSetRef(mutate self, keyLen: int, key: slice, value: cell): void
```

Sets a value as cell reference.

---

```tolk
@pure
fun dict.iDictSetBuilder(mutate self, keyLen: int, key: int, value: builder): void
fun dict.uDictSetBuilder(mutate self, keyLen: int, key: int, value: builder): void
fun dict.sDictSetBuilder(mutate self, keyLen: int, key: slice, value: builder): void
```

Sets a value from builder.

---

### Conditional Set Operations

```tolk
@pure
fun dict.iDictSetIfNotExists(mutate self, keyLen: int, key: int, value: slice): bool
fun dict.uDictSetIfNotExists(mutate self, keyLen: int, key: int, value: slice): bool
```

Sets only if key doesn't exist. Returns whether added.

---

```tolk
@pure
fun dict.iDictSetIfExists(mutate self, keyLen: int, key: int, value: slice): bool
fun dict.uDictSetIfExists(mutate self, keyLen: int, key: int, value: slice): bool
```

Sets only if key exists. Returns whether replaced.

---

### Get Ref Operations

```tolk
@pure
fun dict.iDictGetRef(self, keyLen: int, key: int): (cell?, bool)
fun dict.uDictGetRef(self, keyLen: int, key: int): (cell?, bool)
fun dict.sDictGetRef(self, keyLen: int, key: slice): (cell?, bool)
```

Gets value as cell reference.

---

```tolk
@pure
fun dict.iDictGetRefOrNull(self, keyLen: int, key: int): cell?
fun dict.uDictGetRefOrNull(self, keyLen: int, key: int): cell?
fun dict.sDictGetRefOrNull(self, keyLen: int, key: slice): cell?
```

Gets ref or returns null if not found.

---

### Delete Operations

```tolk
@pure
fun dict.iDictDelete(mutate self, keyLen: int, key: int): bool
fun dict.uDictDelete(mutate self, keyLen: int, key: int): bool
fun dict.sDictDelete(mutate self, keyLen: int, key: slice): bool
```

Deletes an element. Returns whether deleted.

---

### Combined Operations

```tolk
@pure
fun dict.iDictSetAndGet(mutate self, keyLen: int, key: int, value: slice): (slice?, bool)
fun dict.uDictSetAndGet(mutate self, keyLen: int, key: int, value: slice): (slice?, bool)
fun dict.sDictSetAndGet(mutate self, keyLen: int, key: slice, value: slice): (slice?, bool)
```

Sets value and returns previous value.

---

```tolk
@pure
fun dict.iDictDeleteAndGet(mutate self, keyLen: int, key: int): (slice?, bool)
fun dict.uDictDeleteAndGet(mutate self, keyLen: int, key: int): (slice?, bool)
fun dict.sDictDeleteAndGet(mutate self, keyLen: int, key: slice): (slice?, bool)
```

Deletes and returns deleted value.

---

### Min/Max Operations

```tolk
@pure
fun dict.iDictGetFirst(self, keyLen: int): (int?, slice?, bool)
fun dict.uDictGetFirst(self, keyLen: int): (int?, slice?, bool)
fun dict.sDictGetFirst(self, keyLen: int): (slice?, slice?, bool)
```

Gets first (minimal) element. Returns `(key, value, found)`.

---

```tolk
@pure
fun dict.iDictGetLast(self, keyLen: int): (int?, slice?, bool)
fun dict.uDictGetLast(self, keyLen: int): (int?, slice?, bool)
fun dict.sDictGetLast(self, keyLen: int): (slice?, slice?, bool)
```

Gets last (maximal) element.

---

```tolk
@pure
fun dict.iDictDeleteFirstAndGet(mutate self, keyLen: int): (int?, slice?, bool)
fun dict.uDictDeleteFirstAndGet(mutate self, keyLen: int): (int?, slice?, bool)
fun dict.sDictDeleteFirstAndGet(mutate self, keyLen: int): (slice?, slice?, bool)
```

Removes and returns first element.

---

```tolk
@pure
fun dict.iDictDeleteLastAndGet(mutate self, keyLen: int): (int?, slice?, bool)
fun dict.uDictDeleteLastAndGet(mutate self, keyLen: int): (int?, slice?, bool)
fun dict.sDictDeleteLastAndGet(mutate self, keyLen: int): (slice?, slice?, bool)
```

Removes and returns last element.

---

### Navigation

```tolk
@pure
fun dict.iDictGetNext(self, keyLen: int, pivot: int): (int?, slice?, bool)
fun dict.uDictGetNext(self, keyLen: int, pivot: int): (int?, slice?, bool)
fun dict.sDictGetNext(self, keyLen: int, pivot: slice): (slice?, slice?, bool)
```

Gets next element after pivot.

---

```tolk
@pure
fun dict.iDictGetNextOrEqual(self, keyLen: int, pivot: int): (int?, slice?, bool)
fun dict.uDictGetNextOrEqual(self, keyLen: int, pivot: int): (int?, slice?, bool)
fun dict.sDictGetNextOrEqual(self, keyLen: int, pivot: slice): (slice?, slice?, bool)
```

Gets element >= pivot.

---

```tolk
@pure
fun dict.iDictGetPrev(self, keyLen: int, pivot: int): (int?, slice?, bool)
fun dict.uDictGetPrev(self, keyLen: int, pivot: int): (int?, slice?, bool)
fun dict.sDictGetPrev(self, keyLen: int, pivot: slice): (slice?, slice?, bool)
```

Gets previous element before pivot.

---

```tolk
@pure
fun dict.iDictGetPrevOrEqual(self, keyLen: int, pivot: int): (int?, slice?, bool)
fun dict.uDictGetPrevOrEqual(self, keyLen: int, pivot: int): (int?, slice?, bool)
fun dict.sDictGetPrevOrEqual(self, keyLen: int, pivot: slice): (slice?, slice?, bool)
```

Gets element <= pivot.

---

### Prefix Dictionary Operations

```tolk
@pure
fun dict.prefixDictGet(self, keyLen: int, key: slice): (slice, slice?, slice?, bool)
```

Gets value by key prefix.

---

```tolk
@pure
fun dict.prefixDictSet(mutate self, keyLen: int, key: slice, value: slice): bool
```

Sets value by key prefix.

---

```tolk
@pure
fun dict.prefixDictDelete(mutate self, keyLen: int, key: slice): bool
```

Deletes value by key prefix.

---

## exotic-cells.tolk - Exotic Cell Handling

Functions for working with exotic cells (library references, Merkle proofs, etc.).

### ExoticCellType Enum

```tolk
enum ExoticCellType: int8 {
    Ordinary = -1
    PrunedBranch = 1
    LibraryReference = 2
    MerkleProof = 3
    MerkleUpdate = 4
}
```

---

### cell.beginParseSpecial

```tolk
@pure
fun cell.beginParseSpecial(self): (slice, bool)
```

Transforms exotic or ordinary cell into slice. Returns `(slice, isExotic)`.

**Example:**

```tolk
var (s, isExotic) = c.beginParseSpecial();
if (isExotic) {
    val cellType = s.loadInt(8) as ExoticCellType;
    match (cellType) {
        ExoticCellType.LibraryReference => { ... }
        ExoticCellType.MerkleProof => { ... }
    }
}
```

---

### builder.endCellSpecial

```tolk
@pure
fun builder.endCellSpecial(self, isExotic: bool = true): cell
```

Creates an exotic or ordinary cell from builder.

---

### cell.hashOfLevel

```tolk
@pure
fun cell.hashOfLevel(self, level: int): uint256
```

Returns i-th hash of the cell (for Merkle structures).

---

### cell.depthOfLevel

```tolk
@pure
fun cell.depthOfLevel(self, level: int): int
```

Returns i-th depth of the cell.

---

### cell.loadSpecial

```tolk
@pure
fun cell.loadSpecial(self): cell
```

Loads an exotic cell and returns ordinary cell. Does nothing if already ordinary.

---

### cell.toLibraryReference

```tolk
@pure
fun cell.toLibraryReference(self): cell
```

Transforms cell into a library reference (exotic cell with type + hash).

**Example:**

```tolk
var libRef = myCode.toLibraryReference();
// Creates exotic cell: 8 bits (type=2) + 256 bits (hash)
```

---

## gas-payments.tolk - Gas and Fee Calculations

Import with: `import "@stdlib/gas-payments"`

### Gas Consumption

```tolk
fun getGasConsumedAtTheMoment(): int
```

Returns amount of gas (in gas units) consumed so far in Computation Phase.

**Example:**

```tolk
var gasBefore = getGasConsumedAtTheMoment();
// ... do something
var gasUsed = getGasConsumedAtTheMoment() - gasBefore;
```

---

### Gas Limit Management

```tolk
fun acceptExternalMessage(): void
```

Required when processing external messages. Accepts the message to blockchain and agrees to buy gas.

**Example from wallet-v5:**

```tolk
fun onExternalMessage(inMsgBody: slice) {
    // ... validate signature and seqno
    acceptExternalMessage();
    // ... continue processing
}
```

---

```tolk
fun setGasLimitToMaximum(): void
```

Sets gas limit to maximum allowed value.

---

```tolk
fun setGasLimit(limit: int): void
```

Sets gas limit to specific value.

---

### Fee Calculation

```tolk
@pure
fun calculateGasFee(workchain: int8, gasUsed: int): coins
```

Calculates fee (in nanotoncoins) for gas consumption.

**Parameters:**

- `workchain`: Workchain ID (usually 0 for basechain)
- `gasUsed`: Gas units consumed

**Example:**

```tolk
var gasConsumed = 5000;
var fee = calculateGasFee(BASECHAIN, gasConsumed);
```

---

```tolk
@pure
fun calculateGasFeeWithoutFlatPrice(workchain: int8, gasUsed: coins): coins
```

Same as `calculateGasFee` but without flat price component.

---

```tolk
fun calculateStorageFee(workchain: int8, seconds: int, bits: int, cells: int): coins
```

Calculates storage fee for given size and duration.

**Parameters:**

- `workchain`: Workchain ID
- `seconds`: Storage duration
- `bits`: Total data bits
- `cells`: Total cells count

**Example:**

```tolk
var (cells, bits, refs, ok) = dataCell.calculateSize(1000);
var dailyStorageFee = calculateStorageFee(0, 86400, bits, cells);
```

---

```tolk
@pure
fun calculateForwardFee(workchain: int8, bits: int, cells: int): coins
```

Calculates forward fee for message of given size.

**Example:**

```tolk
var msgSize = msgCell.calculateSizeStrict(1000);
var forwardFee = calculateForwardFee(0, msgSize.1, msgSize.0);
```

---

```tolk
@pure
fun calculateForwardFeeWithoutLumpPrice(workchain: int8, bits: int, cells: int): coins
```

Same without lump price component.

---

### Contract Balance Queries

```tolk
fun contract.getStorageDuePayment(): coins
```

Returns the amount of nanotoncoins the contract debts for storage. Returns 0 if no debt.

---

```tolk
@pure
fun contract.getStoragePaidPayment(): coins
```

Returns the amount charged for storage during the storage phase preceding current computation.

---

## lisp-lists.tolk - Lisp-Style Lists

Import with: `import "@stdlib/lisp-lists"`

Lisp-style lists are nested 2-element tuples: `[1, [2, [3, null]]]` represents `[1, 2, 3]`.

### createEmptyList

```tolk
@pure
fun createEmptyList(): tuple
```

Returns empty list (null value).

---

### listPrepend

```tolk
@pure
fun listPrepend<X>(head: X, tail: tuple?): tuple
```

Adds element to the beginning. Returns new list (does not mutate).

**Example:**

```tolk
var list = createEmptyList();
list = listPrepend(3, list);    // [3]
list = listPrepend(2, list);    // [2, 3]
list = listPrepend(1, list);    // [1, 2, 3]
```

---

### listSplit

```tolk
@pure
fun listSplit<X>(list: tuple): (X, tuple?)
```

Extracts head and tail from list.

**Example:**

```tolk
var (head, tail) = listSplit(list);  // head=1, tail=[2,3]
```

---

### listGetHead

```tolk
@pure
fun listGetHead<X>(list: tuple): X
```

Returns the first element.

---

### listGetTail

```tolk
@pure
fun listGetTail(list: tuple): tuple?
```

Returns the rest of the list.

---

## tvm-lowlevel.tolk - Low-Level TVM Operations

Import with: `import "@stdlib/tvm-lowlevel"`

### TVM Register C3 Operations

```tolk
@pure
fun getTvmRegisterC3(): continuation
```

Returns current value of c3 register (contains contract code for function calls).

---

```tolk
fun setTvmRegisterC3(c: continuation): void
```

Updates c3 register. Used for runtime code updates.

**Example:**

```tolk
var newCode = updatedCodeCell;
var newContinuation = transformSliceToContinuation(newCode.beginParse());
setTvmRegisterC3(newContinuation);
// Any subsequent function call will use new code
```

**Note**: Current code and call stack won't change immediately, only future function calls.

---

### transformSliceToContinuation

```tolk
@pure
fun transformSliceToContinuation(s: slice): continuation
```

Converts slice to continuation with empty stack and savelist.

---

### T.stackMoveToTop

```tolk
@pure
fun T.stackMoveToTop(mutate self): void
```

Moves a variable to the top of the stack. Compiled to NOP (optimization hint).

---

## Summary

This stdlib reference covers all 6 modules with 300+ functions. Key takeaways:

**Most Used Modules:**

- `common.tolk` - Always available, use for 95% of contracts
- `gas-payments` - Import for fee calculations
- `tvm-dicts`, `lisp-lists`, `exotic-cells`, `tvm-lowlevel` - Rarely needed

**Best Practices:**

- Use `map<K, V>` instead of low-level dicts
- Use `createMessage` instead of manual message construction
- Use `lazy` loading for message parsing
- Use compile-time functions (`ton()`, `stringCrc32`) for constants
- Check return values (use `isFound`, check `bool` returns, etc.)

**Common Patterns:**

```tolk
// Storage pattern
var storage = lazy Storage.load();
storage.field = newValue;
storage.save();

// Message handling pattern
val msg = lazy AllowedMessage.fromSlice(in.body);
match (msg) {
    MessageType1 => { ... }
    MessageType2 => { ... }
}

// Sending messages pattern
createMessage({
    bounce: BounceMode.NoBounce,
    dest: recipient,
    value: ton("0.05"),
    body: MessageStruct { ... }
}).send(SEND_MODE_REGULAR);
```

For more patterns, see [03-patterns-and-practices.md](./03-patterns-and-practices.md).
