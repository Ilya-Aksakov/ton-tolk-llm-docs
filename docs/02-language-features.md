# Tolk Language Features Guide

Complete guide to Tolk v1.2 syntax, features, and language constructs. This document covers everything from basic structures to advanced patterns with real examples from production contracts.

## Table of Contents

1. [Version Declaration](#version-declaration)
2. [Import System](#import-system)
3. [Constants](#constants)
4. [Structures](#structures)
5. [Union Types](#union-types)
6. [Functions](#functions)
7. [Variables and Assignments](#variables-and-assignments)
8. [Control Flow](#control-flow)
9. [Pattern Matching](#pattern-matching)
10. [Lazy Evaluation](#lazy-evaluation)
11. [Operators](#operators)
12. [Type Casting](#type-casting)
13. [Nullable Types](#nullable-types)
14. [Annotations](#annotations)
15. [Special Constructs](#special-constructs)
16. [Comments](#comments)

---

## Version Declaration

Every Tolk file must start with a version declaration:

```tolk
tolk 1.2
```

This ensures the compiler version compatibility.

---

## Import System

### Importing Standard Library

Standard library modules are imported with `@stdlib/` prefix:

```tolk
import "@stdlib/gas-payments"
import "@stdlib/tvm-dicts"
import "@stdlib/lisp-lists"
```

**Note**: `common.tolk` is automatically imported and doesn't need explicit import.

---

### Importing Local Files

Import local files without extension:

```tolk
import "errors"
import "storage"
import "messages"
import "jetton-utils"
```

**Example from jetton-wallet-contract.tolk:**

```tolk
import "@stdlib/gas-payments"
import "errors"
import "jetton-utils"
import "messages"
import "fees-management"
import "storage"
```

**Note**: Import order matters - imported files are processed first.

---

## Constants

Constants are declared with `const` keyword and must be compile-time computable:

```tolk
const MINIMAL_MESSAGE_VALUE_BOUND = ton("0.01")
const MIN_TONS_FOR_STORAGE = ton("0.01")
const JETTON_WALLET_GAS_CONSUMPTION = ton("0.015")

const MASTERCHAIN = -1
const BASECHAIN = 0

const SIZE_SIGNATURE = 512
```

**From errors.tolk:**

```tolk
const ERR_INVALID_OP = 709
const ERR_NOT_FROM_ADMIN = 73
const ERR_NOT_FROM_OWNER = 705
const ERR_NOT_ENOUGH_TON = 709
const ERR_WRONG_WORKCHAIN = 333
const ERR_NOT_ENOUGH_BALANCE = 706
```

### Human-Readable Toncoin Amounts

Use the `ton()` function for readable coin values:

```tolk
const ONE_TON = ton("1")                  // 1,000,000,000 nanotons
const STORAGE_FEE = ton("0.01")           // 10,000,000 nanotons
const TRANSFER_COST = ton("0.05")         // 50,000,000 nanotons

// In runtime code:
var fee: coins = ton("0.015");
val deposit = ton("2.5");
```

**Why use `ton()`:**

- More readable than raw nanotons
- Prevents mistakes with zeros
- Self-documenting code

---

### Compile-Time Functions

These functions execute during compilation, not runtime:

```tolk
// String hashing (compile-time)
const OP_TRANSFER = stringCrc32("op::transfer")      // CRC32 hash
const OP_BURN = stringCrc32("op::burn")
const HASH_256 = stringSha256("my_secret")           // SHA256 hash
const HASH_32 = stringSha256_32("data")              // SHA256 as int32

// String to integer
const MAGIC = stringToBase256("TON")                 // String to number

// String to slice (hex)
const PREFIX = stringHexToSlice("FFFF")              // Hex string to slice
```

**All compile-time functions:**

- `ton(str)` - Convert string to nanotons
- `stringCrc32(str)` - CRC32 hash
- `stringCrc16(str)` - CRC16 hash
- `stringSha256(str)` - SHA256 hash as uint256
- `stringSha256_32(str)` - SHA256 hash as int32
- `stringToBase256(str)` - String to integer
- `stringHexToSlice(str)` - Hex string to slice
- `address(str)` - Parse address string

**Real usage from contracts:**

```tolk
// From errors.tolk
const OP_INTERNAL_TRANSFER = 0x178d4519
const OP_TRANSFER = 0x0f8a7ea5
const OP_BURN = 0x595f07bc

// Could be written as:
const OP_INTERNAL_TRANSFER = stringCrc32("op::internal_transfer")
const OP_TRANSFER = stringCrc32("op::transfer")
const OP_BURN = stringCrc32("op::burn")
```

---

## Structures

### Basic Structure Definition

```tolk
struct Point {
    x: int
    y: int
}
```

---

### Structures with Types

**From jetton storage.tolk:**

```tolk
struct WalletStorage {
    jettonBalance: coins
    ownerAddress: address
    minterAddress: address
}

struct MinterStorage {
    totalSupply: coins
    adminAddress: address
    content: cell
    jettonWalletCode: cell
}
```

---

### Structures with Opcodes (Prefixes)

Structures can have binary prefixes for message identification:

```tolk
struct (0x0f8a7ea5) AskToTransfer {
    queryId: uint64
    jettonAmount: coins
    transferRecipient: address
    sendExcessesTo: address?
    customPayload: cell?
    forwardTonAmount: coins
    forwardPayload: ForwardPayloadRemainder
}

struct (0x7362d09c) TransferNotificationForRecipient {
    queryId: uint64
    jettonAmount: coins
    transferInitiator: address?
    forwardPayload: ForwardPayloadRemainder
}
```

**Opcode format:**

- `0x12345678` - 32-bit opcode (most common)
- `0xAB` - 8-bit opcode
- Can be any bit width

**From wallet-v5:**

```tolk
struct (0x02) AddExtensionExtraAction {
    addr: address
}

struct (0x6578746E) ExtensionActionRequest {  // "extn" in ASCII
    queryId: uint64
    outActions: OutActionsCell?
    hasExtraActions: bool
    extraActions: SnakedExtraActions
}

struct (0x7369676E) ExternalSignedRequest {  // "sign" in ASCII
    walletId: uint32
    validUntil: uint32
    seqno: uint32
    outActions: OutActionsCell?
    hasExtraActions: bool
    extraActions: SnakedExtraActions
}
```

---

### Structures with Methods

Methods can be defined on structures:

**From storage.tolk:**

```tolk
fun MinterStorage.load() {
    return MinterStorage.fromCell(contract.getData())
}

fun MinterStorage.save(self) {
    contract.setData(self.toCell())
}

fun WalletStorage.load() {
    return WalletStorage.fromCell(contract.getData())
}

fun WalletStorage.save(self) {
    contract.setData(self.toCell())
}
```

---

### Creating Structure Instances

```tolk
// Named fields (recommended)
var storage: WalletStorage = {
    jettonBalance: 0,
    ownerAddress: owner,
    minterAddress: minter,
};

// Shorthand when variable names match field names
val emptyWalletStorage: WalletStorage = {
    jettonBalance: 0,
    ownerAddress,
    minterAddress,
};
```

---

### Field Modifiers: `private` and `readonly`

Fields can have access modifiers to enforce encapsulation:

**`private`** - Field is only accessible within methods of the structure:

```tolk
struct PosInTuple {
    private t: tuple        // Only accessible in methods
    curIndex: int          // Public field
}

fun PosInTuple.first(mutate self) {
    self.curIndex = 0;     // OK
    return self.t.first(); // OK - accessing private field in method
}

// Outside methods:
var pos = PosInTuple { t: someTuple, curIndex: 0 };
pos.curIndex = 5;  // OK
// pos.t = ...     // Error! Private field
```

**`readonly`** - Field cannot be modified after object creation:

```tolk
struct Config {
    readonly maxSupply: coins
    enabled: bool
}

var cfg = Config { maxSupply: ton("1000000"), enabled: true };
cfg.enabled = false;      // OK
// cfg.maxSupply = ...   // Error! Readonly field
```

**Combining `private readonly`:**

```tolk
struct Container {
    private readonly data: cell
    size: int
}

fun Container.hash(self): uint256 {
    return self.data.hash();  // OK - can read in methods
    // self.data = ...         // Error! Cannot modify readonly
}
```

**Use cases:**

- `private` - Hide implementation details
- `readonly` - Immutable configuration values
- `private readonly` - Internal state that never changes

---

### Nested Structures

```tolk
struct Address {
    workchain: int8
    hash: uint256
}

struct UserProfile {
    name: slice
    addr: Address
    balance: coins
}
```

---

## Union Types

Union types define "one of several types":

```tolk
type AllowedMessageToWallet =
    | AskToTransfer
    | AskToBurn
    | InternalTransferStep
```

**From wallet-v5:**

```tolk
type ExtraAction =
    | AddExtensionExtraAction
    | RemoveExtensionExtraAction
    | SetSignatureAllowedExtraAction

type AllowedMessageToWalletV5 =
    | ExtensionActionRequest
    | InternalSignedRequest

type AllowedExternalMessageToWalletV5 =
    | ExternalSignedRequest
```

**Bounced messages handling:**

```tolk
type BounceOpToHandle =
    | InternalTransferStep
    | BurnNotificationForMinter
```

Union types are typically used with `match` for message handling (see [Pattern Matching](#pattern-matching)).

### Type Checking with `is` / `!is`

Instead of pattern matching, you can check union types using `is` and `!is` operators:

```tolk
fun processValue(value: int | slice) {
    if (value is slice) {
        // value is automatically cast to slice here
        var len = value.remainingBitsCount();
        return;
    }
    // value is int here
    return value * 2;
}
```

**Negative check with `!is`:**

```tolk
fun processAddress(addr: address | any_address) {
    if (addr !is address) {
        // addr is any_address here
        if (addr.isInternal()) {
            addr = addr.castToInternal();
        }
    }
    // addr is address here
}
```

**Note**: After `is` check, the compiler performs smart cast - no manual casting needed.

---

## Functions

### Basic Function Syntax

```tolk
fun functionName(param1: Type1, param2: Type2): ReturnType {
    // function body
    return result;
}
```

---

### Functions Without Return Type

Type inference works:

```tolk
fun calcAddressOfJettonWallet(ownerAddress: address, minterAddress: address, jettonWalletCode: cell) {
    val jwDeployed = calcDeployedJettonWallet(ownerAddress, minterAddress, jettonWalletCode);
    return jwDeployed.calculateAddress()
}
```

Compiler automatically infers return type as `address`.

---

### Functions with Default Parameters

```tolk
struct AutoDeployAddress {
    workchain: int8 = BASECHAIN
    stateInit: ContractState | cell
    toShard: AddressShardingOptions? = null
}
```

---

### Functions Returning Multiple Values

```tolk
fun address.getWorkchainAndHash(self): (int8, uint256)

// Usage:
var (workchain, hash) = myAddress.getWorkchainAndHash();
```

---

### Method Functions (Self Parameter)

Methods are functions with `self` parameter:

```tolk
fun WalletStorage.save(self) {
    contract.setData(self.toCell())
}

// Called as:
storage.save();
```

**Mutating methods:**

```tolk
fun slice.loadInt(mutate self, len: int): int

// `mutate` means the slice pointer is shifted
var value = cs.loadInt(32);
```

---

### Pure Functions

`@pure` annotation indicates no side effects:

```tolk
@pure
fun min(x: int, y: int): int
    asm "MIN"

@pure
fun address.getWorkchain(self): int8
    asm "REWRITESTDADDR" "DROP"
```

---

### Inline Functions

```tolk
@inline
fun smallFunction() {
    // Code inlined at call site
}

@inline_ref
fun largeFunction() {
    // Inlined but code stored in separate cell
}
```

**Example from wallet-v5:**

```tolk
@inline_ref
fun processExtraActions(extraActions: SnakedExtraActions, isExtension: bool) {
    while (true) {
        val action = lazy ExtraAction.fromSlice(extraActions);
        match (action) {
            AddExtensionExtraAction => { ... }
            RemoveExtensionExtraAction => { ... }
            SetSignatureAllowedExtraAction => { ... }
            else => throw ERROR_UNSUPPORTED_ACTION
        }
        if (!extraActions.hasNext()) {
            return;
        }
        extraActions = extraActions.getNext();
    }
}
```

---

### Get Methods

Get methods are read-only functions exposed to external queries:

```tolk
get fun get_wallet_data(): JettonWalletDataReply {
    val storage = lazy WalletStorage.load();

    return {
        jettonBalance: storage.jettonBalance,
        ownerAddress: storage.ownerAddress,
        minterAddress: storage.minterAddress,
        jettonWalletCode: contract.getCode(),
    }
}
```

**From wallet-v5:**

```tolk
get fun is_signature_allowed(): bool {
    val storage = lazy Storage.load();
    return storage.isSignatureAllowed;
}

get fun seqno(): int {
    val storage = lazy Storage.load();
    return storage.seqno;
}

get fun get_subwallet_id(): int {
    val storage = lazy Storage.load();
    return storage.subwalletId;
}

get fun get_public_key(): int {
    val storage = lazy Storage.load();
    return storage.publicKey;
}

get fun get_extensions(): map<uint256, bool> {
    val storage = lazy Storage.load();
    return storage.extensions;
}
```

---

### Assembly Functions

Functions can be implemented with raw TVM assembly:

```tolk
@pure
fun min(x: int, y: int): int
    asm "MIN"

@pure
fun max(x: int, y: int): int
    asm "MAX"

@pure
fun slice.loadInt(mutate self, len: int): int
    asm( -> 1 0) "LDINT"
```

**Stack manipulation syntax:**

```tolk
asm(c self) "STREF"           // Reorder: c, self -> self with c stored
asm( -> 1 0) "LDREF"          // Return order: stack positions
asm(-> 0 2 1 3) "DICTIREMMIN" // Complex reordering
```

---

### Message Handlers

Special functions for handling incoming messages:

```tolk
fun onInternalMessage(in: InMessage) {
    val msg = lazy AllowedMessageToWallet.fromSlice(in.body);

    match (msg) {
        InternalTransferStep => { ... }
        AskToTransfer => { ... }
        AskToBurn => { ... }
        else => {
            assert (in.body.isEmpty()) throw 0xFFFF
        }
    }
}

fun onBouncedMessage(in: InMessageBounced) {
    in.bouncedBody.skipBouncedPrefix();
    val msg = lazy BounceOpToHandle.fromSlice(in.bouncedBody);
    // ... handle bounced message
}

fun onExternalMessage(inMsgBody: slice) {
    // ... validate and process external message
}
```

---

### Special Annotation for Bounced Messages

```tolk
@on_bounced_policy("manual")
fun onInternalMessage(in: InMessage) {
    // This tells compiler: don't insert code to filter bounced messages
    // Handle them manually if needed
}
```

---

## Variables and Assignments

### Variable Declaration

```tolk
var x: int = 42;
var balance: coins = ton("1.5");
var addr: address = address("EQC...");
```

---

### Type Inference

```tolk
var x = 42;              // Inferred as int
var msg = in.body;       // Inferred as slice
```

---

### val vs var

```tolk
val constant = 100;      // Immutable (read-only)
var mutable = 200;       // Mutable
```

**Example from jetton:**

```tolk
val msg = lazy AllowedMessageToWallet.fromSlice(in.body);
var storage = lazy WalletStorage.load();
storage.jettonBalance += msg.jettonAmount;
```

---

### Multiple Assignment

```tolk
var (quotient, remainder) = divMod(112, 3);
var (workchain, hash) = addr.getWorkchainAndHash();
var (cells, bits, refs, ok) = dataCell.calculateSize(1000);
```

---

### Compound Assignment

```tolk
storage.jettonBalance += msg.jettonAmount;
storage.jettonBalance -= msg.jettonAmount;
msgValue -= (storageFee + JETTON_WALLET_GAS_CONSUMPTION);
```

---

### Variable Scope Rules

Variables cannot be redeclared within the same scope:

```tolk
var a = 10;
var a = 20;  // Error! Cannot redeclare 'a'

// Use assignment instead:
a = 20;      // OK
```

**Different scopes allow shadowing:**

```tolk
var a = 10;

if (true) {
    var a = 30;  // OK - different scope
    // Inner 'a' shadows outer 'a'
}

// Outer 'a' is still 10 here
```

**Block scopes:**

```tolk
{
    var x = 1;
}
// x is not accessible here

while (condition) {
    var temp = doSomething();
    // temp only exists in this loop iteration's scope
}
```

**Function parameters create their own scope:**

```tolk
fun process(value: int) {
    // var value = 5;  // Error! 'value' already declared as parameter
    value = 5;         // OK - assignment
}
```

---

## Control Flow

### if Statement

```tolk
if (condition) {
    // code
}

if (condition) {
    // code
} else {
    // code
}

if (condition1) {
    // code
} else if (condition2) {
    // code
} else {
    // code
}
```

**Example from jetton:**

```tolk
if (msg.forwardTonAmount) {
    msgValue -= (msg.forwardTonAmount + in.originalForwardFee);

    val notifyOwnerMsg = createMessage({ ... });
    notifyOwnerMsg.send(SEND_MODE_PAY_FEES_SEPARATELY);
}

if (msg.sendExcessesTo != null & (msgValue > 0)) {
    val excessesMsg = createMessage({ ... });
    excessesMsg.send(SEND_MODE_IGNORE_ERRORS);
}
```

---

### while Loop

```tolk
while (condition) {
    // code
}
```

**Example from wallet-v5:**

```tolk
while (true) {
    val action = lazy ExtraAction.fromSlice(extraActions);
    match (action) {
        AddExtensionExtraAction => { ... }
        RemoveExtensionExtraAction => { ... }
        SetSignatureAllowedExtraAction => { ... }
        else => throw ERROR_UNSUPPORTED_ACTION
    }
    if (!extraActions.hasNext()) {
        return;
    }
    extraActions = extraActions.getNext();
}
```

---

### do-while Loop

```tolk
do {
    // code
} while (condition);
```

---

### repeat Loop

```tolk
repeat (10) {
    // Executes exactly 10 times
}

var count = 5;
repeat (count) {
    // Executes 5 times
}
```

---

### return Statement

```tolk
fun calculateFee(): coins {
    if (freeTransfer) {
        return 0;
    }
    return baselineFee + proportionalFee;
}
```

**Early return:**

```tolk
fun onInternalMessage(in: InMessage) {
    val msg = lazy AllowedMessageToWalletV5.fromSlice(in.body);

    match (msg) {
        InternalSignedRequest => {
            if (!bodyLongEnough) {
                return;     // Early return if body too short
            }
            // ... continue processing
        }
    }
}
```

---

### Ternary Operator

```tolk
var forwardedMessagesCount = msg.forwardTonAmount ? 2 : 1;
```

---

## Pattern Matching

### match Statement

Used for discriminating union types:

```tolk
val msg = lazy AllowedMessageToWallet.fromSlice(in.body);

match (msg) {
    InternalTransferStep => {
        // Handle internal transfer
        storage.jettonBalance += msg.jettonAmount;
    }

    AskToTransfer => {
        // Handle transfer request
        assert (msg.transferRecipient.getWorkchain() == BASECHAIN) throw ERR_WRONG_WORKCHAIN;
        storage.jettonBalance -= msg.jettonAmount;
    }

    AskToBurn => {
        // Handle burn request
        storage.jettonBalance -= msg.jettonAmount;
    }

    else => {
        // Default case
        assert (in.body.isEmpty()) throw 0xFFFF
    }
}
```

**From wallet-v5:**

```tolk
val action = lazy ExtraAction.fromSlice(extraActions);
match (action) {
    AddExtensionExtraAction => {
        var storage = lazy Storage.load();
        val inserted = storage.extensions.addIfNotExists(extensionAddrHash, true);
        assert (inserted) throw ERROR_ADD_EXTENSION;
        storage.save();
    }

    RemoveExtensionExtraAction => {
        var storage = lazy Storage.load();
        val removed = storage.extensions.delete(extensionAddrHash);
        assert (removed) throw ERROR_REMOVE_EXTENSION;
        storage.save();
    }

    SetSignatureAllowedExtraAction => {
        var storage = lazy Storage.load();
        storage.isSignatureAllowed = action.allowSignature;
        storage.save();
    }

    else => throw ERROR_UNSUPPORTED_ACTION
}
```

---

### Extracting Values in match

**From jetton bounced message handler:**

```tolk
val msg = lazy BounceOpToHandle.fromSlice(in.bouncedBody);
val restoreAmount = match (msg) {
    InternalTransferStep => msg.jettonAmount,
    BurnNotificationForMinter => msg.jettonAmount,
};
// Both branches return coins, stored in restoreAmount
```

---

### Nested Patterns

```tolk
match (outerUnion) {
    Type1 => {
        match (innerUnion) {
            SubType1 => { ... }
            SubType2 => { ... }
        }
    }
    Type2 => { ... }
}
```

---

## Lazy Evaluation

`lazy` keyword delays deserialization until field access:

### Basic Lazy Loading

```tolk
val msg = lazy AllowedMessageToWallet.fromSlice(in.body);
```

**Key point**: Message is not deserialized yet! Deserialization happens when you access fields.

---

### Lazy with match

```tolk
val msg = lazy AllowedMessageToWallet.fromSlice(in.body);

match (msg) {
    InternalTransferStep => {
        // Only InternalTransferStep fields are loaded here
        storage.jettonBalance += msg.jettonAmount;
    }
    AskToTransfer => {
        // Only AskToTransfer fields are loaded here
        assert (msg.transferRecipient.getWorkchain() == BASECHAIN) throw ERR_WRONG_WORKCHAIN;
    }
}
```

**Benefit**: If message is `AskToTransfer`, fields of `InternalTransferStep` are never loaded (gas savings!).

---

### Lazy Storage Loading

```tolk
var storage = lazy WalletStorage.load();

// Storage cell is parsed only when you access fields:
if (in.senderAddress == storage.ownerAddress) {
    // Only now is ownerAddress loaded from storage
}
```

**Full example from jetton:**

```tolk
fun onInternalMessage(in: InMessage) {
    val msg = lazy AllowedMessageToWallet.fromSlice(in.body);

    match (msg) {
        InternalTransferStep => {
            var storage = lazy WalletStorage.load();  // Not loaded yet

            if (in.senderAddress != storage.minterAddress) {  // minterAddress loaded here
                // Validation logic
            }

            storage.jettonBalance += msg.jettonAmount;  // jettonBalance loaded here
            storage.save();
        }

        AskToTransfer => {
            var storage = lazy WalletStorage.load();
            // Storage loaded on first access
            assert (in.senderAddress == storage.ownerAddress) throw ERR_NOT_FROM_OWNER;
        }
    }
}
```

---

### When NOT to Use Lazy

If you access all fields anyway, don't use `lazy`:

```tolk
// Don't use lazy here - all fields accessed
var storage = WalletStorage.load();
debug.print(storage.jettonBalance);
debug.print(storage.ownerAddress);
debug.print(storage.minterAddress);
```

---

### Lazy Benefits

1. **Gas savings**: Only load fields you actually use
2. **Early validation**: Can check opcode/prefix before full deserialization
3. **Pattern matching optimization**: Match on union without loading all variants
4. **Storage optimization**: Load storage fields on-demand

---

## Operators

### Arithmetic Operators

```tolk
var sum = a + b;
var diff = a - b;
var product = a * b;
var quotient = a / b;
var remainder = a % b;
var negation = -a;
```

---

### Comparison Operators

```tolk
a == b    // Equal
a != b    // Not equal
a < b     // Less than
a > b     // Greater than
a <= b    // Less than or equal
a >= b    // Greater than or equal
```

**Example from jetton:**

```tolk
assert (storage.jettonBalance >= msg.jettonAmount) throw ERR_NOT_ENOUGH_BALANCE;
assert (msg.transferRecipient.getWorkchain() == BASECHAIN) throw ERR_WRONG_WORKCHAIN;
```

---

### Logical Operators

```tolk
a & b     // Bitwise AND (also logical AND for bools)
a | b     // Bitwise OR (also logical OR for bools)
a ^ b     // Bitwise XOR
~a        // Bitwise NOT
!a        // Logical NOT (for bool)
```

**Example from wallet-v5:**

```tolk
assert (storage.isSignatureAllowed | storage.extensions.isEmpty()) throw ERROR_SIGNATURE_DISABLED;

if (msg.sendExcessesTo != null & (msgValue > 0)) {
    // Both conditions must be true
}
```

**Note**: `&` and `|` are short-circuiting when used with booleans.

---

### Shift Operators

```tolk
a << b    // Left shift
a >> b    // Right shift (arithmetic)
a >>~ b   // Right shift (logical)
a >>^ b   // Right shift (round to +infinity)
```

---

### Null Coalescing

```tolk
var value = nullableValue ?? defaultValue;
```

**Example:**

```tolk
var recipient = msg.sendExcessesTo ?? senderAddress;
```

---

### Force Unwrap

```tolk
var value = nullableValue!;
```

Throws if nullable is null.

**Example from jetton:**

```tolk
val excessesMsg = createMessage({
    dest: msg.sendExcessesTo!,  // Force unwrap - we checked it's not null
    value: msgValue,
    body: ReturnExcessesBack { ... }
});
```

---

## Type Casting

### as Operator

```tolk
var intValue = boolValue as int;      // true = -1, false = 0
var sliceAddr = address as slice;     // Address is slice internally
var cellType = rawByte as ExoticCellType;
```

**Example from exotic cells:**

```tolk
var (s, isExotic) = c.beginParseSpecial();
if (isExotic) {
    val cellType = s.loadInt(8) as ExoticCellType;
}
```

---

### Safe vs Unsafe Casting

```tolk
// Unsafe - no runtime check:
var addr = anyAddr as address;

// Safe - with runtime check:
var addr = anyAddr.castToInternal();
```

---

## Nullable Types

### Declaring Nullable

```tolk
var maybeAddress: address? = null;
var maybeCell: cell? = null;
var maybeInt: int? = null;
```

---

### Checking for null

```tolk
if (maybeAddress == null) {
    // Handle null case
}

if (maybeAddress != null) {
    // Use maybeAddress!
}
```

**Example from jetton:**

```tolk
struct (0x0f8a7ea5) AskToTransfer {
    queryId: uint64
    jettonAmount: coins
    transferRecipient: address
    sendExcessesTo: address?        // Nullable
    customPayload: cell?            // Nullable
    forwardTonAmount: coins
    forwardPayload: ForwardPayloadRemainder
}

// Usage:
if (msg.sendExcessesTo != null & (msgValue > 0)) {
    val excessesMsg = createMessage({
        dest: msg.sendExcessesTo!,  // Force unwrap
        value: msgValue,
        body: ReturnExcessesBack { ... }
    });
}
```

---

### Non-null Assertion Operator `!`

When you're certain a nullable value is not null, use `!` to bypass null-safety checks:

```tolk
fun processCell(maybeCell: cell?, hasCell: bool) {
    if (hasCell) {
        // We know it's not null, force unwrap
        var actualCell = maybeCell!;
        actualCell.beginParse();
    }
}
```

**From jetton contract:**

```tolk
if (msg.sendExcessesTo != null) {
    createMessage({
        dest: msg.sendExcessesTo!,  // Bypass null check
        value: msgValue
    }).send(SEND_MODE_REGULAR);
}
```

**Warning**: Using `!` on a null value will cause a runtime exception. Only use when you're absolutely certain the value is not null.

---

### Smart Casts

After null check, compiler automatically narrows the type:

```tolk
var maybeAddr: address? = someAddress;

if (maybeAddr != null) {
    // maybeAddr is automatically cast from address? to address
    var wc = maybeAddr.getWorkchain();  // No ! needed
}
```

---

### Optional Address Pattern

```tolk
// For "internal or none" address, use nullable:
var recipient: address? = null;

// NOT "any_address" which includes external addresses
```

---

## Annotations

### @pure

Function has no side effects:

```tolk
@pure
fun calculateFee(amount: coins): coins {
    return mulDivFloor(amount, feeRate, 10000);
}
```

---

### @inline

Inline function at call site:

```tolk
@inline
fun isValidAddress(addr: address): bool {
    return addr.getWorkchain() == BASECHAIN;
}
```

---

### @inline_ref

Inline but store code in separate cell:

```tolk
@inline_ref
fun processComplexLogic() {
    // Large function body
}
```

---

### @deprecated

Mark function as deprecated:

```tolk
@deprecated("use `address == otherAddress`, not `address.bitsEqual(otherAddress)`")
fun address.bitsEqual(self, b: address): bool
    asm "SDEQ"
```

---

### @on_bounced_policy

Control automatic bounced message filtering:

```tolk
@on_bounced_policy("manual")
fun onInternalMessage(in: InMessage) {
    // Don't insert automatic bounced message filter
}
```

---

### @method_id

Specify method ID for code upgrades and special functions:

```tolk
@method_id(1666)
fun afterCodeUpgrade(oldCode: continuation) {
    // Called after contract code upgrade
    // Method ID 1666 allows TVM to find this function
}

@method_id(0)
fun recv_internal(msg: InMessage) {
    // Entry point for internal messages
}
```

**Use cases:**

- Code upgrade handlers (`afterCodeUpgrade`)
- Explicit method IDs for contract interfaces
- Overriding default method ID calculation

---

## Special Constructs

### assert and throw

```tolk
// Assert with custom error code:
assert (condition) throw errorCode;

// Direct throw:
throw errorCode;
```

**Examples from jetton:**

```tolk
assert (msg.forwardPayload.remainingBitsCount()) throw ERR_INVALID_PAYLOAD;
assert (msg.transferRecipient.getWorkchain() == BASECHAIN) throw ERR_WRONG_WORKCHAIN;
assert (in.senderAddress == storage.ownerAddress) throw ERR_NOT_FROM_OWNER;
assert (storage.jettonBalance >= msg.jettonAmount) throw ERR_NOT_ENOUGH_BALANCE;
```

**From wallet-v5:**

```tolk
assert (isSignatureValid(signedSlice.hash(), signature, storage.publicKey)) throw ERROR_INVALID_SIGNATURE;
assert (msg.seqno == storage.seqno) throw ERROR_INVALID_SEQNO;
assert (msg.validUntil > blockchain.now()) throw ERROR_EXPIRED;
```

---

### asm Blocks

Inline TVM assembly:

```tolk
@pure
fun min(x: int, y: int): int
    asm "MIN"

@pure
fun builder.storeRef(mutate self, c: cell): self
    asm(c self) "STREF"

@pure
fun slice.loadInt(mutate self, len: int): int
    asm( -> 1 0) "LDINT"
```

---

### Type Aliases

```tolk
type ForwardPayloadRemainder = RemainingBitsAndRefs
type SnakedExtraActions = RemainingBitsAndRefs

type dict = cell?  // From stdlib
```

---

## Comments

### Single-Line Comments

```tolk
// This is a single-line comment
var balance = 100;  // Balance in nanotons
```

---

### Multi-Line Comments

```tolk
/*
   This is a multi-line comment.
   It can span multiple lines.
*/
```

---

### Documentation Comments

````tolk
/// Loads a signed len-bit integer from a slice.
/// Example:
/// ```
/// var opcode = cs.loadInt(32);
/// ```
@pure
fun slice.loadInt(mutate self, len: int): int
    builtin
````

---

## Advanced Patterns

### Builder Pattern for Messages

```tolk
val deployMsg = createMessage({
    bounce: BounceMode.Only256BitsOfBody,
    dest: calcDeployedJettonWallet(msg.transferRecipient, storage.minterAddress, contract.getCode()),
    value: 0,
    body: InternalTransferStep {
        queryId: msg.queryId,
        jettonAmount: msg.jettonAmount,
        transferInitiator: storage.ownerAddress,
        sendExcessesTo: msg.sendExcessesTo,
        forwardTonAmount: msg.forwardTonAmount,
        forwardPayload: msg.forwardPayload,
    }
});
deployMsg.send(SEND_MODE_CARRY_ALL_REMAINING_MESSAGE_VALUE);
```

---

### Chaining Method Calls

```tolk
var cell = beginCell()
    .storeUint(0x12345678, 32)
    .storeAddress(recipient)
    .storeCoins(amount)
    .storeRef(payload)
    .endCell();

balances
    .set(user1, balance1)
    .set(user2, balance2)
    .set(user3, balance3);
```

---

### State Init Pattern

```tolk
fun calcDeployedJettonWallet(ownerAddress: address, minterAddress: address, jettonWalletCode: cell): AutoDeployAddress {
    val emptyWalletStorage: WalletStorage = {
        jettonBalance: 0,
        ownerAddress,
        minterAddress,
    };

    return {
        stateInit: {
            code: jettonWalletCode,
            data: emptyWalletStorage.toCell()
        }
    }
}

// Calculate address:
var walletAddr = calcDeployedJettonWallet(owner, minter, code).calculateAddress();
```

---

## Syntax Conveniences

### Trailing Commas

Tolk supports trailing commas in various contexts for better code maintenance:

**In tensors and tuples:**

```tolk
var items = (
    totalSupply,
    verifiedCode,
    validatorsList,  // Trailing comma OK
);

var coords = [
    x,
    y,
    z,  // Makes adding new items easier
];
```

**In function calls:**

```tolk
createMessage({
    bounce: BounceMode.NoBounce,
    value: ton("0.05"),
    dest: recipient,
    body: someData,  // Trailing comma
});
```

**In function parameters:**

```tolk
fun processData(
    value1: int,
    value2: slice,
    value3: cell,  // Trailing comma
) {
    // ...
}
```

**Benefits:**

- Easier to add/remove items
- Cleaner git diffs
- Less prone to syntax errors when modifying lists

---

### Optional Semicolons

Semicolons are optional in certain contexts:

**Last statement in a block:**

```tolk
fun calculateFee(): coins {
    if (freeTransfer) {
        return 0  // No semicolon needed
    }
    return baseFee + additionalFee  // No semicolon needed
}
```

**Top-level declarations:**

```tolk
const MAX_SUPPLY = ton("1000000")  // No semicolon

struct Config {
    timeout: int = 3600
    enabled: bool = true  // No semicolon
}

type MessageType =
    | Transfer
    | Burn  // No semicolon
```

**When semicolons ARE required:**

```tolk
// Multiple statements in same block
var x = 1;  // Semicolon required
var y = 2;  // Semicolon required

// Not the last statement
if (condition) {
    doSomething();  // Semicolon required
    doOtherThing()  // No semicolon (last in block)
}
```

**Best practice:** Use semicolons for clarity, omit them only when it improves readability.

---

## Best Practices

1. **Use `lazy`** for conditional field access
2. **Use `val`** for read-only variables
3. **Use `assert`** for validation with clear error codes
4. **Use union types** for message discrimination
5. **Use `match`** for type-safe message handling
6. **Use structs with opcodes** for message definitions
7. **Use nullable types** (`address?`) instead of "any" types
8. **Use `const`** for error codes and configuration
9. **Use methods on structs** for better organization
10. **Document public APIs** with `///` comments

---

## Summary

Tolk provides:

- **Modern syntax** with type inference
- **Lazy evaluation** for gas optimization
- **Pattern matching** for safe message handling
- **Union types** for clean message discrimination
- **Struct methods** for object-oriented patterns
- **Nullable types** for safe null handling
- **Annotations** for compiler hints
- **Assembly integration** for low-level optimization

For practical patterns, see [03-patterns-and-practices.md](./03-patterns-and-practices.md).
For stdlib functions, see [01-stdlib-reference.md](./01-stdlib-reference.md).
For testing, see [04-testing-guide.md](./04-testing-guide.md).
