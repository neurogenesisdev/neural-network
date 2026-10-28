---
type: resource
status: active
updated: '2026-04-21T12:23:03Z'
title: "What are blockchain rollups?"
source: "https://cointelegraph.com/explained/what-are-blockchain-rollups"
author:
  - "[[Onkar Singh]]"
published: 2024-04-29
created: 2025-08-28
description: "Blockchain rollups are layer-2 scaling solutions that bundle multiple transactions off-chain before committing them to the main chain."
tags:
  - "clippings"
---
[BTC](https://cointelegraph.com/bitcoin-price)

[

$113,216

1.07%

](https://cointelegraph.com/bitcoin-price)[

ETH

$4,561

1.88%

](https://cointelegraph.com/ethereum-price)[

XRP

$3.01

0.03%

](https://cointelegraph.com/xrp-price-index)[

BNB

$874.64

1.06%

](https://cointelegraph.com/binance-coin-price-index)[

SOL

$214.94

2.83%

](https://cointelegraph.com/solana-price-index)[

DOGE

$0.2247

1.29%

](https://cointelegraph.com/doge-price-index)[

ADA

$0.8708

0.25%

](https://cointelegraph.com/ada-price-index)[

STETH

$4,540.71

1.93%

](https://cointelegraph.com/lido-staked-eth-price-index)[

TRX

$0.3455

1.43%

](https://cointelegraph.com/tron-price-index)[

AVAX

$24.92

1.26%

](https://cointelegraph.com/avalanche-price-index)[

SUI

$3.51

0.20%

](https://cointelegraph.com/sui-price-index)[

TON

$3.18

0.37%

](https://cointelegraph.com/toncoin-price-index)[BTC](https://cointelegraph.com/bitcoin-price)

[

$113,213

1.14%

](https://cointelegraph.com/bitcoin-price)[

ETH

$4,562

1.67%

](https://cointelegraph.com/ethereum-price)[

XRP

$3.01

0.04%

](https://cointelegraph.com/xrp-price-index)[

BNB

$874.64

1.06%

](https://cointelegraph.com/binance-coin-price-index)[

SOL

$214.94

2.83%

](https://cointelegraph.com/solana-price-index)[

DOGE

$0.2247

1.29%

](https://cointelegraph.com/doge-price-index)[

ADA

$0.8708

0.25%

](https://cointelegraph.com/ada-price-index)[

STETH

$4,540.71

1.93%

](https://cointelegraph.com/lido-staked-eth-price-index)[

TRX

$0.3455

1.43%

](https://cointelegraph.com/tron-price-index)[

AVAX

$24.92

1.26%

](https://cointelegraph.com/avalanche-price-index)[

SUI

$3.51

0.20%

](https://cointelegraph.com/sui-price-index)[

TON

$3.18

0.37%

](https://cointelegraph.com/toncoin-price-index)

## Blockchain rollups, explained

*Blockchain rollups are a scalability solution that processes and bundles several transactions off-chain before submitting them to the main blockchain, reducing congestion on blockchain networks.*

Consider a busy checkout queue at a store. Blockchain rollups provide quicker processing by acting as a self-checkout lane. To reduce [network congestion](https://cointelegraph.com/explained/what-is-blockchain-network-congestion), they group transactions and manage them off the main blockchain. This reduces the cost per transaction and expedites the process.

Finally, for security purposes, a single transaction summary — which resembles a receipt for each self-checkout customer — is sent back to the main [blockchain](https://cointelegraph.com/learn/how-does-blockchain-work-everything-there-is-to-know). This allows for faster and less expensive side transactions while maintaining the security of the main system. Rollups accomplish this in a variety of ways, but their main goal is to simplify blockchain transactions without sacrificing security.

## Types of blockchain rollups

*ZK-rollups and optimistic rollups are the two primary types of rollups. Both kinds of rollups ease mainnet congestion, resulting in speedier and less expensive transactions.*

### Zero-knowledge rollups (ZK-rollups)

ZK-rollups condense transaction data into a succinct proof using zero-knowledge proofs, which is subsequently sent to the main chain. ZK-rollups accomplish this by lowering the quantity of data that must be processed on-chain while preserving transaction security.

Zero-knowledge proofs ensure privacy by enabling validators to confirm the accuracy of the transactions without requiring access to the transaction details.

![How Zk-rollups work](https://s3.cointelegraph.com/storage/uploads/view/d70d0596b8249c7a7169c6844068a04c.jpg "How Zk-rollups work")

### Optimistic rollups

As implied by the name, optimistic rollups operate under the premise that all transactions are legitimate unless demonstrated to be otherwise.

Optimistic rollups only submit the result to the main chain after [processing transactions off-chain](https://cointelegraph.com/learn/on-chain-vs-off-chain-transactions). The primary chain employs a fraud-proof technique to confirm the legitimacy of the transactions. The transactions are approved if there is no evidence of fraud, but if fraud is detected, the fraudulent transactions are rolled back.

![How optimistic rollups work](https://s3.cointelegraph.com/storage/uploads/view/84575bc4cef735fd19be7a3acbe49606.jpg "How optimistic rollups work")

## How do blockchain rollups work?

*Before uploading several transactions to the main blockchain, blockchain rollups process and bundle them off-chain. Here’s a step-by-step process of how blockchain rollups work:*

![Roll up transactions vs. Transactions on base blockchains](https://s3.cointelegraph.com/storage/uploads/view/cb958eb5217bc8878668ede7d91b1b73.jpg "Roll up transactions vs. Transactions on base blockchains")

### Off-chain processing

Firstly, a rollup [smart contract](https://cointelegraph.com/learn/what-are-smart-contracts-a-beginners-guide-to-automated-agreements) processes transactions off-chain. This enables faster and more efficient transaction processing than dealing directly with the main blockchain.

### Transaction bundling

The rollup smart contract groups several transactions into a single batch. This bundling procedure reduces the amount of data that needs to be uploaded to the main blockchain.

### Validation and compression (for ZK-rollups)

Zero-knowledge proofs are used in ZK-rollups to condense transaction data into a brief proof. After being created off-chain, this proof is added to the primary blockchain.

### Submission to the main blockchain

The rollup smart contract uploads the transaction batch or the compressed proof to the main blockchain after the transactions have been processed and bundled.

### Verification and settlement

The batch or proof that has been uploaded gets validated on the main blockchain. A fraud-proof technique is used to verify the legitimacy of transactions for optimistic rollups. The transactions are finalized on the main blockchain if no fraud is found.

## Applications of rollups

*Rollups can help improve the scalability of blockchain networks, decentralized finance (DeFi) projects, nonfungible tokens (NFTs), gaming and virtual worlds and supply chain businesses.*

Among the important applications of rollups are:

### Scalability solutions

Rollups process and bundle several transactions off-chain before submitting them to the main blockchain, which greatly increases the [scalability of blockchain networks](https://cointelegraph.com/learn/a-beginners-guide-on-blockchain-layer-2-scaling-solutions). This lowers transaction fees and congestion by enabling a larger transaction throughput.

### Decentralized finance

In the [DeFi space](https://cointelegraph.com/learn/defi-a-comprehensive-guide-to-decentralized-finance), where large transaction volumes can cause congestion and expensive gas costs, rollups are very helpful. DeFi applications may be made much more scalable and user-friendly by using rollups, which makes transactions faster and more economical.

### Nonfungible tokens

NFT platforms can benefit from rollups by [minting and processing NFTs](https://cointelegraph.com/learn/what-are-nfts-and-why-are-they-revolutionizing-the-art-world) off-chain before submitting them to the main blockchain. This lowers the time and expense involved in generating and trading NFTs.

### Gaming and virtual worlds

Rollups facilitate faster and more effective in-game transactions, asset transfers and [decentralized governance](https://cointelegraph.com/news/what-are-governance-tokens-and-how-do-they-work), all of which can enhance the scalability of blockchain-based virtual worlds and gaming.

### Supply chain management

Rollups enhance security, efficiency and [transparency in the supply chain](https://cointelegraph.com/explained/how-blockchain-technology-is-used-in-supply-chain-management) by tracking and validating transactions.

Rollups, in general, provide a flexible scaling option that can help a variety of blockchain applications by allowing them to manage increased transaction volumes without compromising the security and decentralization of the underlying blockchain network.

## Benefits of blockchain rollups

*Blockchain networks can benefit from rollups in multiple ways, including scalability, low transaction fees, faster transaction processing, enhanced privacy and user experience.*

[Rollups greatly increase scalability](https://cointelegraph.com/news/ethereum-vitalik-buterin-wants-rollups-stage-1-decentralization) by processing and grouping several transactions off-chain before sending them to the main blockchain. This allows blockchain networks to manage a higher volume of transactions.

Moreover, rollups lessen congestion on the main blockchain, which lowers transaction fees. This increases user accessibility and affordability for blockchain transactions. By shifting a large portion of the computation off-chain, rollups allow for speedier transaction processing. This leads to enhanced user experience and faster confirmation times.

Zero-knowledge proofs are used by some rollup solutions, such ZK-rollups, to ensure transaction privacy. This preserves the security and integrity of the blockchain network while helping to protect user data and transaction details.

Furthermore, by resolving the scalability issues that blockchain networks encounter, rollups provide an effective and intuitive user experience. This makes them appropriate for a variety of applications, including supply chain management, gaming, and DeFi and NFTs.

## Future of blockchain rollups

*Blockchain rollups appear to have a bright future, as these scaling solutions are expected to have a major impact on advancing blockchain technology.*

Rollups present a practical way to increase network capacity, lower transaction costs and improve user experience as blockchain networks expand and encounter scalability issues. Furthermore, rollups are expected to play a bigger role in the DeFi space, where gas fees and large transaction volumes have proven to be major obstacles.

Rollups’ connectivity with Ethereum and other major blockchain platforms is a crucial component of their future. Rollups are likely to be essential in improving the network’s scalability as [Ethereum incorporates danksharding](https://cointelegraph.com/explained/danksharding-and-proto-danksharding-explained).

There will likely be more innovation in this field, such as advancements in security, privacy and interoperability as rollup technology matures. Though rollup systems like ZK-rollups and optimistic rollups are already becoming popular, new advancements could result in rollup solutions that are even more effective and scalable.