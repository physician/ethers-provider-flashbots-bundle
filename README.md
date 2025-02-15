# ethers-provider-flashbots-bundle

Contains the `FlashbotsBundleProvider` ethers.js provider to provide high-level access to `eth_sendBundle` rpc endpoint on [mev-relay](https://github.com/flashbots/mev-relay-js).

Flashbots-enabled relays and miners expose two new jsonrpc endpoint: `eth_sendBundle` and `eth_callBundle`. Since these are brand-new, non-standard endpoints, ethers.js and other libraries do not natively support these requests (like `getTransactionCount`). In order to interact with these endpoints, you will also need access to another full-featured (non-Flashbots) endpoint for nonce-calculation, gas estimation, and transaction status.

This library is not a fully functional ethers.js implementation, just a simple provider class, designed to interact with your existing ethers.js v5 module.

You can pass in a generic ethers.js provider to the Flashbots provider in the constructor:

```ts
const NETWORK_INFO = { chainId: 1, ensAddress: '', name: 'mainnet' }
// Standard json rpc provider directly from ethers.js
const provider = new providers.JsonRpcProvider({ url: ETHEREUM_RPC_URL }, NETWORK_INFO)
// `authSigner` is an Ethereum private key that does NOT store funds and is NOT your bot's primary key.
// This is an identifying key for signing payloads to establish reputation and whitelisting
const authSigner = new Wallet('0x0000000000000000000000000000000000000000000000000000000000000000')
// flashbots provider requires passing in a standard provider
const flashbotsProvider = await FlashbotsBundleProvider.create(provider, authSigner)
```

The flashbotsProvider provides the sendBundle function:

```ts
flashbotsProvider.sendBundle(bundledTransactions: Array<FlashbotsBundleTransaction | FlashbotsBundleRawTransaction>, targetBlockNumber: number)
    => Promise<FlashbotsTransactionResponse>
```

and simulate function:

```ts
flashbotsProvider.simulate(
    signedBundledTransactions: Array<string>,
    blockTag: BlockTag,
    stateBlockTag?: BlockTag,
    blockTimestamp?: number)
      => Promise<SimulationResponse>
```

## Example

```ts
// Using the map below ships two different bundles, targeting the next two blocks
const blockNumber = await provider.getBlockNumber()
const minTimestamp = (await provider.getBlock(blockNumber)).timestamp
const maxTimestamp = minTimestamp + 120
const bundlePromises = [blockNumber + 1, blockNumber + 2].map((targetBlockNumber) =>
  flashbotsProvider.sendBundle(
    [
      {
        signedTransaction: SIGNED_ORACLE_UPDATE_FROM_PENDING_POOL // serialized signed transaction hex
      },
      {
        signer: wallet, // ethers signer
        transaction: transaction // ethers populated transaction object
      }
    ],
    targetBlockNumber, // block number at which this bundle is valid
    {
      minTimestamp, // optional minimum timestamp at which this bundle is valid (inclusive)
      maxTimestamp // optional maximum timestamp at which this bundle is valid (inclusive)
    }
  )
)
```

## bundledTransactions

A Flashbots bundle consists of one or more transactions in strict order to be relayed to the miner directly. While the miner requires signed transactions, `sendBundle()` can receive a mix of pre-signed transaction and `TransactionRequest` + `Signer` (wallet) objects.

These bundles can pay the miner either via gas fees _OR_ via `block.coinbase.transfer(minerReward)`.

## targetBlockNumber

The only block number for which the bundle is to be considered valid. If you would like more than one block to be targeted, submit multiple rpc calls targeting each specific block. This value should be higher than the value of getBlockNumber(). Submitting a bundle with a target block number of the current block, or earlier, is a no-op.

## FlashbotsTransactionResponse

A high-level object which contains metadata available at transaction submission time, as well as the following functions which can wait, track, and simulate the bundle's behavior.

- receipts() - Returns promise of an array of transaction receipts corresponding to the transaction hashes that were relayed as part of the bundle. Will not wait for block to be mined; could return incomplete information
- wait() - Returns a promise which will wait for target block number to be reched _OR_ one of the transactions to become invalid due to nonce-issues (including, but not limited to, one of the transactions from you bundle being included too early). Returns the wait resolution as a status enum
- simulate() - Returns a promise of the transaction simulation, once the proper block height has been reached. Use this function to troubleshoot failing bundles and verify miner profitability

## How to run demo.ts

Included is a simple demo of how to construct the FlashbotsProvider with auth signer authentication and submit a [non-functional] bundle. This will not yield any mev, but could serve as a sample initialization to help integrate into your own functional searcher.
