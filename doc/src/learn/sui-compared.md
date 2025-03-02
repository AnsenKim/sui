---
title: How Sui Differs from Other Blockchains
---

This page summarizes how Sui compares with existing blockchains and is intended for potential adopters of Sui to decide whether it fits their use cases. See [How Sui Works](how-sui-works.md) for an introduction to the Sui architecture.

Here are Sui's key features:

* [Causal order vs. total order](#causal-order-vs-total-order) enables massively parallel execution
* [Sui's variant of Move](../build/move.md) and its object-centric data model make composable objects/NFTs possible
* The blockchain-oriented [Move programming language](https://github.com/MystenLabs/awesome-move) eases the developer experience

## Traditional blockchains

Traditional blockchain validators collectively build a shared accumulator: a representation of the state of the blockchain, a chain to which they add increments over time, called blocks. In blockchains that offer deterministic finality, every time validators want to make an incremental addition to the blockchain, i.e., a block proposal, they sequence the proposal. This protocol lets them form an agreement over the current state of the chain, whether the proposed increment is valid, and what the state of the chain will be after the new addition. 

This method of maintaining common state over time has known practical success over the last 14 years or so, using a wealth of theory from the last 50 years of research in the field of Byzantine Fault Tolerant distributed systems. 

Yet it is inherently sequential: increments to the chain are added one at a time, like pearls on a string. In practice, this approach pauses the influx of transactions (often stored in a "mempool"), while the current block is under consideration.

## Sui's approach to validating new transactions

A lot of transactions do not have complex interdependencies with other, arbitrary parts of the state of the blockchain. Often financial users just want to send an asset to a recipient, and the only data required to gauge whether this simple transaction is admissible is a fresh view of the sender's account. Hence Sui takes the approach of only taking a lock - or "stopping the world" - for the relevant piece of data rather than the whole chain -- in this case, the account of the sender, which can only send one transaction at a time.

Sui further expands this approach to more involved transactions that may explicitly depend on multiple elements under their sender's control, using an [object model](../build/objects.md) and leveraging [Move](../build/move.md)'s strong ownership model. By requiring that dependencies be explicit, Sui applies a "multi-lane" approach to transaction validation, making sure those independent transaction flows can progress without impediment from the others.

This doesn't mean that Sui as a platform never orders transactions with respect to each other, or that it we allows owners to only affect their owned microcosm of objects. Sui will also process transactions that have an effect on some shared state, in a rigorous, consensus-ordered manner. They're just not the default use case.

## A collaborative approach to transaction submission

Sui validates transactions individually, rather than batching them in the traditional blocks. The key advantage of this approach is low latency; each successful transaction quickly obtains a certificate of finality that proves to anyone that the transaction will be processed by the Sui network.

But the process of submitting a transaction is a bit more involved. That little more work occurs on the network. (With bandwidth getting cheaper, this is less of a concern.) Whereas a usual blockchain can accept a bunch of transactions from the same author in a fire-and-forget mode, Sui transaction submission follows these steps:

1. Sender broadcasts a transaction to all Sui validators.
1. Sui validators send individual votes on this transaction to the sender.
1. Each vote has a certain weight since each validator has weight based upon the rules of [Proof of Stake](https://en.wikipedia.org/wiki/Proof_of_work).
1. Sender collects a Byzantine-resistant-majority of these votes into a *certificate* and broadcasts it to all Sui validators, thereby ensuring *finality*, or assurance the transaction will not be dropped (revoked).
1. Optionally, the sender collects a certificate detailing the effects of the transaction.

While those steps demand more of the sender, performing them efficiently can still yield a cryptographic proof of finality with minimum latency. Aside from crafting the original transaction itself, the session management for a transaction does not require access to any private keys and can be delegated to a third party.

## A different approach to state

Because Sui focuses on managing specific objects rather than a single aggregate of state, it also reports on them in a unique way:

* Every object in Sui has a unique version number.
* Every new version is created from a transaction that may involve several dependencies, themselves versioned objects. 

As a consequence, a Sui validator -- or any other validator with a copy of the state -- can exhibit a causal history of an object, showing its history since genesis. Sui explicitly makes the bet that in most cases, the ordering of that causal history with the causal history of another object is irrelevant; and in the few cases where this information is relevant, Sui makes this relationship explicit in the data.

## Causal order vs. total order

Unlike most existing blockchain systems (and as the reader may have guessed from the description of write requests above), Sui does not always impose a total order on the transactions submitted by clients, with shared objects being the exception. Instead, most transactions are *causally* ordered--if a transaction `T1` produces output objects `O1` that are used as input objects in a transaction `T2`, a validator must execute `T1` before it executes `T2`. Note that `T2` need not use these objects directly for a causal relationship to exist--e.g., `T1` might produce output objects which are then used by `T3`, and `T2` might use `T3`'s output objects. However, transactions with no causal relationship can be processed by Sui validators in any order.

## Where Sui excels

This section summarizes the main advantages of Sui with respect to traditional blockchains.

### High performance

Sui’s main selling point is its unprecedented performance. The following bullet points summarize the main performance benefits of Sui with respect to traditional blockchains:

* Sui forgoes consensus for most transactions while other blockchains always totally order them. Causally ordering transactions allows Sui to massively parallelize the execution of most transactions; this reduces latency and allows validators to take advantage of all their CPU cores.
* Sui pushes the complexity at the edges: the client is involved in a number of protocol steps. This minimizes the interactions between validators and keeps their code simpler and more efficient. Sui always gives the possibility to offload most of the client’s workload to a Sui Gateway service for better user experience. In contrast, traditional blockchains follow a fire-and-forget model where clients monitor the blockchain state to assess the success of their transaction submission.
* Sui operates at network speed without waiting for system timeouts between protocol steps. This significantly reduces latency when the network is good and not under attack. In contrast, the security of a number of traditional blockchains (including most proof-of-work based blockchains) need to wait for predefined timeouts before committing transactions.
* Sui can take advantage of more machines per validator to increase its performance. Traditional blockchains are often designed to run on a single machine per validator (or even on a single CPU).

### Performance under faults

Sui runs a leaderless protocol to process common transactions (i.e. containing only owned objects). As a result, faulty validators do not impact performance in any significant way. For transactions involving shared objects, Sui employs a state-of-the-art consensus protocol requiring no [view-change sub-protocol](https://pmg.csail.mit.edu/papers/osdi99.pdf) and thus experiences only slight performance degradations. In contrast, most leader-based blockchains experiencing even a single validator’s crash see their throughput fall and their latency increase (often by more than one order of magnitude).

### Security assumptions

Contrary to many traditional blockchains, Sui does not make strong synchrony assumptions on the network. This means that Sui maintains its security properties under bad network conditions (even excessively bad), network splits/partitions, or even powerful DoS attacks targeted on the validators. Sustained network attacks on synchronous blockchains (i.e., most proof-of-work based blockchains) can lead to double-spend of resources and deadlocks.

### Efficient local read operations

The reading process of Sui enormously differs from other blockchains. Users interested in only a handful of objects and their history perform authenticated reads  at a low granularity and low latency. Sui creates a narrow family tree of objects starting from the [genesis](https://github.com/MystenLabs/sui/blob/main/doc/src/build/wallet.md#genesis) allowing it to read only objects tied to the sender of the transaction. Users requiring a global view of the system (e.g., to audit the system) can take advantage of checkpoints to improve performance.

In traditional blockchains, families are ordered with respect to each other to totally order transactions. This then requires querying a massive blob for the precise information needed. Disk I/O thus becomes a performance bottleneck, and some blockchains [now require SSD drives](https://www.usenix.org/system/files/conference/hotstorage18/hotstorage18-paper-raju.pdf) on their validators as a result.

### Easier developer experience

Sui provides these benefits to developers:

* Move and object-centric data model (enables composable objects/NFTs)
* Asset-centric programming model
* Easier developer experience

## Engineering trade-offs

This section presents the main disadvantages of Sui with respect to traditional blockchains.

### Design complexity

While traditional blockchains require implementing only a single consensus protocol, Sui requires two protocols: (i) a protocol based on Byzantine Consistent Broadcast to handle common transactions, and (ii) a consensus protocol to handle transactions with shared objects. This means the Sui team needs to maintain a much larger codebase.

Transactions involving shared objects require a little overhead (adding two extra round trips - 200ms for well-connected clients using a Sui Gateway service) before submitting it to the consensus protocol. This overhead is required to securely compose the two protocols described above. Other blockchains can instead directly submit the transaction to the consensus protocol. Note the finality for shared object transactions is still in the 2-3 second range even with this overhead.

Building an efficient synchronizer is harder in Sui than in traditional blockchains. The synchronizer sub-protocol allows validators to update each other by sharing data, and it allows slow validators to catch up. Building an efficient synchronizer for traditional blockchains is no easy task, but still simpler than in Sui.

### Sequential writes in the common case

Traditional blockchains totally order all client transactions with respect to each other. This design requires reaching consensus across validators, which is effective but slow. 

As mentioned in previous sections, Sui forgoes consensus for most transactions to reduce their latency. In this manner, Sui enables multi-lane processing and eliminates head-of-line blocking. All other transactions no longer need to wait for the completion of the first transaction’s increment in a single lane. Sui provides a lane of the appropriate breadth for each transaction. Simple transactions require viewing only the sender account, which greatly improves the system’s capacity.

The downside of allowing head-of-line blocking on the sender for these simple transactions is that the sender can send only one transaction at a time. As a result, it is imperative the transactions finalize quickly. 

### Complex total queries

Sui can make total queries more difficult than in traditional blockchains since it does not always impose total order of transactions. Total queries are fairly rare with respect to local reads (see above) but useful in some scenarios. For example, a new validator joins the network and needs to download the total state to disk, or an auditor wishes to audit the entire blockchain.

Sui mitigates this with checkpoints. A checkpoint is established every time an increment is added to a blockchain resulting from a certified transaction. Checkpoints work much like a [write ahead log](https://en.wikipedia.org/wiki/Write-ahead_logging) that stores state prior to full execution of a program. The calls in that program represent a smart contract in a blockchain. A checkpoint contains not only the transactions but also commitments to the state of the blockchain before and after the transactions.

Sui uses the state commitment that arrives upon epoch change. Sui requires a single answer from the multiple validators and leverages an accessory protocol to derive the hash representing the state of the blockchain. This protocol consumes little bandwidth and does not impede the ingestion of transactions. Validators produce checkpoints at every epoch change. Sui requires the validators to also produce checkpoints even more frequently. So users may use these checkpoints to audit the blockchain with some effort.

## Conclusion

In summary, Sui offers many performance and usability gains at the cost of some complexity in less common use cases. Direct sender transactions excel in Sui.
