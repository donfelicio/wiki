<img width="699" alt="screen shot 2019-02-26 at 6 23 18 pm" src="https://user-images.githubusercontent.com/27389/53459978-87dc7700-3a09-11e9-9b5c-cbff329a74c3.png">

We designed the 0x protocol to be extensible exchange infrastructure for the entire crypto economy. 0x's functionality can be extended through extension contracts, which broaden how 0x orders can be filled. There are a variety ways to build extension contracts. An Extension contract can perform work on the users behalf. They can validate that users belong to a specific group. Extension contracts can even be written to join multiple Decentralised Finance projects together.

0x allows for contracts to perform 0x operations, though before version 2 of the protocol there was no way to enforce that an order executed via a specific contract. The 0x order format now contains a field `senderAddress`. When this is defaulted to empty it is completely open for any anyone to fill. The Forwarder uses open orders with an unspecified sender address to perform a fill on behalf of the user. When the sender address is specified the 0x protocol ensures the operation originates from this address. For example it's possible to create a KYC (Know Your Customer) orderbook. This orderbook contains orders specifying the sender address as the KYC attestation contract. 0x protocol enforces all operations originate from the KYC contract in which all addresses have been validated by this contract.

### Forwarder Contract

Wrapping ETH into ERC-20 compliant wETH has been a large barrier to adoption for the greater Decentralised Finance ecosystem. The 0x core team wrote and deployed a Forwarder contract to address these issues. You have used the Forwarder Contract if you have purchased tokens through [0x Instant](https://0x.org/instant). The Forwarder is an example of an extension which performs work on the users behalf.

Users send ETH and a set of orders to the Forwarder contract and these funds are used to trade on 0x. The Forwarder deposits the incoming ETH into wrapped ETH. It then calls fill order with the supplied orders, using the wrapped ETH as its source of funding. Finally it transfers all traded tokens back to the user.

The Forwarder extends the behavior of trading on 0x. It presents a different interface where the user can specify the amount of tokens they wish to buy and use ETH to buy them. This is an improved user experience to interacting with the lower level 0x Exchange contract. From a user's perspective it appears as if they are buying tokens using ETH directly.

<img width="575" alt="screen shot 2019-02-26 at 5 42 25 pm" src="https://user-images.githubusercontent.com/27389/53459986-8dd25800-3a09-11e9-87ce-14eff3464b28.png">

Projects can use the Forwarder contract and a Relayers orders in their own integrations. We've seen great adoption of 0x Instant, which uses the Forwarder under the hood. These projects are able to use the affiliate fee in the Forwarder to collect fees in ETH for the trades they originate.

```typescript
const txHash = await contractWrappers.forwarder.marketBuyOrdersWithEthAsync(
    signedOrders,
    assetAmount,
    taker,
    amountInEth,
    feeOrders,
    affiliateFeePercentage,
    affiliateFeeAddress,
);
```

The 0x Core team have deployed the Forwarder contract on all networks, though there may be some interesting use cases not currently available via the Forwarder contract. We encourage projects to dive into the source code and extend the Forwarder contract for their own usage.

**Related Links**

-   [0x Instant](https://0x.org/instant)
-   [AssetBuyer](https://0x.org/docs/asset-buyer)
-   [0x Starter Project](https://github.com/0xProject/0x-starter-project/blob/master/src/scenarios/forwarder_buy_erc20_tokens.ts#L19)
-   [Forwarder Source Code](https://github.com/0xProject/0x-monorepo/blob/development/contracts/exchange-forwarder/contracts/src/MixinForwarderCore.sol)

### Dutch Auction Contract

The price of an asset decreases over time and the buyer submits a matching order when they believe the asset is appropriately priced. The Dutch Auction contract guarantees the asset is exchanged for the current price, given the block time. The Dutch Auction contract is an example of an extension which validates some state before matching two orders.

The Dutch Auction contract is powered by the 0x Exchange contracts [matchOrders function](https://0x.org/docs/contracts#Exchange-matchOrders) and extends the order format to include additional data. When using match orders, two orders are passed in as parameters, known as the sell order and the buy order. The seller of the asset creates an order for the asset at the lowest price. When a buyer decides the asset is at an acceptable price, they create a matching order for that amount. The buyer submits both the sell order and the buy order to the Dutch Auction contract.

<img width="825" alt="screen shot 2019-02-26 at 9 05 25 pm" src="https://user-images.githubusercontent.com/27389/53460197-4a2c1e00-3a0a-11e9-9cbe-1a143d31cd01.png">

Within the Dutch Auction contract the current price of the asset is calculated given the starting price, the current block time and the expiration time of the order. The contract calculates the correct price and only permits the match to execute if the buy order contains a sufficient amount. Using match orders the caller of `matchOrders` receives any spread or excess in the exchange. In this case the Dutch Auction contract receives the spread and returns it to the seller.

As you may have noticed, the 0x order format does not contain any space for additional parameters. The setup for the Dutch Auction contract encodes additional data in the maker asset data that is compatible with the Asset Proxy. In the case of ERC-20, the asset data is ABI encoded as `ERC20Token(address)`.

<img width="629" alt="screen shot 2019-02-26 at 4 23 51 pm" src="https://user-images.githubusercontent.com/27389/53460021-a17dbe80-3a09-11e9-80e7-061559ed550b.png">

The Dutch Auction encodes additional parameters into the maker asset data, parameters such as starting amount and time the auction begins. These are extracted in the Dutch Auction contract and validated before calling `matchOrders`.

<img width="624" alt="screen shot 2019-02-26 at 4 30 02 pm" src="https://user-images.githubusercontent.com/27389/53460005-99be1a00-3a09-11e9-8f6b-d446672887f7.png">

Encoding of the additional parameters is provided on the [Dutch Auction contract wrapper](https://0x.org/docs/contract-wrappers#DutchAuctionWrapper-encodeDutchAuctionAssetData).

```typescript
// Additional data is encoded in the maker asset data, this includes the begin time and begin amount
// for the auction
const dutchAuctionEncodedAssetData = DutchAuctionWrapper.encodeDutchAuctionAssetData(
    makerAssetData,
    auctionBeginTimeSeconds,
    auctionBeginAmount,
);
```

The Dutch Auction contract implements the same `matchOrders` function as the 0x Exchange contract.

```typescript
// Match the orders via the Dutch Auction contract
const txHash = await contractWrappers.dutchAuction.matchOrdersAsync(buySignedOrder, sellSignedOrder, buyerAddress);
```

Without the `senderAddress` being set to the Dutch Auction contract address, an attacker could try and front-run the transaction and submit the order to the Exchange contract, bypassing the Dutch Auction contract. By setting the `senderAddress` as the Dutch Auction contract the Exchange contract enforces any operations on these orders to be originating from this address.

Note: In the case of calling `matchOrders` the `takerAddress` can be specified in place of the `senderAddress`.

**Related Links**

-   [0x Starter Project](https://github.com/0xProject/0x-starter-project/blob/master/src/scenarios/dutch_auction.ts#L20)
-   [Dutch Auction Source Code](https://github.com/0xProject/0x-monorepo/blob/development/contracts/extensions/contracts/src/DutchAuction/DutchAuction.sol#L29)

### Whitelist Contract

Interacting with a [Whitelist Contract](https://github.com/0xProject/0x-monorepo/blob/69025da2dcb4bded72cbf724c598ee09682d2dfc/contracts/exchange/contracts/examples/Whitelist.sol) is one way to ensure [compliant peer-to-peer trading on 0x](https://blog.0xproject.com/compliant-peer-to-peer-trading-4dab8e5c3162). Relayers intending to offer security tokens or other regulated assets can utilize a whitelist contract to automatically filter through a set of pre-approved Ethereum addresses. The Whitelist contract is an example of an extension which uses the execute transaction functionality of 0x.

<img width="689" alt="screen shot 2019-02-26 at 5 42 11 pm" src="https://user-images.githubusercontent.com/27389/53459994-92970c00-3a09-11e9-8c77-a02267033cc4.png">

This type of extension makes use of the [0x Execute Transaction functionality](https://github.com/0xProject/0x-protocol-specification/blob/master/v2/v2-specification.md#executing-a-transaction). When using execute transaction, the user signs their intent, and this is forwarded on to the 0x protocol via the extension contract. This applies to all operations available on 0x such as fill and cancel. This operation can be submitted by any third party, such as the Relayer itself. When a Relayer submits the operation on behalf of the user, they pay the gas.

```typescript
// Check if maker is on the whitelist.
require(isWhitelisted[order.makerAddress], 'MAKER_NOT_WHITELISTED');
// Check if taker is on the whitelist.
require(isWhitelisted[takerAddress], 'TAKER_NOT_WHITELISTED');
// Submit the 0x transaction to the Exchange contract
EXCHANGE.executeTransaction(salt, signerAddress, signedExchangeTransaction, signature);
```

In order to implement an on-chain whitelist, the contract would first query if the user is present in a list, if the user is present then the 0x transaction is executed. Our contract wrappers package exposes a [transaction encoder](https://0x.org/docs/contract-wrappers#TransactionEncoder) to encode 0x transactions. An detailed example of encoding and executing 0x transactions can be found in our [0x-starter-project](https://github.com/0xProject/0x-starter-project/blob/master/src/scenarios/execute_transaction.ts#L20).

```typescript
// The transaction encoder provides helpers in encoding 0x Exchange transactions to allow
// a third party to submit the transaction. This operates in the context of the signer (taker)
// rather then the context of the submitter (sender)
const transactionEncoder = await contractWrappers.exchange.transactionEncoderAsync();
// This is an ABI encoded function call that the taker wishes to perform
// in this scenario it is a fillOrder
const fillData = transactionEncoder.fillOrderTx(signedOrder, takerAssetAmount);
// Generate a random salt to mitigate replay attacks
const takerTransactionSalt = generatePseudoRandomSalt();
// The taker signs the operation data (fillOrder) with the salt
const executeTransactionHex = transactionEncoder.getTransactionHashHex(fillData, takerTransactionSalt, taker);
const takerSignatureHex = await signatureUtils.ecSignHashAsync(providerEngine, executeTransactionHex, taker);
```

The Whitelist template is a simple example of validating addresses before allowing the `fillOrder` operation to be performed. A project might support more than just `fillOrder` such as `batchFillOrders` or `marketBuyOrders`. For this we have a template which supports validation in all exchange functions. This template extension, named the [BalanceThresholdFilter](https://github.com/0xProject/0x-monorepo/blob/development/contracts/extensions/contracts/src/BalanceThresholdFilter/MixinBalanceThresholdFilterCore.sol#L34) contract, validates by ensuring the user holds a specific ERC721 token.

**Related Links**

-   [0x Starter Project](https://github.com/0xProject/0x-starter-project/blob/master/src/scenarios/execute_transaction.ts#L20)
-   [Whitelist Source Code](https://github.com/0xProject/0x-monorepo/blob/69025da2dcb4bded72cbf724c598ee09682d2dfc/contracts/exchange/contracts/examples/Whitelist.sol#L27)
-   [BalanceThresholdFilter Source Code](https://github.com/0xProject/0x-monorepo/blob/development/contracts/extensions/contracts/src/BalanceThresholdFilter/MixinBalanceThresholdFilterCore.sol#L34)

### Differences to an Asset Proxy

While both an Extension contract and an Asset Proxy change the behavior of exchange on the 0x protocol, they are quite different. Extension contracts cannot change the behavior of settlement. For example, it is not possible to add new token standards with an extension contract.

Since Asset Proxies calculate and finalize settlements, they are written and deployed by the 0x core team. New additions of Asset Proxies undergo security audits and the community votes on the addition to the protocol. If you have a problem that can only be solved via an Asset Proxy, we welcome proposals to the ZEIP repository.
