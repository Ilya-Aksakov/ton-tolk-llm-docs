# Tolk Patterns and Best Practices

Real-world patterns extracted from production TON contracts: jettons, NFTs, wallets, and vesting contracts. This guide shows proven patterns for storage, messages, validation, error handling, and gas management.

## Table of Contents

1. [Storage Management Pattern](#storage-management-pattern)
2. [Message Definition Pattern](#message-definition-pattern)
3. [Message Handling Pattern](#message-handling-pattern)
4. [Error Handling Pattern](#error-handling-pattern)
5. [Fee Management Pattern](#fee-management-pattern)
6. [Address Calculation (StateInit) Pattern](#address-calculation-stateinit-pattern)
7. [Bounced Message Handling Pattern](#bounced-message-handling-pattern)
8. [Excesses Return Pattern](#excesses-return-pattern)
9. [Forward Payload Pattern](#forward-payload-pattern)
10. [Workchain Validation Pattern](#workchain-validation-pattern)
11. [Reserve for Storage Pattern](#reserve-for-storage-pattern)
12. [Admin Transfer Pattern](#admin-transfer-pattern)
13. [Code Upgrade Pattern](#code-upgrade-pattern)
14. [Batch Operations Pattern](#batch-operations-pattern)
15. [External Message + Signature Pattern](#external-message--signature-pattern)
16. [Validation Chain Pattern](#validation-chain-pattern)
17. [Try-Catch Pattern](#try-catch-pattern)

---

## Storage Management Pattern

**Problem**: Need to efficiently save/load contract state from c4 register.

**Solution**: Create storage struct + load/save methods.

### Basic Pattern

```tolk
// storage.tolk
struct WalletStorage {
    jettonBalance: coins
    ownerAddress: address
    minterAddress: address
}

fun WalletStorage.load() {
    return WalletStorage.fromCell(contract.getData())
}

fun WalletStorage.save(self) {
    contract.setData(self.toCell())
}
```

**Usage:**

```tolk
// Load storage (lazy loading for gas optimization)
var storage = lazy WalletStorage.load();

// Modify
storage.jettonBalance += amount;

// Save back to c4
storage.save();
```

---

### Complex Storage with Nested Refs

```tolk
struct ExtraData {
    param1: int
    param2: coins
}

struct MinterStorage {
    totalSupply: coins
    adminAddress: address
    content: cell
    jettonWalletCode: cell
    extraData: Cell<ExtraData>  // Stored in ref
}
```

**Loading nested data:**

```tolk
var storage = MinterStorage.load();
var extra = storage.extraData.load();  // Load from ref
```

---

### Map in Storage Pattern

```tolk
// From wallet-v5
struct Storage {
    isSignatureAllowed: bool
    seqno: uint32
    subwalletId: uint32
    publicKey: uint256
    extensions: map<uint256, bool>  // Address hash -> enabled
}
```

**Usage:**

```tolk
var storage = Storage.load();

// Check extension exists
if (storage.extensions.exists(addressHash)) {
    // Extension is registered
}

// Add extension
storage.extensions.addIfNotExists(newExtHash, true);

// Remove extension
storage.extensions.delete(oldExtHash);

storage.save();
```

---

### Load Only What You Need Pattern

```tolk
fun onInternalMessage(in: InMessage) {
    val msg = lazy AllowedMessage.fromSlice(in.body);

    match (msg) {
        SimpleMessage => {
            // Don't load storage if not needed
            processSimpleMessage(msg);
        }

        ComplexMessage => {
            // Load storage only for complex operations
            var storage = lazy WalletStorage.load();
            storage.balance += msg.amount;
            storage.save();
        }
    }
}
```

**Benefit**: Save gas by not loading storage when not needed.

---

## Message Definition Pattern

**Problem**: Need type-safe message structures with opcodes.

**Solution**: Define structs with opcode prefixes and typed fields.

### Basic Message Definitions

```tolk
// messages.tolk
type ForwardPayloadRemainder = RemainingBitsAndRefs

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

struct (0xd53276db) ReturnExcessesBack {
    queryId: uint64
}
```

**Key points:**

- Opcode in hex format: `struct (0x12345678)`
- Use nullable for optional fields: `address?`, `cell?`
- Use `RemainingBitsAndRefs` for "rest of message" payload
- Include queryId for request correlation

---

### Union Type for Allowed Messages

```tolk
type AllowedMessageToWallet =
    | AskToTransfer
    | AskToBurn
    | InternalTransferStep

type AllowedMessageToMinter =
    | MintNewJettons
    | BurnNotificationForMinter
    | RequestWalletAddress
    | ChangeMinterAdmin
    | ChangeMinterContent
```

---

### ASCII Opcodes Pattern

```tolk
// From wallet-v5 - opcodes are ASCII strings
struct (0x6578746E) ExtensionActionRequest {  // "extn"
    queryId: uint64
    outActions: OutActionsCell?
    hasExtraActions: bool
    extraActions: SnakedExtraActions
}

struct (0x7369676E) ExternalSignedRequest {  // "sign"
    walletId: uint32
    validUntil: uint32
    seqno: uint32
    outActions: OutActionsCell?
    hasExtraActions: bool
    extraActions: SnakedExtraActions
}
```

---

### Nested Cell Messages Pattern

```tolk
struct (0x00000015) MintNewJettons {
    queryId: uint64
    mintRecipient: address
    tonAmount: coins
    internalTransferMsg: Cell<InternalTransferStep>  // Message in ref
}
```

**Usage:**

```tolk
// Load nested message from ref
var internalTransfer = msg.internalTransferMsg.load();

// Send nested message as body
val deployMsg = createMessage({
    dest: walletAddress,
    value: msg.tonAmount,
    body: msg.internalTransferMsg,  // Already a cell ref
});
```

---

## Message Handling Pattern

**Problem**: Need type-safe message parsing and handling.

**Solution**: Lazy loading + pattern matching.

### Standard Handler Pattern

```tolk
fun onInternalMessage(in: InMessage) {
    val msg = lazy AllowedMessageToWallet.fromSlice(in.body);

    match (msg) {
        InternalTransferStep => {
            var storage = lazy WalletStorage.load();

            // Validate sender
            if (in.senderAddress != storage.minterAddress) {
                assert (in.senderAddress == calcAddressOfJettonWallet(
                    msg.transferInitiator!,
                    storage.minterAddress,
                    contract.getCode()
                )) throw ERR_INVALID_WALLET;
            }

            // Update balance
            storage.jettonBalance += msg.jettonAmount;
            storage.save();

            // Forward notification if needed
            if (msg.forwardTonAmount) {
                val notifyMsg = createMessage({
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
                notifyMsg.send(SEND_MODE_PAY_FEES_SEPARATELY);
            }

            // Return excesses
            if (msg.sendExcessesTo != null) {
                returnExcesses(msg.sendExcessesTo!, msgValue, msg.queryId);
            }
        }

        AskToTransfer => {
            var storage = lazy WalletStorage.load();

            // Validate sender is owner
            assert (in.senderAddress == storage.ownerAddress) throw ERR_NOT_FROM_OWNER;

            // Validate balance
            assert (storage.jettonBalance >= msg.jettonAmount) throw ERR_NOT_ENOUGH_BALANCE;

            // Deduct balance
            storage.jettonBalance -= msg.jettonAmount;
            storage.save();

            // Send to recipient
            sendTransferToRecipient(msg);
        }

        else => {
            // Ignore empty messages, throw on unknown opcodes
            assert (in.body.isEmpty()) throw 0xFFFF
        }
    }
}
```

---

### Ignoring Bounced Messages Manually

```tolk
@on_bounced_policy("manual")  // Don't auto-filter bounces
fun onInternalMessage(in: InMessage) {
    val msg = lazy AllowedMessage.fromSlice(in.body);

    match (msg) {
        // Your message handlers
        MessageType1 => { ... }
        MessageType2 => { ... }

        else => {
            // If opcode is strange or body is empty (including bounced),
            // just ignore, do not throw
        }
    }
}
```

---

### Early Return Pattern

```tolk
fun onInternalMessage(in: InMessage) {
    val msg = lazy AllowedMessage.fromSlice(in.body);

    match (msg) {
        ExtensionActionRequest => {
            // Validate sender
            var (senderWorkchain, senderAddrHash) = in.senderAddress.getWorkchainAndHash();
            if (myWorkchain != senderWorkchain) {
                return;  // Wrong workchain, silently ignore
            }

            val storage = lazy Storage.load();
            if (!storage.extensions.exists(senderAddrHash)) {
                return;  // Not an extension, silently accept coins
            }

            // Process request
            processExtensionRequest(msg);
        }
    }
}
```

---

## Error Handling Pattern

**Problem**: Need clear, consistent error reporting.

**Solution**: Define error constants + assert statements.

### Error Constants

```tolk
// errors.tolk
const ERR_INVALID_OP = 709
const ERR_NOT_FROM_ADMIN = 73
const ERR_UNAUTHORIZED_BURN = 74
const ERR_NOT_ENOUGH_AMOUNT_TO_RESPOND = 75
const ERR_NOT_FROM_OWNER = 705
const ERR_NOT_ENOUGH_TON = 709
const ERR_NOT_ENOUGH_GAS = 707
const ERR_INVALID_WALLET = 707
const ERR_WRONG_WORKCHAIN = 333
const ERR_NOT_ENOUGH_BALANCE = 706
const ERR_INVALID_PAYLOAD = 708
```

---

### Validation Chain Pattern

```tolk
fun onInternalMessage(in: InMessage) {
    val msg = lazy AllowedMessage.fromSlice(in.body);

    match (msg) {
        AskToTransfer => {
            // Validation chain with clear error codes
            assert (msg.forwardPayload.remainingBitsCount()) throw ERR_INVALID_PAYLOAD;
            assert (msg.transferRecipient.getWorkchain() == BASECHAIN) throw ERR_WRONG_WORKCHAIN;

            var storage = lazy WalletStorage.load();
            assert (in.senderAddress == storage.ownerAddress) throw ERR_NOT_FROM_OWNER;
            assert (storage.jettonBalance >= msg.jettonAmount) throw ERR_NOT_ENOUGH_BALANCE;
            assert (in.valueCoins > calculateMinimumFee(msg)) throw ERR_NOT_ENOUGH_TON;

            // Process transfer
            processTransfer(msg, storage);
        }
    }
}
```

---

### Custom Exception Numbers

```tolk
// From wallet-v5
const ERROR_INVALID_SIGNATURE = 35
const ERROR_INVALID_SEQNO = 36
const ERROR_INVALID_WALLET_ID = 37
const ERROR_EXPIRED = 38
const ERROR_EXTENSION_WRONG_WORKCHAIN = 39
const ERROR_ADD_EXTENSION = 40
const ERROR_REMOVE_EXTENSION = 41
const ERROR_UNSUPPORTED_ACTION = 42
```

---

## Fee Management Pattern

**Problem**: Need to calculate and reserve fees correctly.

**Solution**: Constants + careful balance tracking.

### Fee Constants

```tolk
// fees-management.tolk
const MINIMAL_MESSAGE_VALUE_BOUND = ton("0.01")
const MIN_TONS_FOR_STORAGE = ton("0.01")
const JETTON_WALLET_GAS_CONSUMPTION = ton("0.015")
```

---

### Fee Calculation Pattern

```tolk
fun onInternalMessage(in: InMessage) {
    val msg = lazy AllowedMessage.fromSlice(in.body);

    match (msg) {
        InternalTransferStep => {
            var storage = lazy WalletStorage.load();
            storage.jettonBalance += msg.jettonAmount;
            storage.save();

            // Calculate fees
            var msgValue = in.valueCoins;
            var tonBalanceBeforeMsg = contract.getOriginalBalance() - msgValue;
            var storageFee = MIN_TONS_FOR_STORAGE - min(tonBalanceBeforeMsg, MIN_TONS_FOR_STORAGE);
            msgValue -= (storageFee + JETTON_WALLET_GAS_CONSUMPTION);

            // Deduct forward fee if forwarding
            if (msg.forwardTonAmount) {
                msgValue -= (msg.forwardTonAmount + in.originalForwardFee);

                val notifyMsg = createMessage({
                    bounce: BounceMode.NoBounce,
                    dest: storage.ownerAddress,
                    value: msg.forwardTonAmount,
                    body: TransferNotificationForRecipient { ... }
                });
                notifyMsg.send(SEND_MODE_PAY_FEES_SEPARATELY);
            }

            // Return remaining as excesses
            if (msg.sendExcessesTo != null & (msgValue > 0)) {
                returnExcesses(msg.sendExcessesTo!, msgValue, msg.queryId);
            }
        }
    }
}
```

---

### Validating Enough TON Attached

```tolk
AskToTransfer => {
    var storage = lazy WalletStorage.load();

    var forwardedMessagesCount = msg.forwardTonAmount ? 2 : 1;

    // Validate sender attached enough TON
    assert (in.valueCoins >
        msg.forwardTonAmount +
        forwardedMessagesCount * in.originalForwardFee +
        (2 * JETTON_WALLET_GAS_CONSUMPTION + MIN_TONS_FOR_STORAGE)
    ) throw ERR_NOT_ENOUGH_TON;

    // Process transfer...
}
```

---

## Address Calculation (StateInit) Pattern

**Problem**: Need to calculate jetton wallet / NFT item addresses.

**Solution**: StateInit-based address calculation.

### Basic Pattern

```tolk
// jetton-utils.tolk
fun calcDeployedJettonWallet(
    ownerAddress: address,
    minterAddress: address,
    jettonWalletCode: cell
): AutoDeployAddress {
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

fun calcAddressOfJettonWallet(
    ownerAddress: address,
    minterAddress: address,
    jettonWalletCode: cell
): address {
    val jwDeployed = calcDeployedJettonWallet(ownerAddress, minterAddress, jettonWalletCode);
    return jwDeployed.calculateAddress()
}
```

---

### Usage: Validate Sender is Jetton Wallet

```tolk
fun onInternalMessage(in: InMessage) {
    val msg = lazy AllowedMessage.fromSlice(in.body);

    match (msg) {
        BurnNotificationForMinter => {
            var storage = lazy MinterStorage.load();

            // Calculate expected wallet address and validate sender
            assert (in.senderAddress == calcAddressOfJettonWallet(
                msg.burnInitiator,
                contract.getAddress(),
                storage.jettonWalletCode
            )) throw ERR_UNAUTHORIZED_BURN;

            // Process burn...
        }
    }
}
```

---

### Usage: Deploy New Wallet

```tolk
AskToTransfer => {
    var storage = lazy WalletStorage.load();
    storage.jettonBalance -= msg.jettonAmount;
    storage.save();

    // Deploy or send to existing wallet
    val deployMsg = createMessage({
        bounce: BounceMode.Only256BitsOfBody,
        dest: calcDeployedJettonWallet(
            msg.transferRecipient,
            storage.minterAddress,
            contract.getCode()
        ),
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
}
```

---

### Sharding Pattern

For sharded jettons (wallet in same shard as owner):

```tolk
// sharding.tolk
const MY_WORKCHAIN = BASECHAIN
const SHARD_DEPTH = 8

fun calcDeployedJettonWallet(
    ownerAddress: address,
    minterAddress: address,
    jettonWalletCode: cell
): AutoDeployAddress {
    val emptyWalletStorage: WalletStorage = {
        jettonBalance: 0,
        ownerAddress,
        minterAddress,
    };

    return {
        workchain: MY_WORKCHAIN,
        stateInit: {
            code: jettonWalletCode,
            data: emptyWalletStorage.toCell()
        },
        toShard: {
            fixedPrefixLength: SHARD_DEPTH,
            closeTo: ownerAddress
        }
    }
}
```

**Result**: Jetton wallet address has same shard prefix as owner (e.g., both start with `01010101`).

---

## Bounced Message Handling Pattern

**Problem**: Need to restore state when message bounces.

**Solution**: Implement `onBouncedMessage` handler.

### Basic Bounced Handler

```tolk
fun onBouncedMessage(in: InMessageBounced) {
    in.bouncedBody.skipBouncedPrefix();  // Skip 0xFFFFFFFF

    val msg = lazy BounceOpToHandle.fromSlice(in.bouncedBody);
    val restoreAmount = match (msg) {
        InternalTransferStep => msg.jettonAmount,
        BurnNotificationForMinter => msg.jettonAmount,
    };

    var storage = lazy WalletStorage.load();
    storage.jettonBalance += restoreAmount;  // Restore balance
    storage.save();
}
```

---

### Union Type for Bounced Messages

```tolk
type BounceOpToHandle =
    | InternalTransferStep
    | BurnNotificationForMinter
```

Only include message types that can bounce and need handling.

---

### Rich Bounce Mode Pattern

```tolk
// Using rich bounce for full context
val deployMsg = createMessage({
    bounce: BounceMode.RichBounce,
    dest: recipient,
    value: amount,
    body: ComplexMessage { ... }
});

fun onBouncedMessage(in: InMessageBounced) {
    // Parse rich bounce body
    val rich = lazy RichBounceBody.fromSlice(in.bouncedBody);

    // Access original body
    val originalBody = rich.originalBody.beginParse();

    // Access error details
    debug.print(rich.exitCode);
    debug.print(rich.bouncedByPhase);

    if (rich.computePhase != null) {
        debug.print(rich.computePhase!.gasUsed);
    }
}
```

---

## Excesses Return Pattern

**Problem**: Need to return unused TON to sender.

**Solution**: Send excesses message with remaining balance.

### Standard Excesses Pattern

```tolk
struct (0xd53276db) ReturnExcessesBack {
    queryId: uint64
}

fun returnExcesses(recipient: address, amount: coins, queryId: uint64) {
    if (amount > 0) {
        val excessesMsg = createMessage({
            bounce: BounceMode.NoBounce,
            dest: recipient,
            value: amount,
            body: ReturnExcessesBack {
                queryId: queryId
            }
        });
        excessesMsg.send(SEND_MODE_IGNORE_ERRORS);
    }
}
```

---

### Usage in Message Handler

```tolk
InternalTransferStep => {
    var storage = lazy WalletStorage.load();
    storage.jettonBalance += msg.jettonAmount;
    storage.save();

    var msgValue = in.valueCoins;

    // Calculate fees
    var tonBalanceBeforeMsg = contract.getOriginalBalance() - msgValue;
    var storageFee = MIN_TONS_FOR_STORAGE - min(tonBalanceBeforeMsg, MIN_TONS_FOR_STORAGE);
    msgValue -= (storageFee + JETTON_WALLET_GAS_CONSUMPTION);

    // Forward notification
    if (msg.forwardTonAmount) {
        msgValue -= (msg.forwardTonAmount + in.originalForwardFee);
        sendNotification(msg);
    }

    // Return remaining as excesses
    if (msg.sendExcessesTo != null & (msgValue > 0)) {
        returnExcesses(msg.sendExcessesTo!, msgValue, msg.queryId);
    }
}
```

---

### Carry All Remaining Value Pattern

```tolk
// When forwarding message, carry all remaining value
val respondMsg = createMessage({
    bounce: BounceMode.Only256BitsOfBody,
    dest: in.senderAddress,
    value: 0,  // Will carry all remaining
    body: ResponseMessage { ... }
});
respondMsg.send(SEND_MODE_CARRY_ALL_REMAINING_MESSAGE_VALUE);
```

---

## Forward Payload Pattern

**Problem**: Need to pass arbitrary data through transfer chain.

**Solution**: Use `RemainingBitsAndRefs` for "rest of message".

### Message Definition

```tolk
type ForwardPayloadRemainder = RemainingBitsAndRefs

struct (0x0f8a7ea5) AskToTransfer {
    queryId: uint64
    jettonAmount: coins
    transferRecipient: address
    sendExcessesTo: address?
    customPayload: cell?
    forwardTonAmount: coins
    forwardPayload: ForwardPayloadRemainder  // Everything after this field
}
```

---

### Forwarding Payload

```tolk
AskToTransfer => {
    // Validate payload (at least some bits present)
    assert (msg.forwardPayload.remainingBitsCount()) throw ERR_INVALID_PAYLOAD;

    // Forward payload to recipient wallet
    val deployMsg = createMessage({
        dest: recipientWallet,
        value: 0,
        body: InternalTransferStep {
            queryId: msg.queryId,
            jettonAmount: msg.jettonAmount,
            transferInitiator: storage.ownerAddress,
            sendExcessesTo: msg.sendExcessesTo,
            forwardTonAmount: msg.forwardTonAmount,
            forwardPayload: msg.forwardPayload,  // Pass through
        }
    });
    deployMsg.send(SEND_MODE_CARRY_ALL_REMAINING_MESSAGE_VALUE);
}
```

---

### Receiving Forward Payload

```tolk
InternalTransferStep => {
    // Update balance
    storage.jettonBalance += msg.jettonAmount;
    storage.save();

    // Forward notification with payload to owner
    if (msg.forwardTonAmount) {
        val notifyMsg = createMessage({
            bounce: BounceMode.NoBounce,
            dest: storage.ownerAddress,
            value: msg.forwardTonAmount,
            body: TransferNotificationForRecipient {
                queryId: msg.queryId,
                jettonAmount: msg.jettonAmount,
                transferInitiator: msg.transferInitiator,
                forwardPayload: msg.forwardPayload  // Delivered to owner
            }
        });
        notifyMsg.send(SEND_MODE_PAY_FEES_SEPARATELY);
    }
}
```

---

## Workchain Validation Pattern

**Problem**: Contract only works in specific workchain.

**Solution**: Validate workchain early.

### Recipient Workchain Validation

```tolk
AskToTransfer => {
    // Only allow transfers to basechain
    assert (msg.transferRecipient.getWorkchain() == BASECHAIN) throw ERR_WRONG_WORKCHAIN;

    // Process transfer...
}
```

---

### Sender Workchain Validation

```tolk
ExtensionActionRequest => {
    var (senderWorkchain, senderAddrHash) = in.senderAddress.getWorkchainAndHash();
    var myWorkchain = contract.getAddress().getWorkchain();

    assert (myWorkchain == senderWorkchain) throw ERROR_EXTENSION_WRONG_WORKCHAIN;

    // Process request...
}
```

---

### Conditional Response Based on Workchain

```tolk
RequestWalletAddress => {
    var walletAddress: address? = null;

    // Only calculate address if owner in basechain
    if (msg.ownerAddress.getWorkchain() == BASECHAIN) {
        var storage = lazy MinterStorage.load();
        walletAddress = calcAddressOfJettonWallet(
            msg.ownerAddress,
            contract.getAddress(),
            storage.jettonWalletCode
        );
    }

    val respondMsg = createMessage({
        dest: in.senderAddress,
        value: 0,
        body: ResponseWalletAddress {
            queryId: msg.queryId,
            jettonWalletAddress: walletAddress,  // null if wrong workchain
            ownerAddress: respondOwnerAddress,
        }
    });
    respondMsg.send(SEND_MODE_CARRY_ALL_REMAINING_MESSAGE_VALUE);
}
```

---

## Reserve for Storage Pattern

**Problem**: Need to ensure contract keeps minimum balance for storage fees.

**Solution**: Reserve storage fee from incoming value.

### Basic Reserve Pattern

```tolk
fun onInternalMessage(in: InMessage) {
    // Reserve minimum for storage
    reserveToncoinsOnBalance(MIN_TONS_FOR_STORAGE, RESERVE_MODE_EXACT_AMOUNT);

    // Now process message
    val msg = lazy AllowedMessage.fromSlice(in.body);
    // ...
}
```

---

### Reserve All But Amount Pattern

```tolk
fun processExpensiveOperation() {
    // Reserve all but 1 TON
    reserveToncoinsOnBalance(
        ton("1.0"),
        RESERVE_MODE_ALL_BUT_AMOUNT
    );

    // Now use that 1 TON for operations
    sendMultipleMessages();
}
```

---

### Manual Storage Fee Calculation

```tolk
InternalTransferStep => {
    var msgValue = in.valueCoins;

    // Calculate storage fee based on balance before message
    var tonBalanceBeforeMsg = contract.getOriginalBalance() - msgValue;
    var storageFee = MIN_TONS_FOR_STORAGE - min(tonBalanceBeforeMsg, MIN_TONS_FOR_STORAGE);

    // Deduct from available value
    msgValue -= (storageFee + JETTON_WALLET_GAS_CONSUMPTION);

    // Use remaining msgValue for operations
}
```

---

## Admin Transfer Pattern

**Problem**: Need to transfer admin privileges safely.

**Solution**: Validate sender + update storage.

### Basic Admin Transfer

```tolk
ChangeMinterAdmin => {
    var storage = lazy MinterStorage.load();

    // Only current admin can change admin
    assert (in.senderAddress == storage.adminAddress) throw ERR_NOT_FROM_ADMIN;

    // Update admin
    storage.adminAddress = msg.newAdminAddress;
    storage.save();
}
```

---

### Admin-Only Operations Pattern

```tolk
type AdminOnlyMessage =
    | ChangeMinterAdmin
    | ChangeMinterContent
    | MintNewJettons

fun onInternalMessage(in: InMessage) {
    val msg = lazy AllowedMessage.fromSlice(in.body);

    match (msg) {
        ChangeMinterAdmin => {
            validateAdmin(in.senderAddress);
            var storage = lazy MinterStorage.load();
            storage.adminAddress = msg.newAdminAddress;
            storage.save();
        }

        ChangeMinterContent => {
            validateAdmin(in.senderAddress);
            var storage = lazy MinterStorage.load();
            storage.content = msg.newContent;
            storage.save();
        }

        MintNewJettons => {
            validateAdmin(in.senderAddress);
            processMint(msg);
        }
    }
}

fun validateAdmin(sender: address) {
    var storage = lazy MinterStorage.load();
    assert (sender == storage.adminAddress) throw ERR_NOT_FROM_ADMIN;
}
```

---

## Code Upgrade Pattern

**Problem**: Need to upgrade contract code after deployment.

**Solution**: Use `setTvmRegisterC3` or `setCodePostponed`.

### Postponed Code Update Pattern

```tolk
struct UpgradeCode {
    newCode: cell
}

fun onInternalMessage(in: InMessage) {
    val msg = lazy AllowedMessage.fromSlice(in.body);

    match (msg) {
        UpgradeCode => {
            var storage = lazy Storage.load();
            assert (in.senderAddress == storage.adminAddress) throw ERR_NOT_FROM_ADMIN;

            // Schedule code update after this transaction succeeds
            contract.setCodePostponed(msg.newCode);

            // Code will be updated after successful completion
            // Next transaction will use new code
        }
    }
}
```

---

### Immediate Code Update with C3 Register

```tolk
fun upgradeCodeNow(newCode: cell) {
    var newContinuation = transformSliceToContinuation(newCode.beginParse());
    setTvmRegisterC3(newContinuation);

    // Any subsequent function call will use new code
    // Current execution continues with old code
}
```

**Warning**: Immediate updates are risky. Prefer `setCodePostponed`.

---

## Batch Operations Pattern

**Problem**: Need to process multiple operations in one message.

**Solution**: Iterate through refs or use snaked structure.

### Snaked Structure Pattern

```tolk
// From wallet-v5
type SnakedExtraActions = RemainingBitsAndRefs

fun SnakedExtraActions.hasNext(self) {
    return self.remainingRefsCount()
}

fun SnakedExtraActions.getNext(self): SnakedExtraActions {
    return self.preloadRef().beginParse()
}
```

**Usage:**

```tolk
fun processExtraActions(extraActions: SnakedExtraActions) {
    while (true) {
        val action = lazy ExtraAction.fromSlice(extraActions);

        match (action) {
            AddExtensionExtraAction => { processAdd(action); }
            RemoveExtensionExtraAction => { processRemove(action); }
            SetSignatureAllowedExtraAction => { processSet(action); }
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

### Batch Whitelist Addition Pattern

```tolk
// From vesting contract
struct RequestAddToWhitelist {
    queryId: uint64
    addrToWhitelist: address
    restRefs: RemainingBitsAndRefs  // More addresses in refs
}

fun onInternalMessage(in: InMessage) {
    match (msg) {
        RequestAddToWhitelist => {
            var storage = lazy Storage.load();

            var addrToWhitelist = msg.addrToWhitelist;
            var nextRef = msg.restRefs;

            do {
                storage.whitelist.addWhitelisted(addrToWhitelist);

                var moreRefs = nextRef.remainingRefsCount();
                if (moreRefs) {
                    nextRef = nextRef.loadRef().beginParse();
                    addrToWhitelist = nextRef.loadAddress();
                }
            } while (moreRefs);

            storage.save();
        }
    }
}
```

---

## External Message + Signature Pattern

**Problem**: Need to process external messages with signature verification.

**Solution**: Validate signature, seqno, expiry.

### Basic External Handler Pattern

```tolk
fun onExternalMessage(inMsgBody: slice) {
    // Extract signature from end of message
    var signature = inMsgBody.getLastBits(SIZE_SIGNATURE);
    var signedSlice = inMsgBody.removeLastBits(SIZE_SIGNATURE);

    // Parse signed portion
    var msg = ExternalSignedRequest.fromSlice(signedSlice, {
        throwIfOpcodeDoesNotMatch: ERROR_INVALID_MESSAGE_OPERATION,
    });

    var storage = lazy Storage.load();

    // Validate signature
    assert (isSignatureValid(signedSlice.hash(), signature, storage.publicKey))
        throw ERROR_INVALID_SIGNATURE;

    // Validate seqno (replay protection)
    assert (msg.seqno == storage.seqno) throw ERROR_INVALID_SEQNO;

    // Validate wallet ID
    assert (msg.walletId == storage.subwalletId) throw ERROR_INVALID_WALLET_ID;

    // Validate not expired
    assert (msg.validUntil > blockchain.now()) throw ERROR_EXPIRED;

    // Accept message (buy gas)
    acceptExternalMessage();

    // Increment seqno to prevent replay
    storage.seqno += 1;
    storage.save();

    // Commit before processing actions
    commitContractDataAndActions();

    // Process actions
    processActions(msg.outActions, msg.hasExtraActions, msg.extraActions);
}
```

---

### Two-Phase Commit Pattern

```tolk
fun onExternalMessage(inMsgBody: slice) {
    // ... validate signature, seqno, expiry

    acceptExternalMessage();

    // Store and commit seqno increment to prevent replays
    // even if subsequent actions fail
    storage.seqno += 1;
    storage.save();

    commitContractDataAndActions();  // Commit c4 (storage) and c5 (actions)

    // Now process actions - if they fail, seqno is still incremented
    processActions(msg.outActions);
}
```

**Benefit**: Prevents replay attacks even if action processing fails.

---

## Validation Chain Pattern

**Problem**: Multiple validations needed before processing.

**Solution**: Chain asserts with clear error codes.

### Validation Chain Example

```tolk
AskToTransfer => {
    // 1. Validate payload format
    assert (msg.forwardPayload.remainingBitsCount()) throw ERR_INVALID_PAYLOAD;

    // 2. Validate workchain
    assert (msg.transferRecipient.getWorkchain() == BASECHAIN) throw ERR_WRONG_WORKCHAIN;

    // 3. Load and validate storage
    var storage = lazy WalletStorage.load();

    // 4. Validate authorization
    assert (in.senderAddress == storage.ownerAddress) throw ERR_NOT_FROM_OWNER;

    // 5. Validate balance
    assert (storage.jettonBalance >= msg.jettonAmount) throw ERR_NOT_ENOUGH_BALANCE;

    // 6. Validate attached TON
    var forwardedMessagesCount = msg.forwardTonAmount ? 2 : 1;
    assert (in.valueCoins >
        msg.forwardTonAmount +
        forwardedMessagesCount * in.originalForwardFee +
        (2 * JETTON_WALLET_GAS_CONSUMPTION + MIN_TONS_FOR_STORAGE)
    ) throw ERR_NOT_ENOUGH_TON;

    // All validations passed - process transfer
    processTransfer(msg, storage);
}
```

---

### Complex Validation with Custom Logic

```tolk
// From vesting contract
fun validateIfLockedAmount(
    msg: AttachedMessage,
    whitelist: Whitelist,
    lockedAmount: coins,
    senderAddress: address
) {
    val (_, _, dest, value, _, _) = msg.msgCell.parseInternalMsgHeader();

    // Validate destination whitelisted
    assert (whitelist.isWhitelisted(dest)) throw ERROR_DEST_NOT_WHITELISTED;

    // Validate amount doesn't exceed unlocked balance
    var unlockedBalance = contract.getOriginalBalance() - lockedAmount;
    assert (value <= unlockedBalance) throw ERROR_VALUE_EXCEEDS_UNLOCKED;

    // More validations...
}
```

---

## Try-Catch Pattern

**Problem**: Need to handle errors gracefully.

**Solution**: Use try-catch blocks.

### Basic Try-Catch

```tolk
fun onExternalMessage(extBody: slice) {
    // ... validate signature, seqno, etc.

    acceptExternalMessage();

    try {
        val attachedMsg = AttachedMessage.fromSlice(msg.rest);
        val vesting = storage.vestingParameters.load();

        var lockedAmount = vesting.getLockedAmount(blockchain.now());
        if (lockedAmount > 0) {
            attachedMsg.validateIfLockedAmount(
                storage.whitelist,
                lockedAmount,
                vesting.senderAddress
            );
        }

        sendRawMessage(attachedMsg.msgCell, attachedMsg.sendMode);
    } catch {
        // Silently ignore errors - message accepted but not sent
        // Gas fee still paid
    }

    storage.seqno += 1;
    storage.save();
}
```

**Use cases:**

- External messages where you want to accept gas even if processing fails
- Optional operations that shouldn't fail the whole transaction
- Experimental/beta features with fallback behavior

---

## Summary

These patterns form the foundation of production TON contracts:

1. **Storage**: Lazy load, save explicitly
2. **Messages**: Typed structs with opcodes, union types for routing
3. **Validation**: Assert chains with clear error codes
4. **Fees**: Calculate carefully, reserve storage, return excesses
5. **Addresses**: StateInit-based calculation for wallets/items
6. **Bounces**: Handle gracefully, restore state
7. **Security**: Validate sender, check workchain, verify signatures
8. **Gas**: Track value, calculate fees, use appropriate send modes
9. **Batch**: Snaked structures for multiple operations
10. **External**: Signature verification, seqno protection, two-phase commit

For stdlib functions, see [01-stdlib-reference.md](./01-stdlib-reference.md).
For language syntax, see [02-language-features.md](./02-language-features.md).
For testing, see [04-testing-guide.md](./04-testing-guide.md).
