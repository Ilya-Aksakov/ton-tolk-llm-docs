# Tolk Language Overview

## What is Tolk?

Tolk is a modern programming language for the TON Virtual Machine (TVM), developed as an alternative to FunC. It compiles to TVM bytecode for executing smart contracts on the TON blockchain.

## Key Features

**Language Version**: Tolk 1.2

**Key Advantages over FunC:**

- Type safety with type inference
- Structures and union types (algebraic data types)
- Built-in support for `map<K, V>` instead of low-level dictionaries
- `lazy` evaluation for gas optimization
- Automatic serialization/deserialization of structures
- Pattern matching with `match`
- Modular system with `import`

## Syntax

```tolk
tolk 1.2

import "@stdlib/gas-payments"
import "errors"

struct MinterStorage {
    totalSupply: coins
    adminAddress: address
    content: cell
}

fun onInternalMessage(in: InMessage) {
    val msg = lazy AllowedMessage.fromSlice(in.body);
    match (msg) {
        MintRequest => { /* handle */ }
        else => throw 0xFFFF
    }
}

get fun get_total_supply(): coins {
    val storage = lazy MinterStorage.load();
    return storage.totalSupply;
}
```

## Project Structure

```
project/
├── contract.tolk          # Main contract (entrypoint)
├── storage.tolk           # Storage structures
├── messages.tolk          # Message definitions
├── errors.tolk            # Error constants
└── utils.tolk            # Helper functions
```

## Entrypoints

**Internal messages**: `fun onInternalMessage(in: InMessage)`
**External messages**: `fun onExternalMessage(inMsgBody: slice)`
**Bounced messages**: `fun onBouncedMessage(in: InMessageBounced)`
**Get methods**: `get fun method_name(): ReturnType`

## Minimal Contract Example

```tolk
tolk 1.2

struct Storage {
    counter: int
}

fun Storage.load() {
    return Storage.fromCell(contract.getData())
}

fun Storage.save(self) {
    contract.setData(self.toCell())
}

fun onInternalMessage(in: InMessage) {
    var storage = Storage.load();
    storage.counter += 1;
    storage.save();
}

get fun get_counter(): int {
    val storage = Storage.load();
    return storage.counter;
}
```

## Compilation and Testing

```bash
# Compilation (via @ton/tolk-js)
import { compileTolk } from '@ton/tolk-js';
const result = await compileTolk({ file: 'contract.tolk' });

# Testing (via @ton/sandbox)
import { Blockchain } from '@ton/sandbox';
const blockchain = await Blockchain.create();
const contract = blockchain.openContract(...);
```

## Next Steps

- [Standard Library Reference](./01-stdlib-reference.md)
- [Language Features](./02-language-features.md)
- [Patterns and Practices](./03-patterns-and-practices.md)
- [Testing Guide](./04-testing-guide.md)
