# Sandbox

This package allows you to emulate arbitrary TON smart contracts, send messages to them and run get methods on them as if they were deployed on a real network.

The key difference of this package from [ton-contract-executor](https://github.com/ton-community/ton-contract-executor) is the fact that the latter only emulates the compute phase of the contract - it does not know about any other phases and thus does not know anything about fees and balances (in a sense that it does not know whether a contract's balance will be enough to process all the out messages that it produces). On the other hand, this package emulates all the phases of a contract, and as a result, the emulation is much closer to what would happen in a real network.

## Content

* [Instalation](#installation)
* [Usage](#usage)
* [Writing Tests](#writing-tests)
  * [Basic test template](#basic-test-template)
  * [Test a transaction with matcher](#test-a-transaction-with-matcher)
  * [Testing transaction fees](#testing-transaction-fees)
  * [Cross contract tests](#cross-contract-tests)
  * [Test examples](#test-examples)
* [Viewing logs](#viewing-logs)
* [Network/Block configuration](#networkblock-configuration)
* [Contributors](#contributors)
* [License](#license)
* [Donations](#donations)

## Installation

Requires node 16 or higher.

```
yarn add @ton/sandbox ton @ton/core ton-crypto
```
or
```
npm i @ton/sandbox ton @ton/core ton-crypto
```

## Usage

To use this package, you need to create an instance of the `Blockchain` class using the static method `Blockchain.create` as follows:
```typescript
import { Blockchain } from "@ton/sandbox";

const blockchain = await Blockchain.create()
```

After that, you can use the low level methods on Blockchain (such as sendMessage) to emulate any messages that you want, but the recommended way to use it is to write wrappers for your contract using the `Contract` interface from `ton-core`. Then you can use `blockchain.openContract` on instances of such contracts, and they will be wrapped in a Proxy that will supply a `ContractProvider` as a first argument to all its methods starting with either `get` or `send`. Also all `send` methods will get Promisified and will return results of running the blockchain message queue along with the original method's result in the `result` field.

A good example of this is the [treasury contract](/src/treasury/Treasury.ts) that is basically a built-in highload wallet meant to help you write tests for your systems of smart contracts. When `blockchain.treasury` is called, an instance of `TreasuryContract` is created and `blockchain.openContract` is called to "open" it. After that, when you call `treasury.send`, `Blockchain` automatically supplies the first `provider` argument.

For your own contracts, you can draw inspiration from the contracts in the [examples](/examples/) - all of them use the `provider.internal` method to send internal messages using the treasuries passed in from the unit test file.
Here is an excerpt of that from [NftItem.ts](/examples/contracts/NftItem.ts):
```typescript
import { Address, beginCell, Cell, Contract, ContractProvider, Sender, toNano, Builder } from "ton-core";

class NftItem implements Contract {
    async sendTransfer(provider: ContractProvider, via: Sender, params: {
        value?: bigint
        to: Address
        responseTo?: Address
        forwardAmount?: bigint
        forwardBody?: Cell | Builder
    }) {
        await provider.internal(via, {
            value: params.value ?? toNano('0.05'),
            body: beginCell()
                .storeUint(0x5fcc3d14, 32) // op
                .storeUint(0, 64) // query id
                .storeAddress(params.to)
                .storeAddress(params.responseTo)
                .storeBit(false) // custom payload
                .storeCoins(params.forwardAmount ?? 0n)
                .storeMaybeRef(params.forwardBody)
                .endCell()
        })
    }
}
```

When you call `nftItem.sendTransfer(treasury.getSender(), { to: recipient })` (with `nftItem` being an "opened" instance of `NftItem`), an external message to the wallet represented by `treasury` will be pushed onto the message queue, then processed, generating an internal message to the NFT contract.

Here is another excerpt that shows the way to interact with get methods from wrappers:
```typescript
import { Contract, ContractProvider } from "ton-core";

export type NftItemData = {
    inited: boolean
    index: number
    collection: Address | null
    owner: Address | null
    content: Cell | null
}

class NftItem implements Contract {
    async getData(provider: ContractProvider): Promise<NftItemData> {
        const { stack } = await provider.get('get_nft_data', [])
        return {
            inited: stack.readBoolean(),
            index: stack.readNumber(),
            collection: stack.readAddressOpt(),
            owner: stack.readAddressOpt(),
            content: stack.readCellOpt(),
        }
    }
}
```

When you call `nftItem.getData()` (note that just like in the `sendTransfer` method, you don't need to supply the `provider` argument - it's done for you on "opened" instances), the `provider` will query the smart contract contained in blockchain and parse the data according to the code. Note that unlike the `send` methods, `get` methods on "opened" instances will return the original result as-is to the caller.

Notes:
- All of the methods of contracts that you want to "open" that start with `get` or `send` **NEED** to accept `provider: ContractProvider` as a first argument (even if not used) due to how the wrapper works.
- You can open any contract at any address, even if it is not yet deployed or was deployed by a "parent" opened contract. The only requirement is that the `address` field (required by the `Contract` interface) is the address of the contract that you want to open, and that `init` is present if you want to deploy using methods on the opened instance (in other cases, `init` is not necessary).
- Ideally, at most one call to **either** `provider.internal` or `provider.external` should be made within a `send` method. Otherwise, you may get hard to interpret (but generally speaking correct) results.
- No calls to `provider.external` or `provider.internal` should be made within `get` methods. Otherwise, you will get weird and wrong results in the following `send` methods of any contract.

---


## Writing tests

### Basic test template

You can install additional `@ton/test-utils` package by running `yarn add @ton/test-utils -D` or `npm i --save-dev @ton/test-utils` (with `.toHaveTransaction` for jest or `.transaction` or `.to.have.transaction` for chai matcher) to add additional helpers for ease of testing. Don't forget to import them in your unit test files though!

Writing tests in Sandbox works through defining arbitary actions with the contract and comparing their results with the expected result, for example:

```typescript
it('should execute with success', async () => {                              // description of the test case
    const res = await main.sendMessage(sender.getSender(), toNano('0.05'));  // performing an action with contract main and saving result in res

    expect(res.transactions).toHaveTransaction({                             // configure the expected result with expect() function
        from: main.address,                                                  // set expected sender for transaction we want to test matcher properties from
        success: true                                                        // set the desirable result using matcher property success
    });

    printTransactionFees(res.transactions);                                  // print table with details on spent fees
});
```


### Test a transaction with matcher

The basic workflow of creating a test is:
1. Create a specific wrapped `Contract` entity using `blockchain.openContract()`.
2. Describe the actions your `Contract` should perform and save the execution result in `res` variable.
3. Verify the properties using the `expect()` function and the matcher `toHaveTransaction()`.

The `toHaveTransaction` matcher expects an object with any combination of fields from the `FlatTransaction` type defined with the following properties

| Name                 | Type          | Description                                                                                                                                                         |
|----------------------|---------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| from                 | Address?      | Contract address of the message sender                                                                                                                              |
| to                   | Address       | Contract address of the message destination                                                                                                                         |
| on                   | Address       | Contract address of the message destination  (Alternative name of the property `to`).                                                                               |
| value                | bigint?       | Amount of Toncoins in the message in nanotons                                                                                                                       |
| body                 | Cell          | Message body defined as a Cell                                                                                                                                              |
| inMessageBounced     | boolean?      | Boolean flag Bounced. True - message is bounced, False - message is not bounced.                                                                                    |
| inMessageBounceable  | boolean?      | Boolean flag Bounceable. True - message can be bounced, False - message can not be bounced.                                                                         |
| op                   | number?       | Op code is the operation identifier number (crc32 from TL-B usually). Expected in the first 32 bits of a message body.                                              |
| initData             | Cell?         | InitData Cell. Used for contract deployment processes.                                                                                                              |
| initCode             | Cell?         | initCode Cell. Used for contract deployment processes.                                                                                                              |
| deploy               | boolean       | Custom Sandbox flag that indicates whether the contract was deployed during this transaction. True if contract before this transaction was not initialized and after this transaction became initialized. Otherwise - False.  |
| lt                   | bigint        | Logical time (set by validators in a normal network, monotonically increases by a set interval in Sandbox). Used for defining order of transactions related to a certain contract                                                           |
| now                  | bigint        | Unix timestamp of the transaction                                                                                                                                       |
| outMessagesCount     | number        | Quantity of outbound messages in a certain transaction                                                                                                              |
| oldStatus            | AccountStatus | AccountStatus before transaction execution                                                                                                                              |
| endStatus            | AccountStatus | AccountStatus after transaction execution                                                                                                                               |
| totalFees            | bigint?        | Number of spent fees in nanotons                                                                                                                                    |
|aborted| boolean?       | True - execution of certain transaction aborted and rollbacked because of errors or insufficient gas. Otherwise - False.                                            |
|destroyed| boolean?       | True - if the existing contract was destroyed due to executing a certain transaction. Otherwise - False.                                                            |
|exitCode| number?        | TVM exit code (from compute phase)                                                                                                                                |
|actionResultCode| number? | Action phase result code |
|success| boolean?       | Custom Sandbox flag that defines the resulting status of a certain transaction. True - if both the compute and the action phase succeeded. Otherwise - False.       |


You can omit those that you're not interested in, and you can also pass in functions accepting those types returning booleans (`true` meaning good) to check for example number ranges, message opcodes, etc. Note however that if a field is optional (like `from?: Address`), then the function needs to accept the optional type, too.



Here is an excerpt of how it's used in the NFT collection example mentioned above:
```typescript
const buyResult = await buyer.send({
    to: sale.address,
    value: price + toNano('1'),
    sendMode: SendMode.PAY_GAS_SEPARATELY,
})

expect(buyResult.transactions).toHaveTransaction({
    from: sale.address,
    to: marketplace.address,
    value: fee,
})
expect(buyResult.transactions).toHaveTransaction({
    from: sale.address,
    to: collection.address,
    value: fee,
})
```
(in that example `jest` is used)


### Testing transaction fees
It is possible to configure and update the current time of the `Blockchain`, which allows one to inspect how much a contract would spend on storage fees.

Suppose we have a `main` instance defined as a wrapped `Contract` instance `main = blockchain.openContract(/* non-wrapped Main instance */)`, and we wish to determine the amount of storage fees that will be accrued between two actions within a specified period.

```typescript
it('should storage fees cost less than 1 TON', async () => {
    const time1 = Math.floor(Date.now() / 1000);                               // current local unix time
    const time2 = time1 + 365 * 24 * 60 * 60;                                  // offset for a year

    blockchain.now = time1;                                                    // set current time
    const res1 = await main.sendMessage(sender.getSender(), toNano('0.05'));   // preview of fees 
    printTransactionFees(res1.transactions);

    blockchain.now = time2;                                                    // set current time
    const res2 = await main.sendMessage(sender.getSender(), toNano('0.05'));   // preview of fees 
    printTransactionFees(res2.transactions);
    
    const tx2 = res2.transactions[1];                                          // extract the transaction that executed in a year
    if (tx2.description.type !== 'generic') {
        throw new Error('Generic transaction expected');
    }

    // Check that the storagePhase fees are less than 1 TON over the course of a year
    expect(tx2.description.storagePhase?.storageFeesCollected).toBeLessThanOrEqual(toNano('1'));   
});
```

### Cross contract tests

The Sandbox emulates the entire process of executing cross-contract interactions as if they occurred on a real blockchain. 
The result of sending a message (transfer) contains basic information about all transactions and actions. 
You can verify all of these by creating specific requirements via `expect()` for each action and transaction.

```typescript
res = await main.sendMessage(...);

expect(res).toHaveTransaction(...) // test case 
        <...>
expect(res).toHaveTransaction(...) // test case
```

For instance, with [Modern Jetton](https://github.com/EmelyanenkoK/modern_jetton) it's possible to test whether a `mint` message results in minting to a new jetton wallet contract and returns the excess to the minter contract.
```typescript
it('minter admin should be able to mint jettons', async () => {
    // can mint from deployer
    let initialTotalSupply = await jettonMinter.getTotalSupply();
    const deployerJettonWallet = await userWallet(deployer.address);
    let initialJettonBalance = toNano('1000.23');
    const mintResult = await jettonMinter.sendMint(deployer.getSender(), deployer.address, initialJettonBalance, toNano('0.05'), toNano('1'));

    expect(mintResult.transactions).toHaveTransaction({ // test transaction of deployment of a jetton wallet
        from: jettonMinter.address,
        to: deployerJettonWallet.address,
        deploy: true,
    });

    expect(mintResult.transactions).toHaveTransaction({ // test transaction of excesses returned to minter
        from: deployerJettonWallet.address,
        to: jettonMinter.address
    });

});
```

### Test examples
You can typically find various tests for Sandbox-based project contracts in the `./tests` directory. Some examples can be found in the following [list](https://docs.ton.org/develop/smart-contracts/examples#examples-of-tests-for-smart-contracts).


## Viewing logs

`Blockchain` and `SmartContract` use `LogsVerbosity` to determine what kinds of logs to print. Here is the definition:
```typescript
type LogsVerbosity = {
    print: boolean
    blockchainLogs: boolean
    vmLogs: Verbosity
    debugLogs: boolean
}

type Verbosity = 'none' | 'vm_logs' | 'vm_logs_full'
```

Setting verbosity on `SmartContract`s works like an override with respect to what is set on `Blockchain`.

`debugLogs` is enabled by default on the `Blockchain` instance (so every `SmartContract` that does not have `debugLogs` overridden will print debug logs), other kinds of logs are turned off.

`print` determines whether to `console.log` all the non-empty logs (if set to `false`, logs will be collected but will only be exposed in the return values of methods on `Blockchain` and `SmartContract`, and not printed to console), defaults to `true` on the `Blockchain` instance.

`'vm_logs'` prints the log of every instruction that was executed, `'vm_logs_full'` also includes code cell hashes, locations, and stack information for every instruction executed.

To override verbosity on a specific contract, use `await blockchain.setVerbosityForAddress(targetAddress, verbosity)`, for example:
```typescript
await blockchain.setVerbosityForAddress(targetAddress, {
    blockchainLogs: true,
    vmLogs: 'vm_logs',
})
```
After that, the target contract will be using `debugLogs` from the `Blockchain` instance to determine whether to print debug logs, but will always print VM logs and blockchain logs.

To set global verbosity, use the `blockchain.verbosity` setter, for example:
```typescript
blockchain.verbosity = {
    blockchainLogs: true,
    vmLogs: 'none',
    debugLogs: false,
}
```
Note that unlike with `setVerbosityForAddress`, with this setter you have to specify all the values from `LogsVerbosity`.

## Setting smart contract state directly

If you want to test some behavior on a contract if it had specific code, data, and other state fields, but do not want to execute all the required transactions for that, you can directly set the full state of the contract as it is stored in sandbox by using this method on the `Blockchain` instance:
```
async setShardAccount(address: Address, account: ShardAccount)
```
There are 2 helpers exported from sandbox that can help you create the `ShardAccount` from the common properties: `createEmptyShardAccount` and `createShardAccount`.

Note that this is a low-level function and does not check any invariants, such as that the address passed as the argument matches the one that is present in the `ShardAccount`, meaning it is possible to break stuff if you're not careful when using it.

## Network/Block configuration

By default, this package will use its [stored network configuration](src/config/defaultConfig.ts) to emulate messages. However, you can set any configuration you want when creating the `Blockchain` instance by passing the configuration cell in the optional `params` argument in the `config` field.

## Contributors

Special thanks to [@dungeon-master-666](https://github.com/dungeon-master-666) for their C++ code of the emulator.

Special thanks to [@TrueCarry](https://github.com/truecarry) for their help with performance and other suggestions.

## License

This package is released under the [MIT License](LICENSE).

## Donations

TON - `EQAQR1d1Q4NaE5EefwUMdrr1QvXg-8mDB0XI2-fwDBD0nYxC`
