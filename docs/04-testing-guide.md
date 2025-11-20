# Tolk Testing Guide

Comprehensive guide to testing Tolk smart contracts using TypeScript, @ton/sandbox, and @ton/test-utils. Real examples from jetton, NFT, and wallet contracts.

## Table of Contents

1. [Test Setup](#test-setup)
2. [Testing Deployment](#testing-deployment)
3. [Testing Minting Operations](#testing-minting-operations)
4. [Testing Transfers](#testing-transfers)
5. [Testing Authorization](#testing-authorization)
6. [Testing Validation](#testing-validation)
7. [Testing Bounced Messages](#testing-bounced-messages)
8. [Testing Forward Payloads](#testing-forward-payloads)
9. [Testing Invalid Messages](#testing-invalid-messages)
10. [Testing Get Methods](#testing-get-methods)
11. [Gas Logging Patterns](#gas-logging-patterns)
12. [Transaction Assertions](#transaction-assertions)
13. [Best Practices](#best-practices)

---

## Test Setup

### Basic Test Structure

```typescript
import {
    Blockchain,
    SandboxContract,
    TreasuryContract,
    internal,
    SendMessageResult
} from '@ton/sandbox';
import { Cell, toNano, beginCell, Address } from '@ton/core';
import { compile } from '@ton/blueprint';
import '@ton/test-utils';

describe('MyContract', () => {
    let blockchain: Blockchain;
    let deployer: SandboxContract<TreasuryContract>;
    let contract: SandboxContract<MyContract>;
    let contractCode: Cell;

    beforeAll(async () => {
        // Compile contract using Blueprint
        // Looks for myContract.compile.ts in the same directory as the wrapper
        contractCode = await compile('MyContract');

        // Create blockchain
        blockchain = await Blockchain.create();

        // Create test accounts
        deployer = await blockchain.treasury('deployer');
    });

    beforeEach(async () => {
        // Deploy fresh contract for each test
        contract = blockchain.openContract(
            MyContract.createFromConfig(
                { /* config */ },
                contractCode
            )
        );

        await contract.sendDeploy(deployer.getSender(), toNano('0.05'));
    });

    it('should work', async () => {
        // Test implementation
    });
});
```

### Contract Compilation Files

For each contract, create a `.compile.ts` file next to its wrapper. Blueprint automatically finds and uses this configuration.

**Project Structure:**
```
project/
├── contracts/
│   └── my-contract.tolk
└── wrappers/
    ├── MyContract.ts          # Wrapper class
    └── MyContract.compile.ts  # Compilation config
```

**Tolk Compilation Config:**

```typescript
// wrappers/MyContract.compile.ts
import { CompilerConfig } from '@ton/blueprint';

export const compile: CompilerConfig = {
    lang: 'tolk',
    entrypoint: 'my-contract.tolk',
    withSrcLineComments: true,
    withStackComments: true,
};
```

**FunC Compilation Config (for reference):**

```typescript
// wrappers/MyContract.compile.ts
import { CompilerConfig } from '@ton/blueprint';

export const compile: CompilerConfig = {
    lang: 'func',
    targets: [
        'imports/stdlib.fc',
        'my-contract.fc',
    ],
};
```

**Configuration Options:**

| Option | Type | Description |
|--------|------|-------------|
| `lang` | `'tolk'` \| `'func'` | Compiler to use |
| `entrypoint` | `string` | Main Tolk file (Tolk only) |
| `targets` | `string[]` | FunC files to compile (FunC only) |
| `withSrcLineComments` | `boolean` | Include source line comments in Fift |
| `withStackComments` | `boolean` | Include stack state comments in Fift |
| `optimizationLevel` | `number` | Optimization level (0-2) |
| `experimentalOptions` | `string` | Experimental compiler flags |

**Usage in Tests:**

```typescript
import { compile } from '@ton/blueprint';

// Blueprint looks for wrappers/MyContract.compile.ts
const contractCode = await compile('MyContract');
```

This approach is simpler than manual compilation and automatically handles file paths and compiler configuration.

---

### Jetton Test Setup Example

```typescript
import { compile } from '@ton/blueprint';

describe('Jetton', () => {
    let walletCode: Cell;
    let minterCode: Cell;
    let blockchain: Blockchain;
    let deployer: SandboxContract<TreasuryContract>;
    let notDeployer: SandboxContract<TreasuryContract>;
    let jettonMinter: SandboxContract<JettonMinter>;
    let defaultContent: Cell;

    beforeAll(async () => {
        // Compile contracts using Blueprint
        // Looks for JettonWallet.compile.ts and JettonMinter.compile.ts
        walletCode = await compile('JettonWallet');
        minterCode = await compile('JettonMinter');

        // Setup blockchain (TVM 12 is activated by default in latest sandbox)
        blockchain = await Blockchain.create();

        // Create test accounts
        deployer = await blockchain.treasury('deployer');
        notDeployer = await blockchain.treasury('notDeployer');

        // Prepare content
        defaultContent = jettonContentToCell({
            type: 1,
            uri: "https://testjetton.org/content.json"
        });

        // Deploy minter
        jettonMinter = blockchain.openContract(
            JettonMinter.createFromConfig(
                {
                    admin: deployer.address,
                    content: defaultContent,
                    wallet_code: jwallet_code,
                },
                minter_code
            )
        );
    });

    afterAll(() => {
        GAS_LOG.saveCurrentRunAfterAll();
    });

    // Helper function to get wallet address
    const userWallet = async (address: Address) =>
        blockchain.openContract(
            JettonWallet.createFromAddress(
                await jettonMinter.getWalletAddress(address)
            )
        );

    // Tests...
});
```

---

## Testing Deployment

### Basic Deployment Test

```typescript
it('should deploy', async () => {
    const deployResult = await jettonMinter.sendDeploy(
        deployer.getSender(),
        toNano('100')
    );

    expect(deployResult.transactions).toHaveTransaction({
        from: deployer.address,
        to: jettonMinter.address,
        deploy: true,
        success: true,
    });
});
```

---

### Testing Initial State After Deployment

```typescript
it('should have correct initial state', async () => {
    await contract.sendDeploy(deployer.getSender(), toNano('0.05'));

    // Check initial values
    expect(await contract.getSeqno()).toBe(0);
    expect(await contract.getAdmin()).toEqualAddress(deployer.address);
    expect(await contract.getTotalSupply()).toBe(0n);
});
```

---

## Testing Minting Operations

### Basic Mint Test

```typescript
it('admin should be able to mint jettons', async () => {
    let initialTotalSupply = await jettonMinter.getTotalSupply();
    const deployerJettonWallet = await userWallet(deployer.address);
    let initialJettonBalance = toNano('1000.23');

    const mintResult = await jettonMinter.sendMint(
        deployer.getSender(),
        deployer.address,
        initialJettonBalance,
        toNano('0.05'),
        toNano('1')
    );

    // Check wallet deployed
    expect(mintResult.transactions).toHaveTransaction({
        from: jettonMinter.address,
        to: deployerJettonWallet.address,
        deploy: true,
    });

    // Check excesses returned
    expect(mintResult.transactions).toHaveTransaction({
        from: deployerJettonWallet.address,
        to: jettonMinter.address
    });

    // Check balance updated
    expect(await deployerJettonWallet.getJettonBalance()).toEqual(initialJettonBalance);
    expect(await jettonMinter.getTotalSupply()).toEqual(initialTotalSupply + initialJettonBalance);
});
```

---

### Testing Unauthorized Mint

```typescript
it('non-admin should not be able to mint', async () => {
    let initialTotalSupply = await jettonMinter.getTotalSupply();
    const deployerJettonWallet = await userWallet(deployer.address);
    let initialJettonBalance = await deployerJettonWallet.getJettonBalance();

    const unAuthMintResult = await jettonMinter.sendMint(
        notDeployer.getSender(),
        deployer.address,
        toNano('777'),
        toNano('0.05'),
        toNano('1')
    );

    expect(unAuthMintResult.transactions).toHaveTransaction({
        from: notDeployer.address,
        to: jettonMinter.address,
        aborted: true,
        exitCode: Errors.not_admin,
    });

    // Check state unchanged
    expect(await deployerJettonWallet.getJettonBalance()).toEqual(initialJettonBalance);
    expect(await jettonMinter.getTotalSupply()).toEqual(initialTotalSupply);
});
```

---

## Testing Transfers

### Successful Transfer Test

```typescript
it('wallet owner should be able to send jettons', async () => {
    const deployerJettonWallet = await userWallet(deployer.address);
    let initialJettonBalance = await deployerJettonWallet.getJettonBalance();
    let initialTotalSupply = await jettonMinter.getTotalSupply();

    const notDeployerJettonWallet = await userWallet(notDeployer.address);
    let initialJettonBalance2 = await notDeployerJettonWallet.getJettonBalance();

    let sentAmount = toNano('0.5');
    let forwardAmount = toNano('0.05');

    const sendResult = await deployerJettonWallet.sendTransfer(
        deployer.getSender(),
        toNano('0.1'),      // TON amount
        sentAmount,          // Jetton amount
        notDeployer.address, // Recipient
        deployer.address,    // Response address
        null,                // Custom payload
        forwardAmount,       // Forward amount
        null                 // Forward payload
    );

    // Check excesses returned
    expect(sendResult.transactions).toHaveTransaction({
        from: notDeployerJettonWallet.address,
        to: deployer.address,
    });

    // Check notification sent
    expect(sendResult.transactions).toHaveTransaction({
        from: notDeployerJettonWallet.address,
        to: notDeployer.address,
        value: forwardAmount
    });

    // Check balances updated
    expect(await deployerJettonWallet.getJettonBalance())
        .toEqual(initialJettonBalance - sentAmount);
    expect(await notDeployerJettonWallet.getJettonBalance())
        .toEqual(initialJettonBalance2 + sentAmount);
    expect(await jettonMinter.getTotalSupply())
        .toEqual(initialTotalSupply);
});
```

---

### Transfer Without Forward Amount

```typescript
it('no forward_ton_amount - no notification', async () => {
    const deployerJettonWallet = await userWallet(deployer.address);
    const notDeployerJettonWallet = await userWallet(notDeployer.address);

    let sentAmount = toNano('0.5');
    let forwardAmount = 0n;  // No forward

    const sendResult = await deployerJettonWallet.sendTransfer(
        deployer.getSender(),
        toNano('0.1'),
        sentAmount,
        notDeployer.address,
        deployer.address,
        null,
        forwardAmount,
        null
    );

    // Check excesses returned
    expect(sendResult.transactions).toHaveTransaction({
        from: notDeployerJettonWallet.address,
        to: deployer.address,
    });

    // Check NO notification sent
    expect(sendResult.transactions).not.toHaveTransaction({
        from: notDeployerJettonWallet.address,
        to: notDeployer.address
    });
});
```

---

## Testing Authorization

### Unauthorized Transfer Test

```typescript
it('not wallet owner should not be able to send jettons', async () => {
    const deployerJettonWallet = await userWallet(deployer.address);
    let initialJettonBalance = await deployerJettonWallet.getJettonBalance();

    let sentAmount = toNano('0.5');

    const sendResult = await deployerJettonWallet.sendTransfer(
        notDeployer.getSender(),  // Wrong sender!
        toNano('0.1'),
        sentAmount,
        notDeployer.address,
        deployer.address,
        null,
        toNano('0.05'),
        null
    );

    expect(sendResult.transactions).toHaveTransaction({
        from: notDeployer.address,
        to: deployerJettonWallet.address,
        aborted: true,
        exitCode: Errors.not_owner,
    });

    // Balance unchanged
    expect(await deployerJettonWallet.getJettonBalance())
        .toEqual(initialJettonBalance);
});
```

---

### Admin Change Test

```typescript
it('minter admin can change admin', async () => {
    const adminBefore = await jettonMinter.getAdminAddress();
    expect(adminBefore).toEqualAddress(deployer.address);

    let res = await jettonMinter.sendChangeAdmin(
        deployer.getSender(),
        notDeployer.address
    );

    expect(res.transactions).toHaveTransaction({
        from: deployer.address,
        on: jettonMinter.address,
        success: true
    });

    const adminAfter = await jettonMinter.getAdminAddress();
    expect(adminAfter).toEqualAddress(notDeployer.address);

    // Change back
    await jettonMinter.sendChangeAdmin(
        notDeployer.getSender(),
        deployer.address
    );
    expect((await jettonMinter.getAdminAddress()).equals(deployer.address))
        .toBe(true);
});
```

---

### Unauthorized Admin Change

```typescript
it('non-admin cannot change admin', async () => {
    const adminBefore = await jettonMinter.getAdminAddress();
    expect(adminBefore).toEqualAddress(deployer.address);

    let changeAdmin = await jettonMinter.sendChangeAdmin(
        notDeployer.getSender(),
        notDeployer.address
    );

    // Admin unchanged
    expect((await jettonMinter.getAdminAddress()).equals(deployer.address))
        .toBe(true);

    expect(changeAdmin.transactions).toHaveTransaction({
        from: notDeployer.address,
        on: jettonMinter.address,
        aborted: true,
        exitCode: Errors.not_admin,
    });
});
```

---

## Testing Validation

### Balance Validation Test

```typescript
it('impossible to send too much jettons', async () => {
    const deployerJettonWallet = await userWallet(deployer.address);
    let initialJettonBalance = await deployerJettonWallet.getJettonBalance();

    const notDeployerJettonWallet = await userWallet(notDeployer.address);
    let initialJettonBalance2 = await notDeployerJettonWallet.getJettonBalance();

    // Try to send more than balance
    let sentAmount = initialJettonBalance + 1n;
    let forwardAmount = toNano('0.05');

    const sendResult = await deployerJettonWallet.sendTransfer(
        deployer.getSender(),
        toNano('0.1'),
        sentAmount,
        notDeployer.address,
        deployer.address,
        null,
        forwardAmount,
        null
    );

    expect(sendResult.transactions).toHaveTransaction({
        from: deployer.address,
        to: deployerJettonWallet.address,
        aborted: true,
        exitCode: Errors.balance_error,
    });

    // Balances unchanged
    expect(await deployerJettonWallet.getJettonBalance())
        .toEqual(initialJettonBalance);
    expect(await notDeployerJettonWallet.getJettonBalance())
        .toEqual(initialJettonBalance2);
});
```

---

### Malformed Payload Test

```typescript
it('malformed forward payload', async () => {
    const deployerJettonWallet = await userWallet(deployer.address);
    const notDeployerJettonWallet = await userWallet(notDeployer.address);

    let sentAmount = toNano('0.5');
    let forwardAmount = toNano('0.05');

    // Build message manually with malformed payload
    let msgPayload = beginCell()
        .storeUint(0xf8a7ea5, 32)     // op
        .storeUint(0, 64)              // queryId
        .storeCoins(sentAmount)
        .storeAddress(notDeployer.address)
        .storeAddress(deployer.address)
        .storeMaybeRef(null)
        .storeCoins(toNano('0.05'))    // forward_ton_amount
        // No forward payload indication!
        .endCell();

    const res = await blockchain.sendMessage(internal({
        from: deployer.address,
        to: deployerJettonWallet.address,
        body: msgPayload,
        value: toNano('0.2')
    }));

    expect(res.transactions).toHaveTransaction({
        from: deployer.address,
        to: deployerJettonWallet.address,
        aborted: true,
        exitCode: Errors.invalid_payload
    });
});
```

---

### Not Enough TON Test

```typescript
it('check revert on not enough tons for forward', async () => {
    const deployerJettonWallet = await userWallet(deployer.address);

    let sentAmount = toNano('0.1');
    let forwardAmount = toNano('0.3');
    let forwardPayload = beginCell().storeUint(0x1234567890abcdefn, 128).endCell();

    const sendResult = await deployerJettonWallet.sendTransfer(
        deployer.getSender(),
        forwardAmount,  // Not enough TON for gas!
        sentAmount,
        notDeployer.address,
        deployer.address,
        null,
        forwardAmount,
        forwardPayload
    );

    expect(sendResult.transactions).toHaveTransaction({
        from: deployer.address,
        on: deployerJettonWallet.address,
        aborted: true,
        exitCode: Errors.not_enough_ton,
    });

    // Check value bounced back
    expect(sendResult.transactions).toHaveTransaction({
        from: deployerJettonWallet.address,
        on: deployer.address,
        inMessageBounced: true,
        success: true
    });
});
```

---

## Testing Bounced Messages

### Internal Transfer Bounce Test

```typescript
it('wallet does not accept internal_transfer not from wallet', async () => {
    const deployerJettonWallet = await userWallet(deployer.address);
    let initialJettonBalance = await deployerJettonWallet.getJettonBalance();

    // Manually construct internal_transfer message
    let internalTransfer = beginCell()
        .storeUint(0x178d4519, 32)  // internal_transfer opcode
        .storeUint(0, 64)            // queryId
        .storeCoins(toNano('0.01'))
        .storeAddress(deployer.address)
        .storeAddress(deployer.address)
        .storeCoins(toNano('0.05'))
        .storeUint(0, 1)
        .endCell();

    const sendResult = await blockchain.sendMessage(internal({
        from: notDeployer.address,  // Not from valid wallet!
        to: deployerJettonWallet.address,
        body: internalTransfer,
        value: toNano('0.3')
    }));

    expect(sendResult.transactions).toHaveTransaction({
        from: notDeployer.address,
        to: deployerJettonWallet.address,
        aborted: true,
        exitCode: Errors.not_valid_wallet,
    });

    // Balance unchanged
    expect(await deployerJettonWallet.getJettonBalance())
        .toEqual(initialJettonBalance);
});
```

---

## Testing Forward Payloads

### Transfer with Forward Payload Test

```typescript
it('correctly sends forward_payload', async () => {
    const deployerJettonWallet = await userWallet(deployer.address);
    const notDeployerJettonWallet = await userWallet(notDeployer.address);

    let sentAmount = toNano('0.5');
    let forwardAmount = toNano('0.05');
    let forwardPayload = beginCell()
        .storeUint(0x1234567890abcdefn, 128)
        .endCell();

    const sendResult = await deployerJettonWallet.sendTransfer(
        deployer.getSender(),
        toNano('0.1'),
        sentAmount,
        notDeployer.address,
        deployer.address,
        null,
        forwardAmount,
        forwardPayload
    );

    // Check excesses
    expect(sendResult.transactions).toHaveTransaction({
        from: notDeployerJettonWallet.address,
        to: deployer.address,
    });

    // Check notification with correct payload
    expect(sendResult.transactions).toHaveTransaction({
        from: notDeployerJettonWallet.address,
        to: notDeployer.address,
        value: forwardAmount,
        body: beginCell()
            .storeUint(Op.transfer_notification, 32)
            .storeUint(0, 64)  // queryId
            .storeCoins(sentAmount)
            .storeAddress(deployer.address)
            .storeUint(1, 1)   // Has ref
            .storeRef(forwardPayload)
            .endCell()
    });
});
```

---

## Testing Invalid Messages

### Unknown Opcode Test

```typescript
it('should throw on unknown opcode', async () => {
    const result = await blockchain.sendMessage(internal({
        from: deployer.address,
        to: contract.address,
        body: beginCell()
            .storeUint(0x12345678, 32)  // Unknown opcode
            .endCell(),
        value: toNano('0.1')
    }));

    expect(result.transactions).toHaveTransaction({
        from: deployer.address,
        to: contract.address,
        aborted: true,
        exitCode: 0xFFFF  // Unknown opcode error
    });
});
```

---

### Empty Message Test

```typescript
it('should accept empty messages silently', async () => {
    const result = await blockchain.sendMessage(internal({
        from: deployer.address,
        to: contract.address,
        body: beginCell().endCell(),  // Empty body
        value: toNano('0.1')
    }));

    expect(result.transactions).toHaveTransaction({
        from: deployer.address,
        to: contract.address,
        success: true  // Should not throw
    });
});
```

---

## Testing Get Methods

### Basic Get Method Test

```typescript
it('should return correct wallet data', async () => {
    const data = await jettonWallet.getWalletData();

    expect(data.jettonBalance).toEqual(expectedBalance);
    expect(data.ownerAddress).toEqualAddress(owner);
    expect(data.minterAddress).toEqualAddress(minter);
});
```

---

### Multiple Get Methods Test

```typescript
it('wallet-v5 get methods should work', async () => {
    expect(await walletV5.getSeqno()).toBe(0);
    expect(await walletV5.getSubwalletId()).toBe(WALLET_ID.serialized);
    expect(await walletV5.getPublicKey()).toBe(bufferToBigInt(keypair.publicKey));
    expect(await walletV5.getIsSignatureAllowed()).toBe(true);

    const extensions = await walletV5.getExtensions();
    expect(extensions.isEmpty()).toBe(true);
});
```

---

### Get Method with Address Calculation

```typescript
it('get_wallet_address should return correct address', async () => {
    const ownerAddress = randomAddress();
    const calculatedAddress = await jettonMinter.getWalletAddress(ownerAddress);

    // Deploy wallet and check address matches
    await jettonMinter.sendMint(
        deployer.getSender(),
        ownerAddress,
        toNano('1'),
        toNano('0.05'),
        toNano('1')
    );

    const actualWallet = await userWallet(ownerAddress);
    expect(actualWallet.address).toEqualAddress(calculatedAddress);
});
```

---

## Gas Logging Patterns

### Setup Gas Logger

```typescript
import { GasLogAndSave } from '../gas-logger';

describe('MyContract', () => {
    let GAS_LOG = new GasLogAndSave('01_project');

    beforeAll(async () => {
        // Compile and setup...

        // Remember contract sizes
        GAS_LOG.rememberBocSize('minter', minterCode);
        GAS_LOG.rememberBocSize('wallet', walletCode);
    });

    afterAll(() => {
        // Save all gas logs
        GAS_LOG.saveCurrentRunAfterAll();
    });

    // Tests...
});
```

---

### Logging Gas for Specific Operations

```typescript
it('[bench] MINT jettons by admin', async () => {
    const mintResult = await jettonMinter.sendMint(
        deployer.getSender(),
        deployer.address,
        initialJettonBalance,
        toNano('0.05'),
        toNano('1')
    );

    // Log gas consumption (skip first tx - it's the send from deployer)
    GAS_LOG.rememberGas('MINT jettons by admin', mintResult.transactions.slice(1));

    // Continue with assertions...
});
```

---

### Comparing Different Scenarios

```typescript
it('[bench] TRANSFER with forward_amount', async () => {
    const sendResult = await wallet.sendTransfer(/* with forward */);
    GAS_LOG.rememberGas('TRANSFER with forward_amount', sendResult.transactions.slice(1));
});

it('[bench] TRANSFER no forward_amount', async () => {
    const sendResult = await wallet.sendTransfer(/* without forward */);
    GAS_LOG.rememberGas('TRANSFER no forward_amount', sendResult.transactions.slice(1));
});
```

---

### Manual Gas Tracking

```typescript
it('track gas consumption', async () => {
    let ggc: bigint = BigInt(0);

    function accountForGas(transactions: BlockchainTransaction[]) {
        transactions.forEach((tx) => {
            ggc += ((tx?.description as TransactionDescriptionGeneric)
                ?.computePhase as TransactionComputeVm)
                ?.gasUsed ?? BigInt(0);
        })
    }

    const result = await contract.sendMessage();
    accountForGas(result.transactions);

    console.log("Total gas used:", ggc);
});
```

---

## Transaction Assertions

### Basic Transaction Assertion

```typescript
expect(result.transactions).toHaveTransaction({
    from: sender.address,
    to: receiver.address,
    success: true,
});
```

---

### Transaction with Specific Value

```typescript
expect(result.transactions).toHaveTransaction({
    from: wallet.address,
    to: recipient.address,
    value: toNano('0.05'),
    success: true,
});
```

---

### Transaction with Body Check

```typescript
expect(result.transactions).toHaveTransaction({
    from: wallet.address,
    to: owner.address,
    body: beginCell()
        .storeUint(Op.transfer_notification, 32)
        .storeUint(0, 64)
        .storeCoins(amount)
        .storeAddress(sender)
        .endCell()
});
```

---

### Aborted Transaction Assertion

```typescript
expect(result.transactions).toHaveTransaction({
    from: sender.address,
    to: contract.address,
    aborted: true,
    exitCode: Errors.not_owner,
});
```

---

### Deployment Transaction

```typescript
expect(result.transactions).toHaveTransaction({
    from: deployer.address,
    to: contract.address,
    deploy: true,
    success: true,
});
```

---

### Bounced Message Assertion

```typescript
expect(result.transactions).toHaveTransaction({
    from: contract.address,
    on: sender.address,
    inMessageBounced: true,
    success: true
});
```

---

### NOT Transaction Assertion

```typescript
expect(result.transactions).not.toHaveTransaction({
    from: wallet.address,
    to: recipient.address
});
```

---

### Multiple Assertions in One Test

```typescript
it('complex transaction flow', async () => {
    const result = await wallet.sendTransfer(/* params */);

    // Check transfer message
    expect(result.transactions).toHaveTransaction({
        from: senderWallet.address,
        to: recipientWallet.address,
        success: true,
    });

    // Check notification
    expect(result.transactions).toHaveTransaction({
        from: recipientWallet.address,
        to: recipient.address,
        value: forwardAmount,
    });

    // Check excesses
    expect(result.transactions).toHaveTransaction({
        from: recipientWallet.address,
        to: sender.address,
    });
});
```

---

## Best Practices

### 1. Use beforeAll for Expensive Setup

```typescript
import { compile } from '@ton/blueprint';

beforeAll(async () => {
    // Compile contracts once
    contractCode = await compile('MyContract');

    // Setup blockchain once
    blockchain = await Blockchain.create();
});
```

---

### 2. Use beforeEach for Fresh State

```typescript
beforeEach(async () => {
    // Deploy fresh contract for each test
    contract = blockchain.openContract(/* ... */);
    await contract.sendDeploy(/* ... */);
});
```

---

### 3. Test Both Success and Failure Cases

```typescript
describe('Transfer', () => {
    it('should succeed with valid params', async () => {
        // Test success path
    });

    it('should fail with insufficient balance', async () => {
        // Test failure path
    });

    it('should fail with unauthorized sender', async () => {
        // Test another failure path
    });
});
```

---

### 4. Use Descriptive Test Names

```typescript
// Good
it('admin can change content when authorized', async () => { });
it('non-admin cannot change content', async () => { });

// Bad
it('test1', async () => { });
it('works', async () => { });
```

---

### 5. Test Edge Cases

```typescript
it('should handle zero amount', async () => { });
it('should handle maximum amount', async () => { });
it('should handle empty payload', async () => { });
it('should handle null address', async () => { });
```

---

### 6. Use Helper Functions

```typescript
async function mintJettons(to: Address, amount: bigint) {
    return await jettonMinter.sendMint(
        deployer.getSender(),
        to,
        amount,
        toNano('0.05'),
        toNano('1')
    );
}

async function getUserWallet(owner: Address) {
    return blockchain.openContract(
        JettonWallet.createFromAddress(
            await jettonMinter.getWalletAddress(owner)
        )
    );
}
```

---

### 7. Check State After Operations

```typescript
it('should update balance after transfer', async () => {
    const balanceBefore = await wallet.getBalance();

    await wallet.sendTransfer(/* params */);

    const balanceAfter = await wallet.getBalance();
    expect(balanceAfter).toEqual(balanceBefore - sentAmount);
});
```

---

### 8. Test Transaction Chain

```typescript
it('should execute full transfer chain', async () => {
    const result = await senderWallet.sendTransfer(/* params */);

    // 1. Sender wallet -> Recipient wallet
    expect(result.transactions).toHaveTransaction({
        from: senderWallet.address,
        to: recipientWallet.address,
    });

    // 2. Recipient wallet -> Recipient owner
    expect(result.transactions).toHaveTransaction({
        from: recipientWallet.address,
        to: recipientOwner.address,
    });

    // 3. Recipient wallet -> Excess receiver
    expect(result.transactions).toHaveTransaction({
        from: recipientWallet.address,
        to: excessReceiver.address,
    });
});
```

---

### 9. Use Gas Benchmarks for Optimization

```typescript
it('[bench] compare gas usage', async () => {
    // Approach 1
    const result1 = await contract.sendMethodA();
    GAS_LOG.rememberGas('Method A', result1.transactions.slice(1));

    // Approach 2
    const result2 = await contract.sendMethodB();
    GAS_LOG.rememberGas('Method B', result2.transactions.slice(1));

    // Compare in saved logs
});
```

---

### 10. Test TEP Compliance

```typescript
describe('TEP-74 Compliance', () => {
    it('should implement get_jetton_data', async () => {
        const data = await minter.getJettonData();
        expect(data).toBeDefined();
    });

    it('should implement get_wallet_address', async () => {
        const addr = await minter.getWalletAddress(owner);
        expect(addr).toBeDefined();
    });
});

describe('TEP-89 Compliance', () => {
    it('should support discovery get method', async () => {
        const interfaces = await contract.supportedInterfaces();
        expect(interfaces).toContain("org.ton.jetton.wallet");
    });
});
```

---

## Summary

Testing Tolk contracts with TypeScript provides:

1. **Type Safety**: Catch errors at compile time
2. **Sandbox Environment**: Test without deploying to mainnet
3. **Transaction Inspection**: Detailed assertion capabilities
4. **Gas Tracking**: Optimize contract performance
5. **Reproducible Tests**: Deterministic blockchain state

**Key Testing Areas:**
- ✅ Deployment and initialization
- ✅ Authorization and access control
- ✅ State transitions (transfers, burns, mints)
- ✅ Validation and error handling
- ✅ Bounced messages
- ✅ Forward payloads
- ✅ Get methods
- ✅ Gas consumption
- ✅ TEP compliance

For contract patterns, see [03-patterns-and-practices.md](./03-patterns-and-practices.md).
For language features, see [02-language-features.md](./02-language-features.md).
For stdlib reference, see [01-stdlib-reference.md](./01-stdlib-reference.md).
