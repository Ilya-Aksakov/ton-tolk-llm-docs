# Tolk Programming Language Documentation

Comprehensive English documentation for Tolk v1.2 - a modern programming language for TON smart contracts.

## About Tolk

Tolk is a high-level programming language designed specifically for developing smart contracts on The Open Network (TON). It compiles to TVM (TON Virtual Machine) bytecode and provides modern language features while maintaining compatibility with TON's architecture.

**Key Features:**

- Modern syntax with type inference
- Lazy evaluation for gas optimization
- Pattern matching for safe message handling
- Union types for message discrimination
- Structure methods for object-oriented patterns
- Nullable types for safe null handling
- Direct compilation to efficient TVM bytecode

## Documentation Structure

This documentation is organized into 4 comprehensive guides:

### [01. Standard Library Reference](./docs/01-stdlib-reference.md)

Complete reference for all 6 Tolk standard library modules with 300+ functions:

- **common.tolk** - Core functions (types, math, crypto, messages, storage)
- **tvm-dicts.tolk** - Low-level dictionary operations
- **exotic-cells.tolk** - Exotic cell handling (library refs, Merkle proofs)
- **gas-payments.tolk** - Gas and fee calculations
- **lisp-lists.tolk** - Lisp-style list operations
- **tvm-lowlevel.tolk** - Low-level TVM register operations

Each function includes:

- Full signature with parameter types
- Detailed description
- Return value explanation
- Real code examples from production contracts
- TypeScript test examples where applicable

### [02. Language Features Guide](./docs/02-language-features.md)

Complete guide to Tolk syntax and language constructs:

- Version declaration and import system
- Constants and type system
- Structures (basic, with opcodes, with methods)
- Union types for message discrimination
- Functions (pure, inline, get methods, assembly)
- Variables and assignments (val vs var)
- Control flow (if, while, repeat, match)
- Pattern matching with lazy evaluation
- Operators (arithmetic, logical, type casting)
- Nullable types and null coalescing
- Annotations (@pure, @inline, @deprecated)
- Special constructs (assert, throw, asm)
- Comments and documentation

### [03. Patterns and Best Practices](./docs/03-patterns-and-practices.md)

Real-world patterns extracted from production TON contracts:

- **Storage Management** - Load/save patterns, lazy loading, maps
- **Message Definitions** - Opcodes, union types, nested cells
- **Message Handling** - Lazy loading + pattern matching
- **Error Handling** - Error constants, validation chains
- **Fee Management** - Fee calculation, reserve patterns
- **Address Calculation** - StateInit-based addresses, sharding
- **Bounced Messages** - Handling bounces, restoring state
- **Excesses Return** - Returning unused TON
- **Forward Payloads** - Passing data through transfers
- **Workchain Validation** - Ensuring correct workchain
- **Admin Transfer** - Safe privilege transfer
- **Code Upgrade** - Updating contract code
- **Batch Operations** - Processing multiple operations
- **External Messages** - Signature verification, replay protection
- **Try-Catch** - Graceful error handling

All patterns include production code examples from jetton, NFT, wallet, and vesting contracts.

### [04. Testing Guide](./docs/04-testing-guide.md)

Comprehensive guide to testing Tolk contracts with TypeScript:

- Test setup with @ton/sandbox and @ton/test-utils
- Testing deployment and initialization
- Testing minting operations
- Testing transfers (successful, unauthorized, insufficient balance)
- Testing authorization and access control
- Testing validation (balance, payload, fees)
- Testing bounced messages
- Testing forward payloads
- Testing invalid messages (unknown opcodes, empty messages)
- Testing get methods
- Gas logging patterns
- Transaction assertions (success, failure, bounced, deployment)
- Best practices for test organization

Includes 50+ complete test examples from production contracts.

## Quick Start

### 1. Basic Contract Structure

```tolk
tolk 1.2

import "@stdlib/gas-payments"
import "storage"
import "messages"

struct Storage {
    owner: address
    balance: coins
}

fun Storage.load() {
    return Storage.fromCell(contract.getData())
}

fun Storage.save(self) {
    contract.setData(self.toCell())
}

fun onInternalMessage(in: InMessage) {
    val msg = lazy AllowedMessage.fromSlice(in.body);

    match (msg) {
        Deposit => {
            var storage = lazy Storage.load();
            storage.balance += in.valueCoins;
            storage.save();
        }

        Withdraw => {
            var storage = lazy Storage.load();
            assert (in.senderAddress == storage.owner) throw ERR_NOT_OWNER;
            assert (storage.balance >= msg.amount) throw ERR_INSUFFICIENT;

            storage.balance -= msg.amount;
            storage.save();

            // Send withdrawal
            createMessage({
                bounce: BounceMode.NoBounce,
                dest: storage.owner,
                value: msg.amount,
                body: {} // empty
            }).send(SEND_MODE_REGULAR);
        }

        else => {
            assert (in.body.isEmpty()) throw ERR_UNKNOWN_OP
        }
    }
}

get fun get_balance(): coins {
    val storage = lazy Storage.load();
    return storage.balance;
}
```

### 2. Testing the Contract

```typescript
import { Blockchain, SandboxContract, TreasuryContract } from "@ton/sandbox";
import { Cell, toNano } from "@ton/core";
import { compile } from "@ton/blueprint";
import { MyContract } from "../wrappers/MyContract";
import "@ton/test-utils";

describe("MyContract", () => {
  let blockchain: Blockchain;
  let deployer: SandboxContract<TreasuryContract>;
  let contract: SandboxContract<MyContract>;

  beforeAll(async () => {
    const code = await compile("contractName");
    blockchain = await Blockchain.create();
    deployer = await blockchain.treasury("deployer");

    contract = blockchain.openContract(
      MyContract.createFromConfig({ owner: deployer.address }, code)
    );

    await contract.sendDeploy(deployer.getSender(), toNano("0.05"));
  });

  it("should accept deposits", async () => {
    const result = await contract.sendDeposit(
      deployer.getSender(),
      toNano("1")
    );

    expect(result.transactions).toHaveTransaction({
      from: deployer.address,
      to: contract.address,
      success: true,
    });

    expect(await contract.getBalance()).toEqual(toNano("1"));
  });

  it("owner can withdraw", async () => {
    const result = await contract.sendWithdraw(
      deployer.getSender(),
      toNano("0.5")
    );

    expect(result.transactions).toHaveTransaction({
      from: contract.address,
      to: deployer.address,
      success: true,
    });

    expect(await contract.getBalance()).toEqual(toNano("0.5"));
  });

  it("non-owner cannot withdraw", async () => {
    const other = await blockchain.treasury("other");

    const result = await contract.sendWithdraw(
      other.getSender(),
      toNano("0.1")
    );

    expect(result.transactions).toHaveTransaction({
      from: other.address,
      to: contract.address,
      aborted: true,
      exitCode: ERR_NOT_OWNER,
    });
  });
});
```

## Real Contract Examples

This documentation is based on 7 production contract projects from the tolk-bench repository:

1. **01_jetton** - TEP-74 compliant jetton (fungible token) implementation
2. **02_nft** - TEP-62 compliant NFT collection and items
3. **03_notcoin** - Optimized jetton with custom features
4. **04_sharded_tgbtc** - Sharded jetton with address optimization
5. **05_wallet-v5** - Latest wallet contract with extensions
6. **06_vesting** - Token vesting contract with whitelist
7. **07_telemint** - Telegram-integrated NFT minting

All examples in this documentation are taken directly from these production-ready contracts.

## Learning Path

**For beginners:**

1. Start with [Language Features Guide](./docs/02-language-features.md) to learn syntax
2. Review [Patterns and Practices](./docs/03-patterns-and-practices.md) for common patterns
3. Study [Testing Guide](./docs/04-testing-guide.md) to write tests
4. Reference [Standard Library](./docs/01-stdlib-reference.md) as needed

**For experienced developers:**

1. Skim [Language Features](./docs/02-language-features.md) for Tolk-specific features
2. Focus on [Patterns and Practices](./docs/03-patterns-and-practices.md) for TON-specific patterns
3. Use [Standard Library Reference](./docs/01-stdlib-reference.md) as API documentation
4. Apply [Testing Guide](./docs/04-testing-guide.md) best practices

**For LLM model training:**

- All 4 documents provide comprehensive coverage of Tolk language
- Examples are from production code, not theoretical
- Patterns follow TON blockchain best practices
- Documentation includes both high-level concepts and low-level details

## Additional Resources

- **TON Documentation**: https://docs.ton.org/
- **Tolk Compiler**: https://github.com/ton-blockchain/ton/tree/master/crypto/smartcont/tolk-tester
- **TON Sandbox**: https://github.com/ton-community/sandbox
- **TEP Standards**: https://github.com/ton-blockchain/TEPs

## Contributing

This documentation is extracted from the tolk-bench repository. To improve it:

1. Study production contracts in `contracts_Tolk/`
2. Review tests in `tests/`
3. Update documentation with new patterns
4. Add more examples from real contracts

## License

Documentation follows the license of the tolk-bench repository and Tolk compiler (LGPL).

---

**Version**: Documentation for Tolk v1.2
**Last Updated**: 2025
**Generated from**: **[tolk-bench](https://github.com/ton-blockchain/tolk-bench)** repository with 7 production contract projects

## Navigation

- **[Standard Library Reference →](./docs/01-stdlib-reference.md)**
- **[Language Features Guide →](./docs/02-language-features.md)**
- **[Patterns and Practices →](./docs/03-patterns-and-practices.md)**
- **[Testing Guide →](./docs/04-testing-guide.md)**
